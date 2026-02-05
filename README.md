<html lang="no">
<head>
  <meta charset="UTF-8" />
  <title>space-station</title>
  <style>
    body {
      margin: 0;
      background: black;
      color: white;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      font-family: Arial, sans-serif;
      overflow: hidden;
    }
    canvas { background: #05080f; border: 2px solid #4af; }
    .ui {
      position: absolute; top: 20px; right: 20px;
      display: flex; flex-direction: column; gap: 10px;
      z-index: 2;
    }
    button { padding: 10px 15px; font-size: 16px; cursor: pointer; }
    #loading {
      position: absolute;
      inset: 0;
      background: black;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      z-index: 5;
    }
  </style>
</head>
<body>

<div id="loading">
  <h1>🚀 space-station</h1>
  <p>Laster romstasjon…</p>
</div>

<div class="ui">
  <button onclick="togglePause()">Pause</button>
  <button onclick="restartGame()">Restart</button>
  <button onclick="upgradeWeapon()">Upgrade Weapon</button>
  <button onclick="resetData()">Reset Data</button>
  <div id="coins">Coins: 0</div>
  <div id="upgrade">Upgrade cost: 0</div>
</div>

<canvas id="game" width="400" height="600"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

const GAME_SPEED = 0.5;

let player, enemies, bullets, explosions, stars;
let score, gameOver=false, paused=false;
let keys={}, shootCooldown=0;

// Lagret data
let coins = Number(localStorage.getItem("coins")) || 0;
let upgradeLevel = Number(localStorage.getItem("upgradeLevel")) || 0;
let highscore = Number(localStorage.getItem("hard_highscore")) || 0;

let bulletSpeed = (8 + upgradeLevel * 2) * GAME_SPEED;
let bulletsPerShot = 1 + upgradeLevel;

function upgradeCost(){ 
  return Math.floor(200 * Math.pow(2, upgradeLevel));
}
function saveProgress(){
  localStorage.setItem("coins", coins);
  localStorage.setItem("upgradeLevel", upgradeLevel);
}

// Reset knapp
function resetData(){
  localStorage.removeItem("coins");
  localStorage.removeItem("upgradeLevel");
  localStorage.removeItem("hard_highscore");
  coins = 0;
  upgradeLevel = 0;
  highscore = 0;
  updateUI();
  alert("Data reset!");
}

function init(){
  player = { x:180, y:540, width:35, height:35, speed:6*GAME_SPEED };
  enemies=[]; bullets=[]; explosions=[];
  stars = Array.from({length:60},()=>({x:Math.random()*400,y:Math.random()*600,s:1+Math.random()*2}));
  score=0; gameOver=false; paused=false; shootCooldown=0;
  updateUI();
}

function restartGame(){ init(); }
function togglePause(){ if(!gameOver) paused=!paused; }

function upgradeWeapon(){
  const cost = upgradeCost();
  if(coins>=cost){
    coins-=cost;
    upgradeLevel++;
    bulletSpeed += 2 * GAME_SPEED;
    bulletsPerShot++;
    saveProgress();
    updateUI();
  }
}

function updateUI(){
  document.getElementById("coins").innerText = `Coins: ${coins}`;
  document.getElementById("upgrade").innerText = `Upgrade cost: ${upgradeCost()}`;
}

document.addEventListener("keydown", e=>{
  keys[e.key.toLowerCase()] = true; 
  if(e.key==='p'||e.key==='P') togglePause();
});
document.addEventListener("keyup", e=>keys[e.key.toLowerCase()] = false);

// Flere fiender
function spawnEnemy(){
  const spawnCount = 3 + Math.floor(Math.random()*3); // 3-5 fiender
  for(let i=0;i<spawnCount;i++){
    const r=Math.random();
    if(r<0.75){
      // Vanlige fiender (nedover) litt tregere
      enemies.push({
        x:Math.random()*370,
        y:-40,
        w:30,
        h:30,
        speedY:(1.8 + score/2000)*GAME_SPEED,
        speedX:0,
        hp:1,
        color:'#f44'
      });
    } else if(r<0.95){
      // Side-fiender litt raskere
      const left=Math.random()<0.5;
      enemies.push({
        x:left?-40:440,
        y:Math.random()*250,
        w:35,
        h:35,
        speedY:1.5*GAME_SPEED,
        speedX:left?2.5*GAME_SPEED:-2.5*GAME_SPEED,
        hp:1,
        color:'#fa0'
      });
    } else {
      // Boss
      enemies.push({
        x:150,
        y:-80,
        w:100,
        h:80,
        speedY:1*GAME_SPEED,
        speedX:1*GAME_SPEED,
        hp:20,
        isBoss:true,
        color:'#a4f'
      });
    }
  }
}

