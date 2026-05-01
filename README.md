<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Super Mario Bros</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #000; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; overflow: hidden; font-family: monospace; }
  #wrapper { position: relative; width: 800px; height: 480px; overflow: hidden; }
  canvas { display: block; image-rendering: pixelated; background: #5c94fc; }
  #ui {
    position: absolute; top: 8px; left: 0; right: 0;
    display: flex; justify-content: space-around;
    color: #fff; font-size: 13px; font-weight: bold;
    text-shadow: 2px 2px #000; pointer-events: none;
    z-index: 10;
  }
  #overlay {
    position: absolute; inset: 0; display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    background: rgba(0,0,0,0.78); color: #fff; text-align: center;
    z-index: 20;
  }
  #overlay h1 { font-size: 30px; color: #ffd700; text-shadow: 3px 3px #c00; margin-bottom: 12px; letter-spacing: 3px; }
  #overlay .sub { font-size: 13px; margin: 4px 0; color: #ccc; }
  #overlay button {
    margin-top: 20px; padding: 12px 32px; font-size: 16px;
    background: #e52; color: #fff; border: 4px solid #fff;
    cursor: pointer; font-family: monospace; font-weight: bold;
    letter-spacing: 1px; border-radius: 4px;
  }
  #overlay button:hover { background: #f74; }
  #msg {
    position: absolute; top: 44%; left: 50%; transform: translate(-50%, -50%);
    color: #ffd700; font-size: 20px; font-weight: bold;
    text-shadow: 2px 2px #000; font-family: monospace;
    pointer-events: none; opacity: 0; transition: opacity 0.4s;
    z-index: 15; white-space: nowrap;
  }
  #controls {
    margin-top: 10px; color: #888; font-size: 12px; text-align: center;
  }
</style>
</head>
<body>
<div id="wrapper">
  <canvas id="c" width="800" height="480"></canvas>
  <div id="ui">
    <div>SCORE: <span id="sc">0</span></div>
    <div>LIVES: <span id="lv">3</span></div>
    <div>COINS: <span id="cn">0</span></div>
    <div>LEVEL: <span id="lv2">1</span></div>
  </div>
  <div id="msg"></div>
  <div id="overlay" id="overlay">
    <h1>SUPER Mario BROS</h1>
    <p class="sub">Arrow Keys / WASD — Move &amp; Run</p>
    <p class="sub">Space / Up / W — Jump</p>
    <p class="sub">Stomp enemies to defeat them!</p>
    <p class="sub">Hit ? blocks for coins &amp; power-ups</p>
    <button id="startBtn">▶ START GAME</button>
  </div>
</div>
<div id="controls">Arrow Keys / WASD to move &nbsp;|&nbsp; Space / W / Up to jump</div>

<script>
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const W = 800, H = 480;
const TILE = 32;
const GRAVITY = 0.52;
const JUMP_FORCE = -12;
const SPEED = 3.8;

let score = 0, lives = 3, coinCount = 0, currentLevel = 1;
let gameRunning = false;
let camera = { x: 0 };
let worldWidth = 0;
let player, tiles, enemies, particles, floatTexts, coins;
let keys = {};
let levelComplete = false;
let levelCompleteTimer = 0;

// ── Input ──────────────────────────────────────────────────────────────────
window.addEventListener('keydown', e => {
  keys[e.key] = true;
  if ([' ','ArrowUp','ArrowDown','ArrowLeft','ArrowRight'].includes(e.key)) e.preventDefault();
});
window.addEventListener('keyup', e => { keys[e.key] = false; });

// ── Helpers ────────────────────────────────────────────────────────────────
function rectsOverlap(a, b) {
  return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
}

function isSolid(t) {
  return ['ground','brick','question','pipe','pipeTop','used','stone','flagPole'].includes(t.type);
}

function showMsg(txt, dur = 1600) {
  const el = document.getElementById('msg');
  el.textContent = txt;
  el.style.opacity = 1;
  clearTimeout(el._t);
  el._t = setTimeout(() => el.style.opacity = 0, dur);
}

// ── Particles ──────────────────────────────────────────────────────────────
function spawnParticles(x, y, color, n = 8) {
  for (let i = 0; i < n; i++) {
    const angle = (Math.PI * 2 / n) * i + Math.random() * 0.4;
    const speed = 2 + Math.random() * 3;
    particles.push({
      x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed - 2,
      color, life: 30 + Math.random() * 20, maxLife: 50, r: 3 + Math.random() * 3
    });
  }
}

function spawnFloatText(x, y, txt, color = '#fff') {
  floatTexts.push({ x, y, txt, color, life: 50, vy: -1.2 });
}

