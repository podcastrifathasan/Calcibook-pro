import React, { useState, useMemo, useCallback } from 'react';
import { ViewType, Event, Language, ChartDataPoint, EntryType } from './types';
import { CATEGORIES, Icons, INITIAL_LOGO, CATEGORY_ICONS } from './constants';
import { translations } from './translations';
import DashboardCard from './components/DashboardCard';
import { SimpleBarChart } from './components/Charts';
import EventItem from './components/EventItem';
import AIAssistant from './components/AIAssistant';

const App: React.FC = () => {
  const [lang, setLang] = useState<Language>('bn');
  const [view, setView] = useState<ViewType>('dashboard');
  const [activeTab, setActiveTab] = useState<'finance' | 'expense'>('finance');
  const [timeFilter, setTimeFilter] = useState<'hour'|'day'|'week'|'month'>('week');
  const [showModal, setShowModal] = useState(false);
  const [showAI, setShowAI] = useState(false);
  const [editingId, setEditingId] = useState<string | null>(null);
  const [appLogo, setAppLogo] = useState(INITIAL_LOGO);
  
  // Persisting events would typically use useEffect/localStorage, simplified here
  const [events, setEvents] = useState<Event[]>([]);
  
  const t = translations[lang];

  // Derived Values
  const totalIncome = useMemo(() => 
    events.filter(e => e.note === 'Income').reduce((acc, curr) => acc + Math.abs(curr.cost), 0)
  , [events]);

  const totalExpense = useMemo(() => 
    events.filter(e => e.note === 'Expense').reduce((acc, curr) => acc + Math.abs(curr.cost), 0)
  , [events]);

  // Chart Data Logic
  const chartData = useMemo<ChartDataPoint[]>(() => {
    let labels: string[] = [];
    
    // Adjust labels based on filter
    if (timeFilter === 'hour') labels = ["10am", "12pm", "2pm", "4pm", "6pm", "8pm", "10pm"];
    else if (timeFilter === 'day') labels = ["Morning", "Noon", "Afternoon", "Evening", "Night"];
    else if (timeFilter === 'week') labels = t.days;
    else labels = ["W1", "W2", "W3", "W4"];

    const baseData = labels.map(label => ({ label, value: 0 }));
    
    const filteredEvents = events.filter(e => 
      activeTab === 'finance' ? e.note === 'Income' : e.note === 'Expense'
    );

    if (filteredEvents.length === 0) {
      // Elegant placeholder curve
      return labels.map((label, i) => ({ 
        label, 
        value: [100, 180, 150, 240, 190, 310, 250][i % 7] 
      }));
    }

    filteredEvents.forEach(e => {
      // Simplified mapping for demo
      const idx = parseInt(e.id.slice(-1)) % labels.length;
      baseData[idx].value += Math.abs(e.cost);
    });

    return baseData;
  }, [events, activeTab, t.days, timeFilter]);

  // Form State
  const [eventName, setEventName] = useState("");
  const [cost, setCost] = useState("");
  const [category, setCategory] = useState(CATEGORIES[0]);
  const [entryType, setEntryType] = useState<EntryType>('expense');

  const openEditModal = (evt: Event) => {
    setEditingId(evt.id);
    setEventName(evt.name);
    setCost(Math.abs(evt.cost).toString());
    setCategory(evt.category);
    setEntryType(evt.note === 'Income' ? 'income' : 'expense');
    setShowModal(true);
  };

  const handleSave = useCallback(() => {
    if (!eventName.trim()) return;
    const costNum = parseFloat(cost) || 0;

    const newEventData = {
      name: eventName,
      cost: entryType === 'income' ? costNum : -costNum,
      category,
      note: entryType === 'income' ? 'Income' as const : 'Expense' as const
    };

    if (editingId) {
      setEvents(prev => prev.map(e => e.id === editingId ? { ...e, ...newEventData } : e));
    } else {
      const newEvent: Event = {
        id: Date.now().toString(),
        date: new Date().toLocaleDateString(lang === 'bn' ? 'bn-BD' : 'en-US', { 
          year: 'numeric', month: 'short', day: 'numeric' 
        }),
        timestamp: Date.now(),
        type: "transaction",
        ...newEventData
      };
      setEvents(prev => [newEvent, ...prev]);
    }

    setEventName(""); setCost(""); setEntryType('expense'); setEditingId(null);
    setShowModal(false);
  }, [eventName, cost, category, entryType, lang, editingId]);

  const toggleLang = () => setLang(prev => prev === 'bn' ? 'en' : 'bn');
  const customizeLogo = () => {
    const newEmoji = prompt(lang === 'bn' ? "লোগো ইমোজি পরিবর্তন করুন:" : "Change Logo Emoji:", appLogo);
    if (newEmoji) setAppLogo(newEmoji.substring(0, 2));
  };

  return (
    <div className="min-h-screen pb-28 max-w-lg mx-auto relative flex flex-col font-['Hind_Siliguri'] overflow-hidden">
      
      {/* Dynamic Background Blurs */}
      <div className="fixed top-0 left-0 w-full h-96 bg-indigo-500/20 rounded-full blur-[100px] -translate-y-1/2 pointer-events-none" />
      <div className="fixed bottom-0 right-0 w-full h-96 bg-purple-500/20 rounded-full blur-[100px] translate-y-1/2 pointer-events-none" />

      {/* Header - Liquid Glass */}
      <header className="glass-panel px-6 pt-12 pb-8 rounded-b-[48px] shadow-2xl shadow-indigo-500/10 z-10 relative mb-4">
        <div className="flex items-center justify-between mb-8">
          <div>
            <h1 className="text-2xl font-black text-slate-800 tracking-tighter uppercase flex items-center gap-2">
              {appLogo} {t.appTitle}
            </h1>
            <p className="text-[10px] text-slate-500 font-bold uppercase tracking-widest mt-1 ml-1 opacity-70">{t.resourceManager}</p>
          </div>
          <div className="flex items-center gap-3">
             <button onClick={() => setShowAI(true)} className="h-11 w-11 bg-white/40 backdrop-blur-md border border-white/50 rounded-2xl flex items-center justify-center text-indigo-600 shadow-sm hover:scale-105 transition-all">
              <Icons.Bot size={22} />
            </button>
            <button onClick={toggleLang} className="px-4 py-2.5 bg-white/40 backdrop-blur-md border border-white/50 rounded-2xl text-[10px] font-black text-slate-700 hover:bg-white/60 uppercase tracking-wider transition-all">{lang === 'bn' ? 'ENG' : 'বাংলা'}</button>
          </div>
        </div>

        <div className="grid grid-cols-2 gap-4">
          <DashboardCard 
            title={t.finance} 
            value={`${totalIncome.toLocaleString()} ${lang === 'bn' ? '৳' : '$'}`} 
            subtext={t.savings}
            icon={Icons.Wallet}
            theme="emerald"
            active={activeTab === 'finance'}
            onClick={() => { setActiveTab('finance'); setView('dashboard'); }}
          />
          <DashboardCard 
            title={t.expenseTitle} 
            value={`${totalExpense.toLocaleString()} ${lang === 'bn' ? '৳' : '$'}`} 
            subtext={t.utilization}
            icon={Icons.Zap}
            theme="rose"
            active={activeTab === 'expense'}
            onClick={() => { setActiveTab('expense'); setView('dashboard'); }}
          />
        </div>
      </header>

      {/* Main Content */}
      <main className="flex-1 overflow-y-auto no-scrollbar p-6 space-y-8 animate-slide-up z-0 relative">
        {view === 'dashboard' ? (
          <>
            {/* Analysis Section */}
            <section className="glass-panel p-6 rounded-[36px] shadow-sm">
              <div className="flex flex-col gap-4 mb-2">
                <div className="flex justify-between items-center">
                  <h2 className="text-lg font-bold text-slate-800 flex items-center gap-2">
                    <Icons.PieChart size={18} className="text-slate-400" />
                    {t.analysis}
                  </h2>
                </div>
                {/* Horizontal Segmented Control for Time */}
                <div className="bg-slate-100/50 p-1.5 rounded-2xl flex gap-1">
                  {(['hour', 'day', 'week', 'month'] as const).map((tf) => (
                    <button
                      key={tf}
                      onClick={() => setTimeFilter(tf)}
                      className={`flex-1 py-2 rounded-xl text-[10px] font-black uppercase tracking-widest transition-all ${
                        timeFilter === tf 
                        ? 'bg-white shadow-sm text-slate-900 scale-100' 
                        : 'text-slate-400 hover:text-slate-600'
                      }`}
                    >
                      {t.intervals[tf]}
                    </button>
                  ))}
                </div>
              </div>
              <SimpleBarChart 
                data={chartData} 
                theme={activeTab === 'finance' ? 'emerald' : 'rose'} 
              />
            </section>

            {/* List Section */}
            <section>
              <div className="flex justify-between items-center mb-4 px-2">
                <h3 className="text-slate-500 text-[10px] font-black uppercase tracking-widest opacity-80">{t.recentLogs}</h3>
                <button onClick={() => setView('events')} className="text-[10px] text-indigo-600 font-black uppercase tracking-wider hover:underline flex items-center gap-1 bg-indigo-50 px-2 py-1 rounded-lg">
                  {t.viewAll} <Icons.ChevronRight size={12} />
                </button>
              </div>
              <div className="space-y-0">
                {events.slice(0, 5).map((evt) => (
                  <div key={evt.id} className="cursor-pointer" onClick={() => openEditModal(evt)}>
                    <EventItem evt={evt} />
                  </div>
                ))}
                {events.length === 0 && (
                  <div className="glass-panel rounded-[32px] flex flex-col items-center justify-center py-12 text-slate-400 gap-3">
                    <Icons.Coins size={48} strokeWidth={1} className="opacity-50" />
                    <span className="text-xs font-bold italic opacity-70">{t.noRecords}</span>
                  </div>
                )}
              </div>
            </section>
          </>
        ) : (
          <section className="space-y-4">
            <h2 className="text-2xl font-bold text-slate-900 px-2 flex items-center gap-2">
              <div className="bg-slate-900 text-white p-2 rounded-xl"><Icons.List size={20} /></div>
              {t.records}
            </h2>
            <div className="glass-panel rounded-[32px] overflow-hidden shadow-sm">
              <table className="w-full text-left text-sm">
                <thead className="bg-slate-50/50 text-[10px] font-black uppercase text-slate-400">
                  <tr>
                    <th className="px-4 py-3">#</th>
                    <th className="px-4 py-3">{lang === 'bn' ? 'তারিখ' : 'Date'}</th>
                    <th className="px-4 py-3">{lang === 'bn' ? 'বিবরণ' : 'Desc'}</th>
                    <th className="px-4 py-3 text-right">{lang === 'bn' ? 'পরিমাণ' : 'Amt'}</th>
                  </tr>
                </thead>
                <tbody className="divide-y divide-slate-100/50">
                  {events.map((evt, idx) => {
                    const isIncome = evt.note === 'Income';
                    return (
                      <tr key={evt.id} className="hover:bg-white/40 transition-colors cursor-pointer" onClick={() => openEditModal(evt)}>
                        <td className="px-4 py-4 text-[10px] font-bold text-slate-300">{events.length - idx}</td>
                        <td className="px-4 py-4 text-slate-500 whitespace-nowrap text-xs font-semibold">{evt.date}</td>
                        <td className="px-4 py-4">
                          <div className="font-bold text-slate-800 text-xs">{evt.name}</div>
                          <div className="flex items-center gap-1 mt-0.5">
                             <span className="text-xs">{CATEGORY_ICONS[evt.category]}</span>
                             <div className="text-[8px] font-black text-slate-400 uppercase tracking-tighter">{evt.category}</div>
                          </div>
                        </td>
                        <td className={`px-4 py-4 text-right font-black ${isIncome ? 'text-emerald-500' : 'text-rose-500'}`}>
                          {isIncome ? '+' : '-'}{Math.abs(evt.cost).toLocaleString()}
                        </td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
              {events.length === 0 && (
                 <div className="flex flex-col items-center justify-center py-20 text-slate-300 gap-3">
                    <Icons.List size={48} strokeWidth={1} />
                    <span className="text-xs font-bold uppercase tracking-widest">{t.noRecords}</span>
                  </div>
              )}
            </div>
          </section>
        )}
      </main>

      {/* Nav - Floating Liquid Glass Bar */}
      <nav className="fixed bottom-6 left-1/2 transform -translate-x-1/2 w-[90%] max-w-sm bg-white/70 backdrop-blur-2xl border border-white/40 rounded-[32px] flex justify-between px-10 py-4 z-40 shadow-2xl shadow-indigo-500/10">
        <button onClick={() => setView('dashboard')} className={`flex flex-col items-center gap-1 transition-all duration-300 ${view === 'dashboard' ? 'text-indigo-600 -translate-y-1' : 'text-slate-400 hover:text-slate-600'}`}>
          <Icons.Home size={24} strokeWidth={view === 'dashboard' ? 3 : 2} />
          {view === 'dashboard' && <span className="w-1 h-1 bg-indigo-600 rounded-full mt-1"></span>}
        </button>
        
        {/* Floating Action Button (FAB) embedded or overlapping */}
        <div className="relative -top-10">
          <button 
            onClick={() => { setEditingId(null); setEventName(""); setCost(""); setShowModal(true); }} 
            className="h-16 w-16 bg-gradient-to-tr from-slate-900 to-slate-800 text-white rounded-full shadow-lg shadow-slate-900/30 flex items-center justify-center hover:scale-110 active:scale-95 transition-all ring-8 ring-white/30 backdrop-blur-sm"
          >
            <Icons.Plus size={32} />
          </button>
        </div>

        <button onClick={() => setView('events')} className={`flex flex-col items-center gap-1 transition-all duration-300 ${view === 'events' ? 'text-indigo-600 -translate-y-1' : 'text-slate-400 hover:text-slate-600'}`}>
          <Icons.List size={24} strokeWidth={view === 'events' ? 3 : 2} />
          {view === 'events' && <span className="w-1 h-1 bg-indigo-600 rounded-full mt-1"></span>}
        </button>
      </nav>

      {/* Entry Modal */}
      {showModal && (
        <div className="fixed inset-0 bg-slate-900/30 backdrop-blur-md z-50 flex items-end sm:items-center justify-center p-0 sm:p-4 animate-slide-up">
          <div className="bg-white/80 backdrop-blur-2xl border border-white/40 w-full max-w-md p-8 rounded-t-[48px] sm:rounded-[48px] shadow-2xl max-h-[95vh] overflow-y-auto no-scrollbar">
            <div className="flex justify-between items-start mb-6">
              <div>
                <h2 className="text-2xl font-black text-slate-900 uppercase tracking-tight">{editingId ? t.editEntry : t.newEntry}</h2>
                <p className="text-xs text-slate-500 font-bold mt-1 uppercase tracking-widest">{t.updateBalance}</p>
              </div>
              <button onClick={() => { setShowModal(false); setEditingId(null); }} className="p-2 bg-slate-100/50 hover:bg-slate-200/50 rounded-full text-slate-500 transition-colors">
                <Icons.X size={20}/>
              </button>
            </div>
            
            <div className="space-y-6">
              <div className="flex p-1.5 bg-slate-100/50 rounded-2xl">
                <button onClick={() => setEntryType('income')} className={`flex-1 py-3 rounded-xl text-xs font-black uppercase transition-all ${entryType === 'income' ? 'bg-white text-emerald-600 shadow-sm ring-1 ring-black/5' : 'text-slate-400 hover:text-slate-600'}`}>{t.inflow}</button>
                <button onClick={() => setEntryType('expense')} className={`flex-1 py-3 rounded-xl text-xs font-black uppercase transition-all ${entryType === 'expense' ? 'bg-white text-rose-600 shadow-sm ring-1 ring-black/5' : 'text-slate-400 hover:text-slate-600'}`}>{t.outflow}</button>
              </div>

              <div>
                <label className="text-[10px] font-black text-slate-400 uppercase tracking-widest block mb-2 px-1">{t.eventName}</label>
                <input type="text" value={eventName} onChange={(e) => setEventName(e.target.value)} placeholder={t.placeholderEvent} className="w-full p-4 bg-white/50 rounded-2xl border-2 border-transparent focus:border-indigo-500/50 focus:bg-white outline-none transition-all font-bold text-slate-800" />
              </div>

              <div>
                <label className={`text-[10px] font-black uppercase tracking-widest block mb-2 px-1 ${entryType === 'income' ? 'text-emerald-500' : 'text-rose-500'}`}>{t.cost}</label>
                <div className="relative">
                   <input type="number" value={cost} onChange={(e) => setCost(e.target.value)} placeholder="0" className={`w-full p-4 rounded-2xl border-2 focus:bg-white outline-none font-black text-xl transition-all pl-8 ${entryType === 'income' ? 'bg-emerald-50/50 border-emerald-100/50 focus:border-emerald-500 text-emerald-900' : 'bg-rose-50/50 border-rose-100/50 focus:border-rose-500 text-rose-900'}`} />
                   <span className={`absolute left-4 top-1/2 -translate-y-1/2 font-black ${entryType === 'income' ? 'text-emerald-300' : 'text-rose-300'}`}>{lang === 'bn' ? '৳' : '$'}</span>
                </div>
              </div>

              <div>
                <label className="text-[10px] font-black text-slate-400 uppercase tracking-widest block mb-2 px-1">{t.categorySelection}</label>
                <div className="grid grid-cols-3 gap-2">
                  {CATEGORIES.map(cat => (
                    <button key={cat} onClick={() => setCategory(cat)} className={`px-2 py-3 rounded-xl flex flex-col items-center gap-1 border transition-all ${category === cat ? 'bg-slate-900 text-white border-slate-900 shadow-lg scale-105' : 'bg-white/50 border-slate-100/50 text-slate-400 hover:bg-white'}`}>
                      <span className="text-xl">{CATEGORY_ICONS[cat]}</span>
                      <span className="text-[9px] font-black uppercase tracking-wide truncate w-full">{cat}</span>
                    </button>
                  ))}
                </div>
              </div>

              <div className="pt-4 flex gap-3">
                {editingId && (
                   <button onClick={() => { if(confirm(t.confirmDelete)) { setEvents(prev => prev.filter(e => e.id !== editingId)); setShowModal(false); setEditingId(null); }}} className="p-5 bg-rose-50 text-rose-600 rounded-[24px] hover:bg-rose-100 transition-all">
                    <Icons.Trash2 size={24} />
                   </button>
                )}
                <button onClick={handleSave} className={`flex-1 py-5 rounded-[24px] font-black text-lg shadow-xl shadow-indigo-500/20 active:scale-95 transition-all text-white ${entryType === 'income' ? 'bg-gradient-to-r from-emerald-500 to-emerald-600' : 'bg-gradient-to-r from-rose-500 to-rose-600'}`}>
                  {editingId ? t.update : t.save}
                </button>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* AI Assistant Modal */}
      <AIAssistant 
        isOpen={showAI} 
        onClose={() => setShowAI(false)} 
        events={events} 
        lang={lang} 
        translations={t}
      />
    </div>
  );
};

export default App;
