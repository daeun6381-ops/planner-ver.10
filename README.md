# planner-ver.10
import React, { useState, useEffect, useRef } from 'react';
import { 
  ChevronLeft, ChevronRight, Plus, Trash2, Settings, Clock, Play, Pause, RotateCcw 
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection } from 'firebase/firestore';

// --- Firebase Config ---
const firebaseConfig = {
  apiKey: "AIzaSyBv14g9crV8vGbobK5cdVxwqlrr0EiM0fA",
  authDomain: "bokk-55eae.firebaseapp.com",
  projectId: "bokk-55eae",
  storageBucket: "bokk-55eae.firebasestorage.app",
  messagingSenderId: "757990079253",
  appId: "1:757990079253:web:f3dbf171b2137f436bc71c",
  measurementId: "G-2PMDNT4LM2"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const APP_ID = 'planner-app-v1';

const App = () => {
  const [user, setUser] = useState(null);
  const [view, setView] = useState('daily');
  const [records, setRecords] = useState({});
  const [loading, setLoading] = useState(true);
  const [dDayTarget, setDDayTarget] = useState('');
  
  const getTodayStr = () => {
    const d = new Date();
    return `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
  };

  const [currentDate, setCurrentDate] = useState(getTodayStr());
  const [isTimerRunning, setIsTimerRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const timerRef = useRef(null);

  const currentRecord = records[currentDate] || { 
    date: currentDate, resolution: "", todos: [], totalTime: 0 
  };

  // --- 테마 설정 ---
  const theme = {
    primary: '#BAE6FD',
    secondary: '#E0F2FE',
    point: '#0284C7',
    sidebar: '#0C4A6E',
    timerBg: '#0C4A6E',
  };

  // --- Auth & Data Sync ---
  useEffect(() => {
    let unsubDDay, unsubRecords;
    const unsubAuth = onAuthStateChanged(auth, async (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
        unsubDDay = onSnapshot(doc(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'settings', 'dday'), (s) => {
          if (s.exists()) setDDayTarget(s.data().target || '');
        });
        unsubRecords = onSnapshot(collection(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'records'), (qs) => {
          const data = {};
          qs.forEach(d => data[d.id] = d.data());
          setRecords(data);
          setLoading(false);
        });
      } else {
        signInAnonymously(auth).catch(e => { console.error(e); setLoading(false); });
      }
    });
    return () => { if (unsubAuth) unsubAuth(); if (unsubDDay) unsubDDay(); if (unsubRecords) unsubRecords(); };
  }, []);

  useEffect(() => {
    setSeconds(currentRecord.totalTime || 0);
    setIsTimerRunning(false);
    if (timerRef.current) clearInterval(timerRef.current);
  }, [currentDate]);

  // --- Handlers ---
  const saveRecord = async (updates) => {
    if (!user) return;
    const ref = doc(db, 'artifacts', APP_ID, 'users', user.uid, 'records', currentDate);
    await setDoc(ref, { ...currentRecord, ...updates }, { merge: true });
  };

  const handleDDayChange = async (val) => {
    setDDayTarget(val);
    if (user) await setDoc(doc(db, 'artifacts', APP_ID, 'users', user.uid, 'settings', 'dday'), { target: val });
  };

  const toggleTimer = () => {
    if (isTimerRunning) {
      clearInterval(timerRef.current);
      saveRecord({ totalTime: seconds });
    } else {
      timerRef.current = setInterval(() => setSeconds(p => p + 1), 1000);
    }
    setIsTimerRunning(!isTimerRunning);
  };

  const resetTimer = () => {
    if (timerRef.current) clearInterval(timerRef.current);
    setIsTimerRunning(false);
    setSeconds(0);
    saveRecord({ totalTime: 0 });
  };

  const addTodo = () => saveRecord({ todos: [...currentRecord.todos, { id: Date.now(), text: "", completed: false }] });
  const updateTodo = (id, text) => saveRecord({ todos: currentRecord.todos.map(t => t.id === id ? { ...t, text } : t) });
  const toggleTodo = (id) => saveRecord({ todos: currentRecord.todos.map(t => t.id === id ? { ...t, completed: !t.completed } : t) });
  const removeTodo = (id) => saveRecord({ todos: currentRecord.todos.filter(t => t.id !== id) });

  const formatDisplayDate = (s) => {
    if(!s) return "";
    const d = new Date(s.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
    return `${s.substring(0,4)}/${s.substring(4,6)}/${s.substring(6,8)} (${["일","월","화","수","목","금","토"][d.getDay()]})`;
  };

  if (loading) return <div style={{display:'flex', height:'100vh', alignItems:'center', justifyContent:'center'}}>데이터 연결 중...</div>;

  return (
    <div style={{ height: '100vh', backgroundColor: '#F0F9FF', display: 'flex', alignItems: 'center', justifyContent: 'center', fontFamily: 'sans-serif' }}>
      <div style={{ width: '90%', maxWidth: '1100px', height: '85vh', backgroundColor: 'white', display: 'flex', borderRadius: '8px', overflow: 'hidden', boxShadow: '0 20px 25px -5px rgba(0,0,0,0.1)', borderLeft: `30px solid ${theme.sidebar}`, position: 'relative' }}>
        
        {/* 사이드바 데코 (스프링 모양) */}
  <div style={{ position: 'absolute', left: '-20px', top: 0, bottom: 0, width: '20px', display: 'flex', flexDirection: 'column', justifyContent: 'space-around', zIndex: 10 }}>
          {[...Array(10)].map((_, i) => (
            <div key={i} style={{ width: '30px', height: '10px', backgroundColor: '#E0F2FE', borderRadius: '10px', border: '1px solid rgba(0,0,0,0.05)' }} />
          ))}
        </div>

  <div style={{ flex: 1, display: 'flex', flexDirection: 'column', padding: '20px' }}>
          {/* 상단 헤더 */}
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', borderBottom: '1px solid #f0f0f0', paddingBottom: '15px' }}>
            <div style={{ display: 'flex', alignItems: 'center', gap: '15px' }}>
              <button onClick={() => {
                const d = new Date(currentDate.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
                d.setDate(d.getDate() - 1);
                setCurrentDate(`${d.getFullYear()}${String(d.getMonth()+1).padStart(2,'0')}${String(d.getDate()).padStart(2,'0')}`);
              }} style={{ border: 'none', background: 'none', cursor: 'pointer' }}><ChevronLeft color="#cbd5e1" /></button>
              <h1 style={{ fontSize: '20px', fontWeight: '900', margin: 0 }}>{formatDisplayDate(currentDate)}</h1>
              <button onClick={() => {
                const d = new Date(currentDate.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
                d.setDate(d.getDate() + 1);
                setCurrentDate(`${d.getFullYear()}${String(d.getMonth()+1).padStart(2,'0')}${String(d.getDate()).padStart(2,'0')}`);
              }} style={{ border: 'none', background: 'none', cursor: 'pointer' }}><ChevronRight color="#cbd5e1" /></button>
            </div>

  <div style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
              <div style={{ padding: '5px 15px', borderRadius: '20px', backgroundColor: theme.secondary, color: theme.point, fontWeight: 'bold' }}>
                { (function() {
                    if (!dDayTarget) return "D-?";
                    const target = new Date(dDayTarget);
                    const today = new Date(); today.setHours(0,0,0,0);
                    const diff = Math.ceil((target - today) / (1000 * 60 * 60 * 24));
                    return diff === 0 ? "D-Day" : diff > 0 ? `D-${diff}` : `D+${Math.abs(diff)}`;
                })() }
                <input type="date" value={dDayTarget} onChange={(e) => handleDDayChange(e.target.value)} style={{ marginLeft: '10px', border: 'none', background: 'transparent', fontSize: '10px', cursor: 'pointer' }} />
              </div>
            </div>
          </div>

          {/* 메인 콘텐츠 */}
  <div style={{ flex: 1, display: 'flex', marginTop: '20px', gap: '20px', overflow: 'hidden' }}>
            {/* 왼쪽: 투두 리스트 */}
            <div style={{ flex: 1.5, display: 'flex', flexDirection: 'column' }}>
              <div style={{ marginBottom: '15px', padding: '10px', backgroundColor: '#F0F9FF', borderRadius: '10px', border: `2px solid ${theme.primary}` }}>
                <textarea 
                  placeholder="오늘의 다짐..." 
                  value={currentRecord.resolution} 
                  onChange={(e) => saveRecord({ resolution: e.target.value })}
                  style={{ width: '100%', border: 'none', background: 'transparent', resize: 'none', fontWeight: 'bold', outline: 'none' }}
                />
              </div>
              <div style={{ flex: 1, overflowY: 'auto' }}>
                <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: '10px' }}>
                  <span style={{ backgroundColor: theme.point, color: 'white', padding: '2px 10px', borderRadius: '10px', fontSize: '12px', fontWeight: 'bold' }}>TASKS</span>
                  <button onClick={addTodo} style={{ background: theme.sidebar, color: 'white', border: 'none', borderRadius: '50%', width: '30px', height: '30px', cursor: 'pointer' }}><Plus size={20}/></button>
                </div>
                {currentRecord.todos.map(todo => (
                  <div key={todo.id} style={{ display: 'flex', alignItems: 'center', gap: '10px', marginBottom: '10px' }}>
                    <input type="checkbox" checked={todo.completed} onChange={() => toggleTodo(todo.id)} style={{ width: '20px', height: '20px' }} />
                    <input 
                      value={todo.text} 
                      onChange={(e) => updateTodo(todo.id, e.target.value)} 
                      style={{ flex: 1, border: 'none', borderBottom: '1px solid #eee', outline: 'none', fontSize: '16px', textDecoration: todo.completed ? 'line-through' : 'none', color: todo.completed ? '#cbd5e1' : '#333' }}
                    />
                    <button onClick={() => removeTodo(todo.id)} style={{ background: 'none', border: 'none', color: '#ffaaaa', cursor: 'pointer' }}><Trash2 size={16}/></button>
                  </div>
                ))}
              </div>
            </div>

            {/* 오른쪽: 타이머 */}
  <div style={{ flex: 1, display: 'flex', flexDirection: 'column', justifyContent: 'flex-end', paddingBottom: '20px' }}>
              <div style={{ backgroundColor: theme.timerBg, padding: '20px', borderRadius: '20px', display: 'flex', alignItems: 'center', justifyContent: 'space-between', boxShadow: '0 10px 15px rgba(0,0,0,0.2)' }}>
                <button onClick={toggleTimer} style={{ width: '50px', height: '50px', borderRadius: '50%', border: 'none', backgroundColor: theme.primary, cursor: 'pointer', display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
                  {isTimerRunning ? <Pause fill={theme.sidebar} /> : <Play fill={theme.sidebar} />}
                </button>
                <div style={{ color: theme.primary, textAlign: 'right' }}>
                  <div style={{ fontSize: '30px', fontWeight: 'bold', fontFamily: 'monospace' }}>
                    {String(Math.floor((seconds % 3600) / 60)).padStart(2, '0')}:{String(seconds % 60).padStart(2, '0')}
                  </div>
                  <div style={{ fontSize: '12px' }}>{Math.floor(seconds / 3600)}H TOTAL</div>
                </div>
                <button onClick={resetTimer} style={{ background: 'rgba(255,255,255,0.1)', border: 'none', borderRadius: '50%', width: '40px', height: '40px', color: 'white', cursor: 'pointer' }}><RotateCcw size={20}/></button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};

export default App;
