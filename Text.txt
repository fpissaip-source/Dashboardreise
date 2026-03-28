import React, { useState, useEffect, useRef } from 'react';
import { Navigation, MapPin, Fuel, MessageSquare, Activity, Send, Car, AlertTriangle, ChevronRight } from 'lucide-react';

const apiKey = ""; // Wird vom System zur Laufzeit bereitgestellt

// Hilfsfunktion: Distanz zwischen zwei Koordinaten (Haversine)
const calcDistance = (lat1, lon1, lat2, lon2) => {
  const R = 6371;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
            Math.sin(dLon/2) * Math.sin(dLon/2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return R * c;
};

// Fetch mit Exponential Backoff für Gemini API
const fetchWithRetry = async (url, options, maxRetries = 5) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
      return await response.json();
    } catch (e) {
      if (i === maxRetries - 1) throw e;
      await new Promise(res => setTimeout(res, Math.pow(2, i) * 1000));
    }
  }
};

// Routenpunkte mit Koordinaten, Namen und Bildern
const ROUTE_WAYPOINTS = [
  {
    name: "Essen",
    country: "Deutschland",
    lat: 51.4556,
    lng: 7.0116,
    image: "https://images.unsplash.com/photo-1570168007204-dfb528c6958f?w=400&q=80",
    gradient: "from-blue-900 to-slate-800",
    emoji: "🏭",
    desc: "Ruhrgebiet"
  },
  {
    name: "Basel",
    country: "Schweiz",
    lat: 47.5596,
    lng: 7.5886,
    image: "https://images.unsplash.com/photo-1587974928442-77dc3e0dba72?w=400&q=80",
    gradient: "from-red-900 to-slate-800",
    emoji: "🇨🇭",
    desc: "Rhein & Kunst"
  },
  {
    name: "Mailand",
    country: "Italien",
    lat: 45.4654,
    lng: 9.1859,
    image: "https://images.unsplash.com/photo-1520175480921-4edfa2983e0f?w=400&q=80",
    gradient: "from-green-900 to-slate-800",
    emoji: "🇮🇹",
    desc: "La Metropolitana"
  },
  {
    name: "Nizza",
    country: "Frankreich",
    lat: 43.7102,
    lng: 7.2620,
    image: "https://images.unsplash.com/photo-1533929736458-ca588d08c8be?w=400&q=80",
    gradient: "from-cyan-900 to-slate-800",
    emoji: "🌊",
    desc: "Côte d'Azur"
  }
];

// Aktuellen Wegpunkt basierend auf Latitude bestimmen
const getCurrentWaypointIndex = (lat) => {
  if (lat > 49) return 0;
  if (lat > 46.5) return 1;
  if (lat > 44) return 2;
  return 3;
};

// Animated Speedometer SVG
const Speedometer = ({ speed }) => {
  const maxSpeed = 200;
  const radius = 88;
  const circumference = 2 * Math.PI * radius;
  const clampedSpeed = Math.min(speed, maxSpeed);
  const dashOffset = circumference - (circumference * clampedSpeed) / maxSpeed;

  const getSpeedColor = () => {
    if (speed < 60) return '#22d3ee';   // cyan
    if (speed < 100) return '#3b82f6';  // blue
    if (speed < 130) return '#f59e0b';  // amber
    return '#ef4444';                    // red
  };

  const needleAngle = -140 + (clampedSpeed / maxSpeed) * 280;

  return (
    <div className="relative flex items-center justify-center" style={{ width: 200, height: 200 }}>
      <svg width="200" height="200" viewBox="0 0 200 200">
        {/* Background ring */}
        <circle cx="100" cy="100" r={radius} fill="none" stroke="#1e293b" strokeWidth="12" />
        {/* Speed arc */}
        <circle
          cx="100" cy="100" r={radius}
          fill="none"
          stroke={getSpeedColor()}
          strokeWidth="12"
          strokeDasharray={circumference}
          strokeDashoffset={dashOffset}
          strokeLinecap="round"
          transform="rotate(-90 100 100)"
          style={{ transition: 'stroke-dashoffset 0.6s ease, stroke 0.4s ease', filter: `drop-shadow(0 0 8px ${getSpeedColor()})` }}
        />
        {/* Tick marks */}
        {[0, 50, 100, 150, 200].map((mark) => {
          const angle = -140 + (mark / maxSpeed) * 280;
          const rad = (angle * Math.PI) / 180;
          const x1 = 100 + 78 * Math.cos(rad);
          const y1 = 100 + 78 * Math.sin(rad);
          const x2 = 100 + 68 * Math.cos(rad);
          const y2 = 100 + 68 * Math.sin(rad);
          return <line key={mark} x1={x1} y1={y1} x2={x2} y2={y2} stroke="#475569" strokeWidth="2" />;
        })}
        {/* Needle */}
        <g transform={`rotate(${needleAngle} 100 100)`} style={{ transition: 'transform 0.6s ease' }}>
          <polygon points="100,24 97,100 103,100" fill={getSpeedColor()} opacity="0.9" />
        </g>
        {/* Center dot */}
        <circle cx="100" cy="100" r="8" fill="#0f172a" stroke={getSpeedColor()} strokeWidth="3" />
      </svg>
      {/* Speed text overlay */}
      <div className="absolute flex flex-col items-center" style={{ top: '55%' }}>
        <span className="text-4xl font-black text-white leading-none">{speed}</span>
        <span className="text-slate-400 text-xs font-semibold tracking-widest">KM/H</span>
      </div>
    </div>
  );
};

