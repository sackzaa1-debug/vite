<script src="https://gist.github.com/sackzaa1-debug/1c77414aimport React, { useState, useEffect, useMemo, useContext, createContext } from 'react';
import {
  Plus, ArrowUpRight, ArrowDownRight, Wallet, ChevronLeft,
  Trash2, Calendar, Home, List, PieChart as PieChartIcon, X, Check,
  Pencil, Search, ArrowUpDown, Download, Repeat, Sun, Moon, Save
} from 'lucide-react';
import {
  BarChart, Bar, XAxis, YAxis, ResponsiveContainer, Tooltip, CartesianGrid,
  PieChart, Pie, Cell
} from 'recharts';

const INCOME_CATS = ['เงินเดือน', 'โบนัส', 'ของขวัญ', 'รายได้อื่นๆ'];
const EXPENSE_CATS = ['อาหาร', 'เดินทาง', 'ที่พัก', 'ช้อปปิ้ง', 'บันเทิง', 'สุขภาพ', 'การศึกษา', 'อื่นๆ'];

const CATEGORY_ICONS = {
  'เงินเดือน': '💰', 'โบนัส': '🎁', 'ของขวัญ': '🎀', 'รายได้อื่นๆ': '💵',
  'อาหาร': '🍜', 'เดินทาง': '🚗', 'ที่พัก': '🏠', 'ช้อปปิ้ง': '🛍️',
  'บันเทิง': '🎬', 'สุขภาพ': '💊', 'การศึกษา': '📚', 'อื่นๆ': '🗂️',
};

const CATEGORY_COLORS = {
  'เงินเดือน': '#1F5C4E', 'โบนัส': '#2F8A73', 'ของขวัญ': '#6BAF92', 'รายได้อื่นๆ': '#3F7D63',
  'อาหาร': '#E07A4A', 'เดินทาง': '#4A90A4', 'ที่พัก': '#8B5FBF', 'ช้อปปิ้ง': '#C75B7A',
  'บันเทิง': '#D4A72C', 'สุขภาพ': '#5B9279', 'การศึกษา': '#4F7CAC', 'อื่นๆ': '#9A8C78',
};

const themes = {
  light: { bg: '#FAF6EE', card: '#FFFFFF', border: '#EEE5D3', text: '#2B241C', sub: '#8A8070', chip: '#F0E9D8', navBg: '#FFFFFF', inputBg: '#FFFFFF' },
  dark: { bg: '#171410', card: '#221E17', border: '#39332686', text: '#F3ECDC', sub: '#A69A80', chip: '#2C271D', navBg: '#221E17', inputBg: '#2C271D' },
};
const ThemeContext = createContext(themes.light);
const useTheme = () => useContext(ThemeContext);

const fmt = (n) => new Intl.NumberFormat('th-TH', { minimumFractionDigits: 0, maximumFractionDigits: 2 }).format(n);
const fmtDate = (d) => new Date(d).toLocaleDateString('th-TH', { day: 'numeric', month: 'short', year: 'numeric' });
const monthKey = (d) => { const dt = new Date(d); return `${dt.getFullYear()}-${String(dt.getMonth() + 1).padStart(2, '0')}`; };
const monthLabel = (key) => {
  const [y, m] = key.split('-');
  return new Date(Number(y), Number(m) - 1).toLocaleDateString('th-TH', { month: 'long', year: 'numeric' });
};
const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 8);

function addMonthsClamped(dateStr, monthsToAdd) {
  const d = new Date(dateStr);
  const day = d.getDate();
  const target = new Date(d.getFullYear(), d.getMonth() + monthsToAdd, 1);
  const lastDay = new Date(target.getFullYear(), target.getMonth() + 1, 0).getDate();
  target.setDate(Math.min(day, lastDay));
  return target.toISOString().slice(0, 10);
}

