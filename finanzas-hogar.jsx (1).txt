import React, { useState, useEffect, useCallback, useRef } from "react";
import { Plus, Trash2, ChevronDown, ChevronRight, Wallet, AlertTriangle, Coins, X, Check, PiggyBank } from "lucide-react";

// ---------- helpers ----------
const fmt = (n) =>
  new Intl.NumberFormat("es-CL", { style: "currency", currency: "CLP", maximumFractionDigits: 0 }).format(
    Math.round(n || 0)
  );

const monthKey = (d = new Date()) => `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`;

const monthLabel = (key) => {
  const [y, m] = key.split("-").map(Number);
  const d = new Date(y, m - 1, 1);
  const s = d.toLocaleDateString("es-CL", { month: "long", year: "numeric" });
  return s.charAt(0).toUpperCase() + s.slice(1);
};

const uid = () => Math.random().toString(36).slice(2, 10);

const PALETTE = [
  { base: "#C9A66B", dark: "#9C7C46", ink: "#3A2E1A" }, // kraft
  { base: "#8FA88C", dark: "#5E7A5B", ink: "#22301F" }, // sage
  { base: "#C08768", dark: "#95603F", ink: "#341F12" }, // terracotta clay
  { base: "#9BA6C4", dark: "#6C7AA0", ink: "#1E2436" }, // slate blue
  { base: "#D3B24B", dark: "#A8891E", ink: "#3A2E05" }, // mustard
  { base: "#B08CB0", dark: "#836083", ink: "#2E1F2E" }, // plum
];

const emptyMonth = () => ({ sueldo: 0, categorias: [] });

// ---------- storage hooks ----------
async function loadMonth(key) {
  try {
    const r = await window.storage.get(`mes:${key}`);
    return r ? JSON.parse(r.value) : null;
  } catch {
    return null;
  }
}
async function saveMonth(key, data) {
  try {
    await window.storage.set(`mes:${key}`, JSON.stringify(data));
  } catch (e) {
    console.error("Error al guardar", e);
  }
}
async function loadMonthList() {
  try {
    const r = await window.storage.get("lista-meses");
    return r ? JSON.parse(r.value) : [];
  } catch {
    return [];
  }
}
async function saveMonthList(list) {
  try {
    await window.storage.set("lista-meses", JSON.stringify(list));
  } catch (e) {
    console.error("Error al guardar lista de meses", e);
  }
}

