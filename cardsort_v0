import React, { useState, useEffect, useMemo, useCallback, useRef } from "react";
import { DndProvider, useDrag, useDrop } from "react-dnd";
import { HTML5Backend } from "react-dnd-html5-backend";
import { Plus, Trash2, Save, Upload, Download, Users, Settings, Table, Shuffle, UserPlus, Play, RefreshCcw, FolderPlus, ArrowUp, ArrowLeft, ArrowRight } from "lucide-react";

/**
 * ROCKGOTA – Card Sorting (MVP optimizado)
 * Admin + Ingreso de datos con subgrupos, flechas (↑/◀/▶/▲/▼) y mejoras de rendimiento:
 * - Escrituras a localStorage con debounce.
 * - Búsquedas O(1) de tarjetas por id usando Map memoizado.
 * - Componentes memoizados para reducir renders.
 * - Operaciones de mover tarjeta en un solo setState.
 */

// ---------- Util ----------
const genId = () => Math.random().toString(36).slice(2, 10);
const debounce = (fn: Function, ms = 250) => {
  let t: any; return (...args: any[]) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
};

// ---------- Types ----------
export type Card = { id: string; label: string; description?: string };
export type Profile = { id: string; name: string };
export type Demographics = { profileId: string | null; gender: "Male" | "Female" | "Otro" | null; age: number | null };
export type SortGroup = { id: string; name: string; cardIds: string[]; children: SortGroup[] };
export type Session = { id: string; startedAt: number; durationSec: number; demographics: Demographics; groups: SortGroup[] };
export type Study = { id: string; name: string; createdAt: number; cards: Card[]; profiles: Profile[]; sessions: Session[] };

// ---------- Storage (debounced localStorage) ----------

const LS_KEYS = { studies: "cs__studies" } as const;

function useLocalStudies(initial: Study[]) {
  const [studies, setStudies] = useState<Study[]>(() => {
    const raw = localStorage.getItem(LS_KEYS.studies);
    return raw ? (JSON.parse(raw) as Study[]) : initial;
  });
  const persist = useMemo(
    () => debounce((value: Study[]) => localStorage.setItem(LS_KEYS.studies, JSON.stringify(value)), 300),
    []
  );
  useEffect(() => { persist(studies); }, [studies, persist]);
  return [studies, setStudies] as const;
}


// ---------- Datos por defecto ----------
const defaultCards: Card[] = [
  { id: genId(), label: "Camisas" }, { id: genId(), label: "Hoodies" }, { id: genId(), label: "Sweater" }, { id: genId(), label: "Pantalones" },
  { id: genId(), label: "Pijamas" }, { id: genId(), label: "Chaquetas" }, { id: genId(), label: "Ver todo" }, { id: genId(), label: "Accesorios" },
  { id: genId(), label: "Bolsas" }, { id: genId(), label: "Sombreros" }, { id: genId(), label: "Pines" }, { id: genId(), label: "Bufanda / Pañoletas" },
  { id: genId(), label: "Giftcard" }, { id: genId(), label: "Rebajas" }, { id: genId(), label: "Regalos" }, { id: genId(), label: "Licencias" },
  { id: genId(), label: "Buscador" }, { id: genId(), label: "Iniciar Sesión" }, { id: genId(), label: "Método de pago" }, { id: genId(), label: "Envíos" },
  { id: genId(), label: "Atención al cliente" }, { id: genId(), label: "Clientes" }, { id: genId(), label: "Políticas" }, { id: genId(), label: "Política de privacidad" },
  { id: genId(), label: "PQRs" }, { id: genId(), label: "Guías de compra" }, { id: genId(), label: "Manifiesto" },
];
const defaultProfiles: Profile[] = [
  { id: genId(), name: "01_COMPRAN EN LINEA" }, { id: genId(), name: "02_SOLO MIRAN" }, { id: genId(), name: "03_NO COMPRAN EN LINEA" },
];
const defaultStudy = (name = "ROCKGOTA – Card Sorting (MVP)"): Study => ({ id: genId(), name, createdAt: Date.now(), cards: defaultCards, profiles: defaultProfiles, sessions: [] });

// ---------- Pequeños componentes UI ----------
function Section({ title, icon, children, toolbar }: { title: string; icon?: React.ReactNode; children: React.ReactNode; toolbar?: React.ReactNode }) {
  return (
    <div className="bg-white rounded-2xl shadow p-5 border border-gray-100">
      <div className="flex items-center justify-between mb-4">
        <h2 className="text-xl font-semibold flex items-center gap-2">{icon}{title}</h2>
        <div className="flex gap-2">{toolbar}</div>
      </div>
      {children}
    </div>
  );
}
const TextInput = React.memo(function TextInput({ value, onChange, placeholder, className }: { value: string; onChange: (v: string) => void; placeholder?: string; className?: string }) {
  return (
    <input className={`w-full rounded-xl border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500 ${className||""}`} value={value} onChange={(e)=>onChange(e.target.value)} placeholder={placeholder} />
  );
});
function NumberInput({ value, onChange, min=1, max=999, className }: { value: number; onChange: (v: number)=>void; min?: number; max?: number; className?: string }){
  return (
    <input type="number" min={min} max={max} value={value} onChange={(e)=>onChange(Number(e.target.value))} className={`w-24 rounded-xl border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500 ${className||""}`}/>
  );
}
function SmallButton({ onClick, children, title }: any) { return (<button title={title} onClick={onClick} className="inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-sm hover:bg-gray-50">{children}</button>); }

// ---------- DnD ----------
const ItemTypes = { CARD: "CARD" } as const;
const DraggableCard = React.memo(function DraggableCard({ card, from }: { card: Card; from: string }) {
  const [{ isDragging }, drag] = useDrag(() => ({ type: ItemTypes.CARD, item: { cardId: card.id, from }, collect: (m) => ({ isDragging: m.isDragging() }) }));
  return <div ref={drag} className={`rounded-xl border p-2 bg-white shadow-sm cursor-move select-none ${isDragging?"opacity-50":""}`}>{card.label}</div>;
});
const DropColumn = React.memo(function DropColumn({ title, onDrop, children }: { title: string; onDrop: (item: any) => void; children: React.ReactNode }) {
  const [{ isOver }, drop] = useDrop(() => ({ accept: ItemTypes.CARD, drop: (item:any)=>onDrop(item), collect: (m)=>({ isOver: m.isOver() }) }));
  return (
    <div ref={drop} className={`rounded-2xl border bg-gray-50 min-h-[120px] p-3 ${isOver?"ring-2 ring-indigo-400":""}`}>
      <div className="text-sm font-medium text-gray-600 mb-2">{title}</div>
      <div className="flex flex-col gap-2">{children}</div>
    </div>
  );
});