// ── Level Builder ──────────────────────────────────────────────────────────
function buildLevel(lvl) {
  tiles = []; enemies = []; particles = []; floatTexts = []; coins = [];
  camera.x = 0; levelComplete = false;

  const layouts = [
    // Level 1
    [
      // ground row: y=13 (13*32=416)
      { type: 'ground', rows: 1, cols: 80, startCol: 0, startRow: 13 },
      // gaps
      { type: 'gap', startCol: 20, endCol: 23, row: 13 },
      { type: 'gap', startCol: 45, endCol: 48, row: 13 },
      // platforms
      { type: 'brick', startCol: 5, startRow: 9, cols: 3 },
      { type: 'question', startCol: 7, startRow: 9, coins: 3 },
      { type: 'question', startCol: 14, startRow: 7 },
      { type: 'brick', startCol: 15, startRow: 7, cols: 2 },
      { type: 'question', startCol: 16, startRow: 7 },
      { type: 'brick', startCol: 17, startRow: 7 },
      { type: 'brick', startCol: 25, startRow: 9, cols: 4 },
      { type: 'question', startCol: 27, startRow: 6 },
      { type: 'brick', startCol: 32, startRow: 8, cols: 2 },
      { type: 'brick', startCol: 35, startRow: 6, cols: 3 },
      { type: 'question', startCol: 36, startRow: 6 },
      { type: 'brick', startCol: 50, startRow: 9, cols: 5 },
      { type: 'question', startCol: 52, startRow: 6, coins: 2 },
      { type: 'brick', startCol: 58, startRow: 8, cols: 3 },
      { type: 'question', startCol: 60, startRow: 5 },
      { type: 'brick', startCol: 65, startRow: 9, cols: 4 },
      // pipes
      { type: 'pipe', col: 10, height: 2 },
      { type: 'pipe', col: 30, height: 3 },
      { type: 'pipe', col: 55, height: 2 },
      { type: 'pipe', col: 70, height: 3 },
      // flag
      { type: 'flag', col: 76 },
    ],
    // Level 2
    [
      { type: 'ground', rows: 1, cols: 100, startCol: 0, startRow: 13 },
      { type: 'gap', startCol: 15, endCol: 18, row: 13 },
      { type: 'gap', startCol: 35, endCol: 39, row: 13 },
      { type: 'gap', startCol: 60, endCol: 65, row: 13 },
      { type: 'brick', startCol: 6, startRow: 8, cols: 4 },
      { type: 'question', startCol: 8, startRow: 8, coins: 4 },
      { type: 'brick', startCol: 20, startRow: 7, cols: 5 },
      { type: 'question', startCol: 22, startRow: 7 },
      { type: 'question', startCol: 40, startRow: 6, coins: 5 },
      { type: 'brick', startCol: 45, startRow: 9, cols: 3 },
      { type: 'brick', startCol: 50, startRow: 7, cols: 3 },
      { type: 'question', startCol: 51, startRow: 7 },
      { type: 'brick', startCol: 68, startRow: 9, cols: 5 },
      { type: 'question', startCol: 70, startRow: 6, coins: 3 },
      { type: 'brick', startCol: 78, startRow: 7, cols: 4 },
      { type: 'pipe', col: 12, height: 2 },
      { type: 'pipe', col: 28, height: 3 },
      { type: 'pipe', col: 55, height: 4 },
      { type: 'pipe', col: 85, height: 2 },
      { type: 'flag', col: 94 },
    ]
  ];

  const layout = layouts[(lvl - 1) % layouts.length];
  let maxCol = 0;

  for (const def of layout) {
    if (def.type === 'ground') {
      const gaps = layout.filter(d => d.type === 'gap');
      for (let c = def.startCol; c < def.startCol + def.cols; c++) {
        const inGap = gaps.some(g => c >= g.startCol && c <= g.endCol);
        if (!inGap) {
          tiles.push({ type: 'ground', x: c * TILE, y: def.startRow * TILE, w: TILE, h: TILE });
          // underground fill
          for (let r = def.startRow + 1; r < 16; r++) {
            tiles.push({ type: 'ground', x: c * TILE, y: r * TILE, w: TILE, h: TILE });
          }
        }
        maxCol = Math.max(maxCol, c);
      }
    } else if (def.type === 'brick') {
      for (let c = 0; c < def.cols; c++) {
        tiles.push({ type: 'brick', x: (def.startCol + c) * TILE, y: def.startRow * TILE, w: TILE, h: TILE });
        maxCol = Math.max(maxCol, def.startCol + c);
      }
    } else if (def.type === 'question') {
      tiles.push({ type: 'question', x: def.startCol * TILE, y: def.startRow * TILE, w: TILE, h: TILE, used: false, coins: def.coins || 1 });
      maxCol = Math.max(maxCol, def.startCol);
    } else if (def.type === 'pipe') {
      const groundY = 13 * TILE;
      const h = def.height;
      // pipe body
      for (let row = 0; row < h; row++) {
        tiles.push({ type: 'pipe', x: def.col * TILE, y: groundY - row * TILE, w: TILE * 2, h: TILE });
        tiles.push({ type: 'pipe', x: (def.col + 1) * TILE, y: groundY - row * TILE, w: TILE, h: TILE, skip: true });
      }
      // pipe top
      tiles.push({ type: 'pipeTop', x: def.col * TILE - 2, y: groundY - h * TILE, w: TILE * 2 + 4, h: TILE });
      tiles.push({ type: 'pipeTop', x: (def.col + 1) * TILE + 2, y: groundY - h * TILE, w: 0, h: TILE, skip: true });
      maxCol = Math.max(maxCol, def.col + 1);
      // spawn Goomba near pipe
      if (def.col > 5) enemies.push(new Goomba((def.col - 2) * TILE, groundY - TILE));
    } else if (def.type === 'flag') {
      // Flag pole
      const gy = 13 * TILE;
      for (let r = 0; r < 11; r++) {
        tiles.push({ type: 'flagPole', x: def.col * TILE + 14, y: gy - r * TILE - TILE, w: 4, h: TILE });
      }
      tiles.push({ type: 'flagBase', x: def.col * TILE + 2, y: gy - TILE, w: 28, h: TILE, isFlagBase: true });
      tiles.push({ type: 'flagBanner', x: def.col * TILE + 18, y: gy - 11 * TILE, w: 24, h: 16, isBanner: true });
      maxCol = Math.max(maxCol, def.col + 1);
    }
  }

  // Scatter some extra coins
  const coinDefs = [[8, 10], [9, 10], [22, 5], [38, 5], [62, 5], [63, 5]];
  for (const [c, r] of coinDefs) {
    coins.push({ x: c * TILE + 8, y: r * TILE, w: 16, h: 16, visible: true, anim: Math.random() * Math.PI * 2 });
  }

  // Extra goombas
  const goombaPositions = lvl === 1
    ? [[6, 12], [18, 12], [28, 12], [40, 12], [53, 12], [63, 12]]
    : [[8, 12], [20, 12], [32, 12], [42, 12], [55, 12], [70, 12], [80, 12]];
  const groundY = 13 * TILE;
  for (const [c] of goombaPositions) {
    enemies.push(new Goomba(c * TILE, groundY - 28));
  }

  // Koopas
  const koopaPos = lvl === 1 ? [[38, 12], [68, 12]] : [[28, 12], [50, 12], [78, 12]];
  for (const [c] of koopaPos) {
    enemies.push(new Koopa(c * TILE, groundY - 36));
  }

  worldWidth = (maxCol + 10) * TILE;
  player = new Player(TILE * 2, groundY - 34);
}

