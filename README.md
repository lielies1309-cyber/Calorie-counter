# Calorie-counimport React, { useState, useEffect, useRef, useMemo } from 'react';
import { Camera, Upload, Utensils, Flame, Info, AlertCircle, Loader2, Trash2, Plus, Settings, X, Save, CheckCircle2, Calendar } from 'lucide-react';

const App = () => {
  const [image, setImage] = useState(null);
  const [base64Image, setBase64Image] = useState(null);
  const [loading, setLoading] = useState(false);
  const [analysis, setAnalysis] = useState(null);
  const [history, setHistory] = useState([]);
  const [error, setError] = useState(null);
  const [showSettings, setShowSettings] = useState(false);
  const [apiKey, setApiKey] = useState("");
  const [dailyGoal, setDailyGoal] = useState(2000);
  const [saveStatus, setSaveStatus] = useState(false);
  
  const fileInputRef = useRef(null);

  // Initialize App: Load History, API Key, and Goal
  useEffect(() => {
    const savedHistory = localStorage.getItem('calorie_history');
    const savedKey = localStorage.getItem('gemini_api_key');
    const savedGoal = localStorage.getItem('daily_calorie_goal');
    
    if (savedHistory) {
      try {
        setHistory(JSON.parse(savedHistory));
      } catch (e) {
        console.error("Failed to load history", e);
      }
    }
    
    if (savedKey) setApiKey(savedKey);
    else setShowSettings(true);

    if (savedGoal) setDailyGoal(parseInt(savedGoal));
  }, []);

  // Sync history to local storage
  useEffect(() => {
    localStorage.setItem('calorie_history', JSON.stringify(history));
  }, [history]);

  // Calculate daily totals
  const dailyStats = useMemo(() => {
    const today = new Date().toLocaleDateString();
    const todayMeals = history.filter(item => {
      const mealDate = new Date(item.timestamp || item.id).toLocaleDateString();
      return mealDate === today;
    });

    const totalCals = todayMeals.reduce((sum, meal) => sum + meal.totalCalories, 0);
    const totalProtein = todayMeals.reduce((sum, meal) => sum + (meal.totalMacros?.protein || 0), 0);
    
    return {
      totalCals,
      totalProtein,
      percent: Math.min(Math.round((totalCals / dailyGoal) * 100), 100),
      isOver: totalCals > dailyGoal
    };
  }, [history, dailyGoal]);

  const saveSettings = () => {
    localStorage.setItem('gemini_api_key', apiKey);
    localStorage.setItem('daily_calorie_goal', dailyGoal.toString());
    setSaveStatus(true);
    setTimeout(() => {
      setSaveStatus(false);
      setShowSettings(false);
    }, 1500);
  };

  const handleImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setImage(reader.result);
        setBase64Image(reader.result.split(',')[1]);
        setAnalysis(null);
        setError(null);
      };
      reader.readAsDataURL(file);
    }
  };

  const analyzeImage = async () => {
    if (!base64Image) return;
    if (!apiKey) {
      setError("Please add your Gemini API Key in Settings first.");
      setShowSettings(true);
      return;
    }

    setLoading(true);
    setError(null);

    const systemPrompt = `You are an expert nutritionist. Analyze the image of the meal.
    1. Identify all food items.
    2. Estimate portion sizes (e.g., 150g, 1 cup).
    3. Calculate calories, protein, carbs, and fats for each.
    4. Provide a total calorie count.
    5. Give a short health tip.
    Return ONLY a JSON object.
    
    Format:
    {
      "mealName": "Name",
      "items": [{ "name": "item", "calories": 100, "protein": 5, "carbs": 10, "fats": 2, "portion": "100g" }],
      "totalCalories": 500,
      "totalMacros": { "protein": 25, "carbs": 50, "fats": 20 },
      "healthTip": "Tip"
    }`;

    const payload = {
      contents: [{
        parts: [
          { text: "Analyze this meal and provide nutritional data in the requested JSON format." },
          { inlineData: { mimeType: "image/png", data: base64Image } }
        ]
      }],
      systemInstruction: { parts: [{ text: systemPrompt }] },
      generationConfig: { responseMimeType: "application/json" }
    };

    const makeRequest = async (attempt = 0) => {
      try {
        const response = await fetch(
          `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
          }
        );

        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error?.message || "API Error");
        }
        
        const data = await response.json();
        const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text;
        const parsed = JSON.parse(resultText);
        
        setAnalysis(parsed);
        const now = new Date();
        setHistory(prev => [{ 
          ...parsed, 
          id: Date.now(), 
          timestamp: now.getTime(),
          date: now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }) 
        }, ...prev]);
        setLoading(false);
      } catch (err) {
        if (attempt < 3) {
          const delay = Math.pow(2, attempt) * 1000;
          setTimeout(() => makeRequest(attempt + 1), delay);
        } else {
          setError(err.message || "Could not analyze image. Check your key.");
          setLoading(false);
        }
      }
    };

    makeRequest();
  };

  const deleteHistoryItem = (id) => {
    setHistory(prev => prev.filter(item => item.id !== id));
  };

  return (
    <div className="min-h-screen bg-slate-50 text-slate-900 font-sans pb-24">
      {/* Header */}
      <header className="bg-white border-b sticky top-0 z-20 p-4 shadow-sm">
        <div className="max-w-md mx-auto flex items-center justify-between">
          <div className="flex items-center gap-2">
            <div className="bg-orange-500 p-2 rounded-lg text-white">
              <Utensils size={20} />
            </div>
            <h1 className="font-bold text-xl tracking-tight">Mom's Calorie Cam</h1>
          </div>
          <button onClick={() => setShowSettings(true)} className="p-2 text-slate-400 hover:text-slate-600 transition-colors">
            <Settings size={22} />
          </button>
        </div>
      </header>

      {/* Settings Modal */}
      {showSettings && (
        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-50 flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-sm rounded-3xl p-6 shadow-2xl">
            <div className="flex justify-between items-center mb-6">
              <h2 className="text-xl font-bold">Settings</h2>
              <button onClick={() => setShowSettings(false)}><X size={24} className="text-slate-400" /></button>
            </div>
            
            <div className="space-y-6">
              <div>
                <label className="block text-xs font-bold text-slate-400 uppercase mb-2">Gemini API Key</label>
                <input 
                  type="password"
                  value={apiKey}
                  onChange={(e) => setApiKey(e.target.value)}
                  placeholder="Paste key here..."
                  className="w-full bg-slate-100 border-none rounded-xl p-3 text-sm focus:ring-2 focus:ring-orange-500 outline-none"
                />
              </div>

              <div>
                <label className="block text-xs font-bold text-slate-400 uppercase mb-2">Daily Calorie Goal</label>
                <input 
                  type="number"
                  value={dailyGoal}
                  onChange={(e) => setDailyGoal(parseInt(e.target.value) || 0)}
                  className="w-full bg-slate-100 border-none rounded-xl p-3 text-sm focus:ring-2 focus:ring-orange-500 outline-none font-bold"
                />
              </div>

              <button 
                onClick={saveSettings}
                disabled={saveStatus}
                className={`w-full flex items-center justify-center gap-2 font-bold py-3 rounded-xl transition-all ${
                  saveStatus ? 'bg-green-500 text-white' : 'bg-slate-900 text-white hover:bg-slate-800'
                }`}
              >
                {saveStatus ? <><CheckCircle2 size={18}/> Saved!</> : <><Save size={18}/> Save Settings</>}
              </button>
            </div>
          </div>
        </div>
      )}

      <main className="max-w-md mx-auto p-4 space-y-6">
        
        {/* Daily Progress Dashboard */}
        {!image && (
          <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100 space-y-4">
            <div className="flex justify-between items-end">
              <div>
                <p className="text-xs font-bold text-slate-400 uppercase tracking-widest">Daily Progress</p>
                <h2 className="text-3xl font-black text-slate-800">
                  {dailyStats.totalCals} <span className="text-sm font-medium text-slate-400">/ {dailyGoal} kcal</span>
                </h2>
              </div>
              <div className="text-right">
                <p className="text-[10px] font-bold text-blue-500 uppercase">Protein</p>
                <p className="text-lg font-bold text-blue-700">{dailyStats.totalProtein}g</p>
              </div>
            </div>

            <div className="h-3 w-full bg-slate-100 rounded-full overflow-hidden">
              <div 
                className={`h-full transition-all duration-1000 ease-out ${dailyStats.isOver ? 'bg-red-500' : 'bg-orange-500'}`}
                style={{ width: `${dailyStats.percent}%` }}
              />
            </div>
            
            {dailyStats.isOver && (
              <div className="flex items-center gap-2 text-red-500 text-[10px] font-bold uppercase">
                <AlertCircle size={14} />
                Daily limit reached
              </div>
            )}
          </div>
        )}

        {/* Upload State */}
        {!image ? (
          <div className="space-y-4">
            <div 
              className="bg-white border-2 border-dashed border-slate-200 rounded-3xl p-10 flex flex-col items-center justify-center text-center gap-4 hover:border-orange-300 transition-colors cursor-pointer"
              onClick={() => fileInputRef.current.click()}
            >
              <div className="bg-orange-100 p-4 rounded-full text-orange-600">
                <Camera size={32} />
              </div>
              <div>
                <p className="font-semibold text-lg">Scan your meal</p>
                <p className="text-slate-500 text-sm">Snap a photo to add to today's total</p>
              </div>
              <input type="file" accept="image/*" className="hidden" ref={fileInputRef} onChange={handleImageUpload} />
            </div>
            
            {history.length > 0 && (
              <div className="pt-2">
                <h2 className="font-bold text-slate-400 uppercase text-[10px] tracking-widest mb-3 flex items-center gap-2">
                  <Calendar size={12}/> Recent Meals
                </h2>
                <div className="space-y-3">
                  {history.map(item => (
                    <div key={item.id} className="bg-white p-3 rounded-2xl flex items-center justify-between shadow-sm border border-slate-100">
                      <div>
                        <p className="font-bold text-slate-800 text-sm">{item.mealName}</p>
                        <p className="text-[10px] text-slate-400 font-bold uppercase">{item.date}</p>
                      </div>
                      <div className="flex items-center gap-3">
                        <span className="bg-orange-50 text-orange-600 px-2 py-1 rounded-lg text-xs font-bold">
                          +{item.totalCalories}
                        </span>
                        <button onClick={() => deleteHistoryItem(item.id)} className="text-slate-200 hover:text-red-400 transition-colors">
                          <Trash2 size={16} />
                        </button>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            )}
          </div>
        ) : (
          /* Analysis / Preview State */
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
            <div className="relative rounded-3xl overflow-hidden shadow-xl aspect-square bg-slate-200">
              <img src={image} alt="Meal preview" className="w-full h-full object-cover" />
              {loading && (
                <div className="absolute inset-0 bg-black/40 backdrop-blur-sm flex flex-col items-center justify-center text-white p-6 text-center">
                  <Loader2 className="animate-spin mb-4" size={40} />
                  <p className="font-medium">Analyzing Plate...</p>
                </div>
              )}
            </div>

            {!analysis && !loading && (
              <button 
                onClick={analyzeImage}
                className="w-full bg-orange-500 text-white font-bold py-4 rounded-2xl shadow-lg flex items-center justify-center gap-2"
              >
                <Flame size={20} /> Calculate & Add to Day
              </button>
            )}

            {error && (
              <div className="bg-red-50 border border-red-100 p-4 rounded-2xl flex gap-3 text-red-600 text-sm">
                <AlertCircle className="shrink-0" />
                <p>{error}</p>
              </div>
            )}

            {analysis && (
              <div className="space-y-4 animate-in zoom-in-95 duration-300">
                <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
                  <h2 className="text-2xl font-bold mb-1">{analysis.mealName}</h2>
                  <p className="text-xs text-green-600 font-bold uppercase">Added to today's total</p>
                  
                  <div className="flex gap-4 mt-4">
                    <div className="flex-1 bg-orange-50 p-3 rounded-2xl text-center">
                      <p className="text-[10px] text-orange-600 font-bold uppercase mb-1">Calories</p>
                      <p className="text-xl font-black text-orange-700">{analysis.totalCalories}</p>
                    </div>
                    <div className="flex-1 bg-blue-50 p-3 rounded-2xl text-center">
                      <p className="text-[10px] text-blue-600 font-bold uppercase mb-1">Protein</p>
                      <p className="text-xl font-black text-blue-700">{analysis.totalMacros.protein}g</p>
                    </div>
                  </div>
                </div>

                <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
                  <h3 className="font-bold text-slate-400 uppercase text-[10px] tracking-widest mb-4">Breakdown</h3>
                  <div className="space-y-4">
                    {analysis.items.map((item, i) => (
                      <div key={i} className="flex justify-between items-center border-b border-slate-50 pb-2 last:border-0 last:pb-0">
                        <div>
                          <p className="font-semibold text-slate-800 text-sm">{item.name}</p>
                          <p className="text-[10px] text-slate-400">{item.portion}</p>
                        </div>
                        <p className="font-bold text-slate-700 text-sm">{item.calories} kcal</p>
                      </div>
                    ))}
                  </div>
                </div>

                <button 
                  onClick={() => { setImage(null); setAnalysis(null); }}
                  className="w-full py-4 bg-slate-900 text-white rounded-2xl font-bold shadow-lg"
                >
                  Done
                </button>
              </div>
            )}
          </div>
        )}
      </main>

      {/* Global Action Bar */}
      {!image && (
        <div className="fixed bottom-6 left-0 right-0 px-4 flex justify-center z-10">
          <button 
            onClick={() => fileInputRef.current.click()}
            className="bg-orange-500 text-white flex items-center gap-3 px-10 py-4 rounded-full shadow-2xl font-black tracking-wide transform active:scale-95 transition-transform"
          >
            <Camera size={24} />
            SCAN MEAL
          </button>
        </div>
      )}
    </div>
  );
};

export default App;ter