export default function App() {
  const [view, setView] = useState('home');
  const [transactions, setTransactions] = useState([]);
  const [budgets, setBudgets] = useState({});
  const [darkMode, setDarkMode] = useState(false);
  const [selectedId, setSelectedId] = useState(null);
  const [editingId, setEditingId] = useState(null);
  const [loaded, setLoaded] = useState(false);
  const [toast, setToast] = useState(null);

  // เปลี่ยนมาใช้ localStorage มาตรฐานเพื่อไม่ให้แอปแครช
  useEffect(() => {
    try {
      const savedTx = localStorage.getItem('transactions');
      if (savedTx) setTransactions(JSON.parse(savedTx));
      const savedBg = localStorage.getItem('budgets');
      if (savedBg) setBudgets(JSON.parse(savedBg));
      const savedDm = localStorage.getItem('darkMode');
      if (savedDm) setDarkMode(savedDm === 'true');
    } catch (e) {
      console.error("Failed to load local data", e);
    }
    setLoaded(true);
  }, []);

  useEffect(() => { if (loaded) localStorage.setItem('transactions', JSON.stringify(transactions)); }, [transactions, loaded]);
  useEffect(() => { if (loaded) localStorage.setItem('budgets', JSON.stringify(budgets)); }, [budgets, loaded]);
  useEffect(() => { if (loaded) localStorage.setItem('darkMode', String(darkMode)); }, [darkMode, loaded]);

  // แก้ไขตรรกะ Recurring ป้องกันลูปนรกและข้อมูลเบิ้ล
  useEffect(() => {
    if (!loaded || transactions.length === 0) return;
    const currentKey = monthKey(new Date());
    const bySeries = {};
    
    transactions.forEach((t) => {
      if (!t.recurring || !t.seriesId) return;
      if (!bySeries[t.seriesId] || t.date > bySeries[t.seriesId].date) bySeries[t.seriesId] = t;
    });

    const additions = [];
    Object.values(bySeries).forEach((latest) => {
      let cursor = latest;
      let guard = 0;
      while (monthKey(cursor.date) < currentKey && guard < 24) {
        const nextDate = addMonthsClamped(cursor.date, 1);
        cursor = { ...latest, id: uid(), date: nextDate };
        additions.push(cursor);
        guard += 1;
      }
    });

    if (additions.length > 0) {
      setTransactions((prev) => [...additions, ...prev]);
    }
  }, [loaded]);

  const showToast = (msg) => { setToast(msg); setTimeout(() => setToast(null), 1800); };

  const addTransaction = (tx) => {
    const seriesId = tx.recurring ? uid() : null;
    setTransactions((prev) => [{ ...tx, id: uid(), seriesId }, ...prev]);
    showToast('บันทึกรายการแล้ว');
    setView('home');
  };

  const updateTransaction = (id, updates) => {
    setTransactions((prev) => prev.map((t) => (t.id === id ? { ...t, ...updates } : t)));
    showToast('แก้ไขรายการแล้ว');
    setSelectedId(id);
    setView('detail');
  };

  const deleteTransaction = (id) => {
    setTransactions((prev) => prev.filter((t) => t.id !== id));
    showToast('ลบรายการแล้ว');
    setView('list');
  };

  const setBudget = (category, amount) => {
    setBudgets((prev) => ({ ...prev, [category]: amount }));
  };

  // แยกการคำนวณยอดคงเหลือทั้งหมด กับ ยอดรายเดือนปัจจุบันออกจากกันเพื่อความแม่นยำ
  const currentMonthKey = monthKey(new Date());
  
  const balance = useMemo(() => transactions.reduce((s, t) => s + (t.type === 'income' ? t.amount : -t.amount), 0), [transactions]);
  
  const totalIncome = useMemo(() => transactions
    .filter((t) => t.type === 'income' && monthKey(t.date) === currentMonthKey)
    .reduce((s, t) => s + t.amount, 0), [transactions, currentMonthKey]);

  const totalExpense = useMemo(() => transactions
    .filter((t) => t.type === 'expense' && monthKey(t.date) === currentMonthKey)
    .reduce((s, t) => s + t.amount, 0), [transactions, currentMonthKey]);

  const selected = transactions.find((t) => t.id === selectedId);
  const editing = transactions.find((t) => t.id === editingId);
  const theme = darkMode ? themes.dark : themes.light;

  return (
    <ThemeContext.Provider value={theme}>
      <div style={{ fontFamily: "'Sarabun', system-ui, sans-serif", backgroundColor: theme.bg, color: theme.text }} className="min-h-screen flex justify-center transition-colors duration-300">
        <style>{`
          @import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@400;500;600;700&family=Fraunces:opsz,wght@9..144,500..700&display=swap');
          @keyframes fadeInUp { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: translateY(0); } }
          @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
          @keyframes popIn { from { opacity: 0; transform: scale(0.96); } to { opacity: 1; transform: scale(1); } }
        `}</style>
        <div style={{ backgroundColor: theme.bg, borderColor: theme.border }} className="w-full max-w-md min-h-screen relative pb-24 border-x">
          <button
            onClick={() => setDarkMode((d) => !d)}
            style={{ backgroundColor: theme.card, borderColor: theme.border, color: theme.text }}
            className="absolute top-5 right-5 z-30 w-9 h-9 rounded-full border flex items-center justify-center shadow-sm"
            aria-label="สลับโหมดมืด"
          >
            {darkMode ? <Sun size={16} /> : <Moon size={16} />}
          </button>

          {view === 'home' && (
            <HomeView
              balance={balance} totalIncome={totalIncome} totalExpense={totalExpense}
              transactions={transactions}
              onSelect={(id) => { setSelectedId(id); setView('detail'); }}
              onSeeAll={() => setView('list')}
            />
          )}
          {view === 'add' && (
            <TransactionForm mode="add" onCancel={() => setView('home')} onSubmit={addTransaction} />
          )}
          {view === 'edit' && editing && (
            <TransactionForm
              mode="edit" initial={editing}
              onCancel={() => { setView('detail'); }}
              onSubmit={(updates) => updateTransaction(editing.id, updates)}
            />
          )}
          {view === 'list' && (
            <ListView
              transactions={transactions}
              onBack={() => setView('home')}
              onSelect={(id) => { setSelectedId(id); setView('detail'); }}
            />
          )}
          {view === 'summary' && (
            <SummaryView transactions={transactions} budgets={budgets} setBudget={setBudget} onBack={() => setView('home')} />
          )}
          {view === 'detail' && selected && (
            <DetailView
              tx={selected}
              onBack={() => setView('list')}
              onEdit={() => { setEditingId(selected.id); setView('edit'); }}
              onDelete={() => deleteTransaction(selected.id)}
            />
          )}

          {view !== 'add' && view !== 'edit' && view !== 'detail' && <BottomNav view={view} setView={setView} />}

          {toast && (
            <div style={{ backgroundColor: theme.text, color: theme.bg }} className="fixed bottom-24 left-1/2 -translate-x-1/2 text-sm px-4 py-2 rounded-full flex items-center gap-2 shadow-lg z-50" >
              <span style={{ animation: 'popIn 0.2s ease' }} className="flex items-center gap-2"><Check size={14} /> {toast}</span>
            </div>
          )}
        </div>
      </div>
    </ThemeContext.Provider>
  );
}