// ── Player ─────────────────────────────────────────────────────────────────
class Player {
  constructor(x, y) {
    this.x = x; this.y = y; this.w = 24; this.h = 32;
    this.vx = 0; this.vy = 0;
    this.onGround = false;
    this.facing = 1;
    this.invincible = 0;
    this.dead = false;
    this.deathTimer = 0;
    this.animFrame = 0;
    this.animTimer = 0;
    this.jumpPressed = false;
  }

  update() {
    if (this.dead) {
      this.deathTimer++;
      if (this.deathTimer === 1) this.vy = -12;
      else this.vy += GRAVITY;
      this.y += this.vy;
      if (this.deathTimer > 90) respawn();
      return;
    }
    if (this.invincible > 0) this.invincible--;

    const left  = keys['ArrowLeft']  || keys['a'] || keys['A'];
    const right  = keys['ArrowRight'] || keys['d'] || keys['D'];
    const jump   = keys['ArrowUp']   || keys['w'] || keys['W'] || keys[' '];

    if (left)  { this.vx = Math.max(this.vx - 0.7, -SPEED); this.facing = -1; }
    else if (right) { this.vx = Math.min(this.vx + 0.7, SPEED); this.facing = 1; }
    else { this.vx *= 0.78; }

    if (jump && !this.jumpPressed && this.onGround) {
      this.vy = JUMP_FORCE;
      this.onGround = false;
      spawnParticles(this.x + this.w / 2, this.y + this.h, '#fff', 5);
    }
    this.jumpPressed = jump;

    this.vy = Math.min(this.vy + GRAVITY, 14);

    this.x += this.vx;
    this.resolveX();
    this.y += this.vy;
    this.onGround = false;
    this.resolveY();

    // Clamp left
    if (this.x < camera.x) { this.x = camera.x; this.vx = 0; }
    // Fall death
    if (this.y > H + 60) this.triggerDeath();

    // Camera
    const target = this.x - W / 3;
    camera.x = Math.max(0, Math.min(target, worldWidth - W));

    // Enemy collision
    if (this.invincible <= 0) {
      for (const e of enemies) {
        if (!e.alive || e.dead) continue;
        if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, { x: e.x, y: e.y, w: e.w, h: e.h })) {
          if (this.vy > 1 && this.y + this.h < e.y + e.h * 0.6 + 8) {
            e.stomp();
            this.vy = -8;
            score += 100;
            updateUI();
            spawnFloatText(e.x + e.w / 2, e.y, '+100');
          } else {
            this.triggerDeath();
          }
        }
      }
    }

    // Coin pickup
    for (const c of coins) {
      if (!c.visible) continue;
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, c)) {
        c.visible = false;
        coinCount++; score += 200;
        updateUI();
        spawnFloatText(c.x + 8, c.y, '+200', '#ffd700');
        spawnParticles(c.x + 8, c.y + 8, '#ffd700', 6);
      }
    }

    // Flag check
    for (const t of tiles) {
      if (t.isBanner && !levelComplete) {
        if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, { x: t.x - 30, y: 0, w: 60, h: H })) {
          triggerLevelComplete();
        }
      }
    }

    // Anim
    if (Math.abs(this.vx) > 0.3 && this.onGround) {
      this.animTimer++;
      if (this.animTimer > 7) { this.animFrame = (this.animFrame + 1) % 3; this.animTimer = 0; }
    } else if (!this.onGround) {
      this.animFrame = 2;
    } else {
      this.animFrame = 0;
    }
  }

  resolveX() {
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vx > 0) this.x = tr.x - this.w;
        else if (this.vx < 0) this.x = tr.x + tr.w;
        this.vx = 0;
      }
    }
  }

  resolveY() {
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vy >= 0) {
          this.y = tr.y - this.h;
          this.vy = 0;
          this.onGround = true;
        } else {
          this.y = tr.y + tr.h;
          this.vy = 1;
          if (t.type === 'question' && !t.used) hitQuestion(t);
          else if (t.type === 'brick') breakBrick(t);
        }
      }
    }
  }

  triggerDeath() {
    if (this.dead) return;
    this.dead = true;
    this.deathTimer = 0;
    lives--;
    updateUI();
    spawnParticles(this.x + this.w / 2, this.y + this.h / 2, '#e00', 14);
  }

  draw() {
    const sx = Math.round(this.x - camera.x);
    const sy = Math.round(this.y);
    if (sy > H + 40) return;

    const flash = this.invincible > 0 && Math.floor(this.invincible / 4) % 2 === 1;
    if (flash) return;

    ctx.save();
    if (this.facing === -1) {
      ctx.translate(sx + this.w / 2, 0);
      ctx.scale(-1, 1);
      ctx.translate(-(sx + this.w / 2), 0);
    }

    // Hat
    ctx.fillStyle = '#d03010';
    ctx.fillRect(sx + 2, sy, 20, 9);
    ctx.fillRect(sx, sy + 7, 24, 4);

    // Face
    ctx.fillStyle = '#ffb08a';
    ctx.fillRect(sx + 3, sy + 11, 18, 10);

    // Eyes
    ctx.fillStyle = '#111';
    ctx.fillRect(sx + 14, sy + 13, 5, 4);

    // Nose
    ctx.fillStyle = '#e07050';
    ctx.fillRect(sx + 9, sy + 16, 7, 3);

    // Mustache
    ctx.fillStyle = '#3a1800';
    ctx.fillRect(sx + 4, sy + 19, 16, 3);

    // Body (overalls)
    ctx.fillStyle = '#2255cc';
    ctx.fillRect(sx + 2, sy + 21, 20, 7);

    // Buttons
    ctx.fillStyle = '#ffd700';
    ctx.fillRect(sx + 6, sy + 22, 3, 3);
    ctx.fillRect(sx + 15, sy + 22, 3, 3);

    // Legs
    ctx.fillStyle = '#d03010';
    const lo = this.animFrame === 1 ? 2 : this.animFrame === 2 ? -1 : 0;
    ctx.fillRect(sx + 2, sy + 28, 8, 4);
    ctx.fillRect(sx + 14, sy + 28, 8, 4);

    // Shoes
    ctx.fillStyle = '#1a1a44';
    ctx.fillRect(sx, sy + 30, 11, 4);
    ctx.fillRect(sx + 13, sy + 30, 11, 4);

    ctx.restore();
  }
}

