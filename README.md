import React, { useState, useEffect, useMemo, useRef } from 'react';
import { Beaker, ThermometerSun, Cylinder, Plus, Minus, Info, RefreshCw } from 'lucide-react';

// --- Math & Helper Functions ---

// Custom hook to smoothly interpolate (lerp) towards a target value over time.
// This is crucial for showing the *gradual* shift in equilibrium after a sudden change.
function useLerpState(target, speed = 0.05) {
  const [current, setCurrent] = useState(target);
  
  useEffect(() => {
    let frameId;
    let lastTime = performance.now();
    
    const tick = (time) => {
      const dt = time - lastTime;
      lastTime = time;
      
      setCurrent(prev => {
        const diff = target - prev;
        if (Math.abs(diff) < 0.0001) return target;
        // Frame-rate independent lerp
        const t = 1 - Math.pow(1 - speed, dt / 16.66);
        return prev + diff * t;
      });
      frameId = requestAnimationFrame(tick);
    };
    
    frameId = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(frameId);
  }, [target, speed]);
  
  return current;
}

// Solves equilibrium for A + B <-> C
const calcConcentrationEq = (Atot, Btot, Kc) => {
  const a = Kc;
  const b = -(Kc * Atot + Kc * Btot + 1);
  const c = Kc * Atot * Btot;
  const discriminant = b * b - 4 * a * c;
  // Use the negative root for physical reality (smaller than Atot and Btot)
  return (-b - Math.sqrt(Math.max(0, discriminant))) / (2 * a);
};

// Solves equilibrium for 2A <-> B
const calcPressureEq = (V, N0, Kc) => {
  const a = 2 * Kc;
  const b = V;
  const c = -V * N0;
  const discriminant = b * b - 4 * a * c;
  return (-b + Math.sqrt(Math.max(0, discriminant))) / (2 * a);
};

// UI Component for the Equilibrium Arrow
const EqArrow = ({ shift }) => (
  <div className="flex flex-col items-center justify-center mx-3 font-bold">
    <span className={`text-2xl leading-[0.5] ${shift === 'right' ? 'text-blue-600 scale-125 transition-transform' : 'text-slate-400'}`}>⟶</span>
    <span className={`text-2xl leading-[0.5] ${shift === 'left' ? 'text-blue-600 scale-125 transition-transform' : 'text-slate-400'}`}>⟵</span>
  </div>
);

// --- Sub-Simulations ---