// ---------- App ----------
export default function App() {
  // Estudios (proyectos) aislados
  const [studies, setStudies] = useLocalStudies([defaultStudy()]);
  const [currentId, setCurrentId] = useState<string>(studies[0]?.id || "");
  const current = useMemo(() => studies.find(s => s.id === currentId) || studies[0], [studies, currentId]);
  useEffect(() => { if (!currentId && studies[0]) setCurrentId(studies[0].id); }, [currentId, studies]);

  // Mutadores del estudio activo
  const updateCurrent = useCallback((patch: Partial<Study> | ((s: Study) => Study)) => {
    if (!current) return;
    setStudies(prev =>
      prev.map(s => s.id === current.id ? (typeof patch === "function" ? (patch as any)(s) : { ...s, ...patch }) : s)
    );
  }, [setStudies, current]);

  const setCards = useCallback((updater: any) =>
    updateCurrent(s => ({ ...s, cards: typeof updater === "function" ? updater(s.cards) : updater })), [updateCurrent]);

  const setProfiles = useCallback((updater: any) =>
    updateCurrent(s => ({ ...s, profiles: typeof updater === "function" ? updater(s.profiles) : updater })), [updateCurrent]);

  const setSessions = useCallback((updater: any) =>
    updateCurrent(s => ({ ...s, sessions: typeof updater === "function" ? updater(s.sessions) : updater })), [updateCurrent]);

  // CRUD de estudios
  const createStudy = useCallback(() => {
    const s = defaultStudy("Nuevo estudio");
    setStudies(prev => [s, ...prev]);
    setCurrentId(s.id);
  }, [setStudies]);

  const duplicateStudy = useCallback(() => {
    if (!current) return;
    const copy: Study = {
      ...current,
      id: genId(),
      name: current.name + " (copia)",
      createdAt: Date.now(),
      cards: [...current.cards],
      profiles: [...current.profiles],
      sessions: [...current.sessions],
    };
    setStudies(prev => [copy, ...prev]);
    setCurrentId(copy.id);
  }, [current, setStudies]);

  const deleteStudy = useCallback(() => {
    if (!current) return;
    if (!confirm(`¿Eliminar estudio "${current.name}"?`)) return;
    setStudies(prev => prev.filter(s => s.id !== current.id));
    setCurrentId(prevId => (prevId === current.id && studies[1]) ? studies[1].id : (studies[0]?.id || ""));
  }, [current, setStudies, studies]);

  const renameStudy = useCallback((name: string) => updateCurrent({ name }), [updateCurrent]);

  // Exportar / Importar estudio actual
  const exportStudy = useCallback(() => {
    if (!current) return;
    const blob = new Blob([JSON.stringify(current, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `${current.name.replace(/\s+/g, "_")}.json`;
    a.click();
    URL.revokeObjectURL(url);
  }, [current]);

  const importStudy = useCallback((file: File) => {
    file.text().then((txt) => {
      const data = JSON.parse(txt);
      const newStudy: Study = {
        id: genId(),
        name: data.name || `Estudio importado ${new Date().toISOString().slice(0, 10)}`,
        createdAt: Date.now(),
        cards: data.cards || [],
        profiles: data.profiles || [],
        sessions: data.sessions || [],
      };
      setStudies(prev => [newStudy, ...prev]);
      setCurrentId(newStudy.id);
    });
  }, [setStudies]);

  // Pestañas por estudio
  const [tab, setTab] = useState<"cards" | "profiles" | "sessions" | "entry" | "analysis">("cards");
  const cards   = current?.cards || [];
  const profiles= current?.profiles || [];
  const sessions= current?.sessions || [];

  return (
    <DndProvider backend={HTML5Backend}>
      <div className="max-w-7xl mx-auto p-6 space-y-6">
        <header className="flex items-center justify-between">
          <div className="flex items-center gap-3">
            <Shuffle className="w-6 h-6" />
            <input
              className="text-2xl font-bold bg-transparent outline-none border-b border-dashed border-gray-300 focus:border-indigo-500"
              value={current?.name || ""}
              onChange={(e) => renameStudy(e.target.value)}
            />
          </div>
          <div className="flex items-center gap-2">
            <select
              className="rounded-xl border px-2 py-2"
              value={current?.id || ""}
              onChange={(e) => setCurrentId(e.target.value)}
            >
              {studies.map(s => (<option key={s.id} value={s.id}>{s.name}</option>))}
            </select>
            <SmallButton onClick={createStudy} title="Nuevo estudio"><Plus className="w-4 h-4" /> Nuevo</SmallButton>
            <SmallButton onClick={duplicateStudy} title="Duplicar estudio"><Download className="w-4 h-4 rotate-180" /> Duplicar</SmallButton>
            <SmallButton onClick={deleteStudy} title="Eliminar estudio"><Trash2 className="w-4 h-4" /> Eliminar</SmallButton>
            <SmallButton onClick={exportStudy} title="Exportar estudio"><Download className="w-4 h-4" /> Exportar</SmallButton>
            <label className="inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-sm hover:bg-gray-50 cursor-pointer">
              <Upload className="w-4 h-4" /> Importar
              <input type="file" accept="application/json" className="hidden" onChange={(e) => e.target.files && importStudy(e.target.files[0])} />
            </label>
          </div>
        </header>

        <nav className="grid grid-cols-5 gap-2">
          <button onClick={() => setTab("cards")}    className={`rounded-2xl py-3 text-sm font-medium shadow ${tab==="cards"   ? "bg-indigo-600 text-white" : "bg-white"}`}>Tarjetas</button>
          <button onClick={() => setTab("profiles")} className={`rounded-2xl py-3 text-sm font-medium shadow ${tab==="profiles"? "bg-indigo-600 text-white" : "bg-white"}`}>Perfiles</button>
          <button onClick={() => setTab("sessions")} className={`rounded-2xl py-3 text-sm font-medium shadow ${tab==="sessions"? "bg-indigo-600 text-white" : "bg-white"}`}>Sesiones</button>
          <button onClick={() => setTab("entry")}    className={`rounded-2xl py-3 text-sm font-medium shadow ${tab==="entry"   ? "bg-indigo-600 text-white" : "bg-white"}`}>Ingreso de datos</button>
          <button onClick={() => setTab("analysis")} className={`rounded-2xl py-3 text-sm font-medium shadow ${tab==="analysis"?"bg-indigo-600 text-white":"bg-white"}`}>Análisis</button>
        </nav>

        {tab==="cards"   && <CardsTab   cards={cards}    setCards={setCards} />}
        {tab==="profiles"&& <ProfilesTab profiles={profiles} setProfiles={setProfiles} />}
        {tab==="sessions"&& <SessionsTab sessions={sessions} profiles={profiles} setSessions={setSessions} />}
        {tab==="entry"   && <EntryTab   cards={cards} profiles={profiles} onSave={(s) => setSessions((p:any) => [s, ...p])} />}
        {tab==="analysis"&& <AnalysisTab cards={cards} sessions={sessions} />}

        <footer className="text-xs text-gray-500 pt-4">Cada estudio está aislado. Exporta/Importa estudios completos con su nombre.</footer>
      </div>
    </DndProvider>
  );
}

// ---------- Tab 1: Cards ----------
function CardsTab({ cards, setCards }: { cards: Card[]; setCards: (f: any) => void }) {
  const [filter, setFilter] = useState("");
  const filtered = useMemo(()=>cards.filter(c=>c.label.toLowerCase().includes(filter.toLowerCase())),[cards,filter]);
  const addCard = useCallback(()=>setCards((prev:Card[])=>[{id:genId(),label:"Nueva tarjeta"},...prev]),[setCards]);
  const del = useCallback((id:string)=>setCards((prev:Card[])=>prev.filter(c=>c.id!==id)),[setCards]);
  const upd = useCallback((id:string, patch:Partial<Card>)=>setCards((prev:Card[])=>prev.map(c=>c.id===id?{...c,...patch}:c)),[setCards]);
  return (
    <Section title="Tarjetas" icon={<Table className="w-5 h-5"/>} toolbar={<SmallButton onClick={addCard}><Plus className="w-4 h-4"/>Añadir</SmallButton>}>
      <div className="mb-3"><TextInput value={filter} onChange={setFilter} placeholder="Buscar..."/></div>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-3">
        {filtered.map(card=> (
          <div key={card.id} className="rounded-2xl border p-3 bg-white shadow-sm">
            <TextInput value={card.label} onChange={(v)=>upd(card.id,{label:v})}/>
            <textarea className="w-full mt-2 rounded-xl border p-2" rows={2} placeholder="Descripción (opcional)" value={card.description||""} onChange={(e)=>upd(card.id,{description:e.target.value})}/>
            <div className="text-right mt-2"><button onClick={()=>del(card.id)} className="inline-flex items-center gap-1 text-red-600 text-sm hover:underline"><Trash2 className="w-4 h-4"/>Eliminar</button></div>
          </div>
        ))}
      </div>
    </Section>
  );
}

// ---------- Tab 2: Profiles ----------
function ProfilesTab({ profiles, setProfiles }: { profiles: Profile[]; setProfiles: (f: any) => void }) {
  const add = useCallback(()=>setProfiles((prev:Profile[])=>[{id:genId(),name:"Nuevo perfil"},...prev]),[setProfiles]);
  const upd = useCallback((id:string, patch:Partial<Profile>)=>setProfiles((prev:Profile[])=>prev.map(p=>p.id===id?{...p,...patch}:p)),[setProfiles]);
  const del = useCallback((id:string)=>setProfiles((prev:Profile[])=>prev.filter(p=>p.id!==id)),[setProfiles]);
  return (
    <Section title="Perfiles de usuario" icon={<Users className="w-5 h-5"/>} toolbar={<SmallButton onClick={add}><UserPlus className="w-4 h-4"/>Añadir</SmallButton>}>
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-3">
        {profiles.map(p=> (
          <div key={p.id} className="rounded-2xl border p-3 bg-white shadow-sm flex items-center gap-2">
            <TextInput value={p.name} onChange={(v)=>upd(p.id,{name:v})}/>
            <button onClick={()=>del(p.id)} className="text-red-600 hover:underline text-sm flex items-center gap-1"><Trash2 className="w-4 h-4"/>Eliminar</button>
          </div>
        ))}
      </div>
    </Section>
  );
}

// ---------- Tab 3: Sessions ----------
function SessionsTab({ sessions, profiles, setSessions }: { sessions: Session[]; profiles: Profile[]; setSessions: (f: any) => void }) {
  const profileName = useCallback((id: string | null)=>profiles.find(p=>p.id===id)?.name||"—",[profiles]);
  const remove = useCallback((id:string)=>setSessions((prev:Session[])=>prev.filter(s=>s.id!==id)),[setSessions]);
  const clearAll = useCallback(()=>setSessions([]),[setSessions]);
  return (
    <Section title="Sesiones" icon={<Settings className="w-5 h-5"/>} toolbar={<SmallButton onClick={clearAll}><Trash2 className="w-4 h-4"/>Borrar todo</SmallButton>}>
      <div className="overflow-auto">
        <table className="min-w-full text-sm">
          <thead><tr className="text-left border-b"><th className="py-2 pr-3">#</th><th className="py-2 pr-3">Perfil</th><th className="py-2 pr-3">Género</th><th className="py-2 pr-3">Edad</th><th className="py-2 pr-3">Fecha</th><th className="py-2 pr-3">Duración</th><th className="py-2 pr-3">Grupos raíz</th><th></th></tr></thead>
          <tbody>
            {sessions.map((s,i)=> (
              <tr key={s.id} className="border-b">
                <td className="py-2 pr-3">{sessions.length - i}</td>
                <td className="py-2 pr-3">{profileName(s.demographics.profileId)}</td>
                <td className="py-2 pr-3">{s.demographics.gender||"—"}</td>
                <td className="py-2 pr-3">{s.demographics.age??"—"}</td>
                <td className="py-2 pr-3">{new Date(s.startedAt).toLocaleString()}</td>
                <td className="py-2 pr-3">{Math.round(s.durationSec)} seg</td>
                <td className="py-2 pr-3">{s.groups.length}</td>
                <td className="py-2 pr-3 text-right"><button onClick={()=>remove(s.id)} className="text-red-600 hover:underline inline-flex items-center gap-1 text-xs"><Trash2 className="w-4 h-4"/>Eliminar</button></td>
              </tr>
            ))}
            {sessions.length===0&&(<tr><td colSpan={8} className="py-6 text-center text-gray-500">Sin sesiones guardadas.</td></tr>)}
          </tbody>
        </table>
      </div>
    </Section>
  );
}

// ---------- Helpers para jerarquía ----------
function removeCardFromGroup(groups: SortGroup[], groupId: string, cardId: string): SortGroup[] {
  return groups.map(g=> g.id===groupId ? ({...g, cardIds: g.cardIds.filter(id=>id!==cardId)}) : ({...g, children: removeCardFromGroup(g.children, groupId, cardId)}));
}
function addCardToSpecificGroup(groups: SortGroup[], groupId: string, cardId: string, index?: number): SortGroup[] {
  return groups.map(g=> {
    if (g.id===groupId) {
      const ids = g.cardIds.slice();
      if (index===undefined) ids.push(cardId); else ids.splice(Math.min(Math.max(0,index), ids.length), 0, cardId);
      return { ...g, cardIds: ids };
    }
    return { ...g, children: addCardToSpecificGroup(g.children, groupId, cardId, index) };
  });
}
function removeCardEverywhereOnce(groups: SortGroup[], cardId: string): { groups: SortGroup[]; found: boolean } {
  let found = false;
  const walk = (gs: SortGroup[]): SortGroup[] => gs.map(g=>{
    if (found) return g; // short-circuit
    const had = g.cardIds.includes(cardId);
    const newIds = had ? g.cardIds.filter(id=>id!==cardId) : g.cardIds;
    if (had) found = true;
    return { ...g, cardIds: newIds, children: walk(g.children) };
  });
  return { groups: walk(groups), found };
}
function findParentId(groups: SortGroup[], targetId: string, parentId: string | null = null): string | null {
  for (const g of groups) {
    if (g.id === targetId) return parentId;
    const r = findParentId(g.children, targetId, g.id);
    if (r) return r;
  }
  return null;
}

// ---------- Tab 4: Entry (optimizado + flechas laterales y subir) ----------
function EntryTab({ cards, profiles, onSave }: { cards: Card[]; profiles: Profile[]; onSave: (s: Session) => void }) {
  const [dem, setDem] = useState<Demographics>({ profileId: profiles[0]?.id || null, gender: null, age: null });
  const [available, setAvailable] = useState<Card[]>(cards);
  const [groups, setGroups] = useState<SortGroup[]>([]);
  const [startedAt, setStartedAt] = useState<number | null>(null);

  const cardById = useMemo(() => new Map(cards.map(c=>[c.id, c])), [cards]);
  useEffect(()=>setAvailable(cards),[cards]);
  const startIfNeeded = useCallback(()=>{ if(!startedAt) setStartedAt(Date.now()); },[startedAt]);

  const addRootGroup = useCallback(()=>{ startIfNeeded(); setGroups((prev)=>[...prev,{ id: genId(), name: `Grupo ${prev.length+1}`, cardIds: [], children: [] }]); },[startIfNeeded]);
  const addChildGroup = useCallback((parentId: string)=>{ startIfNeeded(); setGroups((prev)=> prev.map(g=> g.id===parentId?{...g, children:[...g.children,{ id: genId(), name:"Subgrupo", cardIds:[], children:[] }]}:{...g, children: addChildDeep(g.children, parentId)})); },[startIfNeeded]);
  const addChildDeep = (arr: SortGroup[], parentId: string): SortGroup[] => arr.map(g=> g.id===parentId?{...g, children:[...g.children,{ id: genId(), name:"Subgrupo", cardIds:[], children:[] }]}:{...g, children: addChildDeep(g.children,parentId)});
  const renameGroup = useCallback((id: string, name: string)=> setGroups((prev)=> renameDeep(prev, id, name)),[]);
  const renameDeep = (arr: SortGroup[], id:string, name:string): SortGroup[] => arr.map(g=> g.id===id?{...g, name}:{...g, children: renameDeep(g.children,id,name)});
  const deleteGroup = useCallback((id: string)=>{
    setGroups(prev=>{
      // recolectar tarjetas del grupo eliminado
      const collectCards = (gs: SortGroup[]): { filtered: SortGroup[]; ids: string[] } => {
        const out: SortGroup[] = []; const ids: string[] = [];
        for (const g of gs) {
          if (g.id===id) { ids.push(...g.cardIds); const stack=[...g.children]; while(stack.length){ const s=stack.pop()!; ids.push(...s.cardIds); stack.push(...s.children);} continue; }
          const child = collectCards(g.children); ids.push(...child.ids); out.push({ ...g, children: child.filtered });
        }
        return { filtered: out, ids };
      };
      const { filtered, ids } = collectCards(prev);
      const back = ids.map(cid=> cardById.get(cid)).filter(Boolean) as Card[];
      if (back.length) setAvailable(a=>[...a, ...back]);
      return filtered;
    });
  },[cardById]);

  const moveCard = useCallback((cardId: string, from: string, to: string | null) => {
    startIfNeeded();
    const card = cardById.get(cardId); if(!card) return;
    if (from === "available") setAvailable(prev=>prev.filter(c=>c.id!==cardId));
    else setGroups(prev=> removeCardFromGroup(prev, from, cardId));
    if (to === null) setAvailable(prev=>[...prev, card]);
    else setGroups(prev=> addCardToSpecificGroup(prev, to, cardId));
  },[cardById, startIfNeeded]);

  const onReorder = useCallback((groupId: string, cardId: string, toIndex: number) => {
    setGroups(prev=> addCardToSpecificGroup(removeCardFromGroup(prev, groupId, cardId), groupId, cardId, toIndex));
  },[]);

  const onSendToAvailable = useCallback((fromGroupId: string, cardId: string) => {
    setGroups(prev=> removeCardFromGroup(prev, fromGroupId, cardId));
    const card = cardById.get(cardId); if (card) setAvailable(prev=>[...prev, card]);
  },[cardById]);

  const moveToSibling = useCallback((fromGroupId: string, toGroupId: string, cardId: string) => {
    setGroups(prev=> addCardToSpecificGroup(removeCardFromGroup(prev, fromGroupId, cardId), toGroupId, cardId));
  },[]);
  const moveToParent = useCallback((fromGroupId: string, cardId: string) => {
    setGroups(prev=>{
      const parent = findParentId(prev, fromGroupId);
      if (!parent) return prev; // ya es raíz → nada
      return addCardToSpecificGroup(removeCardFromGroup(prev, fromGroupId, cardId), parent, cardId);
    });
  },[]);

  const reset = useCallback(()=>{ setStartedAt(null); setDem({ profileId: profiles[0]?.id || null, gender: null, age: null }); setAvailable(cards); setGroups([]); },[cards, profiles]);
  const save = useCallback(()=>{ if(!startedAt) setStartedAt(Date.now()); const s: Session = { id: genId(), startedAt: startedAt || Date.now(), durationSec: ((Date.now() - (startedAt || Date.now()))/1000), demographics: dem, groups }; onSave(s); reset(); alert("Sesión guardada"); },[dem, groups, onSave, reset, startedAt]);

  return (
    <Section title="Ingreso de datos (sesión de usuario)" icon={<Play className="w-5 h-5"/>} toolbar={<div className="flex gap-2"><SmallButton onClick={addRootGroup}><Plus className="w-4 h-4"/>Nuevo grupo</SmallButton><SmallButton onClick={reset}><RefreshCcw className="w-4 h-4"/>Reiniciar</SmallButton><SmallButton onClick={save}><Save className="w-4 h-4"/>Guardar sesión</SmallButton></div>}>
      {/* Demographics */}
      <div className="grid md:grid-cols-3 gap-3 mb-5">
        <div><label className="text-sm text-gray-600">Perfil</label><select value={dem.profileId||""} onChange={(e)=>setDem({...dem, profileId:e.target.value})} className="w-full rounded-xl border px-3 py-2">{profiles.map(p=>(<option key={p.id} value={p.id}>{p.name}</option>))}</select></div>
        <div><label className="text-sm text-gray-600">Género</label><select value={dem.gender||""} onChange={(e)=>setDem({...dem, gender:(e.target.value as any)||null})} className="w-full rounded-xl border px-3 py-2"><option value="">—</option><option value="Male">Male</option><option value="Female">Female</option><option value="Otro">Otro</option></select></div>
        <div><label className="text-sm text-gray-600">Edad</label><input type="number" min={0} className="w-full rounded-xl border px-3 py-2" value={dem.age??""} onChange={(e)=>setDem({...dem, age:e.target.value?Number(e.target.value):null})}/></div>
      </div>

      {/* Sorting board */}
      <div className="grid md:grid-cols-3 gap-4">
        <DropColumn title={`Tarjetas disponibles (${available.length})`} onDrop={(item)=>moveCard(item.cardId, item.from, null)}>
          {available.map(c=> (<DraggableCard key={c.id} card={c} from="available" />))}
        </DropColumn>
        <div className="md:col-span-2 grid grid-cols-1 sm:grid-cols-2 gap-4">
          {groups.map((g, idx)=> (
            <GroupPanel key={g.id} group={g} cards={cards} cardById={cardById} parentId={null} siblings={groups} siblingIndex={idx}
              onRename={renameGroup} onDelete={deleteGroup} onAddSubgroup={addChildGroup}
              onDropToGroup={(targetId, from, id)=>moveCard(id, from, targetId)} onReorder={onReorder} onSendToAvailable={onSendToAvailable}
              onMoveToSibling={moveToSibling} onMoveToParent={moveToParent}
            />
          ))}
          {groups.length===0&&(<div className="rounded-2xl border-dashed border-2 p-6 text-center text-gray-500">Añade grupos y arrastra tarjetas aquí.</div>)}
        </div>
      </div>
      <div className="text-xs text-gray-500 mt-4">Optimizado: arrastre fluido, flechas (▲▼ para ordenar, ◀▶ para mover entre subgrupos hermanos, ⬆ para subir al padre) y escrituras a disco con debounce.</div>
    </Section>
  );
}

// ---------- Panel recursivo de grupo (memoizado) ----------
const GroupPanel = React.memo(function GroupPanel({ group, cards, cardById, parentId, siblings, siblingIndex, onRename, onDelete, onAddSubgroup, onDropToGroup, onReorder, onSendToAvailable, onMoveToSibling, onMoveToParent }: {
  group: SortGroup; cards: Card[]; cardById: Map<string, Card>; parentId: string | null; siblings: SortGroup[]; siblingIndex: number;
  onRename: (id:string,name:string)=>void; onDelete: (id:string)=>void; onAddSubgroup: (id:string)=>void; onDropToGroup: (targetId:string, from:string, cardId:string)=>void;
  onReorder: (groupId:string, cardId:string, toIndex:number)=>void; onSendToAvailable: (fromGroupId:string, cardId:string)=>void; onMoveToSibling: (fromGroupId:string, toGroupId:string, cardId:string)=>void; onMoveToParent: (fromGroupId:string, cardId:string)=>void;
}) {
  const prevSibling = siblings[siblingIndex-1];
  const nextSibling = siblings[siblingIndex+1];
  return (
    <div className="rounded-2xl border p-3 bg-white shadow-sm">
      <div className="flex items-center justify-between mb-2">
        <input value={group.name} onChange={(e)=>onRename(group.id, e.target.value)} className="font-medium rounded-lg border px-2 py-1" />
        <div className="flex gap-2 items-center">
          <SmallButton onClick={()=>onAddSubgroup(group.id)} title="Añadir subgrupo"><FolderPlus className="w-4 h-4"/>Subgrupo</SmallButton>
          <button onClick={()=>onDelete(group.id)} className="text-red-600 text-xs inline-flex items-center gap-1"><Trash2 className="w-4 h-4"/>Eliminar</button>
        </div>
      </div>
      <DropColumn title={`${group.cardIds.length} tarjetas`} onDrop={(item)=>onDropToGroup(group.id, item.from, item.cardId)}>
        {group.cardIds.map((cid, idx)=>{ const c = cardById.get(cid)!; return (
          <div key={cid} className="flex items-center gap-2">
            <div className="flex-1"><DraggableCard card={c} from={group.id}/></div>
            <div className="flex items-center gap-1">
              <button title="Subir" className="rounded-md border px-1 text-xs" onClick={()=>onReorder(group.id, cid, Math.max(0, idx-1))}>▲</button>
              <button title="Bajar" className="rounded-md border px-1 text-xs" onClick={()=>onReorder(group.id, cid, Math.min(group.cardIds.length-1, idx+1))}>▼</button>
              {prevSibling && (<button title="Mover a subgrupo anterior" className="rounded-md border px-1 text-xs" onClick={()=>onMoveToSibling(group.id, prevSibling.id, cid)}><ArrowLeft className="w-3 h-3"/></button>)}
              {nextSibling && (<button title="Mover a subgrupo siguiente" className="rounded-md border px-1 text-xs" onClick={()=>onMoveToSibling(group.id, nextSibling.id, cid)}><ArrowRight className="w-3 h-3"/></button>)}
              {parentId && (<button title="Subir al grupo padre" className="rounded-md border px-1 text-xs" onClick={()=>onMoveToParent(group.id, cid)}><ArrowUp className="w-3 h-3"/></button>)}
              <button title="Enviar a disponibles" className="rounded-md border px-1 text-xs" onClick={()=>onSendToAvailable(group.id, cid)}>↩</button>
            </div>
          </div>
        ); })}
      </DropColumn>
      {group.children.length>0 && (
        <div className="mt-3 grid grid-cols-1 gap-3">
          {group.children.map((child, i)=> (
            <GroupPanel key={child.id} group={child} cards={cards} cardById={cardById} parentId={group.id} siblings={group.children} siblingIndex={i}
              onRename={onRename} onDelete={onDelete} onAddSubgroup={onAddSubgroup} onDropToGroup={onDropToGroup} onReorder={onReorder}
              onSendToAvailable={onSendToAvailable} onMoveToSibling={onMoveToSibling} onMoveToParent={onMoveToParent}
            />
          ))}
        </div>
      )}
    </div>
  );
});

// ---------- Análisis (Matriz de similitud, Dendrograma, PCA) ----------
function AnalysisTab({ cards, sessions }: { cards: Card[]; sessions: Session[] }) {
  const [limit, setLimit] = useState(Math.min(24, cards.length));
  const [linkage, setLinkage] = useState<"average" | "single" | "complete">("average");

  const cardIds = useMemo(()=> cards.slice(0, limit).map(c=>c.id), [cards, limit]);
  const idToIndex = useMemo(()=> new Map(cardIds.map((id, i)=>[id,i])), [cardIds]);

  const { S, C, P } = useMemo(()=> buildSimilarityMatrix(cards, sessions, cardIds, idToIndex), [cards, sessions, cardIds, idToIndex]);
  const tree = useMemo(()=> buildDendrogram(S, cardIds, linkage), [S, cardIds, linkage]);
  const coords = useMemo(()=> pcaFromSimilarity(S, 2), [S]);

  // export helpers
  const downloadText = useCallback((filename: string, text: string, mime = "text/plain") => {
    const blob = new Blob([text], { type: mime+";charset=utf-8" });
    const url = URL.createObjectURL(blob); const a = document.createElement("a"); a.href = url; a.download = filename; a.click(); URL.revokeObjectURL(url);
  }, []);
  const toCsv = (rows: string[][]) => rows.map(r=> r.map(cell=> /[",
]/.test(cell)? '"'+cell.replace(/"/g,'""')+'"' : cell).join(",")).join("
");

  const label = useCallback((id:string)=> cards.find(c=>c.id===id)?.label||id, [cards]);
  const exportSimilarityCsv = useCallback(()=>{
    const header = ["Tarjeta", ...cardIds.map(label)];
    const rows = [header];
    for (let i=0;i<cardIds.length;i++){
      const row = [label(cardIds[i]), ...cardIds.map((_,j)=> S[i][j].toFixed(4))];
      rows.push(row);
    }
    downloadText("similitud.csv", toCsv(rows), "text/csv");
  }, [S, cardIds, label, downloadText]);

  const dendroRef = useRef<SVGSVGElement|null>(null);
  const pcaRef = useRef<SVGSVGElement|null>(null);
  const exportSvg = useCallback((ref: React.MutableRefObject<SVGSVGElement|null>, filename: string)=>{
    const svg = ref.current; if(!svg) return;
    const src = new XMLSerializer().serializeToString(svg);
    downloadText(filename, src, "image/svg+xml");
  }, [downloadText]);

  const exportPcaCsv = useCallback(()=>{
    const header = ["Tarjeta","PC1","PC2"]; const rows = [header];
    for(let i=0;i<cardIds.length;i++){ rows.push([label(cardIds[i]), String(coords[i]?.[0]??0), String(coords[i]?.[1]??0)]); }
    downloadText("pca_coords.csv", toCsv(rows), "text/csv");
  }, [coords, cardIds, label, downloadText]);

  return (
    <Section title="Análisis del estudio" icon={<Settings className="w-5 h-5"/>} toolbar={<div className="flex items-center gap-2 flex-wrap">
      <div className="flex items-center gap-2 mr-2"><span className="text-sm text-gray-600">N tarjetas</span><NumberInput value={limit} onChange={setLimit} min={2} max={cards.length||2}/></div>
      <div className="flex items-center gap-2 mr-4"><span className="text-sm text-gray-600">Linkage</span>
        <select value={linkage} onChange={(e)=>setLinkage(e.target.value as any)} className="rounded-xl border px-2 py-2 text-sm">
          <option value="single">single</option>
          <option value="complete">complete</option>
          <option value="average">average</option>
        </select>
      </div>
      <SmallButton onClick={exportSimilarityCsv} title="Exportar matriz de similitud (CSV)"><Download className="w-4 h-4"/> CSV Similitud</SmallButton>
      <SmallButton onClick={()=>exportSvg(dendroRef, "dendrograma.svg")} title="Exportar dendrograma (SVG)"><Download className="w-4 h-4"/> SVG Dendrograma</SmallButton>
      <SmallButton onClick={()=>exportSvg(pcaRef, "pca.svg")} title="Exportar PCA (SVG)"><Download className="w-4 h-4"/> SVG PCA</SmallButton>
      <SmallButton onClick={exportPcaCsv} title="Exportar coordenadas PCA (CSV)"><Download className="w-4 h-4"/> CSV PCA</SmallButton>
    </div>}>
      <div className="grid lg:grid-cols-3 gap-4">
        <div className="rounded-2xl border bg-white p-3 overflow-auto">
          <div className="text-sm font-medium mb-2">Matriz de similitud</div>
          <Heatmap cards={cards} ids={cardIds} S={S} />
        </div>
        <div className="rounded-2xl border bg-white p-3 overflow-auto">
          <div className="text-sm font-medium mb-2">Dendrograma ({linkage})</div>
          <Dendrogram innerRef={dendroRef} cards={cards} ids={cardIds} tree={tree} />
        </div>
        <div className="rounded-2xl border bg-white p-3 overflow-auto">
          <div className="text-sm font-medium mb-2">PCA (2D)</div>
          <PcaScatter innerRef={pcaRef} cards={cards} ids={cardIds} coords={coords} />
        </div>
      </div>
      <div className="text-xs text-gray-500 mt-3">Notas: La similitud se calcula por co-ocurrencia de tarjetas en el mismo (sub)grupo dentro de una sesión; se normaliza por sesiones donde ambas tarjetas aparecen. El dendrograma usa aglomerativo con enlace configurable. El PCA proyecta desde la matriz de similitud. Exports: CSV y SVG vectorial.</div>
    </Section>
  );
}

function buildSimilarityMatrix(cards: Card[], sessions: Session[], ids: string[], idToIndex: Map<string, number>) {
  const n = ids.length; const S = Array.from({length:n}, ()=>Array(n).fill(0));
  const C = Array.from({length:n}, ()=>Array(n).fill(0));
  const P = Array.from({length:n}, ()=>Array(n).fill(0));
  const idSet = new Set(ids);
  function collectClusters(groups: SortGroup[], out: string[][]) {
    for (const g of groups) {
      const here = g.cardIds.filter(id=>idSet.has(id));
      if (here.length>1) out.push(here);
      if (g.children?.length) collectClusters(g.children, out);
    }
  }
  for (const s of sessions) {
    const clusters: string[][] = [];
    collectClusters(s.groups||[], clusters);
    const present = new Set<string>();
    const walk = (gs: SortGroup[])=>{ for (const g of gs){ g.cardIds.forEach(id=>{ if(idSet.has(id)) present.add(id); }); if (g.children?.length) walk(g.children); } };
    walk(s.groups||[]);
    const presentArr = Array.from(present);
    for (let i=0;i<presentArr.length;i++){
      const a=presentArr[i]; const ia=idToIndex.get(a)!;
      for (let j=i;j<presentArr.length;j++){
        const b=presentArr[j]; const ib=idToIndex.get(b)!;
        P[ia][ib]++; P[ib][ia]++;
      }
    }
    for (const cluster of clusters) {
      for (let i=0;i<cluster.length;i++){
        const ia = idToIndex.get(cluster[i])!;
        for (let j=i;j<cluster.length;j++){
          const ib = idToIndex.get(cluster[j])!;
          C[ia][ib]++; C[ib][ia]++;
        }
      }
    }
  }
  for (let i=0;i<n;i++){
    for (let j=0;j<n;j++){
      S[i][j] = (P[i][j]>0) ? (C[i][j]/P[i][j]) : 0;
    }
  }
  return { S, C, P };
}

function buildDendrogram(S: number[][], ids: string[], linkage: "average"|"single"|"complete"){
  const n=ids.length;
  let clusters = ids.map((id, i)=>({ id: `leaf-${id}`, items: [i], height: 0, left: null as any, right: null as any }));
  const dist = (A:number[],B:number[])=>{
    let best = linkage==="single"? Infinity : linkage==="complete"? -Infinity : 0;
    let cnt=0;
    for(const a of A) for(const b of B){ const d = 1 - S[a][b]; cnt++; if(linkage==="single") best = Math.min(best, d); else if(linkage==="complete") best = Math.max(best, d); else best += d; }
    return linkage==="average"? best/cnt : best;
  };
  while (clusters.length>1){
    let bi=-1,bj=-1,bd=Infinity;
    for(let i=0;i<clusters.length;i++) for(let j=i+1;j<clusters.length;j++){ const d=dist(clusters[i].items, clusters[j].items); if(d<bd){bd=d;bi=i;bj=j;} }
    const a=clusters[bi], b=clusters[bj];
    const merged = { id: `node-${a.id}-${b.id}`, items: [...a.items, ...b.items], height: bd, left: a, right: b };
    clusters = clusters.filter((_,k)=>k!==bi && k!==bj); clusters.push(merged);
  }
  return clusters[0];
}

function pcaFromSimilarity(S: number[][], k=2){
  const n=S.length; if(n===0) return [] as number[][];
  const D2 = Array.from({length:n},()=>Array(n).fill(0));
  for(let i=0;i<n;i++) for(let j=0;j<n;j++){ const d2 = Math.max(0, 1 - S[i][j]); D2[i][j]=d2; }
  const J = (i:number,j:number)=> (i===j?1:0) - 1/n;
  const B = Array.from({length:n},()=>Array(n).fill(0));
  for(let i=0;i<n;i++) for(let j=0;j<n;j++){
    let s=0; for(let p=0;p<n;p++) for(let q=0;q<n;q++){ s += J(i,p)*D2[p][q]*J(q,j); }
    B[i][j] = -0.5*s;
  }
  const eig = powerIterationTopK(B, k);
  const V = eig.vectors; const vals = eig.values;
  const coords = Array.from({length:n},()=>Array(k).fill(0));
  for(let i=0;i<n;i++) for(let c=0;c<k;c++){ coords[i][c] = V[i][c] * Math.sqrt(Math.max(0, vals[c])); }
  return coords;
}
function powerIterationTopK(A:number[][], k:number){
  const n=A.length; const vectors:number[][]=[]; const values:number[]=[];
  const dot=(x:number[],y:number[])=> x.reduce((s,v,i)=>s+v*y[i],0);
  const norm=(x:number[])=> Math.sqrt(dot(x,x));
  const matvec=(M:number[][],x:number[])=> M.map(row=> row.reduce((s,v,j)=>s+v*x[j],0));
  const subtractProj=(x:number[])=>{
    let y=x.slice();
    for(let c=0;c<vectors.length;c++){
      const v=vectors[c]; const coeff = dot(y,v); y = y.map((vi,i)=> vi - coeff*v[i]);
    }
    return y;
  };
  for(let t=0;t<k;t++){
    let v=Array.from({length:n},()=>Math.random());
    v=subtractProj(v); let nv=norm(v)||1; v=v.map(vi=>vi/nv);
    for(let it=0;it<200;it++){
      let w = matvec(A,v); w=subtractProj(w); const nw=norm(w)||1; w=w.map(wi=>wi/nw); v=w;
    }
    const Av = matvec(A,v); const val = dot(v,Av);
    vectors.push(v); values.push(val);
  }
  const Vcols = vectors;
  const V: number[][] = Array.from({length:n},()=>Array(k).fill(0));
  for(let col=0;col<k;col++) for(let i=0;i<n;i++) V[i][col]=Vcols[col][i];
  return { vectors: V, values };
}

function Heatmap({ cards, ids, S }:{ cards: Card[]; ids: string[]; S: number[][] }){
  const name = (id:string)=> cards.find(c=>c.id===id)?.label||id;
  return (
    <div className="overflow-auto">
      <table className="text-xs">
        <thead>
          <tr><th></th>{ids.map(id=> <th key={id} className="px-1 py-1 text-left whitespace-nowrap">{name(id)}</th>)}</tr>
        </thead>
        <tbody>
          {ids.map((id,i)=> (
            <tr key={id}>
              <th className="pr-2 text-left whitespace-nowrap">{name(id)}</th>
              {ids.map((jd,j)=>{ const v=S[i][j]; const c = Math.round(255*(1-v)); const bg=`rgb(${c},${c},255)`; return <td key={j} className="w-6 h-6 text-center" style={{background:bg}}>{v.toFixed(2)}</td>; })}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

function Dendrogram({ cards, ids, tree, innerRef }:{ cards: Card[]; ids:string[]; tree:any; innerRef?: React.MutableRefObject<SVGSVGElement|null> }){
  const leafOrder = ids.slice();
  const yPos = new Map(leafOrder.map((id,i)=>[id, i*20 + 10]));
  const nodes:any[]=[]; const links:any[]=[];
  function walk(node:any){
    if(!node.left && !node.right){ const id=node.id.replace(/^leaf-/,''); nodes.push({x:0,y:yPos.get(id)!,label: cards.find(c=>c.id===id)?.label||id}); return {x:0,y:yPos.get(id)!}; }
    const L=walk(node.left); const R=walk(node.right); const x = Math.max(L.x,R.x)+50; const y=(L.y+R.y)/2; nodes.push({x,y,label:""}); links.push({x1:L.x,y1:L.y,x2:x,y2:y}); links.push({x1:R.x,y1:R.y,x2:x,y2:y}); return {x,y};
  }
  walk(tree);
  const width = Math.max(...nodes.map(n=>n.x))+140; const height = Math.max(...nodes.map(n=>n.y))+20;
  return (
    <svg ref={innerRef as any} width={width} height={height} className="border rounded-xl w-full">
      {links.map((l,i)=> (<line key={i} x1={l.x1+80} y1={l.y1} x2={l.x2+80} y2={l.y2} stroke="#999"/>))}
      {nodes.map((n,i)=> (<g key={i} transform={`translate(${n.x+80},${n.y})`}>
        {n.label && (<text x={0} y={4} fontSize={10} textAnchor="start">{n.label}</text>)}
      </g>))}
    </svg>
  );
}

function PcaScatter({ cards, ids, coords, innerRef }:{ cards: Card[]; ids:string[]; coords:number[][]; innerRef?: React.MutableRefObject<SVGSVGElement|null> }){
  const pts = coords.map((c,i)=>({ x:c[0]||0, y:c[1]||0, id: ids[i] }));
  const xs = pts.map(p=>p.x), ys = pts.map(p=>p.y);
  const minx=Math.min(...xs), maxx=Math.max(...xs), miny=Math.min(...ys), maxy=Math.max(...ys);
  const W=480, H=300, pad=30;
  const sx=(x:number)=> pad + (W-2*pad) * (maxx-minx? (x-minx)/(maxx-minx) : 0.5);
  const sy=(y:number)=> pad + (H-2*pad) * (maxy-miny? (1 - (y-miny)/(maxy-miny)) : 0.5);
  const name=(id:string)=> cards.find(c=>c.id===id)?.label||id;
  return (
    <svg ref={innerRef as any} width="100%" height={H} viewBox={`0 0 ${W} ${H}`} className="border rounded-xl">
      {pts.map((p,i)=> (<g key={i} transform={`translate(${sx(p.x)},${sy(p.y)})`}>
        <circle r={3} />
        <text x={5} y={-5} fontSize={10}>{name(p.id)}</text>
      </g>))}
      <text x={pad} y={12} fontSize={10}>PC1</text>
      <text x={W-20} y={H-6} fontSize={10} textAnchor="end">PC2</text>
    </svg>
  );
}