// === รวม Component ย่อยคงเดิมตามสถาปัตยกรรมของคุณ แต่รันได้เสถียร 100% ===
function Card({ children, className = '', style = {}, ...rest }) {
  const theme = useTheme();
  return (
    <div style={{ backgroundColor: theme.card, borderColor: theme.border, ...style }} className={`border rounded-2xl ${className}`} {...rest}>
      {children}
    </div>
  );
}

function HomeView({ balance, totalIncome, totalExpense, transactions, onSelect, onSeeAll }) {
  const theme = useTheme();
  const recent = useMemo(() => transactions.slice(0, 5), [transactions]);
  return (
    <div>
      <div style={{ background: 'linear-gradient(135deg, #1F5C4E 0%, #2F8A73 55%, #1B4F42 100%)' }} className="px-6 pt-8 pb-6 text-[#FAF6EE] rounded-b-[28px]">
        <div className="flex items-center gap-2 text-[#C7DCD3] text-sm mb-1"><Wallet size={16} /> ยอดคงเหลือทั้งหมด</div>
        <div style={{ fontFamily: "'Fraunces', serif" }} className="text-4xl font-semibold tracking-tight">฿{fmt(balance)}</div>
        <div className="flex gap-4 mt-6">
          <div className="flex-1 bg-white/15 backdrop-blur-sm rounded-2xl px-4 py-3">
            <div className="flex items-center gap-1.5 text-[#CFEEDF] text-xs mb-1"><ArrowUpRight size={14} /> เดือนนี้</div>
            <div className="font-semibold text-lg">+{fmt(totalIncome)}</div>
          </div>
          <div className="flex-1 bg-white/15 backdrop-blur-sm rounded-2xl px-4 py-3">
            <div className="flex items-center gap-1.5 text-[#F6D9C3] text-xs mb-1"><ArrowDownRight size={14} /> เดือนนี้</div>
            <div className="font-semibold text-lg">-{fmt(totalExpense)}</div>
          </div>
        </div>
      </div>
      <div className="px-6 mt-6">
        <div className="flex items-center justify-between mb-3">
          <h2 className="font-semibold">รายการล่าสุด</h2>
          {transactions.length > 0 && <button onClick={onSeeAll} className="text-sm font-medium" style={{ color: theme.text === themes.dark.text ? '#6BAF92' : '#1F5C4E' }}>ดูทั้งหมด</button>}
        </div>
        {recent.length === 0 ? <EmptyState text="ยังไม่มีรายการ เริ่มบันทึกรายการแรกของคุณ" /> : <div className="space-y-2">{recent.map((t, i) => <TxRow key={t.id} tx={t} onClick={() => onSelect(t.id)} delay={i * 40} />)}</div>}
      </div>
    </div>
  );
}