const ConcentrationSim = () => {
  const [Atot, setAtot] = useState(0.5); // Fe3+
  const [Btot, setBtot] = useState(0.5); // SCN-
  const [actionMsg, setActionMsg] = useState("System is at equilibrium.");
  
  const Kc = 40;
  
  // Calculate exact target equilibrium
  const target_C = useMemo(() => calcConcentrationEq(Atot, Btot, Kc), [Atot, Btot, Kc]);
  // Animate current state towards target
  const current_C = useLerpState(target_C, 0.04);
  
  const shift = current_C > target_C + 0.005 ? 'left' : current_C < target_C - 0.005 ? 'right' : 'eq';

  // Calculate RGB color based on concentration
  // Fe3+ (yellow) to FeSCN2+ (blood red)
  const t = Math.min(1, Math.max(0, current_C / 1.0)); // Normalize
  const r = Math.round(250 - t * (250 - 140));
  const g = Math.round(230 - t * 230);
  const b = Math.round(100 - t * 100);
  const fluidColor = `rgb(${r}, ${g}, ${b})`;
  
  const fluidStyle = { height: '75%', backgroundColor: fluidColor };

  const handleAddA = () => { setAtot(p => Math.min(2, p + 0.3)); setActionMsg("Added Fe³⁺. System shifts RIGHT to consume it."); };
  const handleSubA = () => { setAtot(p => Math.max(0.1, p - 0.3)); setActionMsg("Removed Fe³⁺. System shifts LEFT to replace it."); };
  const handleAddB = () => { setBtot(p => Math.min(2, p + 0.3)); setActionMsg("Added SCN⁻. System shifts RIGHT to consume it."); };
  const handleSubB = () => { setBtot(p => Math.max(0.1, p - 0.3)); setActionMsg("Removed SCN⁻. System shifts LEFT to replace it."); };
  const handleReset = () => { setAtot(0.5); setBtot(0.5); setActionMsg("System reset to standard equilibrium."); };

  return (
    <div className="flex flex-col md:flex-row gap-8 items-center justify-center p-6 bg-slate-50 rounded-xl">
      {/* Visualizer */}
      <div className="relative w-40 h-56 flex flex-col items-center justify-end">
        <div className="absolute top-0 w-36 h-full border-4 border-slate-300 rounded-b-[2.5rem] bg-white/50 backdrop-blur overflow-hidden flex items-end shadow-inner z-10">
          <div 
            className="w-full transition-colors duration-75"
            style={fluidStyle}
          />
        </div>
        {/* Beaker Lip */}
        <div className="absolute top-[-6px] w-40 h-4 border-4 border-slate-300 rounded-[50%] bg-white/20 z-20"></div>
        <div className="absolute bottom-6 text-white font-bold drop-shadow-md z-30 opacity-80">
          Mixture
        </div>
      </div>

      {/* Controls & Data */}
      <div className="flex-1 max-w-md space-y-6">
        <div className="text-center p-4 bg-white rounded-xl shadow-sm border border-slate-100">
          <div className="flex items-center justify-center text-xl mb-2">
            <span className="text-amber-500 font-semibold">Fe³⁺</span>
            <span className="text-slate-400 mx-2">+</span>
            <span className="text-slate-500 font-semibold">SCN⁻</span>
            <EqArrow shift={shift} />
            <span className="text-red-700 font-bold">FeSCN²⁺</span>
          </div>
          <div className="text-sm text-slate-500 flex justify-center gap-4">
            <span>(Pale Yellow)</span>
            <span>(Colorless)</span>
            <span>(Blood Red)</span>
          </div>
        </div>

        <div className="grid grid-cols-2 gap-4">
          <div className="space-y-2 bg-white p-3 rounded-xl border border-slate-100 shadow-sm">
            <label className="text-sm font-semibold text-slate-600 block text-center">Iron(III) Ion (Fe³⁺)</label>
            <div className="flex justify-center gap-2">
              <button onClick={handleSubA} className="p-2 bg-slate-100 hover:bg-slate-200 rounded-lg text-slate-700"><Minus size={18}/></button>
              <button onClick={handleAddA} className="p-2 bg-amber-100 hover:bg-amber-200 rounded-lg text-amber-700"><Plus size={18}/></button>
            </div>
          </div>
          <div className="space-y-2 bg-white p-3 rounded-xl border border-slate-100 shadow-sm">
            <label className="text-sm font-semibold text-slate-600 block text-center">Thiocyanate (SCN⁻)</label>
            <div className="flex justify-center gap-2">
              <button onClick={handleSubB} className="p-2 bg-slate-100 hover:bg-slate-200 rounded-lg text-slate-700"><Minus size={18}/></button>
              <button onClick={handleAddB} className="p-2 bg-slate-200 hover:bg-slate-300 rounded-lg text-slate-700"><Plus size={18}/></button>
            </div>
          </div>
        </div>

        <div className="p-4 bg-blue-50 text-blue-800 rounded-xl flex items-start gap-3 border border-blue-100">
          <Info className="shrink-0 mt-0.5" size={18} />
          <p className="text-sm leading-relaxed">{actionMsg}</p>
        </div>

        <button onClick={handleReset} className="w-full py-2 flex items-center justify-center gap-2 text-slate-500 hover:bg-slate-100 rounded-lg transition-colors">
          <RefreshCw size={16} /> Reset Simulation
        </button>
      </div>
    </div>
  );
};

