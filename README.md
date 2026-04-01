---
render_with_liquid: false
---
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mom's Calorie Cam</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React and ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap');
        body { font-family: 'Plus Jakarta Sans', sans-serif; }
        .animate-spin-slow { animation: spin 3s linear infinite; }
        @keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
    </style>
</head>
<body class="bg-slate-50 text-slate-900">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef, useMemo } = React;

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

            // Initialize Lucide icons whenever the component renders or state changes
            useEffect(() => {
                if (window.lucide) {
                    window.lucide.createIcons();
                }
            });

            // Initialize App: Load History, API Key, and Goal from local storage
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

            // Sync history to local storage whenever it changes
            useEffect(() => {
                localStorage.setItem('calorie_history', JSON.stringify(history));
            }, [history]);

            // Calculate daily totals using useMemo for performance
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
                }, 1000);
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
                <div className="min-h-screen pb-24">
                    {/* Header */}
                    <header className="bg-white border-b sticky top-0 z-20 p-4 shadow-sm">
                        <div className="max-w-md mx-auto flex items-center justify-between">
                            <div className="flex items-center gap-2">
                                <div className="bg-orange-500 p-2 rounded-lg text-white">
                                    <i data-lucide="utensils" className="w-5 h-5"></i>
                                </div>
                                <h1 className="font-bold text-xl tracking-tight">Mom's Calorie Cam</h1>
                            </div>
                            <button onClick={() => setShowSettings(true)} className="p-2 text-slate-400 hover:text-slate-600 transition-colors">
                                <i data-lucide="settings" className="w-5 h-5"></i>
                            </button>
                        </div>
                    </header>

                    {/* Settings Modal */}
                    {showSettings && (
                        <div className="fixed inset-0 bg-black/60 backdrop-blur-sm z-50 flex items-center justify-center p-4">
                            <div className="bg-white w-full max-w-sm rounded-3xl p-6 shadow-2xl">
                                <div className="flex justify-between items-center mb-6">
                                    <h2 className="text-xl font-bold text-slate-800">Settings</h2>
                                    <button onClick={() => setShowSettings(false)} className="p-1 hover:bg-slate-100 rounded-full transition-colors">
                                        <i data-lucide="x" className="text-slate-400 w-6 h-6"></i>
                                    </button>
                                </div>
                                
                                <div className="space-y-6">
                                    <div>
                                        <label className="block text-xs font-bold text-slate-400 uppercase mb-2 tracking-wider">Gemini API Key</label>
                                        <input 
                                            type="password"
                                            value={apiKey}
                                            onChange={(e) => setApiKey(e.target.value)}
                                            placeholder="Paste key here..."
                                            className="w-full bg-slate-100 border-none rounded-xl p-3 text-sm focus:ring-2 focus:ring-orange-500 outline-none"
                                        />
                                        <p className="text-[10px] text-slate-400 mt-2 italic">Stored locally in your browser</p>
                                    </div>

                                    <div>
                                        <label className="block text-xs font-bold text-slate-400 uppercase mb-2 tracking-wider">Daily Calorie Goal</label>
                                        <input 
                                            type="number"
                                            value={dailyGoal}
                                            onChange={(e) => setDailyGoal(parseInt(e.target.value) || 0)}
                                            className="w-full bg-slate-100 border-none rounded-xl p-3 text-sm focus:ring-2 focus:ring-orange-500 outline-none font-bold text-slate-700"
                                        />
                                    </div>

                                    <button 
                                        onClick={saveSettings}
                                        disabled={saveStatus}
                                        className={`w-full flex items-center justify-center gap-2 font-bold py-3 rounded-xl transition-all shadow-md ${
                                            saveStatus ? 'bg-green-500 text-white' : 'bg-slate-900 text-white hover:bg-slate-800'
                                        }`}
                                    >
                                        {saveStatus ? <i data-lucide="check-circle-2" className="w-5 h-5"></i> : <i data-lucide="save" className="w-5 h-5"></i>}
                                        {saveStatus ? 'Saved!' : 'Save Settings'}
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
                                        <p className="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-1">Today's Progress</p>
                                        <h2 className="text-3xl font-black text-slate-800 leading-none">
                                            {dailyStats.totalCals} <span className="text-sm font-medium text-slate-400 tracking-tight">/ {dailyGoal}</span>
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
                                    <div className="flex items-center gap-2 text-red-500 text-[10px] font-bold uppercase tracking-wide">
                                        <i data-lucide="alert-circle" className="w-3 h-3"></i>
                                        Daily limit reached
                                    </div>
                                )}
                            </div>
                        )}

                        {/* Upload/Scanning State */}
                        {!image ? (
                            <div className="space-y-4">
                                <div 
                                    className="bg-white border-2 border-dashed border-slate-200 rounded-3xl p-10 flex flex-col items-center justify-center text-center gap-4 hover:border-orange-300 transition-colors cursor-pointer group"
                                    onClick={() => fileInputRef.current.click()}
                                >
                                    <div className="bg-orange-100 p-4 rounded-full text-orange-600 group-hover:scale-110 transition-transform">
                                        <i data-lucide="camera" className="w-8 h-8"></i>
                                    </div>
                                    <div>
                                        <p className="font-bold text-lg leading-tight text-slate-700">Add a Meal</p>
                                        <p className="text-slate-400 text-sm mt-1">Snap a photo to count calories</p>
                                    </div>
                                    <input type="file" accept="image/*" className="hidden" ref={fileInputRef} onChange={handleImageUpload} />
                                </div>
                                
                                {history.length > 0 && (
                                    <div className="pt-2 animate-in fade-in slide-in-from-top-2 duration-700">
                                        <h2 className="font-bold text-slate-400 uppercase text-[10px] tracking-widest mb-3 flex items-center gap-2">
                                            <i data-lucide="calendar" className="w-3 h-3"></i> Recent Meals
                                        </h2>
                                        <div className="space-y-3">
                                            {history.slice(0, 10).map(item => (
                                                <div key={item.id} className="bg-white p-4 rounded-2xl flex items-center justify-between shadow-sm border border-slate-100 hover:border-slate-200 transition-colors">
                                                    <div>
                                                        <p className="font-bold text-slate-800 text-sm leading-tight">{item.mealName}</p>
                                                        <p className="text-[10px] text-slate-400 font-bold uppercase mt-1 tracking-tighter">{item.date}</p>
                                                    </div>
                                                    <div className="flex items-center gap-3">
                                                        <span className="bg-orange-50 text-orange-600 px-3 py-1 rounded-full text-[10px] font-black">
                                                            {item.totalCalories} kcal
                                                        </span>
                                                        <button onClick={() => deleteHistoryItem(item.id)} className="text-slate-200 hover:text-red-400 transition-colors p-1">
                                                            <i data-lucide="trash-2" className="w-4 h-4"></i>
                                                        </button>
                                                    </div>
                                                </div>
                                            ))}
                                        </div>
                                    </div>
                                )}
                            </div>
                        ) : (
                            /* Analysis Result State */
                            <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
                                <div className="relative rounded-3xl overflow-hidden shadow-2xl aspect-square bg-slate-200 border-4 border-white">
                                    <img src={image} alt="Meal preview" className="w-full h-full object-cover" />
                                    {loading && (
                                        <div className="absolute inset-0 bg-black/50 backdrop-blur-sm flex flex-col items-center justify-center text-white p-6 text-center">
                                            <div className="animate-spin-slow mb-4">
                                                <i data-lucide="loader-2" className="w-10 h-10"></i>
                                            </div>
                                            <p className="font-black tracking-widest uppercase text-xs">AI Scanning Ingredients...</p>
                                            <p className="text-[10px] opacity-60 mt-2">Checking portion sizes and nutrition</p>
                                        </div>
                                    )}
                                </div>

                                {!analysis && !loading && (
                                    <button 
                                        onClick={analyzeImage}
                                        className="w-full bg-orange-500 text-white font-black py-5 rounded-2xl shadow-xl shadow-orange-200 flex items-center justify-center gap-3 transform active:scale-95 transition-all uppercase tracking-widest"
                                    >
                                        <i data-lucide="flame" className="w-5 h-5"></i> Calculate with AI
                                    </button>
                                )}

                                {error && (
                                    <div className="bg-red-50 border border-red-100 p-4 rounded-2xl flex gap-3 text-red-600 text-xs font-bold animate-shake">
                                        <i data-lucide="alert-circle" className="shrink-0 w-4 h-4"></i>
                                        <p>{error}</p>
                                    </div>
                                )}

                                {analysis && (
                                    <div className="space-y-4 animate-in zoom-in-95 duration-300">
                                        <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
                                            <div className="flex justify-between items-start mb-4">
                                                <div>
                                                    <h2 className="text-2xl font-black text-slate-800 leading-tight">{analysis.mealName}</h2>
                                                    <p className="text-[10px] text-green-600 font-bold uppercase mt-1 flex items-center gap-1">
                                                        <i data-lucide="check" className="w-3 h-3"></i> Logged to Daily Total
                                                    </p>
                                                </div>
                                            </div>
                                            
                                            <div className="grid grid-cols-2 gap-3 mt-4">
                                                <div className="bg-orange-50 p-4 rounded-2xl">
                                                    <p className="text-[10px] text-orange-600 font-bold uppercase mb-1">Calories</p>
                                                    <p className="text-2xl font-black text-orange-700 leading-none">{analysis.totalCalories}</p>
                                                </div>
                                                <div className="bg-blue-50 p-4 rounded-2xl">
                                                    <p className="text-[10px] text-blue-600 font-bold uppercase mb-1">Protein</p>
                                                    <p className="text-2xl font-black text-blue-700 leading-none">{analysis.totalMacros.protein}g</p>
                                                </div>
                                            </div>
                                        </div>

                                        <div className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
                                            <h3 className="font-bold text-slate-400 uppercase text-[10px] tracking-widest mb-4">Nutritional Breakdown</h3>
                                            <div className="space-y-4">
                                                {analysis.items.map((item, i) => (
                                                    <div key={i} className="flex justify-between items-center border-b border-slate-50 pb-2 last:border-0 last:pb-0">
                                                        <div>
                                                            <p className="font-bold text-slate-800 text-sm leading-tight">{item.name}</p>
                                                            <p className="text-[10px] text-slate-400 font-medium tracking-tight mt-0.5">{item.portion}</p>
                                                        </div>
                                                        <p className="font-black text-slate-700 text-sm">{item.calories} <span className="text-[9px] text-slate-400 uppercase ml-0.5">kcal</span></p>
                                                    </div>
                                                ))}
                                            </div>
                                        </div>

                                        <div className="bg-indigo-50 p-5 rounded-3xl flex gap-4 border border-indigo-100">
                                            <div className="bg-white p-2 rounded-xl shadow-sm shrink-0 h-fit">
                                                <i data-lucide="info" className="text-indigo-500 w-5 h-5"></i>
                                            </div>
                                            <div>
                                                <p className="font-black text-indigo-900 text-[10px] uppercase tracking-wider mb-1">Mom's Health Tip</p>
                                                <p className="text-indigo-700 text-sm leading-relaxed italic">"{analysis.healthTip}"</p>
                                            </div>
                                        </div>

                                        <button 
                                            onClick={() => { setImage(null); setAnalysis(null); }}
                                            className="w-full py-5 bg-slate-900 text-white rounded-2xl font-black shadow-xl uppercase tracking-widest text-sm hover:bg-slate-800 active:scale-95 transition-all"
                                        >
                                            Back to Dashboard
                                        </button>
                                    </div>
                                )}
                            </div>
                        )}
                    </main>

                    {/* Fixed Scan Button */}
                    {!image && (
                        <div className="fixed bottom-8 left-0 right-0 px-6 flex justify-center z-10">
                            <button 
                                onClick={() => fileInputRef.current.click()}
                                className="bg-orange-500 text-white flex items-center gap-3 px-12 py-4 rounded-full shadow-2xl shadow-orange-300 font-black tracking-widest transform active:scale-95 transition-all border-4 border-white uppercase"
                            >
                                <i data-lucide="camera" className="w-6 h-6"></i>
                                SCAN MEAL
                            </button>
                        </div>
                    )}
                </div>
            );
        };

        // Render the application
        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