// ── Goomba ─────────────────────────────────────────────────────────────────
class Goomba {
  constructor(x, y) {
    this.x = x; this.y = y; this.w = 28; this.h = 28;
    this.vx = -1; this.vy = 0;
    this.alive = true;
    this.dead = false;
    this.deadTimer = 0;
    this.animTimer = 0;
    this.animFrame = 0;
  }

  update() {
    if (this.dead) {
      this.deadTimer++;
      if (this.deadTimer > 36) this.alive = false;
      return;
    }
    this.vy = Math.min(this.vy + GRAVITY, 14);
    this.x += this.vx;

    let hitWall = false;
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vx < 0) { this.x = tr.x + tr.w; hitWall = true; }
        else { this.x = tr.x - this.w; hitWall = true; }
      }
    }
    if (hitWall) this.vx *= -1;

    this.y += this.vy;
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vy >= 0) { this.y = tr.y - this.h; this.vy = 0; }
      }
    }
    if (this.y > H + 60) this.alive = false;

    this.animTimer++;
    if (this.animTimer > 12) { this.animFrame ^= 1; this.animTimer = 0; }
  }

  stomp() {
    this.dead = true;
    spawnParticles(this.x + this.w / 2, this.y + this.h / 2, '#8B4513', 7);
  }

  draw() {
    if (!this.alive) return;
    const sx = Math.round(this.x - camera.x);
    const sy = Math.round(this.y);
    if (sx < -40 || sx > W + 40) return;

    if (this.dead) {
      ctx.fillStyle = '#6b3010';
      ctx.fillRect(sx, sy + 20, this.w, 8);
      return;
    }

    // Body
    ctx.fillStyle = '#8B4513';
    ctx.fillRect(sx + 2, sy + 2, 24, 18);

    // Eyes (angry)
    ctx.fillStyle = '#fff';
    ctx.fillRect(sx + 3, sy + 4, 9, 8);
    ctx.fillRect(sx + 16, sy + 4, 9, 8);
    ctx.fillStyle = '#111';
    ctx.fillRect(sx + 7, sy + 6, 4, 5);
    ctx.fillRect(sx + 18, sy + 6, 4, 5);

    // Eyebrows
    ctx.fillStyle = '#333';
    ctx.save();
    ctx.translate(sx + 7, sy + 3);
    ctx.rotate(-0.3);
    ctx.fillRect(0, 0, 9, 3);
    ctx.restore();
    ctx.save();
    ctx.translate(sx + 19, sy + 3);
    ctx.rotate(0.3);
    ctx.fillRect(-4, 0, 9, 3);
    ctx.restore();

    // Feet
    const fo = this.animFrame ? 2 : -2;
    ctx.fillStyle = '#5a2000';
    ctx.fillRect(sx + 1 + fo, sy + 20, 12, 8);
    ctx.fillRect(sx + 15 - fo, sy + 20, 12, 8);
  }
}