const PressureSim = () => {
  const [V, setV] = useState(5.0); // Volume slider 2 to 10
  const [actionMsg, setActionMsg] = useState("System is at equilibrium.");
  
  const N0 = 10;
  const Kc = 2.5; // Kc = [N2O4]/[NO2]^2
  
  const target_nA = useMemo(() => calcPressureEq(V, N0, Kc), [V, N0, Kc]);
  const current_nA = useLerpState(target_nA, 0.05); // Faster lerp to see the bounce
  
  const shift = current_nA > target_nA + 0.05 ? 'right' : current_nA < target_nA - 0.05 ? 'left' : 'eq';
  
  // Color calculation based on instant concentration
  const current_conc = current_nA / V;
  // Map concentration to brown intensity
  const t = Math.min(1, current_conc / 2.0);
  const r = Math.round(250 - t * (250 - 139));
  const g = Math.round(250 - t * (250 - 69));
  const b = Math.round(250 - t * (250 - 19));
  const gasColor = `rgb(${r}, ${g}, ${b})`;
  
  const gasContainerStyle = { height: `${(V / 10) * 100}%`, backgroundColor: gasColor };
  const particlesStyle = { backgroundImage: 'radial-gradient(circle, #000 1px, transparent 1px)', backgroundSize: `${V*4}px ${V*4}px` };
  const pistonStyle = { bottom: `calc(${(V / 10) * 14}rem - 8px)` };

  const handleVolumeChange = (e) => {
    const newV = parseFloat(e.target.value);
    setV(newV);
    if (newV < V) setActionMsg("Volume decreased (Pressure increased). Immediate darkening occurs, then system shifts RIGHT (to fewer moles) to relieve pressure, lightening slightly.");
    else if (newV > V) setActionMsg("Volume increased (Pressure decreased). Immediate lightening occurs, then system shifts LEFT (to more moles) to restore pressure, darkening slightly.");
  };

  return (
    <div className="flex flex-col md:flex-row gap-8 items-center justify-center p-6 bg-slate-50 rounded-xl">
      {/* Visualizer */}
      <div className="relative w-32 h-64 flex flex-col items-center">
        {/* Cylinder Body */}
        <div className="absolute bottom-0 w-full h-56 border-x-4 border-b-4 border-slate-400 rounded-b-lg bg-white overflow-hidden shadow-inner flex flex-col justify-end">
          {/* Gas Gas */}
          <div 
            className="w-full transition-colors duration-75 relative"
            style={gasContainerStyle}
          >
             {/* Particles representation (abstract) */}
             <div className="absolute inset-0 opacity-20" style={particlesStyle}></div>
          </div>
        </div>
        {/* Piston */}
        <div 
          className="absolute w-36 h-4 bg-slate-600 rounded-sm z-20 cursor-ns-resize shadow-md transition-all duration-75"
          style={pistonStyle}
        >
          {/* Piston Handle */}
          <div className="absolute bottom-full left-1/2 -translate-x-1/2 w-4 h-32 bg-slate-300"></div>
          <div className="absolute bottom-[calc(100%+8rem)] left-1/2 -translate-x-1/2 w-16 h-4 bg-slate-600 rounded-sm"></div>
        </div>
      </div>

      {/* Controls & Data */}
      <div className="flex-1 max-w-md space-y-6">
        <div className="text-center p-4 bg-white rounded-xl shadow-sm border border-slate-100">
          <div className="flex items-center justify-center text-xl mb-2">
            <span className="text-orange-800 font-semibold flex items-center gap-1">2 NO₂ <span className="text-xs font-normal">(g)</span></span>
            <EqArrow shift={shift} />
            <span className="text-slate-600 font-semibold flex items-center gap-1">N₂O₄ <span className="text-xs font-normal">(g)</span></span>
          </div>
          <div className="text-sm text-slate-500 flex justify-center gap-12">
            <span>(Brown)</span>
            <span>(Colorless)</span>
          </div>
          <div className="mt-2 text-xs font-semibold text-slate-400">2 moles gas ⇌ 1 mole gas</div>
        </div>

        <div className="space-y-4 bg-white p-5 rounded-xl border border-slate-100 shadow-sm">
          <div className="flex justify-between text-sm font-semibold text-slate-600">
            <label>Adjust Volume (Inverse to Pressure)</label>
            <span className="text-blue-600">{V.toFixed(1)} L</span>
          </div>
          <input 
            type="range" min="2" max="10" step="0.1" 
            value={V} onChange={handleVolumeChange}
            className="w-full h-2 bg-slate-200 rounded-lg appearance-none cursor-pointer accent-blue-600"
          />
          <div className="flex justify-between text-xs text-slate-400">
            <span>High Pressure</span>
            <span>Low Pressure</span>
          </div>
        </div>

        <div className="p-4 bg-blue-50 text-blue-800 rounded-xl flex items-start gap-3 border border-blue-100">
          <Info className="shrink-0 mt-0.5" size={18} />
          <p className="text-sm leading-relaxed">{actionMsg}</p>
        </div>
      </div>
    </div>
  );
};