// City Hero Image Card
const CityHeroCard = ({ waypoint, isActive, isPassed }) => {
  return (
    <div className={`relative rounded-xl overflow-hidden transition-all duration-500 ${
      isActive ? 'ring-2 ring-blue-400 shadow-lg shadow-blue-500/20' : ''
    } ${isPassed ? 'opacity-60' : ''}`}>
      <div className={`absolute inset-0 bg-gradient-to-b ${waypoint.gradient} opacity-70`} />
      <img
        src={waypoint.image}
        alt={waypoint.name}
        className="w-full h-24 object-cover"
        onError={(e) => { e.target.style.display = 'none'; }}
      />
      <div className="absolute inset-0 bg-gradient-to-t from-black/80 to-transparent" />
      <div className="absolute bottom-0 left-0 right-0 p-2">
        <div className="flex items-center gap-1">
          <span className="text-sm">{waypoint.emoji}</span>
          <span className="text-white font-bold text-sm">{waypoint.name}</span>
          {isActive && <span className="ml-auto w-2 h-2 bg-blue-400 rounded-full animate-pulse" />}
          {isPassed && <span className="ml-auto text-green-400 text-xs">✓</span>}
        </div>
        <p className="text-slate-300 text-xs">{waypoint.desc}</p>
      </div>
    </div>
  );
};

// Route Timeline Component
const RouteTimeline = ({ currentWaypointIndex, location }) => {
  const totalDistance = 1100; // km Essen to Nizza approx
  const startLat = ROUTE_WAYPOINTS[0].lat;
  const endLat = ROUTE_WAYPOINTS[3].lat;
  const progress = Math.max(0, Math.min(100, ((startLat - location.lat) / (startLat - endLat)) * 100));

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-2xl p-5 shadow-xl">
      <h2 className="text-sm font-bold mb-4 flex items-center gap-2 text-slate-300">
        <Navigation className="w-4 h-4 text-purple-400" />
        Route · Essen → Nizza
        <span className="ml-auto text-xs text-slate-500">{Math.round(progress)}%</span>
      </h2>

      {/* Progress bar */}
      <div className="relative h-1.5 bg-slate-800 rounded-full mb-5 overflow-hidden">
        <div
          className="h-full bg-gradient-to-r from-blue-500 to-cyan-400 rounded-full transition-all duration-1000"
          style={{ width: `${progress}%` }}
        />
      </div>

      {/* Waypoint grid */}
      <div className="grid grid-cols-4 gap-2">
        {ROUTE_WAYPOINTS.map((wp, idx) => (
          <CityHeroCard
            key={wp.name}
            waypoint={wp}
            isActive={idx === currentWaypointIndex}
            isPassed={idx < currentWaypointIndex}
          />
        ))}
      </div>

      {/* Connector line */}
      <div className="flex items-center justify-between mt-3 px-2">
        {ROUTE_WAYPOINTS.map((wp, idx) => (
          <React.Fragment key={wp.name}>
            <div className={`w-3 h-3 rounded-full border-2 transition-colors duration-300 ${
              idx < currentWaypointIndex ? 'bg-green-500 border-green-400' :
              idx === currentWaypointIndex ? 'bg-blue-400 border-blue-300 shadow-[0_0_6px_rgba(96,165,250,0.8)]' :
              'bg-slate-700 border-slate-600'
            }`} />
            {idx < ROUTE_WAYPOINTS.length - 1 && (
              <div className={`flex-1 h-0.5 mx-1 transition-colors duration-300 ${
                idx < currentWaypointIndex ? 'bg-green-500' : 'bg-slate-700'
              }`} />
            )}
          </React.Fragment>
        ))}
      </div>
    </div>
  );
};