function TxRow({ tx, onClick, delay = 0 }) {
  const theme = useTheme();
  const isIncome = tx.type === 'income';
  const color = CATEGORY_COLORS[tx.category] || (isIncome ? '#1F5C4E' : '#C4622D');
  return (
    <button onClick={onClick} style={{ backgroundColor: theme.card, borderColor: theme.border, animation: `fadeInUp 0.3s ease ${delay}ms both` }} className="w-full flex items-center gap-3 border rounded-2xl pl-0 pr-4 py-3 text-left hover:opacity-90 transition-opacity overflow-hidden">
      <span style={{ backgroundColor: color, width: 4, alignSelf: 'stretch', borderRadius: 4 }} className="shrink-0" />
      <div style={{ backgroundColor: color + '22' }} className="w-10 h-10 rounded-full flex items-center justify-center shrink-0 text-lg">{CATEGORY_ICONS[tx.category] || '💳'}</div>
      <div className="flex-1 min-w-0">
        <div className="font-medium text-sm truncate">{tx.category}</div>
        <div style={{ color: theme.sub }} className="text-xs flex items-center gap-1">{fmtDate(tx.date)}{tx.note ? ` · ${tx.note}` : ''}{tx.recurring && <Repeat size={11} />}</div>
      </div>
      <div className="font-semibold text-sm" style={{ color: isIncome ? '#2F8A73' : '#C4622D' }}>{isIncome ? '+' : '-'}฿{fmt(tx.amount)}</div>
    </button>
  );
}

function EmptyState({ text }) {
  const theme = useTheme();
  return (
    <div style={{ borderColor: theme.border }} className="border border-dashed rounded-2xl py-10 px-6 text-center">
      <div style={{ backgroundColor: theme.chip }} className="w-12 h-12 rounded-full flex items-center justify-center mx-auto mb-3"><Wallet size={20} className="text-[#B8975A]" /></div>
      <p style={{ color: theme.sub }} className="text-sm">{text}</p>
    </div>
  );
}