// ── Koopa ──────────────────────────────────────────────────────────────────
class Koopa {
  constructor(x, y) {
    this.x = x; this.y = y; this.w = 28; this.h = 36;
    this.vx = -1; this.vy = 0;
    this.alive = true;
    this.dead = false;
    this.animTimer = 0;
    this.animFrame = 0;
  }

  update() {
    if (this.dead) { this.alive = false; return; }
    this.vy = Math.min(this.vy + GRAVITY, 14);
    this.x += this.vx;

    let hitWall = false;
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vx < 0) { this.x = tr.x + tr.w; hitWall = true; }
        else { this.x = tr.x - this.w; hitWall = true; }
      }
    }
    if (hitWall) this.vx *= -1;

    this.y += this.vy;
    for (const t of tiles) {
      if (!isSolid(t) || t.skip) continue;
      const tr = { x: t.x, y: t.y, w: t.w || TILE, h: t.h || TILE };
      if (rectsOverlap({ x: this.x, y: this.y, w: this.w, h: this.h }, tr)) {
        if (this.vy >= 0) { this.y = tr.y - this.h; this.vy = 0; }
      }
    }
    if (this.y > H + 60) this.alive = false;
    this.animTimer++;
    if (this.animTimer > 10) { this.animFrame ^= 1; this.animTimer = 0; }
  }

  stomp() {
    this.dead = true;
    score += 200;
    updateUI();
    spawnParticles(this.x + this.w / 2, this.y + this.h / 2, '#4a8a00', 10);
    spawnFloatText(this.x + this.w / 2, this.y, '+200', '#4aff00');
  }

  draw() {
    if (!this.alive) return;
    const sx = Math.round(this.x - camera.x);
    const sy = Math.round(this.y);
    if (sx < -40 || sx > W + 40) return;

    // Shell / body
    ctx.fillStyle = '#3a7a00';
    ctx.fillRect(sx + 4, sy + 8, 20, 22);
    ctx.fillStyle = '#5aaa00';
    ctx.fillRect(sx + 7, sy + 12, 14, 14);

    // Head
    ctx.fillStyle = '#7acc20';
    ctx.fillRect(sx + 6, sy, 16, 12);

    // Eyes
    ctx.fillStyle = '#fff';
    ctx.fillRect(sx + 7, sy + 2, 5, 6);
    ctx.fillRect(sx + 16, sy + 2, 5, 6);
    ctx.fillStyle = '#111';
    ctx.fillRect(sx + 9, sy + 4, 3, 4);
    ctx.fillRect(sx + 17, sy + 4, 3, 4);

    // Shell lines
    ctx.fillStyle = '#2a5a00';
    ctx.fillRect(sx + 13, sy + 12, 2, 14);
    ctx.fillRect(sx + 7, sy + 18, 14, 2);

    // Feet
    const fo = this.animFrame ? 2 : 0;
    ctx.fillStyle = '#7acc20';
    ctx.fillRect(sx + 2 + fo, sy + 28, 10, 8);
    ctx.fillRect(sx + 16 - fo, sy + 28, 10, 8);
  }
}