export default function App() {
  const [speed, setSpeed] = useState(0);
  const [location, setLocation] = useState({ lat: 51.4556, lng: 7.0116 });
  const [isSimulating, setIsSimulating] = useState(false);
  const [gpsActive, setGpsActive] = useState(false);

  const [aiInput, setAiInput] = useState("");
  const [chatHistory, setChatHistory] = useState([
    { role: 'system', content: 'Hallo! Ich bin dein Roadtrip-Copilot. Sag mir, worauf du Lust hast (z.B. "Ich habe richtig Bock auf etwas Traditionelles!") und ich suche etwas passend zu deinem aktuellen GPS-Standort.' }
  ]);
  const [isAiLoading, setIsAiLoading] = useState(false);

  const [fuelData, setFuelData] = useState({ country: "Deutschland", currency: "€", stations: [] });

  const chatEndRef = useRef(null);
  const currentWaypointIndex = getCurrentWaypointIndex(location.lat);

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [chatHistory]);

  // Echte GPS-Überwachung
  useEffect(() => {
    if (isSimulating) return;
    if (!("geolocation" in navigator)) return;

    let lastPos = null;
    let lastTime = null;

    const watchId = navigator.geolocation.watchPosition(
      (pos) => {
        setGpsActive(true);
        const { latitude, longitude, speed: gpsSpeed } = pos.coords;
        setLocation({ lat: latitude, lng: longitude });

        let currentSpeed = 0;
        if (gpsSpeed !== null) {
          currentSpeed = gpsSpeed * 3.6;
        } else if (lastPos && lastTime) {
          const dist = calcDistance(lastPos.lat, lastPos.lng, latitude, longitude);
          const timeDiff = (pos.timestamp - lastTime) / 3600000;
          if (timeDiff > 0) currentSpeed = dist / timeDiff;
        }
        setSpeed(Math.round(currentSpeed));
        lastPos = { lat: latitude, lng: longitude };
        lastTime = pos.timestamp;
      },
      (err) => { console.warn("GPS Fehler:", err); setGpsActive(false); },
      { enableHighAccuracy: true, maximumAge: 0 }
    );

    return () => navigator.geolocation.clearWatch(watchId);
  }, [isSimulating]);

  // Simulation: Essen → Schweiz → Italien → Nizza
  useEffect(() => {
    if (!isSimulating) return;

    let simLat = 51.4556;
    let simLng = 7.0116;
    let simSpeed = 0;

    const interval = setInterval(() => {
      simLat -= 0.05;
      simLng += (simLat > 47 ? 0.01 : -0.01);

      if (simSpeed < 120) simSpeed += Math.floor(Math.random() * 10) + 5;
      else simSpeed += (Math.random() > 0.5 ? 2 : -2);

      setLocation({ lat: simLat, lng: simLng });
      setSpeed(Math.round(simSpeed));
      setGpsActive(true);

      if (simLat <= 43.7) {
        setIsSimulating(false);
        setSpeed(0);
      }
    }, 2000);

    return () => clearInterval(interval);
  }, [isSimulating]);

  // Live Spritpreis Update basierend auf GPS
  useEffect(() => {
    const { lat } = location;
    let country = "Deutschland";
    let currency = "€";
    let basePrice = 1.95;

    if (lat < 47.5 && lat >= 45.8) {
      country = "Schweiz"; currency = "CHF"; basePrice = 1.85;
    } else if (lat < 45.8 && lat > 44) {
      country = "Italien"; currency = "€"; basePrice = 2.15;
    } else if (lat <= 44) {
      country = "Frankreich / Italien (Küste)"; currency = "€"; basePrice = 2.05;
    }

    setFuelData({
      country,
      currency,
      stations: [
        { name: "Autobahn Raststätte", price: (basePrice + 0.15).toFixed(2), distance: "2 km" },
        { name: "Supermarkt Tankstelle (Spartipp)", price: (basePrice - 0.08).toFixed(2), distance: "6 km" },
        { name: "Dorf Tankstelle", price: basePrice.toFixed(2), distance: "12 km" }
      ]
    });
  }, [location.lat]);

  // Gemini KI Anfrage
  const handleAskGemini = async () => {
    if (!aiInput.trim()) return;

    const userMsg = aiInput;
    setChatHistory(prev => [...prev, { role: 'user', content: userMsg }]);
    setAiInput("");
    setIsAiLoading(true);

    try {
      const currentCity = ROUTE_WAYPOINTS[currentWaypointIndex];
      const prompt = `Du bist ein ortskundiger Reiseassistent für einen Roadtrip von Essen nach Nizza.
      Die aktuellen GPS-Koordinaten des Nutzers sind: Latitude ${location.lat.toFixed(4)}, Longitude ${location.lng.toFixed(4)}.
      Nächste Stadt/Region: ${currentCity.name}, ${currentCity.country}.
      Der Nutzer sagt: "${userMsg}".
      Antworte extrem kurz (max. 3 Sätze), nenne konkrete real existierende Orte oder Sehenswürdigkeiten in dieser Region.
      Der Nutzer sitzt im Auto und braucht schnelle, praktische Tipps.`;

      const payload = {
        contents: [{ parts: [{ text: prompt }] }],
        systemInstruction: { parts: [{ text: "Du bist ein hilfreicher KI-Copilot im Auto für einen Roadtrip Essen → Nizza." }] }
      };

      const result = await fetchWithRetry(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) }
      );

      const aiText = result.candidates?.[0]?.content?.parts?.[0]?.text || "Ich konnte die Umgebung gerade nicht scannen. Wir fahren zu schnell!";
      setChatHistory(prev => [...prev, { role: 'ai', content: aiText }]);
    } catch (error) {
      console.error("Gemini API Error:", error);
      setChatHistory(prev => [...prev, { role: 'ai', content: "Verbindung zum KI-Server fehlgeschlagen. Sind wir gerade im Gotthardtunnel?" }]);
    } finally {
      setIsAiLoading(false);
    }
  };

  const activeWaypoint = ROUTE_WAYPOINTS[currentWaypointIndex];

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans p-4 md:p-6 flex flex-col items-center">

      {/* Header */}
      <div className="w-full max-w-6xl flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold flex items-center gap-2 text-blue-400">
          <Car className="w-8 h-8" />
          Roadtrip Copilot
          <span className="text-sm font-normal text-slate-400 ml-2 hidden sm:block">
            Essen <ChevronRight className="inline w-3 h-3" /> Nizza
          </span>
        </h1>
        <div className="flex gap-3 items-center">
          <div className={`flex items-center gap-2 text-xs px-3 py-1.5 rounded-full ${gpsActive ? 'bg-green-900/50 text-green-400' : 'bg-red-900/50 text-red-400'}`}>
            <MapPin className="w-3 h-3" />
            {gpsActive ? 'GPS Aktiv' : 'Suche Signal...'}
          </div>
          <button
            onClick={() => setIsSimulating(!isSimulating)}
            className={`text-xs px-3 py-2 rounded-lg border transition-colors ${
              isSimulating
                ? 'bg-red-900/40 border-red-700 text-red-300 hover:bg-red-900/60'
                : 'bg-slate-800 border-slate-600 hover:bg-slate-700'
            }`}
          >
            {isSimulating ? "⏹ Stop" : "▶ Simulieren"}
          </button>
        </div>
      </div>

      {/* Route Timeline (full width) */}
      <div className="w-full max-w-6xl mb-6">
        <RouteTimeline currentWaypointIndex={currentWaypointIndex} location={location} />
      </div>

      <div className="w-full max-w-6xl grid grid-cols-1 md:grid-cols-3 gap-6">

        {/* Linke Spalte: Tacho & Warnungen */}
        <div className="space-y-6 flex flex-col">
          {/* Tacho Card */}
          <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6 flex flex-col items-center justify-center relative overflow-hidden shadow-xl">
            {/* City background glow */}
            <div className={`absolute inset-0 bg-gradient-to-b ${activeWaypoint.gradient} opacity-10 pointer-events-none`} />

            <div className="flex items-center gap-2 mb-3 text-slate-400 text-sm self-start">
              <Activity className="w-4 h-4" />
              <span>Tacho</span>
              <span className="ml-auto text-xs">{activeWaypoint.emoji} {activeWaypoint.name}</span>
            </div>

            <Speedometer speed={speed} />

            <div className="mt-4 flex justify-between w-full text-xs text-slate-400 px-2">
              <div className="flex flex-col items-center bg-slate-800/50 rounded-lg px-3 py-2">
                <span className="text-slate-500">Lat</span>
                <span className="font-mono text-slate-200">{location.lat.toFixed(3)}°</span>
              </div>
              <div className="flex flex-col items-center bg-slate-800/50 rounded-lg px-3 py-2">
                <span className="text-slate-500">Lng</span>
                <span className="font-mono text-slate-200">{location.lng.toFixed(3)}°</span>
              </div>
            </div>
          </div>

          {/* Warnungen Card */}
          <div className="bg-slate-900 border border-slate-800 rounded-2xl p-5 shadow-xl flex-grow">
            <h2 className="text-sm font-bold mb-3 text-slate-300">Regionale Hinweise</h2>
            <div className="text-lg font-semibold text-white mb-1">{fuelData.country}</div>

            {fuelData.country === "Italien" && (
              <div className="flex items-start gap-3 bg-red-950/30 border border-red-900/50 p-3 rounded-lg text-sm text-red-200 mt-3">
                <AlertTriangle className="w-5 h-5 text-red-500 shrink-0 mt-0.5" />
                <p>Mailand Area C voraus! City-Maut Registrierung erforderlich.</p>
              </div>
            )}
            {fuelData.country === "Schweiz" && (
              <div className="flex items-start gap-3 bg-yellow-950/30 border border-yellow-900/50 p-3 rounded-lg text-sm text-yellow-200 mt-3">
                <AlertTriangle className="w-5 h-5 text-yellow-500 shrink-0 mt-0.5" />
                <p>E-Vignette erforderlich. Radarwarner streng verboten!</p>
              </div>
            )}
            {fuelData.country === "Deutschland" && (
              <div className="flex items-start gap-3 bg-blue-950/30 border border-blue-900/50 p-3 rounded-lg text-sm text-blue-200 mt-3">
                <Navigation className="w-5 h-5 text-blue-400 shrink-0 mt-0.5" />
                <p>A3 → A5 Richtung Basel. Tankstop vor der Schweizer Grenze empfohlen!</p>
              </div>
            )}
            {fuelData.country.includes("Frankreich") && (
              <div className="flex items-start gap-3 bg-cyan-950/30 border border-cyan-900/50 p-3 rounded-lg text-sm text-cyan-200 mt-3">
                <MapPin className="w-5 h-5 text-cyan-400 shrink-0 mt-0.5" />
                <p>Côte d'Azur! Riviera-Promenade in Nizza ist mautpflichtig.</p>
              </div>
            )}
          </div>
        </div>

        {/* Mittlere Spalte: Gemini Copilot */}
        <div className="bg-slate-900 border border-slate-800 rounded-2xl flex flex-col shadow-xl md:col-span-1 h-[500px] md:h-auto">
          <div className="p-4 border-b border-slate-800 flex items-center gap-3">
            <div className="w-8 h-8 rounded-full bg-indigo-600/30 border border-indigo-500/50 flex items-center justify-center">
              <MessageSquare className="w-4 h-4 text-indigo-400" />
            </div>
            <div>
              <h2 className="text-sm font-bold text-white">Gemini Copilot</h2>
              <p className="text-xs text-slate-500">KI-Reiseassistent · {activeWaypoint.name} Region</p>
            </div>
          </div>

          <div className="flex-1 overflow-y-auto p-4 space-y-3">
            {chatHistory.map((msg, idx) => (
              <div key={idx} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                <div className={`max-w-[85%] rounded-2xl p-3 text-sm leading-relaxed ${
                  msg.role === 'user'
                    ? 'bg-blue-600 text-white rounded-br-none'
                    : 'bg-slate-800 text-slate-200 rounded-bl-none border border-slate-700'
                }`}>
                  {msg.content}
                </div>
              </div>
            ))}
            {isAiLoading && (
              <div className="flex justify-start">
                <div className="bg-slate-800 text-slate-400 rounded-2xl rounded-bl-none p-3 text-sm flex items-center gap-2 border border-slate-700">
                  <div className="w-2 h-2 bg-indigo-400 rounded-full animate-bounce" />
                  <div className="w-2 h-2 bg-indigo-400 rounded-full animate-bounce" style={{ animationDelay: '0.2s' }} />
                  <div className="w-2 h-2 bg-indigo-400 rounded-full animate-bounce" style={{ animationDelay: '0.4s' }} />
                </div>
              </div>
            )}
            <div ref={chatEndRef} />
          </div>

          <div className="p-4 bg-slate-950/50 rounded-b-2xl border-t border-slate-800">
            <div className="flex gap-2">
              <input
                type="text"
                value={aiInput}
                onChange={(e) => setAiInput(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && handleAskGemini()}
                placeholder={`Tipps für ${activeWaypoint.name}?`}
                className="flex-1 bg-slate-900 border border-slate-700 text-white text-sm rounded-xl px-4 py-3 focus:outline-none focus:border-blue-500 focus:ring-1 focus:ring-blue-500/30 transition-colors"
              />
              <button
                onClick={handleAskGemini}
                disabled={isAiLoading || !aiInput.trim()}
                className="bg-blue-600 hover:bg-blue-500 disabled:bg-slate-800 disabled:text-slate-600 text-white rounded-xl px-4 py-2 flex items-center justify-center transition-colors"
              >
                <Send className="w-5 h-5" />
              </button>
            </div>
          </div>
        </div>

        {/* Rechte Spalte: Live Spritpreise */}
        <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6 shadow-xl flex flex-col">
          <h2 className="text-sm font-bold mb-5 flex items-center gap-2 text-slate-300">
            <Fuel className="w-4 h-4 text-green-400" />
            Live Sprit · {fuelData.country}
          </h2>

          <div className="space-y-3">
            {fuelData.stations.map((station, idx) => (
              <div
                key={idx}
                className={`border rounded-xl p-4 flex justify-between items-center transition-colors ${
                  idx === 1
                    ? 'bg-green-950/20 border-green-900/50 hover:border-green-700/50'
                    : 'bg-slate-950 border-slate-800 hover:border-slate-600'
                }`}
              >
                <div>
                  <div className="font-medium text-slate-200 text-sm">{station.name}</div>
                  <div className="text-xs text-slate-500 mt-1 flex items-center gap-1">
                    <Navigation className="w-3 h-3" /> In {station.distance}
                  </div>
                </div>
                <div className="text-right">
                  <div className={`text-xl font-bold ${idx === 1 ? 'text-green-400' : 'text-white'}`}>
                    {station.price}
                    <span className="text-xs text-slate-500 ml-1">{fuelData.currency}</span>
                  </div>
                  {idx === 1 && <span className="text-xs text-green-500">Bester Preis</span>}
                </div>
              </div>
            ))}
          </div>

          <div className="mt-auto pt-5">
            <div className="bg-blue-950/30 border border-blue-900/50 rounded-xl p-4">
              <p className="text-xs text-blue-200 leading-relaxed">
                <strong className="block text-blue-400 mb-1">KI-Spartipp:</strong>
                Vor der Schweizer Grenze volltanken! In der Schweiz sind Preise in CHF – often günstiger als Deutschland, aber Wechselkurs beachten.
              </p>
            </div>
          </div>
        </div>

      </div>
    </div>
  );
}