const TemperatureSim = () => {
  const [Tcelsius, setTcelsius] = useState(25);
  const [actionMsg, setActionMsg] = useState("System is at standard temperature (25°C).");
  
  const N0 = 10;
  const V = 5.0; // Fixed volume
  
  // Van 't Hoff Equation for Kc(T)
  // Forward reaction is Exothermic (dH is negative)
  // 2NO2 <-> N2O4 + 57.2 kJ
  const dH = -57200; 
  const R = 8.314;
  const T_kelvin = Tcelsius + 273.15;
  const Kc298 = 2.5;
  
  const Kc_T = useMemo(() => {
    return Kc298 * Math.exp((-dH / R) * (1 / T_kelvin - 1 / 298.15));
  }, [T_kelvin, dH]);

  const target_nA = useMemo(() => calcPressureEq(V, N0, Kc_T), [V, N0, Kc_T]);
  const current_nA = useLerpState(target_nA, 0.02); // Slower shift for temperature
  
  const shift = current_nA > target_nA + 0.05 ? 'right' : current_nA < target_nA - 0.05 ? 'left' : 'eq';
  
  const current_conc = current_nA / V;
  // Map concentration to brown intensity
  const t = Math.min(1, current_conc / 2.0);
  const r = Math.round(250 - t * (250 - 139));
  const g = Math.round(250 - t * (250 - 69));
  const b = Math.round(250 - t * (250 - 19));
  const gasColor = `rgb(${r}, ${g}, ${b})`;

  // Bath color based on temperature
  const bathRatio = Tcelsius / 100;
  const bathColor = `rgba(${Math.round(bathRatio * 255)}, 100, ${Math.round((1 - bathRatio) * 255)}, 0.3)`;
  
  const bathBgStyle = { backgroundColor: bathColor };
  const gasBgStyle = { backgroundColor: gasColor };

  const handleTempChange = (e) => {
    const newT = parseInt(e.target.value);
    setTcelsius(newT);
    if (newT > 25) setActionMsg("Temperature increased. Added Heat is treated as a product, so the system shifts LEFT to consume it, turning browner.");
    else if (newT < 25) setActionMsg("Temperature decreased. Removed Heat causes the system to shift RIGHT to produce more, turning lighter.");
    else setActionMsg("System is at standard temperature (25°C).");
  };

  return (
    <div className="flex flex-col md:flex-row gap-8 items-center justify-center p-6 bg-slate-50 rounded-xl">
      {/* Visualizer */}
      <div className="relative w-48 h-56 flex flex-col items-center justify-end">
        {/* Thermal Bath */}
        <div 
          className="absolute bottom-0 w-full h-32 rounded-lg border-2 border-slate-300 transition-colors duration-300 z-0"
          style={bathBgStyle}
        >
           {/* Heater coils/Ice cubes abstract */}
           <div className="absolute inset-0 flex justify-center gap-2 pt-2 opacity-50">
             {Tcelsius > 60 && <div className="text-red-500 font-bold">↑ HEAT ↑</div>}
             {Tcelsius < 10 && <div className="text-blue-500 font-bold text-xl">❄ ❄ ❄</div>}
           </div>
        </div>
        
        {/* Sealed Flask */}
        <div className="absolute bottom-4 w-24 h-24 rounded-full border-4 border-white shadow-lg bg-white/40 backdrop-blur-sm z-10 overflow-hidden flex items-center justify-center">
          <div className="absolute inset-0 transition-colors duration-300" style={gasBgStyle} />
        </div>
        {/* Flask Neck */}
        <div className="absolute bottom-[6.5rem] w-6 h-12 border-x-4 border-white bg-white/40 backdrop-blur-sm z-10 overflow-hidden">
          <div className="absolute inset-0 transition-colors duration-300" style={gasBgStyle} />
          {/* Stopper */}
          <div className="absolute top-0 w-full h-3 bg-slate-800 rounded-sm"></div>
        </div>
      </div>

      {/* Controls & Data */}
      <div className="flex-1 max-w-md space-y-6 z-20">
        <div className="text-center p-4 bg-white rounded-xl shadow-sm border border-slate-100">
          <div className="flex items-center justify-center text-lg mb-2">
            <span className="text-orange-800 font-semibold">2 NO₂</span>
            <EqArrow shift={shift} />
            <span className="text-slate-600 font-semibold">N₂O₄</span>
            <span className="text-red-500 font-bold ml-2 text-sm">+ Heat</span>
          </div>
          <div className="mt-1 text-xs font-semibold text-slate-500 bg-slate-100 inline-block px-3 py-1 rounded-full">
            Forward Reaction is Exothermic (ΔH = -57.2 kJ/mol)
          </div>
        </div>

        <div className="space-y-4 bg-white p-5 rounded-xl border border-slate-100 shadow-sm">
          <div className="flex justify-between text-sm font-semibold text-slate-600">
            <label>Adjust Temperature</label>
            <span className={Tcelsius > 50 ? 'text-red-500' : Tcelsius < 15 ? 'text-blue-500' : 'text-slate-600'}>
              {Tcelsius}°C
            </span>
          </div>
          <input 
            type="range" min="0" max="100" step="1" 
            value={Tcelsius} onChange={handleTempChange}
            className={`w-full h-2 rounded-lg appearance-none cursor-pointer ${Tcelsius > 50 ? 'bg-red-200 accent-red-600' : Tcelsius < 15 ? 'bg-blue-200 accent-blue-600' : 'bg-slate-200 accent-slate-600'}`}
          />
          <div className="flex justify-between text-xs text-slate-400 font-semibold">
            <span className="text-blue-400">0°C (Ice Bath)</span>
            <span className="text-red-400">100°C (Boiling)</span>
          </div>
        </div>

        <div className="p-4 bg-blue-50 text-blue-800 rounded-xl flex items-start gap-3 border border-blue-100">
          <Info className="shrink-0 mt-0.5" size={18} />
          <p className="text-sm leading-relaxed">{actionMsg}</p>
        </div>
      </div>
    </div>
  );
};