// ---------- main ----------
export default function App() {
  const [ready, setReady] = useState(false);
  const [monthsAvail, setMonthsAvail] = useState([]);
  const [currentKey, setCurrentKey] = useState(monthKey());
  const [data, setData] = useState(null); // null = not decided yet (need copy prompt)
  const [showCopyPrompt, setShowCopyPrompt] = useState(false);
  const [prevKeyForCopy, setPrevKeyForCopy] = useState(null);
  const [expanded, setExpanded] = useState(null);
  const [showAddCat, setShowAddCat] = useState(false);
  const [saveState, setSaveState] = useState("idle"); // idle | saving | saved
  const saveTimer = useRef(null);

  // initial load
  useEffect(() => {
    (async () => {
      const list = await loadMonthList();
      setMonthsAvail(list);
      const key = monthKey();
      const existing = await loadMonth(key);
      if (existing) {
        setData(existing);
      } else if (list.length > 0) {
        const lastKey = list[list.length - 1];
        setPrevKeyForCopy(lastKey);
        setShowCopyPrompt(true);
      } else {
        setData(emptyMonth());
      }
      setReady(true);
    })();
  }, []);

  const persist = useCallback(
    (key, next) => {
      setData(next);
      setSaveState("saving");
      if (saveTimer.current) clearTimeout(saveTimer.current);
      saveTimer.current = setTimeout(async () => {
        await saveMonth(key, next);
        if (!monthsAvail.includes(key)) {
          const list = [...monthsAvail, key].sort();
          setMonthsAvail(list);
          await saveMonthList(list);
        }
        setSaveState("saved");
      }, 350);
    },
    [monthsAvail]
  );

  async function switchMonth(key) {
    setExpanded(null);
    setReady(false);
    setCurrentKey(key);
    const existing = await loadMonth(key);
    if (existing) {
      setData(existing);
      setShowCopyPrompt(false);
    } else {
      const list = await loadMonthList();
      const candidates = list.filter((k) => k < key);
      if (candidates.length > 0) {
        setPrevKeyForCopy(candidates[candidates.length - 1]);
        setShowCopyPrompt(true);
        setData(null);
      } else {
        setData(emptyMonth());
      }
    }
    setReady(true);
  }

  function startBlank() {
    setShowCopyPrompt(false);
    persist(currentKey, emptyMonth());
  }

  async function copyFromPrev() {
    const prev = await loadMonth(prevKeyForCopy);
    if (!prev) return startBlank();
    const next = {
      sueldo: prev.sueldo,
      categorias: prev.categorias.map((c) => ({
        ...c,
        id: uid(),
        subcategorias: c.subcategorias.map((s) => ({ ...s, id: uid() })),
        items: [], // gastos no se copian, solo estructura y presupuesto
      })),
    };
    setShowCopyPrompt(false);
    persist(currentKey, next);
  }

  if (!ready || (data === null && !showCopyPrompt)) {
    return (
      <Shell>
        <div className="w-full h-64 flex items-center justify-center text-[#8C7A5B] font-mono text-sm">
          Cargando…
        </div>
      </Shell>
    );
  }

  if (showCopyPrompt) {
    return (
      <Shell>
        <div className="max-w-md mx-auto mt-8 sm:mt-16 bg-[#FAF6EE] border border-[#D9CBB0] rounded-2xl p-5 sm:p-7 shadow-sm">
          <PiggyBank className="w-8 h-8 text-[#9C7C46] mb-3" strokeWidth={1.5} />
          <h2 className="font-serif text-xl text-[#2B2620] mb-2">Nuevo mes: {monthLabel(currentKey)}</h2>
          <p className="text-sm text-[#5C5142] mb-5 leading-relaxed">
            No hay datos para este mes todavía. ¿Quieres partir copiando los sobres (categorías y porcentajes) del
            mes de {monthLabel(prevKeyForCopy)}? Los gastos no se copian, solo la estructura.
          </p>
          <div className="flex flex-col sm:flex-row gap-2">
            <button
              onClick={copyFromPrev}
              className="flex-1 bg-[#4C6444] text-white text-sm rounded-lg py-2.5 font-medium hover:bg-[#3D5237] active:bg-[#334529] transition-colors"
            >
              Copiar estructura
            </button>
            <button
              onClick={startBlank}
              className="flex-1 border border-[#D9CBB0] text-[#5C5142] text-sm rounded-lg py-2.5 font-medium hover:bg-[#F1E9D8] active:bg-[#EDE3CD] transition-colors"
            >
              Empezar en blanco
            </button>
          </div>
        </div>
      </Shell>
    );
  }

  const totalPercent = data.categorias.reduce((a, c) => a + Number(c.percent || 0), 0);
  const asignado = (data.sueldo * totalPercent) / 100;
  const gastadoTotal = data.categorias.reduce(
    (a, c) => a + c.items.reduce((s, it) => s + Number(it.amount || 0), 0),
    0
  );
  const disponible = data.sueldo - gastadoTotal;

  return (
    <Shell>
      <Header
        currentKey={currentKey}
        monthsAvail={monthsAvail}
        onSwitch={switchMonth}
        saveState={saveState}
      />

      <SalaryCard
        sueldo={data.sueldo}
        onChange={(v) => persist(currentKey, { ...data, sueldo: v })}
      />

      <SummaryBar
        sueldo={data.sueldo}
        totalPercent={totalPercent}
        asignado={asignado}
        gastadoTotal={gastadoTotal}
        disponible={disponible}
      />

      <div className="flex flex-wrap items-center justify-between gap-3 mt-8 mb-4">
        <h2 className="font-serif text-lg text-[#2B2620]">Sobres</h2>
        <button
          onClick={() => setShowAddCat(true)}
          className="flex items-center gap-1.5 text-sm bg-[#2B2620] text-[#FAF6EE] rounded-full pl-3 pr-4 py-2.5 hover:bg-[#443C2E] active:bg-[#1E1A13] transition-colors"
        >
          <Plus className="w-4 h-4" /> Nuevo sobre
        </button>
      </div>

      {data.categorias.length === 0 && (
        <div className="border border-dashed border-[#D9CBB0] rounded-xl p-6 sm:p-8 text-center text-[#8C7A5B] text-sm">
          Todavía no hay sobres. Crea uno para empezar a repartir el sueldo.
        </div>
      )}

      <div className="grid gap-3 sm:gap-4 grid-cols-1 sm:grid-cols-2">
        {data.categorias.map((cat, i) => (
          <Envelope
            key={cat.id}
            cat={cat}
            color={PALETTE[i % PALETTE.length]}
            sueldo={data.sueldo}
            expanded={expanded === cat.id}
            onToggle={() => setExpanded(expanded === cat.id ? null : cat.id)}
            onUpdate={(nextCat) =>
              persist(currentKey, {
                ...data,
                categorias: data.categorias.map((c) => (c.id === cat.id ? nextCat : c)),
              })
            }
            onDelete={() =>
              persist(currentKey, {
                ...data,
                categorias: data.categorias.filter((c) => c.id !== cat.id),
              })
            }
          />
        ))}
      </div>

      {showAddCat && (
        <AddCategoryModal
          restante={100 - totalPercent}
          onClose={() => setShowAddCat(false)}
          onAdd={(name, percent) => {
            persist(currentKey, {
              ...data,
              categorias: [...data.categorias, { id: uid(), name, percent, subcategorias: [], items: [] }],
            });
            setShowAddCat(false);
          }}
        />
      )}
    </Shell>
  );
}

