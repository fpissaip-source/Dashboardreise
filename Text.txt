import React, { useState, useEffect, useRef } from 'react';
import { MapPin, Fuel, MessageSquare, Send, Car, AlertTriangle, Activity } from 'lucide-react';

const apiKey = "";

const calcDistance = (lat1, lon1, lat2, lon2) => {
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * Math.sin(dLon / 2) ** 2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
};

const fetchWithRetry = async (url, options, maxRetries = 5) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const r = await fetch(url, options);
      if (!r.ok) throw new Error(`HTTP ${r.status}`);
      return await r.json();
    } catch (e) {
      if (i === maxRetries - 1) throw e;
      await new Promise(res => setTimeout(res, 2 ** i * 1000));
    }
  }
};

const WAYPOINTS = [
  {
    name: "Essen",
    country: "Deutschland",
    lat: 51.4556, lng: 7.0116,
    image: "https://images.unsplash.com/photo-1559827291-72f31e14e1d0?w=1200&q=85",
    accent: "#3b82f6",
    desc: "Ruhrgebiet · Startpunkt",
    warning: { type: "info", text: "A3 → A5 Richtung Basel. Tankstop vor der Schweizer Grenze empfohlen!" }
  },
  {
    name: "Basel",
    country: "Schweiz",
    lat: 47.5596, lng: 7.5886,
    image: "https://images.unsplash.com/photo-1559827260-dc66d52bef19?w=1200&q=85",
    accent: "#ef4444",
    desc: "Rhein & Kunst · Grenzstadt",
    warning: { type: "warning", text: "E-Vignette erforderlich. Radarwarner streng verboten in der Schweiz!" }
  },
  {
    name: "Mailand",
    country: "Italien",
    lat: 45.4654, lng: 9.1859,
    image: "https://images.unsplash.com/photo-1520175480921-4edfa2983e0f?w=1200&q=85",
    accent: "#22c55e",
    desc: "La Metropolitana · Mode & Design",
    warning: { type: "danger", text: "Area C Mailand voraus! City-Maut Registrierung erforderlich." }
  },
  {
    name: "Nizza",
    country: "Frankreich",
    lat: 43.7102, lng: 7.2620,
    image: "https://images.unsplash.com/photo-1533929736458-ca588d08c8be?w=1200&q=85",
    accent: "#06b6d4",
    desc: "Côte d'Azur · Ziel",
    warning: { type: "info", text: "Riviera-Promenade in Nizza ist mautpflichtig. Herzlich willkommen!" }
  }
];

const getWaypointByLat = (lat) => {
  if (lat > 49) return 0;
  if (lat > 46.5) return 1;
  if (lat > 44) return 2;
  return 3;
};

const getFuel = (lat) => {
  if (lat < 47.5 && lat >= 45.8) return { currency: "CHF", base: 1.85 };
  if (lat < 45.8 && lat > 44)    return { currency: "€",   base: 2.15 };
  if (lat <= 44)                  return { currency: "€",   base: 2.05 };
  return { currency: "€", base: 1.95 };
};

const Speedometer = ({ speed, accent }) => {
  const max = 200, r = 72;
  const circ = 2 * Math.PI * r;
  const offset = circ - (circ * Math.min(speed, max)) / max;
  const needleAngle = -140 + (Math.min(speed, max) / max) * 280;
  return (
    <div className="relative flex items-center justify-center" style={{ width: 180, height: 180 }}>
      <svg width="180" height="180" viewBox="0 0 180 180">
        <circle cx="90" cy="90" r={r} fill="none" stroke="#1e293b" strokeWidth="10" />
        <circle cx="90" cy="90" r={r} fill="none" stroke={accent} strokeWidth="10"
          strokeDasharray={circ} strokeDashoffset={offset} strokeLinecap="round"
          transform="rotate(-90 90 90)"
          style={{ transition: 'stroke-dashoffset 0.5s ease', filter: `drop-shadow(0 0 6px ${accent})` }}
        />
        {[0, 50, 100, 150, 200].map(m => {
          const a = ((-140 + (m / max) * 280) * Math.PI) / 180;
          return <line key={m} x1={90 + 63 * Math.cos(a)} y1={90 + 63 * Math.sin(a)} x2={90 + 54 * Math.cos(a)} y2={90 + 54 * Math.sin(a)} stroke="#334155" strokeWidth="2" />;
        })}
        <g transform={`rotate(${needleAngle} 90 90)`} style={{ transition: 'transform 0.5s ease' }}>
          <polygon points="90,18 87,90 93,90" fill={accent} opacity="0.9" />
        </g>
        <circle cx="90" cy="90" r="7" fill="#0f172a" stroke={accent} strokeWidth="3" />
      </svg>
      <div className="absolute flex flex-col items-center" style={{ top: '58%' }}>
        <span className="text-3xl font-black text-white leading-none">{speed}</span>
        <span className="text-xs text-slate-400 tracking-widest">KM/H</span>
      </div>
    </div>
  );
};

