import { useState, useEffect, useCallback, useRef } from "react";

const DEFAULT_MEMBERS = [
  { id: "papa",  name: "パパ",  emoji: "👨", color: "#3b82f6", image: null },
  { id: "mama",  name: "ママ",  emoji: "👩", color: "#ec4899", image: null },
  { id: "girl1", name: "長女",  emoji: "👧", color: "#f97316", image: null },
  { id: "girl2", name: "次女",  emoji: "👧", color: "#a855f7", image: null },
];

const STORAGE_KEY  = "bento-family-v5";
const SETTINGS_KEY = "bento-settings-v5";
const POLL_MS      = 4000;
const DAYS_JP      = ["日","月","火","水","木","金","土"];

function dateKey(offset = 0) {
  const d = new Date();
  d.setDate(d.getDate() + offset);
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,"0")}-${String(d.getDate()).padStart(2,"0")}`;
}

function fmtDk(dk) {
  const [y, m, d] = dk.split("-").map(Number);
  const dt = new Date(y, m - 1, d);
  const dow = DAYS_JP[dt.getDay()];
  const isWeekend = dt.getDay() === 0 || dt.getDay() === 6;
  return {
    label: dk === dateKey(1) ? `明日（${dow}）` : `${m}/${d}（${dow}）`,
    dow, isWeekend, month: m, date: d
  };
}

function get7Days() { return Array.from({ length: 7 }, (_, i) => dateKey(i + 1)); }

async function loadShared() {
  try { const r = await window.storage.get(STORAGE_KEY, true); return r ? JSON.parse(r.value) : null; }
  catch { return null; }
}

async function saveShared(data) {
  try { await window.storage.set(STORAGE_KEY, JSON.stringify(data), true); }
  catch(e) { console.error(e); }
}

async function loadSettings() {
  try { const r = await window.storage.get(SETTINGS_KEY, false); return r ? JSON.parse(r.value) : null; }
  catch { return null; }
}

async function saveSettings(s) {
  try { await window.storage.set(SETTINGS_KEY, JSON.stringify(s), false); }
  catch(e) { console.error(e); }
}

async function requestNotifPerm() {
  if (!("Notification" in window)) return false;
  if (Notification.permission === "granted") return true;
  return (await Notification.requestPermission()) === "granted";
}

function sendNotif(title, body) {
  if (typeof Notification !== "undefined" && Notification.permission === "granted")
    new Notification(title, { body });
}

export default function BentoApp() {
  const [members,     setMembers]     = useState(DEFAULT_MEMBERS);
  const [statuses,    setStatuses]    = useState({});
  const [nocook,      setNocook]      = useState({});
  const [tab,         setTab]         = useState("main");
  const [editId,      setEditId]      = useState(null);
  const [editName,    setEditName]    = useState("");
  const [syncSt,      setSyncSt]      = useState("idle");
  const [lastSync,    setLastSync]    = useState(null);
  const [settings,    setSettings]    = useState({
    deadlineHour: 21, deadlineMin: 0,
    reminderHour: 20, reminderMin: 0,
    notifEnabled: false,
  });
  const [notifPerm, setNotifPerm] = useState(
    typeof Notification !== "undefined" ? Notification.permission : "denied"
  );
  const [weekDk,      setWeekDk]      = useState(dateKey(1));
  const [uploadingId, setUploadingId] = useState(null);

  const prevAllRef = useRef({});
  const timerRef   = useRef([]);
  const days7      = get7Days();
  const todayDk    = dateKey(1);

  useEffect(() => {
    (async () => {
      const shared = await loadShared();
      if (shared) {
        if (shared.members) setMembers(shared.members);
        setStatuses(shared.statuses ?? {});
        setNocook(shared.nocook ?? {});
        setLastSync(new Date()); setSyncSt("ok");
      }
      const cfg = await loadSettings();
      if (cfg) setSettings(s => ({ ...s, ...cfg }));
    })();
  }, []);

  useEffect(() => {
    const id = setInterval(async () => {
      setSyncSt("syncing");
      const shared = await loadShared();
      if (shared) {
        if (shared.members && shared.members.length > 0) {
          setMembers(prev => {
            const merged = prev.map(localMember => {
              const storageMember = shared.members.find(m => m.id === localMember.id);
              return storageMember ? { ...localMember, ...storageMember } : localMember;
            });
            return merged;
          });
        }
        setStatuses(prev => ({ ...prev, ...(shared.statuses ?? {}) }));
        setNocook(prev => ({ ...prev, ...(shared.nocook ?? {}) }));
        setLastSync(new Date()); setSyncSt("ok");
      } else { setSyncSt("err"); }
    }, POLL_MS);
    return () => clearInterval(id);
  }, []);

  useEffect(() => {
    days7.forEach(dk => {
      if (nocook[dk]) return;
      const st = statuses[dk] ?? {};
      const done = members.every(m => st[m.id] !== undefined && st[m.id] !== null);
      if (done && !prevAllRef.current[dk] && settings.notifEnabled) {
        const cnt = members.filter(m => st[m.id] === true).length;
        const { label } = fmtDk(dk);
        sendNotif("🍱 全員回答完了！", `${label}は${cnt > 0 ? cnt+"個必要" : "不要"}です`);
      }
      prevAllRef.current[dk] = done;
    });
  }, [statuses, nocook, members, settings.notifEnabled]);

  useEffect(() => {
    timerRef.current.forEach(clearTimeout); timerRef.current = [];
    if (!settings.notifEnabled) return;
    const schedule = (h, m, title, body) => {
      const now = new Date(), t = new Date();
      t.setHours(h, m, 0, 0);
      if (t <= now) return;
      timerRef.current.push(setTimeout(() => sendNotif(title, body), t - now));
    };
    schedule(settings.reminderHour, settings.reminderMin,
      "🍱 リマインダー", `締め切り${settings.deadlineHour}:${String(settings.deadlineMin).padStart(2,"0")}までに回答してね`);
    schedule(settings.deadlineHour, settings.deadlineMin,
      "⏰ 締め切り！", "まだ回答していないメンバーがいます");
    return () => timerRef.current.forEach(clearTimeout);
  }, [settings]);

  const answer = useCallback(async (memberId, val, dk) => {
    const newStatuses = {
      ...statuses,
      [dk]: { 
        ...(statuses[dk] ?? {}), 
        [memberId]: val 
      },
    };
    setStatuses(newStatuses);
    const shared = await loadShared() ?? { members, statuses: {}, nocook: {} };
    await saveShared({
      members: shared.members ?? members,
      statuses: newStatuses,
      nocook: shared.nocook ?? nocook,
    });
    setSyncSt("ok");
    setLastSync(new Date());
  }, [statuses, nocook, members]);

  const toggleNocook = useCallback(async (dk) => {
    const newVal = !(nocook[dk] ?? false);
    const newNocook = { ...nocook, [dk]: newVal };
    const newStatuses = newVal
      ? { ...statuses, [dk]: {} }
      : statuses;
    
    setNocook(newNocook);
    setStatuses(newStatuses);
    
    const shared = await loadShared() ?? { members, statuses: {}, nocook: {} };
    await saveShared({
      members: shared.members ?? members,
      statuses: newStatuses,
      nocook: newNocook,
    });
    setSyncSt("ok");
    setLastSync(new Date());
  }, [statuses, nocook, members]);

  const saveName = useCallback(async () => {
    if (!editName.trim()) { setEditId(null); return; }
    const nm = members.map(m => m.id === editId ? { ...m, name: editName.trim() } : m);
    setMembers(nm);
    
    const shared = await loadShared() ?? { members: [], statuses: {}, nocook: {} };
    await saveShared({
      members: nm,
      statuses: shared.statuses ?? statuses,
      nocook: shared.nocook ?? nocook,
    });
    setSyncSt("ok");
    setLastSync(new Date());
    setEditId(null);
  }, [editId, editName, members, statuses, nocook]);

  const handlePhotoUpload = useCallback(async (memberId, file) => {
    if (!file) return;
    setUploadingId(memberId);
    const reader = new FileReader();
    reader.onload = async (evt) => {
      try {
        const imgData = evt.target?.result;
        if (typeof imgData === 'string') {
          const nm = members.map(x => x.id === memberId ? { ...x, image: imgData } : x);
          setMembers(nm);
          const shared = await loadShared() ?? { members: [], statuses: {}, nocook: {} };
          await saveShared({
            members: nm,
            statuses: shared.statuses ?? statuses,
            nocook: shared.nocook ?? nocook,
          });
          setSyncSt("ok");
          setLastSync(new Date());
        }
      } catch (err) {
        console.error("Photo upload error:", err);
      } finally {
        setUploadingId(null);
      }
    };
    reader.readAsDataURL(file);
  }, [members, statuses, nocook]);

  const handlePhotoDelete = useCallback(async (memberId) => {
    const nm = members.map(x => x.id === memberId ? { ...x, image: null } : x);
    setMembers(nm);
    const shared = await loadShared() ?? { members: [], statuses: {}, nocook: {} };
    await saveShared({
      members: nm,
      statuses: shared.statuses ?? statuses,
      nocook: shared.nocook ?? nocook,
    });
    setSyncSt("ok");
    setLastSync(new Date());
  }, [members, statuses, nocook]);

  const handleSaveSettings = async (s) => { setSettings(s); await saveSettings(s); };
  const enableNotif = async () => {
    const ok = await requestNotifPerm();
    setNotifPerm(ok ? "granted" : "denied");
    if (ok) handleSaveSettings({ ...settings, notifEnabled: true });
  };

  const daySummary = (dk) => {
    if (nocook[dk]) return { icon:"🙅", text:"作れない", color:"#ef4444", bg:"#fee2e2" };
    const st   = statuses[dk] ?? {};
    const need = members.filter(m => st[m.id] === true).length;
    const ans  = members.filter(m => st[m.id] !== undefined && st[m.id] !== null).length;
    if (ans < members.length) return { icon:"…", text:`${ans}/${members.length}`, color:"#94a3b8", bg:"#f8fafc" };
    if (need === 0) return { icon:"✅", text:"不要",  color:"#22c55e", bg:"#f0fdf4" };
    return { icon:"🍱", text:`${need}個必要`, color:"#3b82f6", bg:"#eff6ff" };
  };

  const AvatarDisplay = ({ member }) => (
    member.image ? (
      <img src={member.image} style={{ width:"100%", height:"100%", borderRadius:"50%", objectFit:"cover" }} alt={member.name} />
    ) : (
      <span style={{ fontSize: member.id === "mainCard" ? 28 : 18 }}>{member.emoji}</span>
    )
  );

  const MainView = () => {
    const dk  = todayDk;
    const nc  = nocook[dk] ?? false;
    const st  = statuses[dk] ?? {};
    const fmt = fmtDk(dk);
    const need = members.filter(m => st[m.id] === true).length;
    const ans  = members.filter(m => st[m.id] !== undefined && st[m.id] !== null).length;
    const done = ans === members.length && !nc;

    return (
      <div>
        <div style={S.dateHeader}>
          <div>
            <div style={{ fontSize:13, fontWeight:700, color:"#94a3b8" }}>明日のお弁当</div>
            <div style={{ fontSize:22, fontWeight:900, color: fmt.isWeekend ? "#f97316" : "#1e293b" }}>
              {fmt.label}
            </div>
          </div>
          <div style={{ ...S.sumPill, background: daySummary(dk).bg, color: daySummary(dk).color, border: `1.5px solid ${daySummary(dk).color}44` }}>
            <span>{daySummary(dk).icon}</span>
            <span style={{ fontWeight:800 }}>{daySummary(dk).text}</span>
          </div>
        </div>

        {nc && (
          <div style={S.nocookBanner}>
            <span style={{ fontSize:32 }}>🙅‍♀️</span>
            <div style={{ flex:1 }}>
              <div style={{ fontWeight:900, fontSize:15, color:"#b91c1c" }}>この日はお弁当が作れません</div>
              <div style={{ fontSize:12, color:"#ef4444", marginTop:2 }}>各自で準備をお願いします</div>
            </div>
            <button style={S.cancelBtn} onClick={() => toggleNocook(dk)}>解除</button>
          </div>
        )}

        {done && (
          <div style={{
            ...S.doneBanner,
            background: need > 0 ? "linear-gradient(120deg,#fde68a,#fbbf24)" : "linear-gradient(120deg,#bbf7d0,#34d399)",
          }}>
            <span style={{ fontSize:22 }}>{need > 0 ? "🍱" : "✅"}</span>
            <span style={{ fontWeight:900, color: need > 0 ? "#78350f" : "#14532d", fontSize:15 }}>
              {need > 0 ? `${need}個お弁当が必要！` : "明日はお弁当不要です"}
            </span>
          </div>
        )}

        {!nc && (
          <button style={S.nocookBtn} onClick={() => toggleNocook(dk)}>
            🙅‍♀️ この日はお弁当が作れない
          </button>
        )}

        {!nc && (
          <div style={S.cardGrid}>
            {members.map((m, i) => {
              const val = st[m.id] ?? null;
              return (
                <div key={m.id} style={{
                  ...S.card,
                  borderColor: val === true ? m.color : "#e2e8f0",
                  background:  val === true
                    ? `linear-gradient(145deg,${m.color}16,${m.color}06)`
                    : val === false ? "#f8fafc" : "white",
                  animationDelay: `${i * 70}ms`,
                }}>
                  <div style={{ ...S.ava, background:`${m.color}22`, border:`2.5px solid ${m.color}55` }}>
                    <AvatarDisplay member={m} />
                  </div>

                  {editId === m.id ? (
                    <div style={S.nameEditRow}>
                      <input style={S.nameInp} value={editName} autoFocus maxLength={8}
                        onChange={e => setEditName(e.target.value)}
                        onKeyDown={e => e.key === "Enter" && saveName()} />
                      <button style={S.nameOkBtn} onClick={saveName}>✓</button>
                    </div>
                  ) : (
                    <div style={S.nameRow} onClick={() => { setEditId(m.id); setEditName(m.name); }}>
                      <span style={S.nameText}>{m.name}</span>
                      <span style={{ fontSize:11, opacity:.38 }}>✏️</span>
                    </div>
                  )}

                  <span style={{
                    ...S.chip,
                    background: val === true ? m.color : val === false ? "#94a3b8" : "#e2e8f0",
                    color: val === null ? "#94a3b8" : "white",
                  }}>
                    {val === true ? "🍱 いる" : val === false ? "✋ いらない" : "未回答"}
                  </span>

                  <div style={S.btnPair}>
                    <button style={{
                      ...S.btnYes,
                      border: `2px solid ${m.color}`,
                      background: val === true ? m.color : "white",
                      color:      val === true ? "white" : m.color,
                      boxShadow:  val === true ? `0 4px 14px ${m.color}55` : "none",
                    }} onClick={() => answer(m.id, val === true ? null : true, dk)}>
                      🍱 いる
                    </button>
                    <button style={{
                      ...S.btnNo,
                      background: val === false ? "#64748b" : "white",
                      color:      val === false ? "white" : "#64748b",
                      boxShadow:  val === false ? "0 4px 14px #64748b44" : "none",
                    }} onClick={() => answer(m.id, val === false ? null : false, dk)}>
                      ✋ いらない
                    </button>
                  </div>
                </div>
              );
            })}
          </div>
        )}

        {!nc && (
          <div style={S.prog}>
            <div style={S.progLabel}>
              <span style={{ color:"#64748b", fontSize:13, fontWeight:600 }}>
                回答済み {ans}/{members.length}
              </span>
              {done && <span style={{ color:"#10b981", fontWeight:800, fontSize:13 }}>✅ 全員完了！</span>}
            </div>
            <div style={S.progTrack}>
              <div style={{ ...S.progFill, width:`${(ans / members.length) * 100}%` }} />
            </div>
          </div>
        )}

        <button style={S.resetBtn} onClick={async () => {
          const shared = await loadShared() ?? { members, statuses:{}, nocook:{} };
          const ns = { ...shared.statuses, [dk]: {} };
          const nn = { ...(shared.nocook ?? {}), [dk]: false };
          setStatuses(ns);
          setNocook(nn);
          await saveShared({ members: shared.members ?? members, statuses: ns, nocook: nn });
        }}>🔄 リセット</button>
      </div>
    );
  };

  const WeekView = () => {
    const nc   = nocook[weekDk] ?? false;
    const st   = statuses[weekDk] ?? {};
    const fmt  = fmtDk(weekDk);
    const need = members.filter(m => st[m.id] === true).length;
    const ans  = members.filter(m => st[m.id] !== undefined && st[m.id] !== null).length;
    const done = ans === members.length && !nc;

    return (
      <div>
        <div style={S.dayTabs}>
          {days7.map(dk => {
            const { month, date, dow, isWeekend } = fmtDk(dk);
            const nc_  = nocook[dk] ?? false;
            const sum  = daySummary(dk);
            const isSel = dk === weekDk;
            return (
              <button key={dk} onClick={() => setWeekDk(dk)} style={{
                ...S.dayTab,
                background: isSel ? "#3b82f6" : nc_ ? "#fff0f0" : "white",
                color:      isSel ? "white"   : nc_ ? "#ef4444" : isWeekend ? "#f97316" : "#334155",
                border:     `2px solid ${isSel ? "#3b82f6" : nc_ ? "#fca5a5" : "#e2e8f0"}`,
                boxShadow:  isSel ? "0 4px 12px #3b82f655" : "none",
              }}>
                {dk === todayDk && <span style={{ fontSize:8, fontWeight:900, opacity:.8 }}>明日</span>}
                <span style={{ fontSize:11, fontWeight:700 }}>{month}/{date}</span>
                <span style={{ fontSize:15, fontWeight:900 }}>{dow}</span>
                <span style={{
                  fontSize:9, fontWeight:800,
                  color: isSel ? "#bfdbfe" : sum.color,
                }}>{sum.icon}{sum.text}</span>
              </button>
            );
          })}
        </div>

        <div style={S.weekDetail}>
          <div style={S.weekDetailHeader}>
            <span style={{ fontSize:17, fontWeight:900, color: fmt.isWeekend ? "#f97316" : "#1e293b" }}>
              {fmt.label}
            </span>
            <div style={{ ...S.sumPill, background: daySummary(weekDk).bg, color: daySummary(weekDk).color, border:`1.5px solid ${daySummary(weekDk).color}44` }}>
              <span>{daySummary(weekDk).icon}</span>
              <span style={{ fontWeight:800 }}>{daySummary(weekDk).text}</span>
            </div>
          </div>

          {done && (
            <div style={{
              ...S.doneBanner,
              background: need > 0 ? "linear-gradient(120deg,#fde68a,#fbbf24)" : "linear-gradient(120deg,#bbf7d0,#34d399)",
              marginBottom: 12,
            }}>
              <span style={{ fontSize:20 }}>{need > 0 ? "🍱" : "✅"}</span>
              <span style={{ fontWeight:900, color: need > 0 ? "#78350f" : "#14532d", fontSize:14 }}>
                {need > 0 ? `${need}個お弁当が必要！` : "お弁当不要です"}
              </span>
            </div>
          )}

          {nc && (
            <div style={{ ...S.nocookBanner, marginBottom:12 }}>
              <span style={{ fontSize:28 }}>🙅‍♀️</span>
              <div style={{ flex:1 }}>
                <div style={{ fontWeight:900, fontSize:14, color:"#b91c1c" }}>お弁当が作れません</div>
              </div>
              <button style={S.cancelBtn} onClick={() => toggleNocook(weekDk)}>解除</button>
            </div>
          )}

          {!nc && (
            <button style={{ ...S.nocookBtn, marginBottom:14 }} onClick={() => toggleNocook(weekDk)}>
              🙅‍♀️ この日はお弁当が作れない
            </button>
          )}

          {!nc && (
            <div style={{ display:"flex", flexDirection:"column", gap:10 }}>
              {members.map(m => {
                const val = st[m.id] ?? null;
                return (
                  <div key={m.id} style={{
                    ...S.weekMemberRow,
                    borderLeft: `4px solid ${val === true ? m.color : val === false ? "#cbd5e1" : "#e2e8f0"}`,
                    background: val === true ? `${m.color}0d` : "white",
                  }}>
                    <div style={{ ...S.miniAva, background:`${m.color}22`, border:`2px solid ${m.color}44` }}>
                      <AvatarDisplay member={m} />
                    </div>
                    <span style={{ fontWeight:800, fontSize:15, color:"#1e293b", flex:1 }}>{m.name}</span>

                    <div style={S.weekBtnGroup}>
                      <button style={{
                        ...S.weekBtn,
                        background: val === true ? m.color : "white",
                        color:      val === true ? "white" : m.color,
                        border:     `2px solid ${m.color}`,
                        boxShadow:  val === true ? `0 3px 10px ${m.color}55` : "none",
                      }} onClick={() => answer(m.id, val === true ? null : true, weekDk)}>
                        🍱 いる
                      </button>
                      <button style={{
                        ...S.weekBtn,
                        background: val === false ? "#64748b" : "white",
                        color:      val === false ? "white"   : "#64748b",
                        border:     "2px solid #cbd5e1",
                        boxShadow:  val === false ? "0 3px 10px #64748b44" : "none",
                      }} onClick={() => answer(m.id, val === false ? null : false, weekDk)}>
                        ✋ いらない
                      </button>
                    </div>
                  </div>
                );
              })}
            </div>
          )}

          {!nc && (
            <div style={{ ...S.prog, marginTop:16 }}>
              <div style={S.progLabel}>
                <span style={{ color:"#64748b", fontSize:13, fontWeight:600 }}>回答済み {ans}/{members.length}</span>
                {done && <span style={{ color:"#10b981", fontWeight:800, fontSize:13 }}>✅ 全員完了！</span>}
              </div>
              <div style={S.progTrack}>
                <div style={{ ...S.progFill, width:`${(ans / members.length) * 100}%` }} />
              </div>
            </div>
          )}

          <button style={{ ...S.resetBtn, marginTop:14 }} onClick={async () => {
            const shared = await loadShared() ?? { members, statuses:{}, nocook:{} };
            const ns = { ...shared.statuses, [weekDk]: {} };
            const nn = { ...(shared.nocook ?? {}), [weekDk]: false };
            setStatuses(ns);
            setNocook(nn);
            await saveShared({ members: shared.members ?? members, statuses: ns, nocook: nn });
          }}>🔄 この日をリセット</button>
        </div>
      </div>
    );
  };

  const SettingsView = () => (
    <div style={S.settingsBox}>
      <h2 style={S.settingsTitle}>⚙️ 設定</h2>

      <section style={S.section}>
        <h3 style={S.sTitle}>🔔 通知</h3>
        {notifPerm !== "granted"
          ? <button style={S.permBtn} onClick={enableNotif}>通知を許可する</button>
          : <label style={S.toggle}>
              <input type="checkbox" checked={settings.notifEnabled}
                onChange={e => handleSaveSettings({ ...settings, notifEnabled: e.target.checked })} />
              <span style={S.toggleLbl}>通知を有効にする</span>
            </label>}
      </section>

      <section style={S.section}>
        <h3 style={S.sTitle}>⏰ リマインダー時刻</h3>
        <p style={S.sDesc}>締め切り前にお知らせ</p>
        <TimeInput hour={settings.reminderHour} min={settings.reminderMin} color="#f97316"
          onChange={(h, m) => handleSaveSettings({ ...settings, reminderHour: h, reminderMin: m })} />
      </section>

      <section style={S.section}>
        <h3 style={S.sTitle}>🚨 締め切り時刻</h3>
        <p style={S.sDesc}>この時刻に未回答者へ通知</p>
        <TimeInput hour={settings.deadlineHour} min={settings.deadlineMin} color="#ef4444"
          onChange={(h, m) => handleSaveSettings({ ...settings, deadlineHour: h, deadlineMin: m })} />
      </section>

      <section style={S.section}>
        <h3 style={S.sTitle}>👨‍👩‍👧‍👧 メンバー設定</h3>
        <p style={S.sDesc}>名前変更・写真設定</p>
        <div style={{ display:"flex", flexDirection:"column", gap:10 }}>
          {members.map(m => (
            <div key={m.id} style={{ ...S.mRow, borderLeft:`4px solid ${m.color}`, flexDirection:"column", gap:10, alignItems:"stretch" }}>
              <div style={{ display:"flex", alignItems:"center", gap:10 }}>
                <div style={{ ...S.miniAva, width:40, height:40, background:`${m.color}22`, border:`2px solid ${m.color}44` }}>
                  <AvatarDisplay member={m} />
                </div>
                {editId === m.id ? (
                  <div style={{ display:"flex", gap:4, flex:1 }}>
                    <input style={S.nameInp} value={editName} autoFocus maxLength={8}
                      onChange={e => setEditName(e.target.value)}
                      onKeyDown={e => e.key === "Enter" && saveName()} />
                    <button style={S.nameOkBtn} onClick={saveName}>✓</button>
                  </div>
                ) : (
                  <span style={{ fontSize:14, fontWeight:800, color:"#334155", cursor:"pointer", flex:1 }}
                    onClick={() => { setEditId(m.id); setEditName(m.name); }}>
                    {m.name}<span style={{ opacity:.4, marginLeft:4 }}>✏️</span>
                  </span>
                )}
              </div>

              <div style={{ display:"flex", gap:6 }}>
                <label style={{
                  flex:1, borderRadius:10, border:"2px dashed #cbd5e1",
                  padding:"8px 12px", textAlign:"center", cursor:"pointer",
                  fontSize:12, fontWeight:800, color:"#64748b", transition:"all .2s",
                  opacity: uploadingId === m.id ? 0.5 : 1,
                  pointerEvents: uploadingId === m.id ? "none" : "auto",
                }}>
                  📷 {uploadingId === m.id ? "処理中..." : (m.image ? "変更" : "写真設定")}
                  <input type="file" accept="image/*" style={{ display:"none" }}
                    onChange={e => handlePhotoUpload(m.id, e.target.files?.[0] ?? null)} />
                </label>
                {m.image && uploadingId !== m.id && (
                  <button style={{
                    borderRadius:10, border:"2px solid #fca5a5",
                    padding:"8px 12px", background:"white", color:"#ef4444",
                    cursor:"pointer", fontSize:12, fontWeight:800, fontFamily:"inherit"
                  }} onClick={() => handlePhotoDelete(m.id)}>削除</button>
                )}
              </div>
            </div>
          ))}
        </div>
      </section>
    </div>
  );

  return (
    <div style={S.page}>
      <div style={S.blob1} /><div style={S.blob2} /><div style={S.blob3} />
      <div style={S.wrap}>

        <header style={S.header}>
          <div style={S.hLeft}>
            <span style={{ fontSize:36 }}>🍱</span>
            <h1 style={S.title}>お弁当いる？</h1>
          </div>
          <div style={S.syncChip}>
            <span style={{ ...S.dot, background: syncSt==="ok"?"#22c55e":syncSt==="syncing"?"#fbbf24":"#ef4444" }} />
            <span style={{ fontSize:10, color:"#64748b", fontWeight:700 }}>
              {syncSt==="ok"&&lastSync
                ? `${lastSync.getHours()}:${String(lastSync.getMinutes()).padStart(2,"0")} 同期`
                : syncSt==="syncing" ? "同期中" : "エラー"}
            </span>
          </div>
        </header>

        <div style={S.tabBar}>
          {[
            { key:"main",     label:"🏠 今日",   sub: fmtDk(todayDk).label },
            { key:"week",     label:"📅 1週間",  sub: "先の回答" },
            { key:"settings", label:"⚙️ 設定",   sub: "" },
          ].map(t => (
            <button key={t.key} style={{
              ...S.tabBtn,
              background: tab === t.key ? "#3b82f6" : "white",
              color:      tab === t.key ? "white"   : "#64748b",
              border:     `2px solid ${tab === t.key ? "#3b82f6" : "#e2e8f0"}`,
              boxShadow:  tab === t.key ? "0 4px 12px #3b82f655" : "none",
            }} onClick={() => setTab(t.key)}>
              <span style={{ fontSize:13, fontWeight:800 }}>{t.label}</span>
              {t.sub && <span style={{ fontSize:9, opacity:.75, fontWeight:700 }}>{t.sub}</span>}
            </button>
          ))}
        </div>

        <div style={{ animation:"fadeUp .3s ease both" }}>
          {tab === "main"     && <MainView />}
          {tab === "week"     && <WeekView />}
          {tab === "settings" && <SettingsView />}
        </div>

        <p style={S.footer}>🍀 家族みんなで共有 · リアルタイム同期</p>
      </div>

      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;700;800;900&display=swap');
        @keyframes fadeUp { from{opacity:0;transform:translateY(12px)} to{opacity:1;transform:none} }
        button:active { transform: scale(.97); }
        * { -webkit-font-smoothing: antialiased; -moz-osx-font-smoothing: grayscale; }
      `}</style>
    </div>
  );
}

