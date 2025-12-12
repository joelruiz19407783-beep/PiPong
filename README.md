<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1.0"/>
<title>Pong Neon - Acode</title>
<style>
  :root{
    --neon-c1:#00ffff;
    --neon-c2:#39ff14;
    --neon-t:#ff00ff;
    --bg:#000;
  }
  html,body{height:100%;margin:0;background:var(--bg);font-family:Arial,Helvetica,sans-serif;color:var(--neon-t);-webkit-user-select:none;user-select:none}
  .center{display:flex;flex-direction:column;align-items:center;justify-content:center;height:100%;gap:12px}
  #startBtn{
    background:linear-gradient(90deg,var(--neon-c1),var(--neon-c2));
    color:#000;padding:14px 36px;font-size:20px;border:none;border-radius:12px;font-weight:700;
    box-shadow:0 0 18px var(--neon-c1),0 0 38px var(--neon-c2);cursor:pointer;touch-action:manipulation;
  }
  #startBtn:active{transform:scale(.97)}
  #game{border-radius:10px;width:92vw;max-width:360px;height:auto;box-shadow:0 0 30px var(--neon-c1);background:#000;display:block}
  #hud{width:92vw;max-width:360px;display:flex;justify-content:space-between;align-items:center;padding:0 6px;margin-bottom:6px}
  .score{color:var(--neon-t);font-size:18px;text-shadow:0 0 8px var(--neon-t)}
  .touch-controls{width:92vw;max-width:360px;display:flex;justify-content:space-between;gap:12px;margin-top:8px}
  .controls-column{display:flex;flex-direction:column;gap:10px;flex:1;align-items:center}
  .touch-btn{width:100%;background:rgba(0,255,255,0.06);border:2px solid rgba(0,255,255,0.14);color:var(--neon-c1);font-size:20px;padding:10px 0;border-radius:10px;box-shadow:0 0 12px rgba(0,255,255,0.08);touch-action:none}
  .touch-btn:active{transform:scale(.98)}
  .title{color:var(--neon-t);font-size:28px;text-shadow:0 0 14px var(--neon-t);pointer-events:none}
  .small{font-size:12px;color:#9f9;opacity:.9}
</style>
</head>
<body>

<div class="center" id="menuScreen">
  <div class="title" id="titleText">P O N G &nbsp; N E Ó N</div>
  <button id="startBtn">JUGAR</button>
  <div class="small">Controles táctiles: botones ⬆ / ⬇ o arrastra la zona izquierda</div>
</div>

<div id="gameContainer" style="display:none;flex-direction:column;align-items:center;">
  <div id="hud">
    <div class="score" id="scoreLeft">0</div>
    <div class="score" id="scoreRight">0</div>
  </div>

  <canvas id="game"></canvas>

  <div class="touch-controls">
    <div style="flex:1;display:flex;gap:8px">
      <button class="touch-btn" id="upBtn">⬆ SUBIR</button>
      <button class="touch-btn" id="downBtn">⬇ BAJAR</button>
    </div>
    <div style="width:8px"></div>
  </div>
</div>

<script>
/* Pong Neon - single file, Acode-friendly */

// Canvas logical size (keeps small for phones)
const W = 300, H = 450;
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
canvas.width = W;
canvas.height = H;

const menuScreen = document.getElementById('menuScreen');
const gameContainer = document.getElementById('gameContainer');
const startBtn = document.getElementById('startBtn');
const titleText = document.getElementById('titleText');

const scoreLeftEl = document.getElementById('scoreLeft');
const scoreRightEl = document.getElementById('scoreRight');
const upBtn = document.getElementById('upBtn');
const downBtn = document.getElementById('downBtn');

// Theme
const neonBall = '#39ff14';
const neonPaddle = '#00ffff';
const neonText = '#ff00ff';
const bg = '#000';

// Game state
let playing = false;
let playerScore = 0, aiScore = 0;

// Paddles
const paddleW = 12, paddleH = 70;
let playerY = (H - paddleH)/2;
let aiY = (H - paddleH)/2;
const playerSpeed = 6;

// Ball
let ballX = W/2, ballY = H/2, ballR = 8;
let ballVX = 3, ballVY = 2.6;
const speedMult = 1.06; // increases on hit

// AI human-like
let aiBaseSpeed = 3.2;
let aiErrorChance = 0.17;
let aiErrorMax = 30;

// Touch state
let movingUp = false, movingDown = false;
let dragging = false, dragStartY = 0;

// WebAudio
const AudioCtx = window.AudioContext || window.webkitAudioContext;
const audioCtx = new AudioCtx();

// Hit sound (short beep)
function playHitSound(){
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'sine';
  o.frequency.value = 900 + Math.random()*140;
  g.gain.value = 0.0001;
  o.connect(g);
  g.connect(audioCtx.destination);
  const now = audioCtx.currentTime;
  g.gain.exponentialRampToValueAtTime(0.12, now + 0.01);
  g.gain.exponentialRampToValueAtTime(0.0001, now + 0.14);
  o.start(now);
  o.stop(now + 0.16);
}

// Score sound (descending bell)
function playScoreSound(){
  const now = audioCtx.currentTime;
  const g = audioCtx.createGain();
  g.connect(audioCtx.destination);
  g.gain.value = 0.0001;
  const freqs = [900,700,520];
  freqs.forEach((f,i)=>{
    const o = audioCtx.createOscillator();
    o.type = 'triangle';
    o.frequency.value = f;
    o.connect(g);
    o.start(now + i*0.06);
    o.stop(now + i*0.06 + 0.14);
  });
  g.gain.exponentialRampToValueAtTime(0.12, now + 0.01);
  g.gain.exponentialRampToValueAtTime(0.0001, now + 0.45);
}

// Menu floating animation
let floatingPhase = 0;

// Start button
startBtn.addEventListener('click', ()=> {
  if (audioCtx.state === 'suspended') audioCtx.resume();
  menuScreen.style.display = 'none';
  gameContainer.style.display = 'flex';
  playing = true;
  resetAll();
});

// Touch buttons hold
upBtn.addEventListener('touchstart', e => { e.preventDefault(); movingUp = true; }, {passive:false});
upBtn.addEventListener('touchend', e => { e.preventDefault(); movingUp = false; }, {passive:false});
downBtn.addEventListener('touchstart', e => { e.preventDefault(); movingDown = true; }, {passive:false});
downBtn.addEventListener('touchend', e => { e.preventDefault(); movingDown = false; }, {passive:false});

// Mouse support for testing
upBtn.addEventListener('mousedown', ()=> movingUp = true);
upBtn.addEventListener('mouseup', ()=> movingUp = false);
downBtn.addEventListener('mousedown', ()=> movingDown = true);
downBtn.addEventListener('mouseup', ()=> movingDown = false);

// Dragging on left half to move paddle
canvas.addEventListener('touchstart', e => {
  const t = e.touches[0];
  const rect = canvas.getBoundingClientRect();
  const x = t.clientX - rect.left;
  const y = t.clientY - rect.top;
  if (x < rect.width * 0.6) { dragging = true; dragStartY = y; }
}, {passive:false});

canvas.addEventListener('touchmove', e => {
  if (!dragging) return;
  e.preventDefault();
  const t = e.touches[0];
  const rect = canvas.getBoundingClientRect();
  const y = t.clientY - rect.top;
  const dy = y - dragStartY;
  playerY += dy * 1.0;
  dragStartY = y;
}, {passive:false});

canvas.addEventListener('touchend', e => { dragging = false; }, {passive:false});

// clamp paddles inside canvas
function clampPaddles(){
  if (playerY < 0) playerY = 0;
  if (playerY + paddleH > H) playerY = H - paddleH;
  if (aiY < 0) aiY = 0;
  if (aiY + paddleH > H) aiY = H - paddleH;
}

// resets
function resetBall(){
  ballX = W/2; ballY = H/2;
  ballVX = (Math.random()>.5?1:-1) * (3 + Math.random()*0.8);
  ballVY = (Math.random()>.5?1:-1) * (2.2 + Math.random()*0.8);
}

function resetAll(){
  playerScore = 0; aiScore = 0;
  scoreLeftEl.textContent = playerScore;
  scoreRightEl.textContent = aiScore;
  playerY = (H - paddleH)/2;
  aiY = (H - paddleH)/2;
  resetBall();
}

// draw functions
function drawBackground(){
  ctx.fillStyle = bg;
  ctx.fillRect(0,0,W,H);
  ctx.strokeStyle = 'rgba(255,255,255,0.06)';
  ctx.setLineDash([8,8]);
  ctx.beginPath();
  ctx.moveTo(W/2,0); ctx.lineTo(W/2,H); ctx.stroke();
  ctx.setLineDash([]);
}

function drawPaddle(x,y){
  ctx.fillStyle = neonPaddle;
  ctx.shadowBlur = 16; ctx.shadowColor = neonPaddle;
  ctx.fillRect(x,y,paddleW,paddleH);
  ctx.shadowBlur = 0;
}

function drawBall(){
  ctx.beginPath();
  ctx.arc(ballX, ballY, ballR, 0, Math.PI*2);
  ctx.fillStyle = neonBall;
  ctx.shadowBlur = 16; ctx.shadowColor = neonBall;
  ctx.fill();
  ctx.shadowBlur = 0;
}

function drawScores(){
  ctx.fillStyle = neonText;
  ctx.font = '20px Arial';
  ctx.fillText(playerScore, W/2 - 40, 28);
  ctx.fillText(aiScore, W/2 + 20, 28);
}

// AI: predict and sometimes err
function updateAI(){
  // avoid division by zero
  const vx = ballVX === 0 ? 0.0001 : ballVX;
  let predicted = ballY + (ballVY * ((W - 20 - ballX) / vx));
  predicted = Math.max(0, Math.min(H - paddleH, predicted - paddleH/2));
  if (Math.random() < aiErrorChance) predicted += (Math.random()*aiErrorMax*2 - aiErrorMax);
  const speed = aiBaseSpeed;
  if (aiY + paddleH/2 < predicted) aiY += speed; else aiY -= speed;
}

// ball physics & collisions
function updateBall(){
  ballX += ballVX; ballY += ballVY;

  // top/bottom collision
  if (ballY - ballR < 0 || ballY + ballR > H){
    ballVY *= -1;
    playHitSound();
  }

  // player paddle collision (left)
  if (ballX - ballR < paddleW + 6){
    if (ballY > playerY && ballY < playerY + paddleH){
      ballVX = Math.abs(ballVX) * speedMult;
      const hitPos = (ballY - (playerY + paddleH/2)) / (paddleH/2);
      ballVY += hitPos * 1.2;
      playHitSound();
    }
  }

  // ai paddle collision (right)
  if (ballX + ballR > W - paddleW - 6){
    if (ballY > aiY && ballY < aiY + paddleH){
      ballVX = -Math.abs(ballVX) * speedMult;
      const hitPos = (ballY - (aiY + paddleH/2)) / (paddleH/2);
      ballVY += hitPos * 1.0;
      playHitSound();
    }
  }

  // scoring
  if (ballX < -10){
    aiScore++; scoreRightEl.textContent = aiScore; playScoreSound(); resetBall();
  }
  if (ballX > W + 10){
    playerScore++; scoreLeftEl.textContent = playerScore; playScoreSound(); resetBall();
  }
}

// main loop
function step(){
  requestAnimationFrame(step);

  if (!playing){
    // floating menu animation
    floatingPhase += 0.04;
    const ty = Math.sin(floatingPhase) * 8;
    titleText.style.transform = `translateY(${ty}px)`;
    startBtn.style.transform = `translateY(${ -ty*0.6 }px)`;
    return;
  }

  // apply controls
  if (movingUp) playerY -= playerSpeed;
  if (movingDown) playerY += playerSpeed;
  clampPaddles();

  updateAI();
  updateBall();

  // drawing
  drawBackground();
  drawPaddle(6, playerY);
  drawPaddle(W - paddleW - 6, aiY);
  drawBall();
  drawScores();
}

// resume audio on first user gesture (some browsers require)
document.addEventListener('click', ()=> { if (audioCtx.state === 'suspended') audioCtx.resume(); }, {once:true});

// small keyboard support for testing
window.addEventListener('keydown', e => {
  if (e.key === 'ArrowUp') playerY -= 20;
  if (e.key === 'ArrowDown') playerY += 20;
});

// start loop
step();
</script>
</body>
</html>