// ---------- shell / layout ----------
function Shell({ children }) {
  return (
    <div className="min-h-screen w-full bg-[#FAF6EE] text-[#2B2620] overflow-x-hidden" style={{ fontFamily: "'IBM Plex Sans', sans-serif" }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,600&family=IBM+Plex+Sans:wght@400;500;600&family=IBM+Plex+Mono:wght@400;500;600&display=swap');
        .font-serif { font-family: 'Fraunces', serif; }
        .font-mono { font-family: 'IBM Plex Mono', monospace; }
        *, *::before, *::after { box-sizing: border-box; }
        input, select, textarea, button { font-size: 16px; }
        input[type="number"] { -moz-appearance: textfield; }
        input[type="number"]::-webkit-outer-spin-button,
        input[type="number"]::-webkit-inner-spin-button {
          -webkit-appearance: none;
          margin: 0;
        }
        button, a { touch-action: manipulation; -webkit-tap-highlight-color: transparent; }
        input:focus, select:focus, textarea:focus, button:focus-visible {
          outline: 2px solid #9C7C46;
          outline-offset: 1px;
        }
        @media (prefers-reduced-motion: reduce) {
          * { animation: none !important; transition: none !important; }
        }
      `}</style>
      <div className="max-w-3xl mx-auto px-4 sm:px-5 py-6 sm:py-8 pb-20">{children}</div>
    </div>
  );
}

function Header({ currentKey, monthsAvail, onSwitch, saveState }) {
  const idx = monthsAvail.indexOf(currentKey);
  const hasPrev = idx > 0 || !monthsAvail.includes(currentKey);
  const sorted = [...new Set([...monthsAvail, currentKey])].sort();
  const pos = sorted.indexOf(currentKey);

  const go = (dir) => {
    const [y, m] = currentKey.split("-").map(Number);
    const d = new Date(y, m - 1 + dir, 1);
    onSwitch(monthKey(d));
  };

  return (
    <div className="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-3 mb-6">
      <div className="flex items-center gap-2.5 min-w-0">
        <Wallet className="w-6 h-6 shrink-0 text-[#9C7C46]" strokeWidth={1.5} />
        <h1 className="font-serif text-xl sm:text-2xl text-[#2B2620] truncate">Finanzas de casa</h1>
      </div>
      <div className="flex items-center justify-between sm:justify-end gap-1 text-sm">
        <button onClick={() => go(-1)} className="p-2.5 -m-1 rounded-full hover:bg-[#F1E9D8] active:bg-[#EDE3CD] text-[#8C7A5B]" aria-label="Mes anterior">
          <ChevronRight className="w-4 h-4 rotate-180" />
        </button>
        <span className="font-mono text-[#5C5142] min-w-[8.5rem] text-center text-sm sm:text-base">{monthLabel(currentKey)}</span>
        <button onClick={() => go(1)} className="p-2.5 -m-1 rounded-full hover:bg-[#F1E9D8] active:bg-[#EDE3CD] text-[#8C7A5B]" aria-label="Mes siguiente">
          <ChevronRight className="w-4 h-4" />
        </button>
      </div>
    </div>
  );
}

function SalaryCard({ sueldo, onChange }) {
  const [editing, setEditing] = useState(false);
  const [val, setVal] = useState(sueldo || "");

  useEffect(() => setVal(sueldo || ""), [sueldo]);

  return (
    <div className="bg-[#2B2620] text-[#FAF6EE] rounded-2xl px-5 sm:px-6 py-5 flex flex-wrap items-center justify-between gap-3">
      <div className="min-w-0 flex-1">
        <div className="text-xs uppercase tracking-wide text-[#C9BBA0] mb-1">Sueldo líquido del mes</div>
        {editing ? (
          <div className="flex items-center gap-2">
            <input
              autoFocus
              type="number"
              inputMode="numeric"
              value={val}
              onChange={(e) => setVal(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === "Enter") {
                  onChange(Number(val) || 0);
                  setEditing(false);
                }
              }}
              className="font-mono text-xl sm:text-2xl bg-transparent border-b border-[#9C7C46] outline-none w-full max-w-[11rem] text-[#FAF6EE]"
            />
            <button
              onClick={() => {
                onChange(Number(val) || 0);
                setEditing(false);
              }}
              className="p-2 bg-[#4C6444] rounded-full shrink-0"
              aria-label="Guardar sueldo"
            >
              <Check className="w-4 h-4" />
            </button>
          </div>
        ) : (
          <div className="font-mono text-xl sm:text-2xl truncate">{fmt(sueldo)}</div>
        )}
      </div>
      {!editing && (
        <button
          onClick={() => setEditing(true)}
          className="text-xs border border-[#5C5142] rounded-full px-3 py-2 hover:bg-[#3A3428] active:bg-[#443C2E] transition-colors shrink-0"
        >
          Editar
        </button>
      )}
    </div>
  );
}

function SummaryBar({ sueldo, totalPercent, asignado, gastadoTotal, disponible }) {
  const over = totalPercent > 100;
  return (
    <div className="grid grid-cols-3 gap-2 sm:gap-3 mt-4">
      <Stat label="Asignado" value={`${totalPercent}%`} sub={fmt(asignado)} warn={over} />
      <Stat label="Gastado" value={fmt(gastadoTotal)} sub={sueldo ? `${Math.round((gastadoTotal / sueldo) * 100)}% del sueldo` : ""} />
      <Stat label="Disponible" value={fmt(disponible)} warn={disponible < 0} />
    </div>
  );
}

function Stat({ label, value, sub, warn }) {
  return (
    <div className="bg-[#F1E9D8] rounded-xl px-2.5 sm:px-4 py-2.5 sm:py-3 min-w-0">
      <div className="text-[10px] sm:text-[11px] uppercase tracking-wide text-[#8C7A5B] mb-0.5 truncate">{label}</div>
      <div className={`font-mono text-sm sm:text-lg truncate ${warn ? "text-[#A63D40]" : "text-[#2B2620]"}`}>{value}</div>
      {sub && <div className="text-[10px] sm:text-xs text-[#8C7A5B] mt-0.5 font-mono truncate">{sub}</div>}
    </div>
  );
}

// ---------- envelope (category) ----------
function Envelope({ cat, color, sueldo, expanded, onToggle, onUpdate, onDelete }) {
  const budget = (sueldo * Number(cat.percent || 0)) / 100;
  const spent = cat.items.reduce((s, it) => s + Number(it.amount || 0), 0);
  const remaining = budget - spent;
  const pct = budget > 0 ? Math.min(100, (spent / budget) * 100) : 0;
  const over = spent > budget;

  const [addingItem, setAddingItem] = useState(false);
  const [addingSub, setAddingSub] = useState(false);

  return (
    <div
      className="relative rounded-2xl overflow-hidden border transition-shadow hover:shadow-md"
      style={{ borderColor: color.dark + "55", background: "#FAF6EE" }}
    >
      {/* flap */}
      <div
        className="h-10 relative cursor-pointer select-none"
        style={{
          background: color.base,
          clipPath: "polygon(0 0, 50% 70%, 100% 0, 100% 100%, 0 100%)",
        }}
        onClick={onToggle}
      />
      <div className="px-4 pb-4 -mt-1">
        <div className="flex items-start justify-between gap-2">
          <div className="cursor-pointer flex-1 min-w-0" onClick={onToggle}>
            <div className="flex items-center flex-wrap gap-x-1.5 gap-y-0.5">
              <span className="font-serif text-base break-words" style={{ color: color.ink }}>
                {cat.name}
              </span>
              <span className="font-mono text-xs text-[#8C7A5B] shrink-0">{cat.percent}%</span>
            </div>
            <div className="font-mono text-sm text-[#5C5142] mt-0.5 truncate">{fmt(budget)} presupuestado</div>
          </div>
          <button
            onClick={onDelete}
            className="text-[#B8A98A] hover:text-[#A63D40] active:text-[#A63D40] p-2 -m-1 shrink-0"
            aria-label="Eliminar sobre"
          >
            <Trash2 className="w-4 h-4" />
          </button>
        </div>

        <div className="mt-3 h-2 rounded-full bg-[#EDE3CD] overflow-hidden">
          <div
            className="h-full rounded-full"
            style={{ width: `${pct}%`, background: over ? "#A63D40" : color.dark }}
          />
        </div>
        <div className="flex flex-wrap justify-between gap-x-2 mt-1 text-xs font-mono">
          <span className={over ? "text-[#A63D40]" : "text-[#8C7A5B]"}>{fmt(spent)} gastado</span>
          <span className={remaining < 0 ? "text-[#A63D40] flex items-center gap-1" : "text-[#5C5142]"}>
            {remaining < 0 && <AlertTriangle className="w-3 h-3 shrink-0" />}
            {fmt(remaining)} {remaining < 0 ? "en rojo" : "disponible"}
          </span>
        </div>

        <button
          onClick={onToggle}
          className="mt-3 -ml-2 px-2 py-1.5 text-xs text-[#8C7A5B] flex items-center gap-1 hover:text-[#5C5142] active:text-[#2B2620]"
        >
          {expanded ? <ChevronDown className="w-3.5 h-3.5" /> : <ChevronRight className="w-3.5 h-3.5" />}
          {expanded ? "Ocultar detalle" : `Ver detalle (${cat.items.length})`}
        </button>

        {expanded && (
          <div className="mt-3 border-t border-[#EDE3CD] pt-3">
            <ItemList cat={cat} onUpdate={onUpdate} />

            <div className="flex flex-wrap gap-2 mt-3">
              <button
                onClick={() => setAddingItem(true)}
                className="text-xs flex items-center gap-1 bg-[#2B2620] text-[#FAF6EE] rounded-full px-3 py-2 hover:bg-[#443C2E] active:bg-[#1E1A13]"
              >
                <Plus className="w-3.5 h-3.5" /> Gasto
              </button>
              <button
                onClick={() => setAddingSub(true)}
                className="text-xs flex items-center gap-1 border border-[#D9CBB0] text-[#5C5142] rounded-full px-3 py-2 hover:bg-[#F1E9D8] active:bg-[#EDE3CD]"
              >
                <Plus className="w-3.5 h-3.5" /> Subcategoría
              </button>
            </div>

            {addingItem && (
              <AddItemForm
                subcategorias={cat.subcategorias}
                onCancel={() => setAddingItem(false)}
                onAdd={(item) => {
                  onUpdate({ ...cat, items: [...cat.items, { id: uid(), ...item }] });
                  setAddingItem(false);
                }}
              />
            )}
            {addingSub && (
              <AddSubForm
                onCancel={() => setAddingSub(false)}
                onAdd={(name) => {
                  onUpdate({ ...cat, subcategorias: [...cat.subcategorias, { id: uid(), name }] });
                  setAddingSub(false);
                }}
              />
            )}
          </div>
        )}
      </div>
    </div>
  );
}

function ItemList({ cat, onUpdate }) {
  if (cat.items.length === 0 && cat.subcategorias.length === 0) {
    return <div className="text-xs text-[#B8A98A] italic">Sin gastos todavía.</div>;
  }

  const bySub = (subId) => cat.items.filter((it) => (it.subId || null) === subId);
  const removeItem = (id) => onUpdate({ ...cat, items: cat.items.filter((it) => it.id !== id) });

  const Row = ({ it }) => (
    <div className="flex items-center justify-between gap-2 text-sm py-1.5">
      <span className="text-[#5C5142] break-words min-w-0">{it.desc}</span>
      <div className="flex items-center gap-1 shrink-0">
        <span className="font-mono text-[#2B2620]">{fmt(it.amount)}</span>
        <button
          onClick={() => removeItem(it.id)}
          className="text-[#C9BBA0] hover:text-[#A63D40] active:text-[#A63D40] p-1.5"
          aria-label="Eliminar gasto"
        >
          <X className="w-3.5 h-3.5" />
        </button>
      </div>
    </div>
  );

  return (
    <div className="space-y-2">
      {cat.subcategorias.map((sub) => (
        <div key={sub.id}>
          <div className="text-xs font-medium text-[#8C7A5B] uppercase tracking-wide mb-0.5">{sub.name}</div>
          {bySub(sub.id).length === 0 ? (
            <div className="text-xs text-[#B8A98A] italic pl-1">Sin gastos.</div>
          ) : (
            bySub(sub.id).map((it) => <Row key={it.id} it={it} />)
          )}
        </div>
      ))}
      {bySub(null).length > 0 && (
        <div>
          {cat.subcategorias.length > 0 && (
            <div className="text-xs font-medium text-[#8C7A5B] uppercase tracking-wide mb-0.5">Directo</div>
          )}
          {bySub(null).map((it) => (
            <Row key={it.id} it={it} />
          ))}
        </div>
      )}
    </div>
  );
}

function AddItemForm({ subcategorias, onAdd, onCancel }) {
  const [desc, setDesc] = useState("");
  const [amount, setAmount] = useState("");
  const [subId, setSubId] = useState("");

  const submit = () => {
    if (!desc.trim() || !amount) return;
    onAdd({ desc: desc.trim(), amount: Number(amount), subId: subId || null, date: new Date().toISOString() });
  };

  return (
    <div className="mt-3 bg-[#F1E9D8] rounded-xl p-3 space-y-2">
      <input
        autoFocus
        placeholder="Descripción (ej: cuenta de luz)"
        value={desc}
        onChange={(e) => setDesc(e.target.value)}
        className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none focus:border-[#9C7C46]"
      />
      <input
        type="number"
        inputMode="decimal"
        placeholder="Monto"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none focus:border-[#9C7C46] font-mono"
      />
      {subcategorias.length > 0 && (
        <select
          value={subId}
          onChange={(e) => setSubId(e.target.value)}
          className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none"
        >
          <option value="">Sin subcategoría</option>
          {subcategorias.map((s) => (
            <option key={s.id} value={s.id}>
              {s.name}
            </option>
          ))}
        </select>
      )}
      <div className="flex gap-2 pt-1">
        <button onClick={submit} className="flex-1 bg-[#4C6444] text-white text-sm rounded-lg py-2.5 hover:bg-[#3D5237] active:bg-[#334529]">
          Guardar
        </button>
        <button onClick={onCancel} className="flex-1 border border-[#D9CBB0] text-[#5C5142] text-sm rounded-lg py-2.5 hover:bg-white active:bg-[#EDE3CD]">
          Cancelar
        </button>
      </div>
    </div>
  );
}

function AddSubForm({ onAdd, onCancel }) {
  const [name, setName] = useState("");
  const submit = () => {
    if (!name.trim()) return;
    onAdd(name.trim());
  };
  return (
    <div className="mt-3 bg-[#F1E9D8] rounded-xl p-3 space-y-2">
      <input
        autoFocus
        placeholder="Nombre de la subcategoría (ej: luz)"
        value={name}
        onChange={(e) => setName(e.target.value)}
        onKeyDown={(e) => e.key === "Enter" && submit()}
        className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none focus:border-[#9C7C46]"
      />
      <div className="flex gap-2">
        <button onClick={submit} className="flex-1 bg-[#4C6444] text-white text-sm rounded-lg py-2.5 hover:bg-[#3D5237] active:bg-[#334529]">
          Guardar
        </button>
        <button onClick={onCancel} className="flex-1 border border-[#D9CBB0] text-[#5C5142] text-sm rounded-lg py-2.5 hover:bg-white active:bg-[#EDE3CD]">
          Cancelar
        </button>
      </div>
    </div>
  );
}

function AddCategoryModal({ restante, onClose, onAdd }) {
  const [name, setName] = useState("");
  const [percent, setPercent] = useState("");

  const submit = () => {
    if (!name.trim() || !percent) return;
    onAdd(name.trim(), Number(percent));
  };

  return (
    <div
      className="fixed inset-0 bg-[#2B2620]/80 backdrop-blur-[2px] flex items-end sm:items-center justify-center z-50 overflow-y-auto"
      onClick={onClose}
    >
      <div
        className="bg-[#FAF6EE] rounded-t-2xl sm:rounded-2xl p-6 w-full sm:max-w-sm shadow-xl max-h-[92vh] overflow-y-auto my-0 sm:my-8"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="flex items-center justify-between mb-4">
          <h3 className="font-serif text-lg">Nuevo sobre</h3>
          <button onClick={onClose} className="text-[#8C7A5B] p-2 -m-2" aria-label="Cerrar">
            <X className="w-4 h-4" />
          </button>
        </div>
        <div className="space-y-3">
          <div>
            <label className="text-xs text-[#8C7A5B] uppercase tracking-wide">Nombre</label>
            <input
              autoFocus
              placeholder="Ej: Deudas, Comida, Ahorro"
              value={name}
              onChange={(e) => setName(e.target.value)}
              className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none focus:border-[#9C7C46] mt-1"
            />
          </div>
          <div>
            <label className="text-xs text-[#8C7A5B] uppercase tracking-wide">Porcentaje del sueldo</label>
            <input
              type="number"
              inputMode="decimal"
              placeholder="Ej: 20"
              value={percent}
              onChange={(e) => setPercent(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && submit()}
              className="w-full text-sm bg-white rounded-lg px-3 py-2.5 border border-[#D9CBB0] outline-none focus:border-[#9C7C46] mt-1 font-mono"
            />
            <div className="text-xs text-[#8C7A5B] mt-1 font-mono">
              {restante >= 0 ? `Te queda ${restante}% sin asignar` : `Ya asignaste ${100 - restante}% (más del 100%)`}
            </div>
          </div>
        </div>
        <div className="flex gap-2 mt-5">
          <button onClick={submit} className="flex-1 bg-[#2B2620] text-[#FAF6EE] text-sm rounded-lg py-2.5 hover:bg-[#443C2E] active:bg-[#1E1A13]">
            Crear sobre
          </button>
          <button onClick={onClose} className="flex-1 border border-[#D9CBB0] text-[#5C5142] text-sm rounded-lg py-2.5 hover:bg-[#F1E9D8] active:bg-[#EDE3CD]">
            Cancelar
          </button>
        </div>
      </div>
    </div>
  );
}