function shoot(){
  for(let i=0;i<bulletsPerShot;i++){
    const off=(i-Math.floor(bulletsPerShot/2))*10;
    bullets.push({x:player.x+player.width/2-3+off,y:player.y,w:6,h:12,speed:bulletSpeed});
  }
}

function explode(x,y){
  for(let i=0;i<8;i++) explosions.push({x,y,dx:(Math.random()-0.5)*4,dy:(Math.random()-0.5)*4,life:15});
}

function update(){
  if(gameOver||paused) return;

  stars.forEach(s=>{ s.y += s.s*GAME_SPEED; if(s.y>600)s.y=0; });

  if((keys['arrowleft'] || keys['a']) && player.x>0) player.x-=player.speed;
  if((keys['arrowright'] || keys['d']) && player.x<365) player.x+=player.speed;

  if(keys[' '] && shootCooldown<=0){ shoot(); shootCooldown=18; }
  if(shootCooldown>0) shootCooldown--;

  bullets.forEach(b=>b.y-=b.speed);
  bullets = bullets.filter(b=>b.y>-20);

  enemies.forEach(e=>{ e.y+=e.speedY; e.x+=e.speedX; });

  bullets.forEach((b,bi)=>{
    enemies.forEach((e,ei)=>{
      if(b.x<e.x+e.w && b.x+b.w>e.x && b.y<e.y+e.h && b.y+b.h>e.y){
        explode(e.x+e.w/2,e.y+e.h/2);
        e.hp--; bullets.splice(bi,1);
        if(e.hp<=0){
          enemies.splice(ei,1);
          coins += e.isBoss?300:50;
          score += e.isBoss?1000:200;
          saveProgress(); updateUI();
        }
      }
    });
  });

  enemies.forEach(e=>{
    if(player.x<e.x+e.w && player.x+player.width>e.x && player.y<e.y+e.h && player.y+player.height>e.y){
      gameOver=true;
      if(score>highscore){ highscore=score; localStorage.setItem('hard_highscore',highscore); }
    }
  });

  explosions.forEach(p=>{p.x+=p.dx;p.y+=p.dy;p.life--;});
  explosions = explosions.filter(p=>p.life>0);

  score += 0.5 * GAME_SPEED;
}

function draw(){
  ctx.clearRect(0,0,400,600);

  stars.forEach(s=>{ ctx.fillStyle='white'; ctx.fillRect(s.x,s.y,2,2); });

  ctx.fillStyle='#0f0'; ctx.fillRect(player.x,player.y,player.width,player.height);
  ctx.fillStyle='white'; bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.w,b.h));

  enemies.forEach(e=>{
    ctx.fillStyle=e.color; ctx.fillRect(e.x,e.y,e.w,e.h);
    if(e.isBoss){
      ctx.fillStyle='black'; ctx.fillRect(e.x,e.y-10,e.w,5);
      ctx.fillStyle='#0f0'; ctx.fillRect(e.x,e.y-10,e.w*(e.hp/20),5);
    }
  });

  explosions.forEach(p=>{ ctx.fillStyle='yellow'; ctx.fillRect(p.x,p.y,3,3); });

  ctx.fillStyle='white'; ctx.font='16px Arial';
  ctx.fillText(`Score: ${Math.floor(score)}`,10,20);
  ctx.fillText(`Highscore: ${highscore}`,10,40);

  if(paused){ ctx.font='30px Arial'; ctx.fillText('PAUSE',140,300); }
  if(gameOver){ ctx.font='30px Arial'; ctx.fillText('GAME OVER',100,300); }
}

// Loading screen
if(!localStorage.getItem('visited')){
  setTimeout(()=>{
    document.getElementById('loading').style.display='none';
    localStorage.setItem('visited','yes');
  },2000);
}else{
  document.getElementById('loading').style.display='none';
}

init();
setInterval(spawnEnemy,700); // spawn oftere for flere fiender
(function loop(){ update(); draw(); requestAnimationFrame(loop); })();
</script>
</body>
</html>
