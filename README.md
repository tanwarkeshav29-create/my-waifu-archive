# my-waifu-archive
the ultimate list
import React, { useEffect, useState, useRef } from "react";
import { ResponsiveContainer, PieChart, Pie, Cell, Tooltip, Legend } from "recharts";

// Ultimate Character Gallery — single-file React component
// - Organized by media (anime, manhwa, manhua, vn, donghua, novel, book)
// - Ranks (Waifu, Best Character, Best Fiction, Custom ranks)
// - Genres per series, multi-select filters
// - Fate Universe grouping & quick charting
// - CSV / JSON export-import, shareable link (URL hash), printable view
// - Move up/down ordering, bulk actions, and visual stats

const RANKS_DEFAULT = ["Waifu", "Best Character", "Best Fiction"];
const MEDIA_TYPES = ["anime","manhwa","manhua","web novel","vn","donghua","book","other"];
const COLORS = ["#8884d8","#82ca9d","#ffc658","#ff7f50","#a28fd0","#f78fb3","#74b9ff","#fdcb6e"];

export default function UltimateCharacterGallery() {
  const STORAGE_KEY = "ultimate_character_gallery_v1";
  const [characters, setCharacters] = useState([]);
  const [ranks, setRanks] = useState(RANKS_DEFAULT);
  const [isModalOpen, setModalOpen] = useState(false);
  const [editingIndex, setEditingIndex] = useState(null);
  const [form, setForm] = useState(emptyForm());
  const [query, setQuery] = useState("");
  const [selectedMedia, setSelectedMedia] = useState([]);
  const [selectedRanks, setSelectedRanks] = useState([]);
  const [selectedGenres, setSelectedGenres] = useState([]);
  const fileRef = useRef(null);

  useEffect(() => {
    const hash = window.location.hash.slice(1);
    if (hash) {
      try {
        const decoded = JSON.parse(decodeURIComponent(atob(hash)));
        if (Array.isArray(decoded)) {
          setCharacters(decoded);
          return;
        }
      } catch (e) {
        // ignore
      }
    }
    const saved = localStorage.getItem(STORAGE_KEY);
    if (saved) setCharacters(JSON.parse(saved));
    else setCharacters(sampleData());
  }, []);

  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(characters));
  }, [characters]);

  function emptyForm() {
    return {
      name: "",
      series: "",
      media: "anime",
      role: "",
      rank: "",
      genres: [],
      description: "",
      image: "",
      fate_universe: false
    };
  }

  function sampleData() {
    return [
      { name: "Example Girl", series: "Sample Series", media: "anime", role: "Lead", rank: "Waifu", genres: ["action","romance"], description: "Sample.", image: "", fate_universe: false },
    ];
  }

  // CRUD
  function openAdd() {
    setForm(emptyForm());
    setEditingIndex(null);
    setModalOpen(true);
  }
  function openEdit(idx) {
    setForm({ ...characters[idx], genres: characters[idx].genres || [] });
    setEditingIndex(idx);
    setModalOpen(true);
  }
  function saveForm() {
    if (!form.name.trim()) return alert("Name required");
    const data = { ...form, genres: Array.isArray(form.genres) ? form.genres : (form.genres || "").split(",").map(s=>s.trim()).filter(Boolean) };
    if (editingIndex === null) setCharacters([data, ...characters]);
    else {
      const next = [...characters]; next[editingIndex] = data; setCharacters(next);
    }
    setModalOpen(false);
  }
  function removeCharacter(idx) { if (!confirm("Delete?")) return; setCharacters(characters.filter((_,i)=>i!==idx)); }

  function onFileChange(e) {
    const f = e.target.files && e.target.files[0]; if (!f) return;
    const reader = new FileReader(); reader.onload = () => setForm({ ...form, image: reader.result }); reader.readAsDataURL(f); e.target.value = null;
  }

  // Ordering
  function moveUp(i){ if(i===0) return; const a=[...characters]; [a[i-1],a[i]]=[a[i],a[i-1]]; setCharacters(a); }
  function moveDown(i){ if(i===characters.length-1) return; const a=[...characters]; [a[i+1],a[i]]=[a[i],a[i+1]]; setCharacters(a); }

  // Bulk / Import / Export
  function exportJSON(){ const data=JSON.stringify({ranks,characters},null,2); downloadBlob(data,'gallery.json'); }
  function exportCSV(){ const rows=[['name','series','media','role','rank','genres','fate_universe','description']]; characters.forEach(c=>rows.push([c.name,c.series,c.media,c.role,c.rank,(c.genres||[]).join('|'),c.fate_universe? '1':'0', c.description.replace(/
/g,' ')])); const csv = rows.map(r=>r.map(cell=>`"${(''+cell).replace(/"/g,'""')}"`).join(',')).join('
'); downloadBlob(csv,'gallery.csv'); }
  function downloadBlob(str, filename){ const blob=new Blob([str],{type:'application/octet-stream'}); const href=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=href; a.download=filename; a.click(); URL.revokeObjectURL(href); }
  function importJSON(e){ const f = e.target.files && e.target.files[0]; if(!f) return; const r=new FileReader(); r.onload=()=>{ try{ const parsed=JSON.parse(r.result); if(parsed.ranks) setRanks(parsed.ranks); if(Array.isArray(parsed.characters)) setCharacters(parsed.characters); else setCharacters(parsed); alert('Imported'); }catch(err){ alert('Import error: '+err.message);} }; r.readAsText(f); e.target.value=null; }

  function copyShareableLink(){ try{ const payload = JSON.stringify(characters); const encoded = btoa(encodeURIComponent(payload)); const url = `${window.location.origin}${window.location.pathname}#${encoded}`; navigator.clipboard.writeText(url).then(()=>alert('Link copied')); }catch(e){ alert('Could not create link'); } }

  // Filters
  function toggleFilter(arraySetter, item){ arraySetter(prev => prev.includes(item) ? prev.filter(x=>x!==item) : [...prev, item]); }

  function clearFilters(){ setQuery(''); setSelectedMedia([]); setSelectedGenres([]); setSelectedRanks([]); }

  const allGenres = Array.from(new Set(characters.flatMap(c=>c.genres||[]))).sort();

  const filtered = characters.filter(c=>{
    if(query && !(`${c.name} ${c.series} ${c.description}`.toLowerCase().includes(query.toLowerCase()))) return false;
    if(selectedMedia.length && !selectedMedia.includes(c.media)) return false;
    if(selectedRanks.length && !selectedRanks.includes(c.rank)) return false;
    if(selectedGenres.length && !(c.genres||[]).some(g=>selectedGenres.includes(g))) return false;
    return true;
  });

  // Stats for fate universe & media distribution
  const mediaCounts = MEDIA_TYPES.map(m=>({ name:m, value: characters.filter(c=>c.media===m).length }));
  const rankCounts = ranks.map(r=>({ name:r, value: characters.filter(c=>c.rank===r).length }));
  const fateCount = characters.filter(c=>c.fate_universe).length;

  // Utility: add custom rank
  function addRank(name){ if(!name.trim()) return; if(ranks.includes(name)) return alert('Rank exists'); setRanks([name,...ranks]); }
  function removeRank(r){ if(!confirm('Remove rank from list? existing characters keep the value.')) return; setRanks(ranks.filter(x=>x!==r)); }

  // Print view
  function openPrint(){ const w = window.open('','_blank'); const html = `<html><head><title>Print — Character Gallery</title><style>body{font-family:Arial,Helvetica,sans-serif;padding:20px} .card{border:1px solid #ddd;border-radius:12px;padding:12px;margin:8px;display:inline-block;width:240px;vertical-align:top}</style></head><body>${characters.map(c=>`<div class="card"><h3>${escapeHtml(c.name)}</h3><div>${escapeHtml(c.series)}</div><div>${escapeHtml(c.media)} — ${escapeHtml(c.rank||'')}</div><div>${escapeHtml((c.genres||[]).join(', '))}</div><p>${escapeHtml(c.description||'')}</p></div>`).join('')}</body></html>`; w.document.write(html); w.document.close(); w.print(); }

  function escapeHtml(s=''){ return (''+s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

  return (
    <div className="p-6 max-w-7xl mx-auto">
      <header className="mb-6">
        <h1 className="text-3xl font-extrabold">Ultimate Character Gallery</h1>
        <p className="mt-1 text-sm opacity-80">Ranks, genres, Fate-universe tools, charts, CSV/JSON export, printable view, and shareable links.</p>
      </header>

      <section className="mb-4 flex gap-2 flex-wrap">
        <button onClick={openAdd} className="px-4 py-2 rounded bg-gradient-to-r from-purple-500 to-indigo-500 text-white">Add character</button>
        <button onClick={exportJSON} className="px-3 py-2 rounded border">Export JSON</button>
        <button onClick={exportCSV} className="px-3 py-2 rounded border">Export CSV</button>
        <label className="px-3 py-2 rounded border cursor-pointer">Import JSON<input ref={fileRef} type="file" accept="application/json" onChange={importJSON} className="hidden" /></label>
        <button onClick={copyShareableLink} className="px-3 py-2 rounded border">Copy shareable link</button>
        <button onClick={openPrint} className="px-3 py-2 rounded border">Print view</button>
        <button onClick={()=>{ if(!confirm('Clear all?')) return; setCharacters([]); }} className="px-3 py-2 rounded border text-red-600">Clear all</button>
      </section>

      <section className="grid md:grid-cols-4 gap-6">
        <aside className="md:col-span-1 bg-white/80 rounded-lg p-4 shadow">
          <div className="mb-3">
            <label className="text-xs">Search</label>
            <input value={query} onChange={e=>setQuery(e.target.value)} className="w-full p-2 border rounded mt-1" placeholder="name, series, notes..." />
          </div>

          <div className="mb-3">
            <label className="text-xs">Media</label>
            <div className="flex flex-wrap gap-2 mt-2">
              {MEDIA_TYPES.map(m=> (
                <button key={m} onClick={()=>toggleFilter(setSelectedMedia,m)} className={`px-2 py-1 rounded border ${selectedMedia.includes(m)? 'bg-gray-100':''}`}>{m}</button>
              ))}
            </div>
          </div>

          <div className="mb-3">
            <label className="text-xs">Ranks</label>
            <div className="flex gap-2 flex-wrap mt-2">
              {ranks.map(r=> (
                <button key={r} onClick={()=>toggleFilter(setSelectedRanks,r)} className={`px-2 py-1 rounded border ${selectedRanks.includes(r)? 'bg-gray-100':''}`}>{r}</button>
              ))}
            </div>
            <div className="mt-2 flex gap-2">
              <input placeholder="Add rank" onKeyDown={e=>{ if(e.key==='Enter'){ addRank(e.target.value); e.target.value=''; }}} className="p-1 border rounded text-sm w-full" />
            </div>
          </div>

          <div className="mb-3">
            <label className="text-xs">Genres</label>
            <div className="flex flex-wrap gap-2 mt-2">
              {allGenres.length===0 ? <div className="text-sm opacity-70">No genres yet</div> : allGenres.map(g=> (
                <button key={g} onClick={()=>toggleFilter(setSelectedGenres,g)} className={`px-2 py-1 rounded border ${selectedGenres.includes(g)? 'bg-gray-100':''}`}>{g}</button>
              ))}
            </div>
          </div>

          <div className="mt-4 text-xs opacity-80">
            <div>Fate-universe characters: <strong>{fateCount}</strong></div>
            <div className="mt-2">Quick actions:</div>
            <div className="flex gap-2 mt-2">
              <button onClick={()=>{ const fateOnly=characters.filter(c=>c.fate_universe); setCharacters(fateOnly); }} className="px-2 py-1 border rounded">Show only Fate</button>
              <button onClick={()=>{ setCharacters(characters.sort((a,b)=> (b.rank||'').localeCompare(a.rank||''))); }} className="px-2 py-1 border rounded">Sort by rank</button>
            </div>
          </div>

          <div className="mt-4">
            <button onClick={clearFilters} className="px-3 py-2 rounded border">Clear filters</button>
          </div>
        </aside>

        <main className="md:col-span-3">
          <section className="mb-4 grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div className="bg-white/90 rounded-lg p-3 shadow">
              <h3 className="font-semibold">Media distribution</h3>
              <div style={{height:200}}>
                <ResponsiveContainer width="100%" height={200}>
                  <PieChart>
                    <Pie dataKey="value" data={mediaCounts} outerRadius={70} label>
                      {mediaCounts.map((entry, idx) => <Cell key={idx} fill={COLORS[idx % COLORS.length]} />)}
                    </Pie>
                    <Tooltip />
                    <Legend />
                  </PieChart>
                </ResponsiveContainer>
              </div>
            </div>

            <div className="bg-white/90 rounded-lg p-3 shadow">
              <h3 className="font-semibold">Ranks distribution</h3>
              <div style={{height:200}}>
                <ResponsiveContainer width="100%" height={200}>
                  <PieChart>
                    <Pie dataKey="value" data={rankCounts} outerRadius={70} label>
                      {rankCounts.map((entry, idx) => <Cell key={idx} fill={COLORS[idx % COLORS.length]} />)}
                    </Pie>
                    <Tooltip />
                    <Legend />
                  </PieChart>
                </ResponsiveContainer>
              </div>
            </div>
          </section>

          <section>
            <div className="grid md:grid-cols-2 gap-4">
              {filtered.map((c, idx) => (
                <article key={idx} className="bg-white/90 rounded-2xl shadow p-3 flex gap-3">
                  <div className="w-28 h-28 bg-gray-100 rounded overflow-hidden flex items-center justify-center">
                    {c.image ? <img src={c.image} alt={c.name} className="w-full h-full object-cover" /> : <div className="text-xs opacity-70">No image</div>}
                  </div>
                  <div className="flex-1">
                    <div className="flex justify-between items-start">
                      <div>
                        <h2 className="font-semibold">{c.name}</h2>
                        <div className="text-xs opacity-80">{c.series} • {c.media} • {c.rank}</div>
                      </div>
                      <div className="text-right text-xs opacity-80">{(c.genres||[]).slice(0,3).join(', ')}</div>
                    </div>

                    <p className="mt-2 text-sm line-clamp-3">{c.description}</p>
                    <div className="mt-3 flex gap-2 items-center">
                      <button onClick={()=>openEdit(idx)} className="px-2 py-1 rounded border">Edit</button>
                      <button onClick={()=>removeCharacter(idx)} className="px-2 py-1 rounded border text-red-600">Delete</button>
                      <button onClick={()=>moveUp(idx)} className="px-2 py-1 rounded border">↑</button>
                      <button onClick={()=>moveDown(idx)} className="px-2 py-1 rounded border">↓</button>
                      <div className="ml-auto text-xs opacity-70">Fate: {c.fate_universe ? 'Yes' : 'No'}</div>
                    </div>
                  </div>
                </article>
              ))}

              {filtered.length===0 && <div className="p-6 border rounded-lg text-center opacity-80">No characters match your filters.</div>}
            </div>
          </section>
        </main>
      </section>

      {/* Modal */}
      {isModalOpen && (
        <div className="fixed inset-0 bg-black/50 flex items-center justify-center p-4 z-50">
          <div className="bg-white rounded-2xl w-full max-w-3xl p-6 shadow-lg">
            <div className="flex items-start justify-between">
              <h3 className="text-xl font-semibold">{editingIndex===null ? 'Add character' : 'Edit character'}</h3>
              <button onClick={()=>setModalOpen(false)} className="text-sm opacity-70">Close</button>
            </div>

            <div className="mt-4 grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label className="text-xs">Name</label>
                <input value={form.name} onChange={e=>setForm({...form,name:e.target.value})} className="w-full p-2 border rounded mt-1" />

                <label className="text-xs mt-2 block">Series</label>
                <input value={form.series} onChange={e=>setForm({...form,series:e.target.value})} className="w-full p-2 border rounded mt-1" />

                <label className="text-xs mt-2 block">Media</label>
                <select value={form.media} onChange={e=>setForm({...form,media:e.target.value})} className="w-full p-2 border rounded mt-1">
                  {MEDIA_TYPES.map(m=> <option key={m} value={m}>{m}</option>)}
                </select>

                <label className="text-xs mt-2 block">Rank</label>
                <select value={form.rank} onChange={e=>setForm({...form,rank:e.target.value})} className="w-full p-2 border rounded mt-1">
                  <option value="">(none)</option>
                  {ranks.map(r=> <option key={r} value={r}>{r}</option>)}
                </select>

                <label className="text-xs mt-2 block">Role</label>
                <input value={form.role} onChange={e=>setForm({...form,role:e.target.value})} className="w-full p-2 border rounded mt-1" />

                <label className="text-xs mt-2 block">Fate-universe?</label>
                <div className="flex items-center gap-2 mt-1"><input type="checkbox" checked={form.fate_universe} onChange={e=>setForm({...form,fate_universe:e.target.checked})} /> <span className="text-sm opacity-80">Mark as Fate Universe character</span></div>
              </div>

              <div>
                <label className="text-xs">Genres (type and press Enter to add)</label>
                <GenreInput value={form.genres||[]} onChange={g=>setForm({...form,genres:g})} />

                <label className="text-xs mt-2 block">Description</label>
                <textarea value={form.description} onChange={e=>setForm({...form,description:e.target.value})} className="w-full p-2 border rounded mt-1 h-32"></textarea>

                <label className="text-xs mt-2 block">Image</label>
                <div className="flex gap-2 items-center">
                  <input type="file" accept="image/*" onChange={onFileChange} />
                  {form.image && <button onClick={()=>setForm({...form,image:''})} className="text-sm opacity-80">Remove</button>}
                </div>
                {form.image && <div className="mt-2 w-full h-36 rounded overflow-hidden border"><img src={form.image} alt="preview" className="w-full h-full object-cover" /></div>}
              </div>
            </div>

            <div className="mt-4 flex gap-2 justify-end">
              <button onClick={()=>setModalOpen(false)} className="px-4 py-2 rounded border">Cancel</button>
              <button onClick={saveForm} className="px-4 py-2 rounded bg-indigo-600 text-white">Save</button>
            </div>
          </div>
        </div>
      )}

      <footer className="mt-6 text-sm opacity-80">Tip: use <strong>Copy shareable link</strong> to send your gallery; the link embeds the list so others can open it directly. You can also export CSV/JSON for backups or upload to a personal GitHub gist for hosting.</footer>
    </div>
  );
}

function GenreInput({value = [], onChange}){
  const [text, setText] = useState('');
  useEffect(()=> setText(''), [value]);
  function add(){ if(!text.trim()) return; const next = Array.from(new Set([...(value||[]), text.trim().toLowerCase()])); onChange(next); setText(''); }
  function remove(g){ onChange((value||[]).filter(x=>x!==g)); }
  return (
    <div>
      <div className="flex gap-2">
        <input value={text} onChange={e=>setText(e.target.value)} onKeyDown={e=>{ if(e.key==='Enter'){ e.preventDefault(); add(); } }} className="p-2 border rounded w-full" placeholder="add genre and press Enter" />
        <button onClick={add} className="px-3 py-2 border rounded">Add</button>
      </div>
      <div className="mt-2 flex gap-2 flex-wrap">
        {(value||[]).map(g=> <span key={g} className="px-2 py-1 rounded-full border">{g} <button className="ml-2 text-xs" onClick={()=>remove(g)}>×</button></span>)}
      </div>
    </div>
  );
}

