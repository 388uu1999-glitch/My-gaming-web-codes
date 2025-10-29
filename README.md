# My-gaming-web-codes
codes for HTML
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Gaming News — HTML Aggregator</title>
  <style>
    :root{--bg:#0f1724;--card:#0b1220;--muted:#94a3b8;--accent:#7c3aed}
    *{box-sizing:border-box}
    body{margin:0;font-family:Inter,ui-sans-serif,system-ui,-apple-system,"Segoe UI",Roboto,'Helvetica Neue';background:linear-gradient(180deg,#020617 0%, #071029 100%);color:#e6eef8}
    header{padding:20px;display:flex;gap:16px;align-items:center;justify-content:space-between}
    h1{margin:0;font-size:20px}
    .controls{display:flex;gap:8px;align-items:center}
    input[type=text]{padding:8px 10px;border-radius:8px;border:1px solid rgba(255,255,255,0.06);backdrop-filter:blur(4px);background:rgba(255,255,255,0.02);color:inherit}
    button{padding:8px 12px;border-radius:10px;border:0;background:var(--accent);color:white;cursor:pointer}
    .yt-button{background:red;color:white;font-weight:600;text-decoration:none;padding:8px 14px;border-radius:10px;display:inline-block;margin-top:6px}
    .yt-button.copied{background:green;}
    main{padding:18px;display:grid;grid-template-columns:1fr 360px;gap:18px}
    .feed{background:var(--card);padding:12px;border-radius:12px;min-height:60vh;border:1px solid rgba(255,255,255,0.03)}
    .sidebar{background:transparent;padding:12px}
    .sources{display:flex;flex-direction:column;gap:8px}
    .source-item{display:flex;justify-content:space-between;align-items:center;padding:8px;border-radius:8px;background:rgba(255,255,255,0.02)}
    .article{display:flex;gap:10px;padding:12px;border-radius:10px;background:linear-gradient(180deg, rgba(255,255,255,0.01), rgba(255,255,255,0.00));margin-bottom:10px}
    .thumb{width:120px;height:80px;background:#0b1220;border-radius:8px;object-fit:cover}
    .meta{font-size:12px;color:var(--muted)}
    .title{font-weight:600;margin:4px 0}
    .summary{color:var(--muted);font-size:13px}
    footer{padding:10px;text-align:center;color:var(--muted);font-size:12px}
    @media (max-width:900px){main{grid-template-columns:1fr} .thumb{display:none}}
  </style>
</head>
<body>
  <header>
    <div>
      <h1>Gaming News — HTML Aggregator</h1>
      <div style="font-size:13px;color:var(--muted)">Aggregates public RSS feeds (client-side)</div>
      <div style="font-size:12px;color:var(--muted)"><em>by Mystrey45</em></div>
      <button id="copyLinkBtn" class="yt-button">Copy My YouTube Link</button>
    </div>
    <div class="controls">
      <input id="newSrc" type="text" placeholder="Add RSS feed URL or site URL" style="width:340px" />
      <button id="addBtn">Add</button>
      <button id="refreshBtn">Refresh</button>
    </div>
  </header>

  <main>
    <section class="feed" id="feed">
      <div id="status" style="margin-bottom:12px;color:var(--muted)">Loading...</div>
      <div id="articles"></div>
    </section>

    <aside class="sidebar">
      <div style="margin-bottom:12px;font-weight:600">Sources</div>
      <div class="sources" id="sourcesList"></div>

      <div style="margin-top:18px;font-weight:600">Options</div>
      <div style="margin-top:8px;color:var(--muted);font-size:13px">Uses <code>api.allorigins.win</code> to bypass CORS. If a source blocks fetching or returns empty, try another RSS or proxy.</div>
    </aside>
  </main>

  <footer>Tip: save this file as <code>gaming-news.html</code> and open in a browser. To run a production feed, use a server to avoid CORS limits.</footer>

<script>
// Copy YouTube link functionality
const copyButton = document.getElementById('copyLinkBtn');
const ytLink = 'https://www.youtube.com/@MysteryAnimation-t8q4y';

copyButton.addEventListener('click', async () => {
  try {
    await navigator.clipboard.writeText(ytLink);
    copyButton.textContent = 'Copied!';
    copyButton.classList.add('copied');
    setTimeout(() => {
      copyButton.textContent = 'Copy My YouTube Link';
      copyButton.classList.remove('copied');
    }, 2000);
  } catch (err) {
    alert('Failed to copy link: ' + err);
  }
});

// Default gaming RSS sources (editable)
const defaultSources = [
  {name: 'IGN - Games', url: 'https://feeds.ign.com/ign/games-all'},
  {name: 'GameSpot - News', url: 'https://www.gamespot.com/feeds/news/'},
  {name: 'PC Gamer', url: 'https://www.pcgamer.com/rss/'},
  {name: 'Kotaku', url: 'https://kotaku.com/rss'},
  {name: 'Eurogamer', url: 'https://www.eurogamer.net/?format=rss'}
];

const STORAGE_KEY = 'gaming_news_sources_v1';
let sources = loadSources();

function loadSources(){
  try{
    const raw = localStorage.getItem(STORAGE_KEY);
    if(!raw) { localStorage.setItem(STORAGE_KEY, JSON.stringify(defaultSources)); return defaultSources }
    return JSON.parse(raw);
  }catch(e){console.warn('loadSources',e); localStorage.removeItem(STORAGE_KEY); return defaultSources}
}

function saveSources(){ localStorage.setItem(STORAGE_KEY, JSON.stringify(sources)) }

function renderSources(){
  const el = document.getElementById('sourcesList'); el.innerHTML='';
  sources.forEach((s,idx)=>{
    const div = document.createElement('div'); div.className='source-item';
    div.innerHTML = `<div style="font-size:13px">${escapeHtml(s.name || s.url)}</div><div style='display:flex;gap:8px'><button data-idx='${idx}' class='view'>View</button><button data-idx='${idx}' class='del'>✕</button></div>`;
    el.appendChild(div);
  })
  el.querySelectorAll('.del').forEach(b=>b.addEventListener('click',e=>{sources.splice(+e.currentTarget.dataset.idx,1); saveSources(); renderSources(); fetchAll();}))
  el.querySelectorAll('.view').forEach(b=>b.addEventListener('click',e=>{const s=sources[+e.currentTarget.dataset.idx]; fetchAndShow(s);}))
}

// helpers
function escapeHtml(str){return String(str).replace(/[&<>"']/g,c=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":"&#39;" })[c])}
function timeAgo(d){ try{const diff=(Date.now()-new Date(d).getTime())/1000; if(diff<60) return Math.floor(diff)+'s ago'; if(diff<3600) return Math.floor(diff/60)+'m ago'; if(diff<86400) return Math.floor(diff/3600)+'h ago'; return Math.floor(diff/86400)+'d ago'; }catch(e){return ''} }

async function fetchAll(){
  document.getElementById('status').textContent = 'Refreshing...';
  const promises = sources.map(s => fetchFeed(s)).map(p => p.catch(e=>({error:e.message})))
  const results = await Promise.all(promises);
  const items = [];
  results.forEach((r,i)=>{
    if(r && r.items) r.items.forEach(it=>items.push({...it, _source: sources[i].name || sources[i].url}))
  })
  items.sort((a,b)=> new Date(b.pubDate || b.isoDate || b.date || 0) - new Date(a.pubDate || a.isoDate || a.date || 0))
  renderArticles(items);
  document.getElementById('status').textContent = `Showing ${items.length} articles from ${sources.length} sources`;
}

async function fetchAndShow(source){
  document.getElementById('status').textContent = `Loading ${source.name || source.url}...`;
  try{
    const res = await fetchFeed(source);
    renderArticles((res.items||[]).map(it=>({...it,_source:source.name||source.url})))
    document.getElementById('status').textContent = `Showing ${res.items.length} articles from ${escapeHtml(source.name||source.url)}`;
  }catch(e){document.getElementById('status').textContent = 'Error: '+e.message}
}

async function fetchFeed(source){
  const endpoint = 'https://api.allorigins.win/raw?url=' + encodeURIComponent(source.url);
  const resp = await fetch(endpoint);
  if(!resp.ok) throw new Error('Failed to fetch '+source.url+' ('+resp.status+')');
  const text = await resp.text();
  const doc = new DOMParser().parseFromString(text,'application/xml');
  if(doc.querySelector('parsererror')){
    throw new Error('Received invalid XML from '+source.url);
  }
  const items = [];
  const rssItems = Array.from(doc.querySelectorAll('item'));
  if(rssItems.length){
    rssItems.forEach(item=>{
      const title = item.querySelector('title')?.textContent || '';
      const link = item.querySelector('link')?.textContent || item.querySelector('guid')?.textContent || '';
      const desc = item.querySelector('description')?.textContent || item.querySelector('summary')?.textContent || '';
      const pubDate = item.querySelector('pubDate')?.textContent || item.querySelector('dc\:date')?.textContent || '';
      let img = '';
      const media = item.querySelector('media\:content, enclosure, image');
      if(media) img = media.getAttribute('url') || media.textContent || '';
      items.push({title, link, summary:desc, pubDate, image:img});
    })
    return {items}
  }
  const atomEntries = Array.from(doc.querySelectorAll('entry'));
  if(atomEntries.length){
    atomEntries.forEach(entry=>{
      const title = entry.querySelector('title')?.textContent || '';
      const link = entry.querySelector('link')?.getAttribute('href') || entry.querySelector('id')?.textContent || '';
      const summary = entry.querySelector('summary')?.textContent || entry.querySelector('content')?.textContent || '';
      const pubDate = entry.querySelector('updated')?.textContent || entry.querySelector('published')?.textContent || '';
      items.push({title, link, summary, pubDate});
    })
    return {items}
  }
  return {items:[]}
}

function renderArticles(items){
  const container = document.getElementById('articles'); container.innerHTML='';
  if(items.length===0){ container.innerHTML = '<div style="color:var(--muted)">No articles found.</div>'; return }
  items.forEach(it=>{
    const a = document.createElement('a'); a.href = it.link || '#'; a.target='_blank'; a.rel='noopener noreferrer';
    a.className='article';
    const thumb = document.createElement('img'); thumb.className='thumb'; thumb.src = it.image || '';
    thumb.onerror = ()=>{thumb.style.display='none'};
    const meta = document.createElement('div'); meta.style.flex='1';
    const title = document.createElement('div'); title.className='title'; title.innerHTML = escapeHtml(it.title || '(no title)');
    const src = document.createElement('div'); src.className='meta'; src.textContent = `${it._source || ''} • ${it.pubDate ? timeAgo(it.pubDate) : ''}`;
    const summary = document.createElement('div'); summary.className='summary'; summary.innerHTML = (it.summary||'').replace(/<[^>]+>/g,'').slice(0,320) + ( (it.summary||'').length>320 ? '…' : '');
    meta.appendChild(title); meta.appendChild(src); meta.appendChild(summary);
    a.appendChild(thumb); a.appendChild(meta);
    container.appendChild(a);
  })
}

document.getElementById('addBtn').addEventListener('click',()=>{
  const v = (document.getElementById('newSrc').value||'').trim(); if(!v) return;
  const name = v.replace(/https?:\/\//,'').split('/')[0];
  sources.unshift({name, url:v}); saveSources(); renderSources(); fetchAll(); document.getElementById('newSrc').value='';
})

document.getElementById('refreshBtn').addEventListener('click',fetchAll);

renderSources(); fetchAll();
</script>
</body>
</html>
