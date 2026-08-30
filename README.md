<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Debug Dash — squash the bug before it ships</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;700&family=JetBrains+Mono:wght@400;500;700&display=swap" rel="stylesheet">
<style>
  :root{
    --bg: #171225;
    --panel: #211a35;
    --panel-2: #2a2144;
    --gutter: #4a4066;
    --text: #d8d3ee;
    --muted: #8b81ad;
    --pink: #ff6fa5;
    --amber: #ffcc66;
    --mint: #4fd8c4;
    --bug: #ff5f5f;
    --bug-glow: rgba(255, 95, 95, 0.35);
  }
  *{box-sizing:border-box;}
  html,body{height:100%;}
  body{
    margin:0;
    min-height:100vh;
    display:flex;
    align-items:center;
    justify-content:center;
    background:
      radial-gradient(circle at 20% 15%, rgba(255,111,165,0.08), transparent 40%),
      radial-gradient(circle at 85% 80%, rgba(79,216,196,0.08), transparent 40%),
      var(--bg);
    font-family:'JetBrains Mono', monospace;
    color:var(--text);
    padding:24px;
  }
  .display{ font-family:'Space Grotesk', sans-serif; }

  .window{
    width:100%;
    max-width:720px;
    background:var(--panel);
    border-radius:14px;
    overflow:hidden;
    box-shadow: 0 30px 80px -20px rgba(0,0,0,0.6), 0 0 0 1px rgba(255,255,255,0.04);
  }

  .titlebar{
    display:flex;
    align-items:center;
    gap:10px;
    padding:12px 16px;
    background:var(--panel-2);
    border-bottom:1px solid rgba(255,255,255,0.06);
  }
  .dot{width:11px;height:11px;border-radius:50%;}
  .dot.r{background:#ff5f56;}
  .dot.y{background:#ffbd2e;}
  .dot.g{background:#27c93f;}
  .tab{
    margin-left:10px;
    font-size:12.5px;
    color:var(--muted);
    background:rgba(255,255,255,0.04);
    padding:4px 10px;
    border-radius:6px;
  }

  .statbar{
    display:flex;
    justify-content:space-between;
    align-items:center;
    padding:12px 18px;
    font-size:13px;
    color:var(--muted);
    border-bottom:1px dashed rgba(255,255,255,0.08);
  }
  .statbar b{ color:var(--text); font-weight:600; }
  .lives span{ filter:grayscale(0); margin-left:2px; }
  .combo{ color:var(--amber); }

  .editor{
    position:relative;
    min-height:340px;
    padding:14px 0;
  }
  .row{
    display:flex;
    align-items:center;
    height:44px;
    padding:0 18px;
    font-size:14.5px;
    white-space:pre;
    position:relative;
  }
  .lineno{
    width:28px;
    color:var(--gutter);
    user-select:none;
    flex-shrink:0;
    font-size:13px;
  }
  .code{ opacity:0; transition:opacity .15s ease; }
  .row.active .code{ opacity:1; }

  .bug{
    background:var(--bug-glow);
    color:var(--bug);
    padding:1px 4px;
    border-radius:4px;
    cursor:pointer;
    font-weight:700;
    animation: pulse 0.9s ease-in-out infinite;
    box-shadow: 0 0 0 1px rgba(255,95,95,0.4);
  }
  @keyframes pulse{
    0%,100%{ box-shadow:0 0 0 0 var(--bug-glow); }
    50%{ box-shadow:0 0 0 6px rgba(255,95,95,0); }
  }
  .bug.squashed{
    animation:none;
    background:rgba(79,216,196,0.25);
    color:var(--mint);
    text-decoration:line-through;
  }
  .bug.shipped{
    animation:none;
    background:rgba(255,95,95,0.55);
    color:#fff;
  }

  .timerbar{
    position:absolute;
    left:18px; right:18px; bottom:2px;
    height:2px;
    background:rgba(255,255,255,0.08);
    border-radius:2px;
    overflow:hidden;
  }
  .timerbar i{
    display:block; height:100%;
    background:var(--pink);
    width:100%;
    transform-origin:left;
  }

  .overlay{
    position:absolute; inset:0;
    background:rgba(15,11,26,0.92);
    display:flex; flex-direction:column;
    align-items:center; justify-content:center;
    text-align:center;
    padding:32px;
    gap:14px;
  }
  .overlay h1{ font-family:'Space Grotesk', sans-serif; font-size:26px; margin:0; }
  .overlay p{ color:var(--muted); margin:0; max-width:380px; line-height:1.6; font-size:13.5px; }
  .btn{
    font-family:'Space Grotesk', sans-serif;
    font-weight:700;
    background:var(--pink);
    color:#1a0f18;
    border:none;
    padding:12px 26px;
    border-radius:8px;
    cursor:pointer;
    font-size:14px;
    letter-spacing:0.02em;
    transition: transform .15s ease;
  }
  .btn:hover{ transform:translateY(-2px); }
  .btn:active{ transform:translateY(0); }

  .stage{ position:relative; }
  .hidden{ display:none !important; }

  .badge-row{ display:flex; gap:10px; }
  .kbd{
    font-size:11px; color:var(--muted); background:rgba(255,255,255,0.06);
    padding:2px 6px; border-radius:4px;
  }
</style>
</head>
<body>

<div class="window">
  <div class="titlebar">
    <div class="dot r"></div><div class="dot y"></div><div class="dot g"></div>
    <div class="tab">debug_dash.js — unsaved</div>
  </div>

  <div class="statbar">
    <div>SCORE <b id="score">0</b> &nbsp;·&nbsp; BEST <b id="best">0</b></div>
    <div class="combo" id="combo">combo x1</div>
    <div class="lives" id="lives">❤️❤️❤️</div>
  </div>

  <div class="editor stage" id="editor">
    <!-- rows injected by JS -->

    <div class="overlay" id="startOverlay">
      <h1 class="display">Debug Dash</h1>
      <p>Bugs are sneaking into the codebase. Click each glowing token before its timer runs out — miss three and it ships to prod. Gets faster as your score climbs.</p>
      <button class="btn" id="startBtn">Start Debugging</button>
      <div class="badge-row"><span class="kbd">JS</span><span class="kbd">Python</span><span class="kbd">CSS</span><span class="kbd">SQL</span><span class="kbd">git</span></div>
    </div>

    <div class="overlay hidden" id="endOverlay">
      <h1 class="display">Shipped too many bugs 🚨</h1>
      <p id="finalScoreText">Final score: 0</p>
      <button class="btn" id="retryBtn">Try Again</button>
    </div>
  </div>
</div>

<script>
(function(){
  const ROWS = 5;
  const LIVES_START = 3;
  const HIGH_SCORE_KEY = 'debugDashHighScore';

  const pool = [
    { plain: 'array.', bug: 'lenght', rest: '' },
    { plain: 'if (a ', bug: '=', rest: ' b) {' },
    { plain: 'for (let i = 0; i < 10; ', bug: 'i--', rest: ') {' },
    { plain: 'def foo():\n  ', bug: 'retrun', rest: ' x' },
    { plain: 'h1 { ', bug: 'colour', rest: ': red; }' },
    { plain: 'SELECT * ', bug: 'FORM', rest: ' users;' },
    { plain: "import ", bug: 'Reactt', rest: " from 'react';" },
    { plain: '<div ', bug: 'clas', rest: '="container">' },
    { plain: 'while (true) { ', bug: 'brake', rest: '; }' },
    { plain: 'function sum(a,b){ ', bug: 'retun', rest: ' a+b }' },
    { plain: 'git commit -m "fix"; ', bug: 'got', rest: ' push' },
    { plain: 'npm ', bug: 'instal', rest: ' express' },
    { plain: '#header { ', bug: 'widht', rest: ': 100%; }' },
    { plain: 'WHERE id ', bug: '==', rest: ' 1;' },
    { plain: 'const ', bug: 'lett', rest: ' x = 5;' },
  ];

  const editor = document.getElementById('editor');
  const scoreEl = document.getElementById('score');
  const bestEl = document.getElementById('best');
  const comboEl = document.getElementById('combo');
  const livesEl = document.getElementById('lives');
  const startOverlay = document.getElementById('startOverlay');
  const endOverlay = document.getElementById('endOverlay');
  const finalScoreText = document.getElementById('finalScoreText');
  const startBtn = document.getElementById('startBtn');
  const retryBtn = document.getElementById('retryBtn');

  let rows = [];
  let score = 0, lives = LIVES_START, combo = 1;
  let spawnTimer = null;
  let running = false;
  let best = parseInt(localStorage.getItem(HIGH_SCORE_KEY) || '0', 10);
  bestEl.textContent = best;

  function buildRows(){
    for(let i=0;i<ROWS;i++){
      const row = document.createElement('div');
      row.className = 'row';
      row.innerHTML = `<span class="lineno">${i+10}</span><span class="code"></span><div class="timerbar"><i></i></div>`;
      editor.insertBefore(row, startOverlay);
      rows.push({ el: row, codeEl: row.querySelector('.code'), bar: row.querySelector('.timerbar i'), busy:false, timeoutId:null, animId:null });
    }
  }

  function pickFreeRow(){
    const free = rows.filter(r => !r.busy);
    if(!free.length) return null;
    return free[Math.floor(Math.random()*free.length)];
  }

  function difficulty(){
    const spawnMs = Math.max(650, 1500 - score*8);
    const liveMs = Math.max(1400, 4200 - score*15);
    return { spawnMs, liveMs };
  }

  function spawnBug(){
    if(!running) return;
    const row = pickFreeRow();
    const { liveMs } = difficulty();
    if(row){
      const line = pool[Math.floor(Math.random()*pool.length)];
      row.busy = true;
      row.el.classList.add('active');
      row.codeEl.innerHTML = `${escapeHtml(line.plain)}<span class="bug" data-row="${rows.indexOf(row)}">${escapeHtml(line.bug)}</span>${escapeHtml(line.rest)}`;
      row.bar.style.transition = 'none';
      row.bar.style.transform = 'scaleX(1)';
      requestAnimationFrame(()=>{
        row.bar.style.transition = `transform ${liveMs}ms linear`;
        row.bar.style.transform = 'scaleX(0)';
      });

      const bugSpan = row.codeEl.querySelector('.bug');
      bugSpan.addEventListener('click', () => squash(row));

      row.timeoutId = setTimeout(() => shipIt(row), liveMs);
    }
    const { spawnMs } = difficulty();
    spawnTimer = setTimeout(spawnBug, spawnMs);
  }

  function escapeHtml(str){
    return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  }

  function squash(row){
    if(!row.busy || !running) return;
    clearTimeout(row.timeoutId);
    const bugSpan = row.codeEl.querySelector('.bug');
    if(bugSpan) bugSpan.classList.add('squashed');
    score += 10 * combo;
    combo += 1;
    scoreEl.textContent = score;
    comboEl.textContent = `combo x${combo}`;
    freeRowSoon(row);
  }

  function shipIt(row){
    if(!row.busy) return;
    const bugSpan = row.codeEl.querySelector('.bug');
    if(bugSpan) bugSpan.classList.add('shipped');
    combo = 1;
    comboEl.textContent = `combo x${combo}`;
    lives -= 1;
    renderLives();
    freeRowSoon(row);
    if(lives <= 0){
      endGame();
    }
  }

  function freeRowSoon(row){
    setTimeout(() => {
      row.el.classList.remove('active');
      row.busy = false;
      row.codeEl.innerHTML = '';
    }, 380);
  }

  function renderLives(){
    livesEl.textContent = '❤️'.repeat(Math.max(lives,0)) + '🖤'.repeat(LIVES_START - Math.max(lives,0));
  }

  function startGame(){
    score = 0; lives = LIVES_START; combo = 1;
    scoreEl.textContent = score;
    comboEl.textContent = 'combo x1';
    renderLives();
    rows.forEach(r => { r.busy=false; r.el.classList.remove('active'); r.codeEl.innerHTML=''; clearTimeout(r.timeoutId); });
    startOverlay.classList.add('hidden');
    endOverlay.classList.add('hidden');
    running = true;
    spawnBug();
  }

  function endGame(){
    running = false;
    clearTimeout(spawnTimer);
    rows.forEach(r => clearTimeout(r.timeoutId));
    if(score > best){
      best = score;
      localStorage.setItem(HIGH_SCORE_KEY, String(best));
    }
    bestEl.textContent = best;
    finalScoreText.textContent = `Final score: ${score} — best: ${best}`;
    endOverlay.classList.remove('hidden');
  }

  buildRows();
  renderLives();
  startBtn.addEventListener('click', startGame);
  retryBtn.addEventListener('click', startGame);
})();
</script>
</body>
</html>