function TimeInput({ hour, min, onChange, color }) {
  return (
    <div style={{ display:"flex", gap:8, alignItems:"center" }}>
      <select value={hour} style={{ ...S.sel, borderColor: color }}
        onChange={e => onChange(Number(e.target.value), min)}>
        {Array.from({length:24}, (_, i) => (
          <option key={i} value={i}>{String(i).padStart(2,"0")}</option>
        ))}
      </select>
      <span style={{ fontWeight:900, color:"#64748b" }}>:</span>
      <select value={min} style={{ ...S.sel, borderColor: color }}
        onChange={e => onChange(hour, Number(e.target.value))}>
        {[0,15,30,45].map(v => (
          <option key={v} value={v}>{String(v).padStart(2,"0")}</option>
        ))}
      </select>
    </div>
  );
}

const S = {
  page: {
    minHeight: "100vh",
    background: "linear-gradient(160deg,#fff7ed 0%,#fef9f0 50%,#fdf4ff 100%)",
    fontFamily: "'Nunito','Hiragino Maru Gothic ProN','BIZ UDPGothic',sans-serif",
    position: "relative", overflow: "hidden", padding: "14px 13px 56px",
  },
  blob1: { position:"fixed", top:-90, right:-70, width:250, height:250, borderRadius:"50%", background:"radial-gradient(circle,#fde68a55,transparent 70%)", pointerEvents:"none" },
  blob2: { position:"fixed", bottom:-70, left:-50, width:220, height:220, borderRadius:"50%", background:"radial-gradient(circle,#fbcfe855,transparent 70%)", pointerEvents:"none" },
  blob3: { position:"fixed", top:"42%", left:"58%", width:160, height:160, borderRadius:"50%", background:"radial-gradient(circle,#bbf7d044,transparent 70%)", pointerEvents:"none" },
  wrap: { maxWidth:520, margin:"0 auto", position:"relative", zIndex:1 },

  header: { display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:12 },
  hLeft:  { display:"flex", alignItems:"center", gap:10 },
  title:  { fontSize:22, fontWeight:900, color:"#1e293b", margin:0, letterSpacing:-.5 },
  syncChip: { display:"flex", alignItems:"center", gap:5, background:"white", borderRadius:20, padding:"5px 10px", boxShadow:"0 1px 6px #0001" },
  dot: { width:7, height:7, borderRadius:"50%", flexShrink:0 },

  tabBar: { display:"flex", gap:8, marginBottom:16 },
  tabBtn: {
    flex:1, borderRadius:14, padding:"9px 6px 8px", cursor:"pointer",
    fontFamily:"inherit", transition:"all .2s cubic-bezier(.34,1.56,.64,1)",
    display:"flex", flexDirection:"column", alignItems:"center", gap:2,
  },

  dateHeader: { display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:14 },
  sumPill: { display:"flex", alignItems:"center", gap:5, borderRadius:20, padding:"6px 12px", fontSize:13, fontWeight:800 },

  nocookBanner: { display:"flex", alignItems:"center", gap:12, background:"linear-gradient(120deg,#fee2e2,#fecaca)", border:"2px solid #fca5a5", borderRadius:16, padding:"13px 15px", marginBottom:12, boxShadow:"0 4px 14px #ef444422" },
  cancelBtn: { background:"white", border:"2px solid #fca5a5", color:"#ef4444", borderRadius:10, padding:"5px 12px", fontWeight:800, cursor:"pointer", fontSize:12, fontFamily:"inherit", whiteSpace:"nowrap" },
  doneBanner: { borderRadius:16, padding:"12px 16px", display:"flex", alignItems:"center", gap:10, marginBottom:12, boxShadow:"0 4px 14px #fbbf2433" },
  nocookBtn: { display:"block", width:"100%", marginBottom:12, background:"white", border:"2px dashed #fca5a5", borderRadius:14, padding:"11px 0", fontSize:13, fontWeight:800, color:"#ef4444", cursor:"pointer", fontFamily:"inherit" },

  cardGrid: { display:"grid", gridTemplateColumns:"1fr 1fr", gap:11, marginBottom:16 },
  card: {
    background:"white", borderRadius:20, border:"2.5px solid #e2e8f0",
    padding:"15px 11px 13px", display:"flex", flexDirection:"column",
    alignItems:"center", gap:8, boxShadow:"0 2px 12px #00000009",
    transition:"all .25s cubic-bezier(.34,1.56,.64,1)",
    animation:"fadeUp .35s ease both",
  },
  ava: { width:52, height:52, borderRadius:"50%", display:"flex", alignItems:"center", justifyContent:"center" },
  nameRow: { display:"flex", alignItems:"center", gap:3, cursor:"pointer" },
  nameText: { fontSize:15, fontWeight:800, color:"#1e293b" },
  nameEditRow: { display:"flex", gap:4 },
  nameInp: { width:62, padding:"2px 7px", borderRadius:8, border:"2px solid #93c5fd", fontSize:12, fontFamily:"inherit", outline:"none", textAlign:"center" },
  nameOkBtn: { background:"#3b82f6", color:"white", border:"none", borderRadius:8, padding:"2px 8px", cursor:"pointer", fontWeight:800, fontFamily:"inherit", fontSize:13 },
  chip: { fontSize:11, fontWeight:800, borderRadius:20, padding:"3px 10px", letterSpacing:.3 },
  btnPair: { display:"flex", flexDirection:"column", gap:5, width:"100%" },
  btnYes: { borderRadius:11, padding:"8px 0", fontSize:13, fontWeight:800, cursor:"pointer", transition:"all .2s", fontFamily:"inherit", width:"100%" },
  btnNo:  { borderRadius:11, padding:"8px 0", fontSize:13, fontWeight:800, cursor:"pointer", border:"2px solid #cbd5e1", transition:"all .2s", fontFamily:"inherit", width:"100%" },

  prog: { marginBottom:4 },
  progLabel: { display:"flex", justifyContent:"space-between", marginBottom:5 },
  progTrack: { height:7, background:"#e2e8f0", borderRadius:99, overflow:"hidden" },
  progFill:  { height:"100%", background:"linear-gradient(90deg,#34d399,#10b981)", borderRadius:99, transition:"width .5s cubic-bezier(.34,1.56,.64,1)" },
  resetBtn: { display:"block", margin:"12px auto 0", background:"white", border:"2px solid #e2e8f0", borderRadius:12, padding:"8px 22px", fontSize:13, fontWeight:800, color:"#64748b", cursor:"pointer", fontFamily:"inherit" },

  dayTabs: { display:"flex", gap:5, marginBottom:14, overflowX:"auto", paddingBottom:3 },
  dayTab: {
    flexShrink:0, width:60, minHeight:76, borderRadius:14,
    display:"flex", flexDirection:"column", alignItems:"center",
    justifyContent:"center", gap:1, cursor:"pointer", fontFamily:"inherit",
    transition:"all .22s cubic-bezier(.34,1.56,.64,1)", padding:"6px 2px",
  },
  weekDetail: { background:"white", borderRadius:20, padding:"16px 14px", boxShadow:"0 4px 20px #0000000c" },
  weekDetailHeader: { display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:12 },
  weekMemberRow: {
    display:"flex", alignItems:"center", gap:10,
    background:"white", borderRadius:14, padding:"11px 13px",
    boxShadow:"0 1px 6px #0000000a",
  },
  miniAva: { width:34, height:34, borderRadius:"50%", display:"flex", alignItems:"center", justifyContent:"center", flexShrink:0 },
  weekBtnGroup: { display:"flex", gap:6 },
  weekBtn: { borderRadius:10, padding:"7px 10px", fontSize:12, fontWeight:800, cursor:"pointer", transition:"all .2s", fontFamily:"inherit", whiteSpace:"nowrap" },

  settingsBox: { background:"white", borderRadius:22, padding:"18px 16px", boxShadow:"0 4px 20px #0000000c" },
  settingsTitle: { fontSize:18, fontWeight:900, color:"#1e293b", margin:"0 0 16px" },
  section: { marginBottom:20, paddingBottom:18, borderBottom:"1px solid #f1f5f9" },
  sTitle: { fontSize:14, fontWeight:800, color:"#334155", margin:"0 0 3px" },
  sDesc:  { fontSize:11, color:"#94a3b8", margin:"0 0 9px", fontWeight:600 },
  permBtn: { background:"linear-gradient(135deg,#3b82f6,#6366f1)", color:"white", border:"none", borderRadius:12, padding:"9px 18px", fontSize:13, fontWeight:800, cursor:"pointer", fontFamily:"inherit" },
  toggle:    { display:"flex", alignItems:"center", gap:9, cursor:"pointer" },
  toggleLbl: { fontSize:14, fontWeight:700, color:"#334155" },
  sel: { borderRadius:10, border:"2px solid #e2e8f0", padding:"5px 9px", fontSize:15, fontWeight:800, fontFamily:"inherit", color:"#1e293b", outline:"none", background:"white", cursor:"pointer" },
  mRow: { display:"flex", alignItems:"center", gap:10, background:"#f8fafc", borderRadius:12, padding:"9px 13px" },

  footer: { textAlign:"center", fontSize:11, color:"#cbd5e1", margin:"18px 0 0", fontWeight:700 },
};