export default function App() {
  const [speed, setSpeed] = useState(0);
  const [location, setLocation] = useState({ lat: 51.4556, lng: 7.0116 });
  const [isSimulating, setIsSimulating] = useState(false);
  const [gpsActive, setGpsActive] = useState(false);
  const [gpsError, setGpsError] = useState(null);
  const [displayIdx, setDisplayIdx] = useState(0);
  const [manualCity, setManualCity] = useState(false);
  const [aiInput, setAiInput] = useState("");
  const [chatHistory, setChatHistory] = useState([
    { role: 'ai', content: 'Hallo! Sag mir, worauf du Lust hast – ich finde etwas passend zu deinem GPS-Standort.' }
  ]);
  const [isAiLoading, setIsAiLoading] = useState(false);
  const chatEndRef = useRef(null);

  const gpsIdx = getWaypointByLat(location.lat);
  const active = WAYPOINTS[displayIdx];
  const startLat = WAYPOINTS[0].lat, endLat = WAYPOINTS[3].lat;
  const progress = Math.max(0, Math.min(100, ((startLat - location.lat) / (startLat - endLat)) * 100));
  const fuel = getFuel(location.lat);

  useEffect(() => { chatEndRef.current?.scrollIntoView({ behavior: "smooth" }); }, [chatHistory]);

  // GPS
  useEffect(() => {
    if (isSimulating) return;
    if (!("geolocation" in navigator)) {
      setGpsError("GPS nicht verfügbar");
      return;
    }
    setGpsError(null);
    let lastPos = null, lastTime = null;
    const id = navigator.geolocation.watchPosition(
      (pos) => {
        setGpsActive(true);
        setGpsError(null);
        const { latitude, longitude, speed: gs } = pos.coords;
        setLocation({ lat: latitude, lng: longitude });
        let s = gs !== null ? gs * 3.6 : 0;
        if (!s && lastPos && lastTime) {
          const d = calcDistance(lastPos.lat, lastPos.lng, latitude, longitude);
          const t = (pos.timestamp - lastTime) / 3600000;
          if (t > 0) s = d / t;
        }
        setSpeed(Math.round(s));
        if (!manualCity) setDisplayIdx(getWaypointByLat(latitude));
        lastPos = { lat: latitude, lng: longitude };
        lastTime = pos.timestamp;
      },
      (err) => {
        setGpsActive(false);
        const msgs = {
          1: "GPS-Zugriff verweigert – bitte erlauben",
          2: "GPS-Signal nicht verfügbar",
          3: "GPS-Anfrage Timeout"
        };
        setGpsError(msgs[err.code] || "GPS-Fehler");
      },
      { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
    );
    return () => navigator.geolocation.clearWatch(id);
  }, [isSimulating, manualCity]);

  // Simulation
  useEffect(() => {
    if (!isSimulating) return;
    let sLat = 51.4556, sLng = 7.0116, sSpeed = 0;
    const iv = setInterval(() => {
      sLat -= 0.05;
      sLng += sLat > 47 ? 0.01 : -0.01;
      sSpeed = sSpeed < 120 ? sSpeed + Math.floor(Math.random() * 10) + 5 : sSpeed + (Math.random() > 0.5 ? 2 : -2);
      setLocation({ lat: sLat, lng: sLng });
      setSpeed(Math.round(sSpeed));
      setGpsActive(true);
      if (!manualCity) setDisplayIdx(getWaypointByLat(sLat));
      if (sLat <= 43.7) { setIsSimulating(false); setSpeed(0); }
    }, 2000);
    return () => clearInterval(iv);
  }, [isSimulating, manualCity]);

  const handleAskGemini = async () => {
    if (!aiInput.trim()) return;
    if (!apiKey) {
      setChatHistory(p => [...p,
        { role: 'user', content: aiInput },
        { role: 'ai', content: "Kein API-Key hinterlegt. Bitte füge deinen Gemini API-Key in der App ein." }
      ]);
      setAiInput("");
      return;
    }
    const msg = aiInput;
    setChatHistory(p => [...p, { role: 'user', content: msg }]);
    setAiInput("");
    setIsAiLoading(true);
    try {
      const prompt = `Du bist ein Roadtrip-Assistent (Essen → Nizza).
GPS: ${location.lat.toFixed(4)}, ${location.lng.toFixed(4)}. Region: ${active.name}, ${active.country}.
Nutzer: "${msg}"
Antworte in max. 2 Sätzen mit konkreten Orten in der Nähe.`;
      const res = await fetchWithRetry(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] }) }
      );
      const text = res.candidates?.[0]?.content?.parts?.[0]?.text || "Ich konnte die Umgebung nicht scannen.";
      setChatHistory(p => [...p, { role: 'ai', content: text }]);
    } catch {
      setChatHistory(p => [...p, { role: 'ai', content: "Verbindung fehlgeschlagen. Sind wir im Gotthardtunnel?" }]);
    } finally {
      setIsAiLoading(false);
    }
  };

  const warningColors = {
    info:    'bg-blue-950/60 border-blue-800/60 text-blue-200',
    warning: 'bg-yellow-950/60 border-yellow-800/60 text-yellow-200',
    danger:  'bg-red-950/60 border-red-800/60 text-red-200',
  };

  return (
    <div className="min-h-screen bg-slate-950 text-white font-sans overflow-x-hidden">

      {/* ── HERO ─────────────────────────────────────── */}
      <div className="relative w-full" style={{ height: '75vh', minHeight: 480 }}>
        {WAYPOINTS.map((wp, i) => (
          <div key={wp.name} className="absolute inset-0 transition-opacity duration-700"
            style={{ opacity: i === displayIdx ? 1 : 0 }}>
            <img src={wp.image} alt={wp.name} className="w-full h-full object-cover" />
          </div>
        ))}
        {/* Fade: dark top for readability, fades to bg-slate-950 at bottom */}
        <div className="absolute inset-0 bg-gradient-to-b from-black/65 via-black/5 to-slate-950 pointer-events-none" />

        {/* Header */}
        <div className="relative z-10 flex items-center justify-between px-5 pt-5">
          <div className="flex items-center gap-2">
            <Car className="w-5 h-5 text-white" />
            <span className="font-bold text-white text-sm tracking-wide">Roadtrip Copilot</span>
          </div>
          <div className="flex items-center gap-2">
            <div className={`text-xs px-2.5 py-1 rounded-full flex items-center gap-1 backdrop-blur-sm border ${
              gpsActive ? 'bg-green-500/20 text-green-300 border-green-500/30' : 'bg-red-500/20 text-red-300 border-red-500/30'
            }`}>
              <MapPin className="w-3 h-3" />
              {gpsActive ? 'GPS' : 'Suche...'}
            </div>
            <button
              onClick={() => setIsSimulating(s => !s)}
              className={`text-xs px-3 py-1 rounded-full backdrop-blur-sm border transition-all ${
                isSimulating ? 'bg-red-500/30 border-red-400/40 text-red-200' : 'bg-white/10 border-white/20 text-white'
              }`}
            >
              {isSimulating ? '⏹ Stop' : '▶ Demo'}
            </button>
          </div>
        </div>

        {/* GPS error banner */}
        {gpsError && (
          <div className="relative z-10 mx-4 mt-3 bg-red-950/70 border border-red-700/50 rounded-xl px-4 py-2 text-xs text-red-200 backdrop-blur-sm flex items-center gap-2">
            <AlertTriangle className="w-4 h-4 shrink-0" />
            {gpsError}
          </div>
        )}

        {/* City name — large */}
        <div className="absolute bottom-14 left-0 right-0 px-6 z-10">
          <div className="text-xs font-semibold tracking-widest uppercase mb-1" style={{ color: active.accent }}>
            {active.country} · {Math.round(progress)}% der Route
          </div>
          <h1 className="text-6xl font-black tracking-tight text-white drop-shadow-2xl leading-none">
            {active.name}
          </h1>
          <p className="text-white/55 text-sm mt-1">{active.desc}</p>
          <div className="mt-3 h-1 bg-white/10 rounded-full w-44 overflow-hidden">
            <div className="h-full rounded-full transition-all duration-1000"
              style={{ width: `${progress}%`, backgroundColor: active.accent, boxShadow: `0 0 8px ${active.accent}` }} />
          </div>
        </div>
      </div>

      {/* ── CITY TILES ──────────────────────────────── */}
      <div className="px-4 -mt-2 relative z-20">
        <p className="text-xs text-slate-500 uppercase tracking-widest mb-3 px-1">Route</p>
        <div className="flex gap-3 overflow-x-auto pb-1" style={{ scrollbarWidth: 'none' }}>
          {WAYPOINTS.map((wp, i) => {
            const isPassed = i < gpsIdx;
            const isActive = i === displayIdx;
            return (
              <button key={wp.name} onClick={() => { setDisplayIdx(i); setManualCity(true); }}
                className="relative flex-shrink-0 rounded-2xl overflow-hidden focus:outline-none active:scale-95 transition-transform"
                style={{
                  width: 120, height: 85,
                  boxShadow: isActive ? `0 0 0 2.5px ${wp.accent}, 0 6px 20px ${wp.accent}50` : 'none',
                  opacity: isPassed && !isActive ? 0.5 : 1
                }}
              >
                <img src={wp.image} alt={wp.name} className="absolute inset-0 w-full h-full object-cover" />
                <div className="absolute inset-0 bg-gradient-to-t from-black/85 via-black/20 to-transparent" />
                {isPassed && <div className="absolute top-2 right-2 w-4 h-4 rounded-full bg-green-500 flex items-center justify-center text-xs">✓</div>}
                {isActive && <div className="absolute top-2 right-2 w-2 h-2 rounded-full animate-pulse" style={{ backgroundColor: wp.accent }} />}
                <div className="absolute bottom-0 left-0 right-0 p-2 text-left">
                  <div className="text-white font-bold text-xs leading-tight">{wp.name}</div>
                  <div className="text-white/45 text-xs">{wp.country}</div>
                </div>
              </button>
            );
          })}
        </div>
        {manualCity && (
          <button onClick={() => { setManualCity(false); setDisplayIdx(gpsIdx); }}
            className="mt-2 text-xs text-slate-500 hover:text-slate-300 transition-colors">
            ← Zurück zu GPS
          </button>
        )}
      </div>

      {/* ── DASHBOARD ───────────────────────────────── */}
      <div className="px-4 mt-5 pb-12 space-y-4">

        {/* Tacho + Sprit */}
        <div className="grid grid-cols-2 gap-4">
          <div className="bg-slate-900/80 backdrop-blur border border-slate-800 rounded-2xl p-4 flex flex-col items-center">
            <div className="flex items-center gap-1.5 self-start mb-1 text-slate-400 text-xs">
              <Activity className="w-3 h-3" /> Tacho
            </div>
            <Speedometer speed={speed} accent={active.accent} />
            <div className="mt-1 text-xs text-slate-500 font-mono">{location.lat.toFixed(3)}° {location.lng.toFixed(3)}°</div>
          </div>

          <div className="bg-slate-900/80 backdrop-blur border border-slate-800 rounded-2xl p-4">
            <div className="flex items-center gap-1.5 mb-3 text-slate-400 text-xs">
              <Fuel className="w-3 h-3 text-green-400" /> Live Sprit
            </div>
            <div className="space-y-2.5">
              {[
                { label: "Autobahn",   price: (fuel.base + 0.15).toFixed(2) },
                { label: "Supermarkt", price: (fuel.base - 0.08).toFixed(2), best: true },
                { label: "Dorf",       price: fuel.base.toFixed(2) },
              ].map((s, i) => (
                <div key={i} className="flex justify-between items-center">
                  <span className={`text-xs ${s.best ? 'text-green-400' : 'text-slate-400'}`}>{s.label}</span>
                  <span className={`font-bold text-sm ${s.best ? 'text-green-400' : 'text-white'}`}>
                    {s.price} <span className="text-xs font-normal text-slate-500">{fuel.currency}</span>
                  </span>
                </div>
              ))}
            </div>
            <div className="mt-3 text-xs text-slate-500 border-t border-slate-800 pt-2">
              {active.country} · live
            </div>
          </div>
        </div>

        {/* Regional warning */}
        {active.warning && (
          <div className={`flex items-start gap-3 border rounded-2xl p-4 text-sm ${warningColors[active.warning.type]}`}>
            <AlertTriangle className="w-4 h-4 shrink-0 mt-0.5" />
            <p>{active.warning.text}</p>
          </div>
        )}

        {/* Gemini */}
        <div className="bg-slate-900/80 backdrop-blur border border-slate-800 rounded-2xl overflow-hidden">
          <div className="flex items-center gap-3 px-4 py-3 border-b border-slate-800">
            <div className="w-8 h-8 rounded-full flex items-center justify-center shrink-0"
              style={{ backgroundColor: `${active.accent}25`, border: `1px solid ${active.accent}40` }}>
              <MessageSquare className="w-4 h-4" style={{ color: active.accent }} />
            </div>
            <div>
              <div className="text-sm font-semibold">Gemini Copilot</div>
              <div className="text-xs text-slate-500">{active.name} Region</div>
            </div>
          </div>

          <div className="h-52 overflow-y-auto p-4 space-y-3">
            {chatHistory.map((msg, i) => (
              <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                <div className={`max-w-[85%] rounded-2xl px-3 py-2 text-sm leading-relaxed ${
                  msg.role === 'user'
                    ? 'text-white rounded-br-none'
                    : 'bg-slate-800 text-slate-200 rounded-bl-none border border-slate-700'
                }`} style={msg.role === 'user' ? { backgroundColor: active.accent } : {}}>
                  {msg.content}
                </div>
              </div>
            ))}
            {isAiLoading && (
              <div className="flex justify-start">
                <div className="bg-slate-800 border border-slate-700 rounded-2xl rounded-bl-none px-3 py-2 flex gap-1">
                  {[0, 0.15, 0.3].map((d, i) => (
                    <div key={i} className="w-1.5 h-1.5 rounded-full animate-bounce"
                      style={{ backgroundColor: active.accent, animationDelay: `${d}s` }} />
                  ))}
                </div>
              </div>
            )}
            <div ref={chatEndRef} />
          </div>

          <div className="p-3 border-t border-slate-800">
            <div className="flex gap-2">
              <input type="text" value={aiInput}
                onChange={e => setAiInput(e.target.value)}
                onKeyPress={e => e.key === 'Enter' && handleAskGemini()}
                placeholder={`Tipps für ${active.name}?`}
                className="flex-1 bg-slate-900 border border-slate-700 text-white text-sm rounded-xl px-4 py-2.5 focus:outline-none focus:border-blue-500 transition-colors"
              />
              <button onClick={handleAskGemini} disabled={isAiLoading || !aiInput.trim()}
                className="rounded-xl px-3 py-2 flex items-center justify-center transition-all disabled:opacity-30"
                style={{ backgroundColor: active.accent }}>
                <Send className="w-4 h-4 text-white" />
              </button>
            </div>
          </div>
        </div>

      </div>
    </div>
  );
}
