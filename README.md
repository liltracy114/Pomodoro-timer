import { useState, useEffect, useRef, useCallback } from "react";

const MODES = {
  pomodoro: { label: "Focus", duration: 25 * 60, color: "#e8857a" },
  short: { label: "Short Break", duration: 5 * 60, color: "#7ab8e8" },
  long: { label: "Long Break", duration: 15 * 60, color: "#a8d8a0" },
};

const QUOTES = [
  "Small steps every day.",
  "Progress, not perfection.",
  "One task at a time.",
  "You've got this.",
  "Stay focused, stay kind.",
  "Rest is part of the work.",
];

const PALETTES = {
  rose:    { bg: "#fff5f4", card: "#fff0ee", accent: "#e8857a", text: "#4a2020", muted: "#b07070", ring: "#f5c5c0" },
  sky:     { bg: "#f4f8ff", card: "#eef4ff", accent: "#6fa3e8", text: "#1a2e4a", muted: "#6080a0", ring: "#c0d8f5" },
  sage:    { bg: "#f4faf5", card: "#eef8f0", accent: "#6db880", text: "#1a3a22", muted: "#5a8a65", ring: "#b8dfbe" },
  lavender:{ bg: "#f7f4ff", card: "#f2eeff", accent: "#9b7ee8", text: "#2a1a4a", muted: "#7a60b0", ring: "#d0c0f5" },
};

function pad(n) { return String(n).padStart(2, "0"); }
function fmt(s) { return `${pad(Math.floor(s / 60))}:${pad(s % 60)}`; }

function uuid() { return Math.random().toString(36).slice(2, 10); }