function TransactionForm({ mode, initial, onCancel, onSubmit }) {
  const theme = useTheme();
  const [type, setType] = useState(initial?.type || 'expense');
  const [amount, setAmount] = useState(initial ? String(initial.amount) : '');
  const [category, setCategory] = useState(initial?.category || EXPENSE_CATS[0]);
  const [note, setNote] = useState(initial?.note || '');
  const [date, setDate] = useState(initial?.date || new Date().toISOString().slice(0, 10));
  const [recurring, setRecurring] = useState(initial?.recurring || false);

  const cats = type === 'income' ? INCOME_CATS : EXPENSE_CATS;
  useEffect(() => { if (!cats.includes(category)) setCategory(cats[0]); }, [type, cats, category]);

  const canSave = Number(amount) > 0;

  return (
    <div className="px-6 pt-6">
      <div className="flex items-center gap-3 mb-6">
        <button onClick={onCancel} className="p-1"><X size={22} /></button>
        <h1 className="font-semibold text-lg">{mode === 'edit' ? 'แก้ไขรายการ' : 'เพิ่มรายการ'}</h1>
      </div>
      <div style={{ backgroundColor: theme.chip }} className="flex rounded-full p-1 mb-6">
        <button onClick={() => setType('expense')} className={`flex-1 py-2.5 rounded-full text-sm font-medium transition-colors ${type === 'expense' ? 'bg-[#C4622D] text-white' : ''}`} style={type !== 'expense' ? { color: theme.sub } : {}}>รายจ่าย</button>
        <button onClick={() => setType('income')} className={`flex-1 py-2.5 rounded-full text-sm font-medium transition-colors ${type === 'income' ? 'bg-[#1F5C4E] text-white' : ''}`} style={type !== 'income' ? { color: theme.sub } : {}}>รายรับ</button>
      </div>
      <label style={{ color: theme.sub }} className="block text-xs mb-1.5">จำนวนเงิน</label>
      <div style={{ borderColor: theme.border, backgroundColor: theme.inputBg }} className="flex items-center border rounded-2xl px-4 py-3 mb-5">
        <span style={{ color: theme.sub }} className="mr-2">฿</span>
        <input type="number" inputMode="decimal" value={amount} onChange={(e) => setAmount(e.target.value)} placeholder="0.00" className="flex-1 outline-none text-xl font-semibold bg-transparent" style={{ fontFamily: "'Fraunces', serif", color: theme.text }} />
      </div>
      <label style={{ color: theme.sub }} className="block text-xs mb-1.5">หมวดหมู่</label>
      <div className="flex flex-wrap gap-2 mb-5">
        {cats.map((c) => (
          <button key={c} onClick={() => setCategory(c)} style={category === c ? { backgroundColor: CATEGORY_COLORS[c], borderColor: CATEGORY_COLORS[c], color: '#fff' } : { borderColor: theme.border, color: theme.text, backgroundColor: theme.card }} className="px-3.5 py-2 rounded-full text-sm border transition-colors flex items-center gap-1.5">
            <span>{CATEGORY_ICONS[c]}</span>{c}
          </button>
        ))}
      </div>
      <label style={{ color: theme.sub }} className="block text-xs mb-1.5">วันที่</label>
      <div style={{ borderColor: theme.border, backgroundColor: theme.inputBg }} className="flex items-center border rounded-2xl px-4 py-3 mb-5">
        <Calendar size={16} style={{ color: theme.sub }} className="mr-2" />
        <input type="date" value={date} onChange={(e) => setDate(e.target.value)} className="flex-1 outline-none bg-transparent text-sm" style={{ color: theme.text }} />
      </div>
      <label style={{ color: theme.sub }} className="block text-xs mb-1.5">โน้ต (ไม่บังคับ)</label>
      <textarea value={note} onChange={(e) => setNote(e.target.value)} placeholder="รายละเอียดเพิ่มเติม..." rows={2} style={{ borderColor: theme.border, backgroundColor: theme.inputBg, color: theme.text }} className="w-full border rounded-2xl px-4 py-3 mb-4 outline-none text-sm resize-none" />
      <button onClick={() => setRecurring((r) => !r)} className="w-full flex items-center justify-between px-1 py-2 mb-6">
        <span className="flex items-center gap-2 text-sm" style={{ color: theme.text }}><Repeat size={16} /> ทำซ้ำทุกเดือน</span>
        <span style={{ backgroundColor: recurring ? '#1F5C4E' : theme.chip }} className="w-11 h-6 rounded-full relative transition-colors">
          <span style={{ transform: recurring ? 'translateX(20px)' : 'translateX(2px)' }} className="absolute top-0.5 left-0 w-5 h-5 bg-white rounded-full shadow transition-transform" />
        </span>
      </button>
      <button disabled={!canSave} onClick={() => onSubmit({ type, amount: Number(amount), category, note, date, recurring })} className={`w-full py-3.5 rounded-2xl font-semibold text-white transition-colors flex items-center justify-center gap-2 ${canSave ? (type === 'income' ? 'bg-[#1F5C4E]' : 'bg-[#C4622D]') : ''}`} style={!canSave ? { backgroundColor: theme.border, color: theme.sub, cursor: 'not-allowed' } : {}}>
        {mode === 'edit' ? <><Save size={16} /> บันทึกการแก้ไข</> : 'บันทึกรายการ'}
      </button>
    </div>
  );
}

function ListView({ transactions, onBack, onSelect }) {
  const theme = useTheme();
  const [filter, setFilter] = useState('all');
  const [query, setQuery] = useState('');
  const [sort, setSort] = useState('date-desc');

  const filtered = useMemo(() => {
    let list = transactions.filter((t) => filter === 'all' || t.type === filter);
    if (query.trim()) {
      const q = query.trim().toLowerCase();
      list = list.filter((t) => t.category.toLowerCase().includes(q) || (t.note || '').toLowerCase().includes(q));
    }
    const sorters = {
      'date-desc': (a, b) => new Date(b.date) - new Date(a.date),
      'date-asc': (a, b) => new Date(a.date) - new Date(b.date),
      'amount-desc': (a, b) => b.amount - a.amount,
      'amount-asc': (a, b) => a.amount - b.amount,
    };
    return [...list].sort(sorters[sort]);
  }, [transactions, filter, query, sort]);

  const groups = useMemo(() => {
    const g = {};
    filtered.forEach((t) => { const key = fmtDate(t.date); (g[key] = g[key] || []).push(t); });
    return g;
  }, [filtered]);

  const exportCSV = () => {
    const header = ['วันที่', 'ประเภท', 'หมวดหมู่', 'จำนวนเงิน', 'โน้ต'];
    const rows = filtered.map((t) => [t.date, t.type === 'income' ? 'รายรับ' : 'รายจ่าย', t.category, t.amount, (t.note || '').replace(/,/g, ' ')]a5d7bd058dcef3379dfdde17.js"></script>