// ── Block interactions ─────────────────────────────────────────────────────
function hitQuestion(t) {
  t.used = true;
  t.type = 'used';
  spawnParticles(t.x + TILE / 2, t.y, '#ffd700', 10);

  const n = t.coins || 1;
  for (let i = 0; i < n; i++) {
    setTimeout(() => {
      coinCount++; score += 100;
      updateUI();
      spawnFloatText(t.x + TILE / 2, t.y - 20, '+100 COIN', '#ffd700');
      spawnParticles(t.x + TILE / 2, t.y, '#ffd700', 4);
    }, i * 100);
  }
}

function breakBrick(t) {
  const idx = tiles.indexOf(t);
  if (idx !== -1) tiles.splice(idx, 1);
  spawnParticles(t.x + TILE / 2, t.y + TILE / 2, '#c84b0c', 10);
  score += 50;
  updateUI();
  spawnFloatText(t.x + TILE / 2, t.y, '+50');
}

// ── Level complete ─────────────────────────────────────────────────────────
function triggerLevelComplete() {
  if (levelComplete) return;
  levelComplete = true;
  levelCompleteTimer = 0;
  showMsg('LEVEL CLEAR! +1000', 3000);
  score += 1000;
  updateUI();
  spawnParticles(player.x + player.w / 2, player.y + player.h / 2, '#ffd700', 20);
}

// ── Respawn / game over ────────────────────────────────────────────────────
function respawn() {
  if (lives <= 0) {
    gameOver();
    return;
  }
  buildLevel(currentLevel);
}

function gameOver() {
  gameRunning = false;
  const ov = document.getElementById('overlay');
  ov.innerHTML = `<h1>GAME OVER</h1>
    <p class="sub">Score: ${score}</p>
    <p class="sub">Coins: ${coinCount}</p>
    <button id="startBtn">▶ PLAY AGAIN</button>`;
  ov.style.display = 'flex';
  document.getElementById('startBtn').onclick = () => { resetGame(); startGame(); };
}

function resetGame() {
  score = 0; lives = 3; coinCount = 0; currentLevel = 1;
  updateUI();
}

function updateUI() {
  document.getElementById('sc').textContent = score;
  document.getElementById('lv').textContent = lives;
  document.getElementById('cn').textContent = coinCount;
  document.getElementById('lv2').textContent = currentLevel;
}

// ── Drawing ────────────────────────────────────────────────────────────────
const CLOUD_DEFS = [
  { x: 200, y: 60 }, { x: 520, y: 40 }, { x: 900, y: 80 },
  { x: 1300, y: 55 }, { x: 1700, y: 70 }, { x: 2100, y: 45 },
  { x: 2500, y: 65 }, { x: 2900, y: 50 }
];

const HILL_DEFS = [
  { x: 300, r: 80 }, { x: 700, r: 60 }, { x: 1100, r: 90 },
  { x: 1600, r: 70 }, { x: 2000, r: 100 }, { x: 2400, r: 65 }, { x: 2800, r: 85 }
];

function drawCloud(x, y) {
  const sx = x - camera.x;
  if (sx < -120 || sx > W + 120) return;
  ctx.fillStyle = '#fff';
  ctx.beginPath();
  ctx.arc(sx + 30, y + 20, 20, 0, Math.PI * 2);
  ctx.arc(sx + 55, y + 12, 28, 0, Math.PI * 2);
  ctx.arc(sx + 80, y + 20, 22, 0, Math.PI * 2);
  ctx.fill();
}