export default function App() {
  const [palette, setPalette] = useState("rose");
  const P = PALETTES[palette];

  // Timer state
  const [mode, setMode] = useState("pomodoro");
  const [timeLeft, setTimeLeft] = useState(MODES.pomodoro.duration);
  const [running, setRunning] = useState(false);
  const [sessions, setSessions] = useState(0);
  const [quote] = useState(QUOTES[Math.floor(Math.random() * QUOTES.length)]);
  const intervalRef = useRef(null);

  // Task state
  const [lists, setLists] = useState([
    { id: uuid(), name: "Today", tasks: [
      { id: uuid(), text: "Review project proposal", done: false, poms: 0 },
      { id: uuid(), text: "Send morning emails", done: true, poms: 1 },
    ]},
    { id: uuid(), name: "This Week", tasks: [
      { id: uuid(), text: "Weekly planning", done: false, poms: 0 },
    ]},
  ]);
  const [activeList, setActiveList] = useState(null);
  const [newListName, setNewListName] = useState("");
  const [newTask, setNewTask] = useState("");
  const [activeTab, setActiveTab] = useState("timer");
  const [editingList, setEditingList] = useState(null);
  const [editName, setEditName] = useState("");
  const [showAddList, setShowAddList] = useState(false);

  useEffect(() => {
    if (lists.length > 0 && !activeList) setActiveList(lists[0].id);
  }, []);

  // Timer
  useEffect(() => {
    if (running) {
      intervalRef.current = setInterval(() => {
        setTimeLeft(t => {
          if (t <= 1) {
            clearInterval(intervalRef.current);
            setRunning(false);
            if (mode === "pomodoro") setSessions(s => s + 1);
            return 0;
          }
          return t - 1;
        });
      }, 1000);
    } else {
      clearInterval(intervalRef.current);
    }
    return () => clearInterval(intervalRef.current);
  }, [running, mode]);

  const switchMode = (m) => {
    setMode(m);
    setRunning(false);
    setTimeLeft(MODES[m].duration);
  };

  const resetTimer = () => {
    setRunning(false);
    setTimeLeft(MODES[mode].duration);
  };

  const progress = 1 - timeLeft / MODES[mode].duration;
  const r = 88;
  const circ = 2 * Math.PI * r;
  const dash = circ * progress;

  // Tasks
  const currentList = lists.find(l => l.id === activeList);

  const addTask = () => {
    if (!newTask.trim() || !activeList) return;
    setLists(ls => ls.map(l => l.id === activeList
      ? { ...l, tasks: [...l.tasks, { id: uuid(), text: newTask.trim(), done: false, poms: 0 }] }
      : l));
    setNewTask("");
  };

  const toggleTask = (tid) => {
    setLists(ls => ls.map(l => l.id === activeList
      ? { ...l, tasks: l.tasks.map(t => t.id === tid ? { ...t, done: !t.done } : t) }
      : l));
  };

  const deleteTask = (tid) => {
    setLists(ls => ls.map(l => l.id === activeList
      ? { ...l, tasks: l.tasks.filter(t => t.id !== tid) }
      : l));
  };

  const addList = () => {
    if (!newListName.trim()) return;
    const nl = { id: uuid(), name: newListName.trim(), tasks: [] };
    setLists(ls => [...ls, nl]);
    setActiveList(nl.id);
    setNewListName("");
    setShowAddList(false);
  };

  const deleteList = (lid) => {
    setLists(ls => ls.filter(l => l.id !== lid));
    if (activeList === lid) setActiveList(lists.find(l => l.id !== lid)?.id || null);
  };

  const saveListName = () => {
    setLists(ls => ls.map(l => l.id === editingList ? { ...l, name: editName } : l));
    setEditingList(null);
  };

  const addPomToTask = (tid) => {
    setLists(ls => ls.map(l => l.id === activeList
      ? { ...l, tasks: l.tasks.map(t => t.id === tid ? { ...t, poms: t.poms + 1 } : t) }
      : l));
  };

  const totalTasks = lists.reduce((a, l) => a + l.tasks.length, 0);
  const doneTasks = lists.reduce((a, l) => a + l.tasks.filter(t => t.done).length, 0);

  const styles = {
    app: {
      minHeight: "100vh",
      background: P.bg,
      fontFamily: "'Crimson Pro', 'Georgia', serif",
      display: "flex",
      flexDirection: "column",
      alignItems: "center",
      padding: "0 0 3rem",
      color: P.text,
      transition: "background 0.4s",
    },
    header: {
      width: "100%",
      maxWidth: 520,
      padding: "2rem 1.5rem 0",
      display: "flex",
      justifyContent: "space-between",
      alignItems: "center",
    },
    title: {
      fontSize: 22,
      fontWeight: 600,
      letterSpacing: "0.02em",
      color: P.accent,
      margin: 0,
    },
    palettePicker: {
      display: "flex",
      gap: 6,
    },
    dot: (k) => ({
      width: 16,
      height: 16,
      borderRadius: "50%",
      background: PALETTES[k].accent,
      border: palette === k ? `2.5px solid ${P.text}` : "2px solid transparent",
      cursor: "pointer",
      transition: "transform 0.15s",
    }),
    card: {
      background: P.card,
      borderRadius: 24,
      padding: "1.75rem",
      width: "100%",
      maxWidth: 520,
      boxShadow: `0 4px 32px ${P.ring}aa`,
      margin: "1.25rem 1.5rem 0",
    },
    tabs: {
      display: "flex",
      gap: 8,
      marginBottom: "1.5rem",
    },
    tab: (active) => ({
      flex: 1,
      padding: "0.5rem 0",
      borderRadius: 12,
      border: "none",
      background: active ? P.accent : "transparent",
      color: active ? "#fff" : P.muted,
      fontFamily: "inherit",
      fontSize: 15,
      fontWeight: active ? 600 : 400,
      cursor: "pointer",
      transition: "all 0.2s",
      letterSpacing: "0.02em",
    }),
    modeRow: {
      display: "flex",
      gap: 6,
      marginBottom: "1.75rem",
      justifyContent: "center",
    },
    modeBtn: (active, m) => ({
      padding: "5px 14px",
      borderRadius: 20,
      border: `1.5px solid ${active ? MODES[m].color : P.ring}`,
      background: active ? MODES[m].color + "22" : "transparent",
      color: active ? MODES[m].color : P.muted,
      fontFamily: "inherit",
      fontSize: 13,
      cursor: "pointer",
      transition: "all 0.2s",
    }),
    timerWrap: {
      display: "flex",
      justifyContent: "center",
      marginBottom: "1.5rem",
      position: "relative",
    },
    timerTime: {
      position: "absolute",
      top: "50%",
      left: "50%",
      transform: "translate(-50%, -50%)",
      textAlign: "center",
    },
    timeNum: {
      fontSize: 52,
      fontWeight: 300,
      letterSpacing: "0.04em",
      color: P.text,
      lineHeight: 1,
      fontVariantNumeric: "tabular-nums",
    },
    modeLbl: {
      fontSize: 13,
      color: P.muted,
      marginTop: 4,
      letterSpacing: "0.06em",
      textTransform: "uppercase",
    },
    btnRow: {
      display: "flex",
      gap: 10,
      justifyContent: "center",
      marginBottom: "1.25rem",
    },
    mainBtn: {
      padding: "0.65rem 2.5rem",
      borderRadius: 50,
      border: "none",
      background: P.accent,
      color: "#fff",
      fontSize: 17,
      fontFamily: "inherit",
      fontWeight: 600,
      cursor: "pointer",
      letterSpacing: "0.04em",
      boxShadow: `0 4px 16px ${P.accent}66`,
      transition: "transform 0.12s, box-shadow 0.12s",
    },
    resetBtn: {
      padding: "0.65rem 1.2rem",
      borderRadius: 50,
      border: `1.5px solid ${P.ring}`,
      background: "transparent",
      color: P.muted,
      fontSize: 17,
      fontFamily: "inherit",
      cursor: "pointer",
      transition: "all 0.15s",
    },
    sessionRow: {
      display: "flex",
      justifyContent: "center",
      gap: 6,
      flexWrap: "wrap",
    },
    tomato: (filled) => ({
      width: 20,
      height: 20,
      borderRadius: "50%",
      background: filled ? P.accent : P.ring,
      display: "inline-block",
      transition: "background 0.3s",
    }),
    quoteBox: {
      textAlign: "center",
      color: P.muted,
      fontSize: 14,
      fontStyle: "italic",
      marginTop: "1rem",
      letterSpacing: "0.02em",
    },
    // Task styles
    listTabs: {
      display: "flex",
      gap: 6,
      flexWrap: "wrap",
      marginBottom: "1rem",
      alignItems: "center",
    },
    listTab: (active) => ({
      padding: "4px 12px",
      borderRadius: 14,
      border: `1.5px solid ${active ? P.accent : P.ring}`,
      background: active ? P.accent : "transparent",
      color: active ? "#fff" : P.muted,
      fontFamily: "inherit",
      fontSize: 13,
      cursor: "pointer",
      transition: "all 0.18s",
      display: "flex",
      alignItems: "center",
      gap: 5,
    }),
    addListBtn: {
      padding: "4px 10px",
      borderRadius: 14,
      border: `1.5px dashed ${P.ring}`,
      background: "transparent",
      color: P.muted,
      fontFamily: "inherit",
      fontSize: 13,
      cursor: "pointer",
    },
    taskItem: (done) => ({
      display: "flex",
      alignItems: "center",
      gap: 10,
      padding: "0.65rem 0.75rem",
      borderRadius: 12,
      background: done ? P.ring + "44" : P.bg,
      marginBottom: 6,
      transition: "background 0.2s",
    }),
    check: (done) => ({
      width: 20,
      height: 20,
      borderRadius: "50%",
      border: `2px solid ${done ? P.accent : P.ring}`,
      background: done ? P.accent : "transparent",
      cursor: "pointer",
      flexShrink: 0,
      display: "flex",
      alignItems: "center",
      justifyContent: "center",
      transition: "all 0.18s",
    }),
    taskText: (done) => ({
      flex: 1,
      fontSize: 15,
      color: done ? P.muted : P.text,
      textDecoration: done ? "line-through" : "none",
      letterSpacing: "0.01em",
    }),
    pomBadge: {
      fontSize: 11,
      color: P.accent,
      background: P.ring + "88",
      borderRadius: 8,
      padding: "2px 6px",
      cursor: "pointer",
      flexShrink: 0,
    },
    delBtn: {
      background: "none",
      border: "none",
      color: P.ring,
      cursor: "pointer",
      fontSize: 16,
      lineHeight: 1,
      padding: "0 2px",
      flexShrink: 0,
    },
    addRow: {
      display: "flex",
      gap: 8,
      marginTop: "1rem",
    },
    input: {
      flex: 1,
      padding: "0.55rem 0.9rem",
      borderRadius: 12,
      border: `1.5px solid ${P.ring}`,
      background: P.bg,
      color: P.text,
      fontFamily: "inherit",
      fontSize: 15,
      outline: "none",
    },
    addBtn: {
      padding: "0.55rem 1.1rem",
      borderRadius: 12,
      border: "none",
      background: P.accent,
      color: "#fff",
      fontFamily: "inherit",
      fontSize: 15,
      cursor: "pointer",
    },
    statsRow: {
      display: "flex",
      gap: 12,
      marginBottom: "1rem",
    },
    statCard: {
      flex: 1,
      background: P.bg,
      borderRadius: 14,
      padding: "0.75rem",
      textAlign: "center",
    },
    statNum: {
      fontSize: 24,
      fontWeight: 500,
      color: P.accent,
    },
    statLbl: {
      fontSize: 12,
      color: P.muted,
      letterSpacing: "0.04em",
      textTransform: "uppercase",
    },
    progressBar: {
      height: 4,
      borderRadius: 4,
      background: P.ring,
      margin: "0.5rem 0 1rem",
      overflow: "hidden",
    },
    progressFill: {
      height: "100%",
      borderRadius: 4,
      background: P.accent,
      width: totalTasks > 0 ? `${(doneTasks / totalTasks) * 100}%` : "0%",
      transition: "width 0.4s",
    },
  };

  return (
    <div style={styles.app}>
      <link href="https://fonts.googleapis.com/css2?family=Crimson+Pro:ital,wght@0,300;0,400;0,600;1,400&display=swap" rel="stylesheet" />
      <div style={styles.header}>
        <h1 style={styles.title}>✿ pomo flow</h1>
        <div style={styles.palettePicker}>
          {Object.keys(PALETTES).map(k => (
            <div key={k} style={styles.dot(k)} onClick={() => setPalette(k)} title={k} />
          ))}
        </div>
      </div>

      <div style={styles.card}>
        <div style={styles.tabs}>
          <button style={styles.tab(activeTab === "timer")} onClick={() => setActiveTab("timer")}>Timer</button>
          <button style={styles.tab(activeTab === "tasks")} onClick={() => setActiveTab("tasks")}>Tasks</button>
          <button style={styles.tab(activeTab === "stats")} onClick={() => setActiveTab("stats")}>Overview</button>
        </div>

        {activeTab === "timer" && (
          <>
            <div style={styles.modeRow}>
              {Object.entries(MODES).map(([k, v]) => (
                <button key={k} style={styles.modeBtn(mode === k, k)} onClick={() => switchMode(k)}>{v.label}</button>
              ))}
            </div>

            <div style={styles.timerWrap}>
              <svg width={210} height={210} viewBox="0 0 210 210">
                <circle cx={105} cy={105} r={r} fill="none" stroke={P.ring} strokeWidth={10} />
                <circle
                  cx={105} cy={105} r={r}
                  fill="none"
                  stroke={MODES[mode].color}
                  strokeWidth={10}
                  strokeLinecap="round"
                  strokeDasharray={`${dash} ${circ}`}
                  transform="rotate(-90 105 105)"
                  style={{ transition: "stroke-dasharray 0.5s linear" }}
                />
              </svg>
              <div style={styles.timerTime}>
                <div style={styles.timeNum}>{fmt(timeLeft)}</div>
                <div style={styles.modeLbl}>{MODES[mode].label}</div>
              </div>
            </div>

            <div style={styles.btnRow}>
              <button style={styles.mainBtn} onClick={() => setRunning(r => !r)}>
                {running ? "Pause" : "Start"}
              </button>
              <button style={styles.resetBtn} onClick={resetTimer}>↺</button>
            </div>

            <div style={styles.sessionRow}>
              {Array.from({ length: Math.max(4, sessions + 1) }, (_, i) => (
                <div key={i} style={styles.tomato(i < sessions)} />
              ))}
            </div>
            <div style={styles.quoteBox}>{quote}</div>
          </>
        )}

        {activeTab === "tasks" && (
          <>
            <div style={styles.listTabs}>
              {lists.map(l => (
                <button key={l.id} style={styles.listTab(activeList === l.id)} onClick={() => setActiveList(l.id)}>
                  {editingList === l.id ? (
                    <input
                      value={editName}
                      onChange={e => setEditName(e.target.value)}
                      onBlur={saveListName}
                      onKeyDown={e => e.key === "Enter" && saveListName()}
                      style={{ width: 80, fontFamily: "inherit", fontSize: 13, border: "none", background: "transparent", color: "inherit", outline: "none" }}
                      autoFocus
                      onClick={e => e.stopPropagation()}
                    />
                  ) : (
                    <span onDoubleClick={() => { setEditingList(l.id); setEditName(l.name); }}>{l.name}</span>
                  )}
                  <span style={{ opacity: 0.6, fontSize: 11 }}>
                    {l.tasks.filter(t => t.done).length}/{l.tasks.length}
                  </span>
                </button>
              ))}
              {showAddList ? (
                <input
                  value={newListName}
                  onChange={e => setNewListName(e.target.value)}
                  onKeyDown={e => { if (e.key === "Enter") addList(); if (e.key === "Escape") setShowAddList(false); }}
                  placeholder="List name…"
                  style={{ ...styles.input, flex: "none", width: 110, padding: "3px 8px", fontSize: 13 }}
                  autoFocus
                  onBlur={() => { if (!newListName.trim()) setShowAddList(false); }}
                />
              ) : (
                <button style={styles.addListBtn} onClick={() => setShowAddList(true)}>+ list</button>
              )}
            </div>

            {currentList && (
              <>
                {currentList.tasks.length === 0 && (
                  <div style={{ textAlign: "center", color: P.muted, fontStyle: "italic", padding: "1.5rem 0", fontSize: 14 }}>
                    No tasks yet — add one below ✦
                  </div>
                )}
                {currentList.tasks.map(task => (
                  <div key={task.id} style={styles.taskItem(task.done)}>
                    <div style={styles.check(task.done)} onClick={() => toggleTask(task.id)}>
                      {task.done && <span style={{ color: "#fff", fontSize: 12, lineHeight: 1 }}>✓</span>}
                    </div>
                    <span style={styles.taskText(task.done)}>{task.text}</span>
                    <span style={styles.pomBadge} title="Log a pomodoro" onClick={() => addPomToTask(task.id)}>
                      🍅 {task.poms}
                    </span>
                    <button style={styles.delBtn} onClick={() => deleteTask(task.id)}>×</button>
                  </div>
                ))}
                <div style={styles.addRow}>
                  <input
                    style={styles.input}
                    placeholder="Add a task…"
                    value={newTask}
                    onChange={e => setNewTask(e.target.value)}
                    onKeyDown={e => e.key === "Enter" && addTask()}
                  />
                  <button style={styles.addBtn} onClick={addTask}>+</button>
                </div>
                <div style={{ marginTop: "0.75rem", display: "flex", justifyContent: "flex-end" }}>
                  <button
                    style={{ ...styles.delBtn, fontSize: 13, color: P.muted }}
                    onClick={() => { if (window.confirm("Delete this list?")) deleteList(currentList.id); }}
                  >
                    Delete list
                  </button>
                </div>
              </>
            )}
          </>
        )}

        {activeTab === "stats" && (
          <>
            <div style={styles.statsRow}>
              <div style={styles.statCard}>
                <div style={styles.statNum}>{sessions}</div>
                <div style={styles.statLbl}>Sessions</div>
              </div>
              <div style={styles.statCard}>
                <div style={styles.statNum}>{doneTasks}</div>
                <div style={styles.statLbl}>Done</div>
              </div>
              <div style={styles.statCard}>
                <div style={styles.statNum}>{totalTasks - doneTasks}</div>
                <div style={styles.statLbl}>Remaining</div>
              </div>
            </div>

            <div style={{ fontSize: 13, color: P.muted, marginBottom: 4, letterSpacing: "0.04em" }}>
              Overall progress — {totalTasks > 0 ? Math.round((doneTasks / totalTasks) * 100) : 0}%
            </div>
            <div style={styles.progressBar}><div style={styles.progressFill} /></div>

            {lists.map(l => {
              const done = l.tasks.filter(t => t.done).length;
              const pct = l.tasks.length > 0 ? (done / l.tasks.length) * 100 : 0;
              const totalPoms = l.tasks.reduce((a, t) => a + t.poms, 0);
              return (
                <div key={l.id} style={{ marginBottom: "1rem" }}>
                  <div style={{ display: "flex", justifyContent: "space-between", fontSize: 14, marginBottom: 4 }}>
                    <span style={{ color: P.text }}>{l.name}</span>
                    <span style={{ color: P.muted }}>{done}/{l.tasks.length} · 🍅 {totalPoms}</span>
                  </div>
                  <div style={{ height: 6, borderRadius: 4, background: P.ring, overflow: "hidden" }}>
                    <div style={{ height: "100%", borderRadius: 4, background: P.accent, width: `${pct}%`, transition: "width 0.4s" }} />
                  </div>
                </div>
              );
            })}

            <div style={{ ...styles.quoteBox, marginTop: "1.25rem" }}>
              {sessions > 0
                ? `You've completed ${sessions} pomodoro${sessions > 1 ? "s" : ""} today — beautiful work ✦`
                : "Start your first session when you're ready ✦"}
            </div>
          </>
        )}
      </div>
    </div>
  );
}