// --- Main Application ---

export default function App() {
  const [activeTab, setActiveTab] = useState('concentration');

  const tabs = [
    { id: 'concentration', label: 'Concentration', icon: Beaker },
    { id: 'pressure', label: 'Pressure', icon: Cylinder },
    { id: 'temperature', label: 'Temperature', icon: ThermometerSun },
  ];

  return (
    <div className="min-h-screen bg-slate-100 text-slate-800 font-sans p-4 md:p-8 flex flex-col items-center">
      <div className="w-full max-w-4xl bg-white shadow-xl rounded-2xl overflow-hidden border border-slate-200">
        
        {/* Header */}
        <div className="bg-slate-900 text-white p-6 text-center">
          <h1 className="text-2xl md:text-3xl font-bold tracking-tight mb-2">Le Chatelier's Principle</h1>
          <p className="text-slate-300 text-sm md:text-base">
            When a system at equilibrium is disturbed, it shifts to counteract the disturbance and establish a new equilibrium.
          </p>
        </div>

        {/* Tab Navigation */}
        <div className="flex border-b border-slate-200 bg-slate-50 overflow-x-auto">
          {tabs.map(tab => {
            const Icon = tab.icon;
            const isActive = activeTab === tab.id;
            return (
              <button
                key={tab.id}
                onClick={() => setActiveTab(tab.id)}
                className={`flex-1 min-w-[120px] py-4 px-6 flex items-center justify-center gap-2 text-sm font-semibold transition-colors
                  ${isActive ? 'bg-white text-blue-600 border-b-2 border-blue-600' : 'text-slate-500 hover:text-slate-700 hover:bg-slate-100'}`}
              >
                <Icon size={18} />
                {tab.label}
              </button>
            );
          })}
        </div>

        {/* Active Simulation Container */}
        <div className="p-6 md:p-8">
          {activeTab === 'concentration' && <ConcentrationSim />}
          {activeTab === 'pressure' && <PressureSim />}
          {activeTab === 'temperature' && <TemperatureSim />}
        </div>
        
      </div>
    </div>
  );
}