function drawHill(x, r) {
  const sx = x - camera.x;
  if (sx < -r - 20 || sx > W + r + 20) return;
  const groundY = 13 * TILE;
  ctx.fillStyle = '#5da832';
  ctx.beginPath();
  ctx.arc(sx, groundY, r, Math.PI, 0);
  ctx.fill();
  ctx.fillStyle = '#4a9022';
  for (let i = -r / 2; i < r / 2; i += 18) {
    ctx.fillRect(sx + i, groundY - r * 0.6 + Math.abs(i) * 0.4, 8, 24);
  }
}

function drawTile(t) {
  const sx = Math.round(t.x - camera.x);
  const sy = Math.round(t.y);
  if (t.skip) return;
  if (sx < -TILE - 4 || sx > W + 4) return;

  switch (t.type) {
    case 'ground': {
      // Top green layer
      const isTop = !tiles.some(o => o.type === 'ground' && o.x === t.x && o.y === t.y - TILE);
      if (isTop) {
        ctx.fillStyle = '#6dbd3a';
        ctx.fillRect(sx, sy, TILE, 10);
        ctx.fillStyle = '#5aad2a';
        ctx.fillRect(sx, sy + 10, TILE, 4);
        ctx.fillStyle = '#c07540';
        ctx.fillRect(sx, sy + 14, TILE, TILE - 14);
        // dirt texture
        ctx.fillStyle = '#a8622e';
        for (let i = 0; i < 2; i++) {
          ctx.fillRect(sx + 5 + i * 14, sy + 20, 4, 4);
        }
      } else {
        ctx.fillStyle = '#c07540';
        ctx.fillRect(sx, sy, TILE, TILE);
        ctx.fillStyle = '#a8622e';
        ctx.fillRect(sx, sy, TILE, 2);
        for (let i = 0; i < 2; i++) ctx.fillRect(sx + 5 + i * 14, sy + 8, 4, 4);
      }
      break;
    }
    case 'brick': {
      ctx.fillStyle = '#c05028';
      ctx.fillRect(sx, sy, TILE, TILE);
      ctx.fillStyle = '#e06030';
      ctx.fillRect(sx + 1, sy + 1, TILE - 2, 12);
      ctx.fillRect(sx + 1, sy + 15, TILE / 2 - 2, 12);
      ctx.fillRect(sx + TILE / 2 + 1, sy + 15, TILE / 2 - 2, 12);
      ctx.fillStyle = '#a04020';
      ctx.fillRect(sx, sy + 13, TILE, 2);
      ctx.fillRect(sx + TILE / 2, sy + 1, 2, 12);
      ctx.fillRect(sx, sy + 15, 2, 14);
      ctx.fillRect(sx + TILE / 2 - 2, sy + 15, 2, 14);
      break;
    }
    case 'question': {
      const pulse = Math.sin(Date.now() / 200) * 4;
      ctx.fillStyle = t.used ? '#888' : '#e8a000';
      ctx.fillRect(sx, sy, TILE, TILE);
      if (!t.used) {
        ctx.fillStyle = '#ffd040';
        ctx.fillRect(sx + 2, sy + 2, TILE - 4, 10);
        ctx.fillStyle = '#c07000';
        ctx.fillRect(sx + 2, sy + 12, TILE - 4, 2);
        // ? mark
        ctx.fillStyle = '#fff';
        ctx.font = 'bold 20px monospace';
        ctx.fillText('?', sx + 9, sy + 24);
      } else {
        ctx.fillStyle = '#aaa';
        ctx.fillRect(sx + 2, sy + 2, TILE - 4, TILE - 4);
      }
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 2;
      ctx.strokeRect(sx + 1, sy + 1, TILE - 2, TILE - 2);
      break;
    }
    case 'used': {
      ctx.fillStyle = '#666';
      ctx.fillRect(sx, sy, TILE, TILE);
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 2;
      ctx.strokeRect(sx + 1, sy + 1, TILE - 2, TILE - 2);
      break;
    }
    case 'pipe': {
      ctx.fillStyle = '#1a8a00';
      ctx.fillRect(sx + 3, sy, TILE - 6, TILE);
      ctx.fillStyle = '#2aaa20';
      ctx.fillRect(sx + 4, sy, 8, TILE);
      ctx.fillStyle = '#0a6000';
      ctx.fillRect(sx + TILE - 9, sy, 6, TILE);
      break;
    }
    case 'pipeTop': {
      ctx.fillStyle = '#1a8a00';
      ctx.fillRect(sx, sy, t.w, TILE);
      ctx.fillStyle = '#2aaa20';
      ctx.fillRect(sx + 2, sy + 2, 14, TILE - 4);
      ctx.fillStyle = '#0a6000';
      ctx.fillRect(sx + t.w - 8, sy, 8, TILE);
      ctx.strokeStyle = '#000';
      ctx.lineWidth = 2;
      ctx.strokeRect(sx + 1, sy + 1, t.w - 2, TILE - 2);
      break;
    }
    case 'flagPole': {
      ctx.fillStyle = '#888';
      ctx.fillRect(sx, sy, 4, TILE);
      break;
    }
    case 'flagBase': {
      ctx.fillStyle = '#888';
      ctx.fillRect(sx + 10, sy, 8, TILE);
      break;
    }
    case 'flagBanner': {
      ctx.fillStyle = '#e00';
      ctx.beginPath();
      ctx.moveTo(sx, sy);
      ctx.lineTo(sx + 24, sy + 8);
      ctx.lineTo(sx, sy + 16);
      ctx.fill();
      break;
    }
    case 'stone': {
      ctx.fillStyle = '#888';
      ctx.fillRect(sx, sy, TILE, TILE);
      ctx.fillStyle = '#aaa';
      ctx.fillRect(sx + 2, sy + 2, TILE - 4, 10);
      break;
    }
  }
}

