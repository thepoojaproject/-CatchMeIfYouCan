
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Pixel Collector</title>
<style>
  :root{
    --bg:#111; --fg:#ddd; --accent:#ffd166; --danger:#ef233c;
  }
  html,body{height:100%;margin:0;background:var(--bg);color:var(--fg);font-family:system-ui,Segoe UI,Roboto,Arial}
  .wrap{height:100vh;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:12px}
  canvas{image-rendering:pixelated;border:6px solid #222;background:#000;display:block}
  .ui{display:flex;gap:12px;align-items:center;color:var(--fg);font-weight:600}
  .small{font-size:0.9rem;color:#bbb}
  button{background:#222;color:var(--fg);border:1px solid #333;padding:6px 10px;border-radius:6px;cursor:pointer}
  .hint{color:#888;font-size:0.9rem;margin-top:6px}
  @media (max-width:520px){
    canvas{width:320px;height:320px}
  }

  /* footer / badge */
  .madeby{
    position:fixed;
    right:12px;
    bottom:8px;
    font-size:12px;
    color:#ddd;
    background:rgba(255,255,255,0.03);
    padding:6px 8px;
    border-radius:8px;
    border:1px solid rgba(255,255,255,0.03);
    pointer-events:none;
    user-select:none;
    font-weight:700;
    letter-spacing:0.2px;
    display:flex;
    gap:6px;
    align-items:center;
  }

  /* optional: accessible hidden footer for screen-readers (keeps visual UI clean) */
  .sr-only {
    position: absolute !important;
    height: 1px; width: 1px;
    overflow: hidden;
    clip: rect(1px, 1px, 1px, 1px);
    white-space: nowrap;
  }
</style>
</head>
<body>
  <div class="wrap">
    <div class="ui">
      <div>Score: <span id="score">0</span></div>
      <div class="small">Lives: <span id="lives">3</span></div>
      <div class="small">Level: <span id="level">1</span></div>
      <button id="restart">Restart (R)</button>
      <button id="pause">Pause (Space)</button>
    </div>

    <!-- Pixel canvas (logical size 256 × 256 for crisp pixels) -->
    <canvas id="game" width="256" height="256"></canvas>

    <div class="hint">Move: ← ↑ → ↓ or WASD — Collect coins, avoid enemies</div>
  </div>


  <!-- visible to screen readers (improves accessibility) -->
  <div class="sr-only">Made in heart by Armeen</div>

<script>
/* Pixel Collector — simple pixel-style canvas game
   - grid = 16px tiles on a 256x256 canvas (16x16 tiles)
   - player, coins, enemies drawn with solid rects (no images)
*/

// ---- Config ----
const TILE = 16;
const TILES = 16;
const CANVAS_SIZE = TILE * TILES; // 256
const PLAYER_COLOR = '#ffffff';
const COIN_COLOR = '#ffd166';
const ENEMY_COLOR = '#ef233c';
const BG_COLOR = '#000000';

// ---- Canvas setup ----
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
ctx.imageSmoothingEnabled = false;

// scale canvas for bigger display but keep pixelated look by CSS (handled)
function clear() {
  ctx.fillStyle = BG_COLOR;
  ctx.fillRect(0,0,CANVAS_SIZE,CANVAS_SIZE);
}

// ---- Game state ----
let score = 0;
let lives = 3;
let level = 1;
let running = true;
let paused = false;

// DOM refs
const scoreEl = document.getElementById('score');
const livesEl = document.getElementById('lives');
const levelEl = document.getElementById('level');
const restartBtn = document.getElementById('restart');
const pauseBtn = document.getElementById('pause');

// Player (grid coords)
const player = {
  x: Math.floor(TILES/2),
  y: Math.floor(TILES/2),
  speed: 6,      // pixels per second (we implement smooth lerp)
  size: TILE-2,
  px: 0, py: 0,  // pixel position for smooth movement
  targetX: null, targetY: null
};

// Input
const keys = {};
window.addEventListener('keydown', e=>{ keys[e.key.toLowerCase()] = true; if(e.key===' ' ) { togglePause(); e.preventDefault(); }});
window.addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });

// Coins and enemies
let coin = null;
let enemies = [];

// --- Utilities ---
function randTile(){ return Math.floor(Math.random()*TILES); }
function placeCoin(){
  // avoid placing on player or enemy
  let tries=0;
  while(tries<200){
    const x=randTile(), y=randTile();
    if ((x===player.x && y===player.y) || enemies.some(en=>Math.round(en.x)===x && Math.round(en.y)===y)) { tries++; continue; }
    coin = {x,y};
    return;
  }
  coin = {x:0,y:0};
}
function addEnemy(){
  // spawn at edge, not on player
  const side = Math.random() < 0.5 ? 'h' : 'v';
  let ex, ey;
  if(side==='h'){
    ex = Math.random()<0.5?0:TILES-1;
    ey = randTile();
  } else {
    ey = Math.random()<0.5?0:TILES-1;
    ex = randTile();
  }
  // avoid immediate collision
  if (ex===player.x && ey===player.y) return;
  enemies.push({x:ex,y:ey,spd: 0.6 + Math.random()*0.6}); // speed is grid per second-ish
}

// ---- Game init ----
function resetGame(){
  score = 0; lives = 3; level = 1; running = true; paused=false;
  player.x = Math.floor(TILES/2); player.y = Math.floor(TILES/2);
  player.px = player.x*TILE; player.py = player.y*TILE;
  enemies = [];
  placeCoin();
  enemies.push({x:0,y:0,spd:0.5});
  updateUI();
}

// ---- Update UI ----
function updateUI(){
  scoreEl.textContent = score;
  livesEl.textContent = lives;
  levelEl.textContent = level;
}

// ---- Movement helpers ----
function tryMovePlayer(dx,dy){
  const nx = Math.floor(player.x) + dx, ny = Math.floor(player.y) + dy;
  if (nx < 0 || ny < 0 || nx >= TILES || ny >= TILES) return;
  player.targetX = nx; player.targetY = ny;
}

// ---- Game loop ----
let last = 0;
function gameTick(ts){
  if(!running) return;
  if(!last) last = ts;
  const dt = (ts-last)/1000; // seconds
  last = ts;
  if(!paused){
    handleInput();
    updateEntities(dt);
    checkCollisions();
  }
  render();
  requestAnimationFrame(gameTick);
}

// handle keyboard input (grid moves)
let moveCooldown = 0;
function handleInput(){
  // simple grid-step input with short cooldown so holding doesn't spam too fast
  moveCooldown -= 1/60;
  if(moveCooldown>0) return;
  const left = keys['arrowleft'] || keys['a'];
  const right = keys['arrowright'] || keys['d'];
  const up = keys['arrowup'] || keys['w'];
  const down = keys['arrowdown'] || keys['s'];
  if(left){ tryMovePlayer(-1,0); moveCooldown = 0.12; }
  else if(right){ tryMovePlayer(1,0); moveCooldown = 0.12; }
  else if(up){ tryMovePlayer(0,-1); moveCooldown = 0.12; }
  else if(down){ tryMovePlayer(0,1); moveCooldown = 0.12; }
}

// Update enemies & smooth player pixel position
function updateEntities(dt){
  // player smooth interpolation toward target tile
  if(player.targetX !== null){
    const tx = player.targetX * TILE;
    const ty = player.targetY * TILE;
    const lerpSpeed = 14; // pixels per frame smoothing strength
    // snap if close
    const dx = tx - player.px, dy = ty - player.py;
    const dist = Math.hypot(dx,dy);
    if(dist < 1){
      player.px = tx; player.py = ty;
      player.x = player.targetX; player.y = player.targetY;
      player.targetX = player.targetY = null;
    } else {
      const step = Math.min(dist, lerpSpeed * (1/60) * 60); // keep consistent
      player.px += dx * (step/dist);
      player.py += dy * (step/dist);
    }
  } else {
    // ensure pixel pos equals grid
    player.px = player.x * TILE;
    player.py = player.y * TILE;
  }

  // enemies chase player tile-by-tile using simple velocity
  for(const en of enemies){
    const dx = player.x - en.x;
    const dy = player.y - en.y;
    if(Math.abs(dx) > Math.abs(dy)){ en.x += Math.sign(dx)*Math.min(1, en.spd*dt*2); }
    else { en.y += Math.sign(dy)*Math.min(1, en.spd*dt*2); }
    // clamp and round to grid for collision checks (we allow fractional but check proximity)
    en.x = Math.max(0, Math.min(TILES-1, en.x));
    en.y = Math.max(0, Math.min(TILES-1, en.y));
  }
}

// collisions with coin/enemy
function checkCollisions(){
  // coin
  if(coin && Math.round(player.x) === coin.x && Math.round(player.y) === coin.y){
    score += 10;
    placeCoin();
    // every 30 points spawn an enemy & increase level
    if(score % 30 === 0){
      addEnemy();
      level = Math.min(99, 1 + Math.floor(score/30));
    }
    updateUI();
  }

  // enemies: if any enemy is within 0.6 tile of player => hit
  for(const en of enemies){
    if(Math.hypot(en.x - player.x, en.y - player.y) < 0.6){
      // lose life and respawn player to center
      lives -= 1;
      updateUI();
      player.x = Math.floor(TILES/2); player.y = Math.floor(TILES/2);
      player.px = player.x*TILE; player.py = player.y*TILE;
      player.targetX = player.targetY = null;
      // small knockback: move enemy a bit away
      en.x = Math.max(0, Math.min(TILES-1, en.x + (en.x < player.x ? -1 : 1)));
      en.y = Math.max(0, Math.min(TILES-1, en.y + (en.y < player.y ? -1 : 1)));
      if(lives <= 0){
        // game over
        running = false;
        setTimeout(()=>showGameOver(), 200);
      }
      break;
    }
  }
}

// show game over overlay (simple)
function showGameOver(){
  paused = true;
  ctx.fillStyle = 'rgba(0,0,0,0.6)';
  ctx.fillRect(0,0,CANVAS_SIZE,CANVAS_SIZE);
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 18px monospace';
  ctx.textAlign = 'center';
  ctx.fillText('GAME OVER', CANVAS_SIZE/2, CANVAS_SIZE/2 - 6);
  ctx.font = '12px monospace';
  ctx.fillText('Press R to restart', CANVAS_SIZE/2, CANVAS_SIZE/2 + 14);
}

// ---- Render ----
function render(){
  clear();
  // grid subtle (optional) - keep minimal: small dotted grid
  ctx.fillStyle = '#0a0a0a';
  for(let gx=0;gx<TILES;gx++){
    for(let gy=0;gy<TILES;gy++){
      if((gx+gy)%2===0) ctx.fillRect(gx*TILE, gy*TILE, TILE, TILE);
    }
  }

  // coin
  if(coin){
    ctx.fillStyle = COIN_COLOR;
    const cx = coin.x * TILE + 3;
    const cy = coin.y * TILE + 3;
    ctx.fillRect(cx, cy, TILE-6, TILE-6);
  }

  // enemies
  for(const en of enemies){
    ctx.fillStyle = ENEMY_COLOR;
    // en.x/y can be fractional; draw centered rect scaled to TILE-4
    const ex = Math.round(en.x * TILE + 1);
    const ey = Math.round(en.y * TILE + 1);
    ctx.fillRect(ex, ey, TILE-2, TILE-2);
  }

  // player
  ctx.fillStyle = PLAYER_COLOR;
  ctx.fillRect(Math.round(player.px)+1, Math.round(player.py)+1, player.size, player.size);
}

// ---- Controls: restart & pause ----
function togglePause(){
  if(!running) return;
  paused = !paused;
  pauseBtn.textContent = paused ? 'Resume (Space)' : 'Pause (Space)';
}
pauseBtn.onclick = togglePause;
restartBtn.onclick = ()=>{ 
  running = true; paused=false; last=0;
  resetGame();
  requestAnimationFrame(gameTick);
};

// Keyboard R for restart
window.addEventListener('keydown', e=>{
  if(e.key.toLowerCase() === 'r'){ resetGame(); last=0; if(!running){ running = true; } requestAnimationFrame(gameTick); }
});

// start
resetGame();
requestAnimationFrame(gameTick);
</script>
<footer style="
    text-align: center;
    padding: 15px;
    background: #fff;
    color: #000;
    font-family: Arial, sans-serif;
    font-size: 16px;
">
    Made in <span style="color: red;">❤</span> By Armeen
</footer>

</body>
</html>
# -CatchMeIfYouCan