function drawCoin(c) {
  if (!c.visible) return;
  const sx = Math.round(c.x - camera.x);
  const sy = Math.round(c.y);
  if (sx < -20 || sx > W + 20) return;
  c.anim += 0.08;
  const scaleX = Math.abs(Math.cos(c.anim));
  ctx.save();
  ctx.translate(sx + 8, sy + 8);
  ctx.scale(scaleX, 1);
  ctx.fillStyle = '#ffd700';
  ctx.beginPath();
  ctx.arc(0, 0, 8, 0, Math.PI * 2);
  ctx.fill();
  ctx.fillStyle = '#ffee88';
  ctx.fillRect(-3, -5, 6, 5);
  ctx.restore();
}

// ── Background ─────────────────────────────────────────────────────────────
function drawBackground() {
  // Sky
  ctx.fillStyle = '#5c94fc';
  ctx.fillRect(0, 0, W, H);

  // Hills
  for (const h of HILL_DEFS) drawHill(h.x, h.r);

  // Clouds
  for (const c of CLOUD_DEFS) drawCloud(c.x, c.y);
}

// ── Main loop ──────────────────────────────────────────────────────────────
let lastTime = 0;
let ticker = 0;

function loop(ts) {
  if (!gameRunning) return;
  requestAnimationFrame(loop);
  ticker++;

  ctx.clearRect(0, 0, W, H);
  drawBackground();

  // Draw tiles
  for (const t of tiles) drawTile(t);

  // Draw coins
  for (const c of coins) drawCoin(c);

  // Update + draw enemies
  for (const e of enemies) e.update();
  for (const e of enemies) e.draw();

  // Update + draw player
  player.update();
  player.draw();

  // Particles
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx; p.y += p.vy; p.vy += 0.15;
    p.life--;
    if (p.life <= 0) { particles.splice(i, 1); continue; }
    ctx.globalAlpha = p.life / p.maxLife;
    ctx.fillStyle = p.color;
    ctx.fillRect(p.x - p.r / 2, p.y - p.r / 2, p.r, p.r);
    ctx.globalAlpha = 1;
  }

  // Float texts
  for (let i = floatTexts.length - 1; i >= 0; i--) {
    const ft = floatTexts[i];
    ft.y += ft.vy;
    ft.life--;
    if (ft.life <= 0) { floatTexts.splice(i, 1); continue; }
    ctx.globalAlpha = ft.life / 50;
    ctx.fillStyle = ft.color || '#fff';
    ctx.font = 'bold 14px monospace';
    ctx.fillText(ft.txt, ft.x - camera.x - 20, ft.y);
    ctx.globalAlpha = 1;
  }

  // Level complete advance
  if (levelComplete) {
    levelCompleteTimer++;
    if (levelCompleteTimer > 120) {
      currentLevel++;
      if (currentLevel > 2) {
        // Victory!
        gameRunning = false;
        const ov = document.getElementById('overlay');
        ov.innerHTML = `<h1>YOU WIN!</h1>
          <p class="sub">Final Score: ${score}</p>
          <p class="sub">Coins: ${coinCount}</p>
          <button id="startBtn">▶ PLAY AGAIN</button>`;
        ov.style.display = 'flex';
        document.getElementById('startBtn').onclick = () => { resetGame(); startGame(); };
        return;
      }
      updateUI();
      buildLevel(currentLevel);
      showMsg('WORLD ' + currentLevel + '-1', 2000);
    }
  }
}

// ── Start ──────────────────────────────────────────────────────────────────
function startGame() {
  document.getElementById('overlay').style.display = 'none';
  resetGame();
  buildLevel(1);
  gameRunning = true;
  showMsg('WORLD 1-1', 2000);
  requestAnimationFrame(loop);
}

document.getElementById('startBtn').onclick = startGame;
</script>
</body>
</html>
