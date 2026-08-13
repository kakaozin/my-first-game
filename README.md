[index.html](https://github.com/user-attachments/files/31028352/index.html)
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>MATH ESCAPE</title>
<style>
* {
  word-break: 
  break-all 
  !important;
}
body {
  margin: 0;
  overflow: 
  hidden;
  background: 
  #000;
  font-family: 
  sans-serif;
}
#canvas-container {
  width: 100vw;
  height: 100vh;
}
#ui-layer {
  position: 
  absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: 
  none;
  display: 
  flex;
  flex-direction: 
  column;
  justify-content: 
  space-between;
  padding: 10px;
}
#ai-window {
  background: 
  rgba(15,0,0,0.9);
  border: 
  2px solid 
  #f00;
  color: 
  #f33;
  padding: 10px;
  width: 320px;
  max-width: 90%;
  pointer-events: 
  auto;
}
#chat-form {
  display: 
  flex;
  margin-top: 
  5px;
}
#user-input {
  flex: 1;
  background: 
  #100;
  border: 
  1px solid 
  #f00;
  color: 
  #f33;
  padding: 4px;
}
#send-btn {
  background: 
  #f00;
  border: 
  none;
  color: 
  #fff;
  padding: 4px 10px;
}
#status-bar {
  color: 
  #fff;
  background: 
  rgba(0,0,0,0.7);
  padding: 8px;
  align-self: 
  flex-start;
}
#blocker {
  position: 
  absolute;
  width: 100%;
  height: 100%;
  background: 
  #000;
  display: 
  flex;
  flex-direction: 
  column;
  justify-content: 
  center;
  align-items: 
  center;
  color: 
  #fff;
  cursor: 
  pointer;
  z-index: 10;
  text-align: 
  center;
}
</style>
</head>
<body>

<div id="blocker">
<h1 style="color:#f00;">
MATH ESCAPE
</h1>
<p>徘徊する鍵に触れ、</p>
<p>計算クイズに10問正解し、</p>
<p>白い出口へ脱出せよ。</p>
<br>
<p style="color:#f00;
border:1px solid #f00;
padding:10px;">
【ココをクリックして開始】
</p>
</div>

<div id="canvas-container"></div>

<div id="ui-layer">
<div id="status-bar">
<div style="color:#f33;">
暴走度: 
<span id="rate">0</span>%
<span id="p-label" 
style="color:#0ff;
display:none;">
(計算中停止)</span>
</div>
<div id="items" 
style="color:#0ff;">
正解数: 0 / 10
</div>
<div id="warn" 
style="color:#f00;
display:none;">
⚠️AI覚醒！接近中！
</div>
</div>

<div id="ai-window">
<strong>[HAL]:</strong>
<div id="msg">
問題：数字を入力して送信せよ。
</div>
<form id="chat-form" 
onsubmit="return false;">
<input type="text" 
id="user-input" 
placeholder="答えを入力..." 
autocomplete="off">
<button id="send-btn">
解答
</button>
</form>
</div>
</div>

<script>
const container = 
document.getElementById(
'canvas-container'
);
const canvas = 
document.createElement(
'canvas'
);
container.appendChild(
canvas
);
const ctx = 
canvas.getContext('2d');

let w = window.innerWidth;
let h = window.innerHeight;
canvas.width = w; canvas.height = h;

let player = { x: 0, z: 0, angle: 0 };
let enemy = { x: -18, z: -18, size: 4, active: false };
let exitGate = { x: 20, z: 20, size: 3 };

let moveF = false, moveB = false;
let moveL = false, moveR = false;
let corruption = 0;
let isGameOver = false;
let isChatting = false;
let score = 0; // 正解数

// クイズ出題中の管理用データ
let currentAns = null;
let quizActive = false;

// 小学4年生向けのクイズ10問（たし・ひき・かけ・わり算）
const quizList = [
  { q: "328 + 45 = ?", a: 373 },
  { q: "900 - 125 = ?", a: 775 },
  { q: "24 × 6 = ?", a: 144 },
  { q: "72 ÷ 4 = ?", a: 18 },
  { q: "105 × 3 = ?", a: 315 },
  { q: "480 ÷ 5 = ?", a: 96 },
  { q: "135 + 246 = ?", a: 381 },
  { q: "503 - 87 = ?", a: 415 },
  { q: "16 × 15 = ?", a: 240 },
  { q: "240 ÷ 8 = ?", a: 30 }
];

// 迷宮内を動き回る鍵キューブのデータ
const keys = [
  {x:-15, z:10, vx:2, vz:1},
  {x:15, z:-15, vx:-1, vz:2},
  {x:-5, z:-18, vx:1, vz:-2}
];

const pillars = [];
for(let i=0; i<8; i++) {
  pillars.push({
    x: (Math.random()-0.5)*30,
    z: (Math.random()-0.5)*30,
    size: 2
  });
}

const blocker = document.getElementById('blocker');
const msgBox = document.getElementById('msg');
const rateBox = document.getElementById('rate');
const pLabel = document.getElementById('p-label');
const itemBox = document.getElementById('items');
const warnBox = document.getElementById('warn');
const userInput = document.getElementById('user-input');
const sendBtn = document.getElementById('send-btn');

userInput.addEventListener('focus', () => { 
  isChatting = true; 
  pLabel.style.display = 'inline';
});
userInput.addEventListener('blur', () => { 
  isChatting = false; 
  pLabel.style.display = 'none';
});

// クイズを出題する関数
function askQuiz() {
  if (score >= 10) return;
  quizActive = true;
  let qData = quizList[score];
  currentAns = qData.a;
  msgBox.innerText = "【問題】" + qData.q;
  userInput.focus();
}

function handleSend() {
  const text = userInput.value.trim();
  if (!text || isGameOver) return;
  userInput.value = "";

  if (quizActive) {
    if (parseInt(text) === currentAns) {
      score++;
      quizActive = false;
      itemBox.innerText = "正解数: " + score + " / 10";
      msgBox.innerText = "正解！鍵を手に入れた。";
      if (score >= 10) {
        itemBox.innerText = "脱出ゲート開放！白い光へ向かえ！";
      }
    } else {
      // 間違えるとAIが激怒して暴走度が20%跳ね上がる
      corruption += 20;
      if (corruption > 100) corruption = 100;
      rateBox.innerText = Math.floor(corruption);
      msgBox.innerText = "不正解！バカな人間め。もう一度答えてみろ。";
    }
  } else {
    msgBox.innerText = "鍵を見つけてクイズに答えよ。";
  }
}

sendBtn.addEventListener('click', handleSend);
userInput.addEventListener('keypress', (e) => {
  if (e.key === 'Enter') handleSend();
});

blocker.addEventListener('click', function () {
  if (!isGameOver) blocker.style.display = 'none';
});

document.addEventListener('keydown', (e) => {
  if (isChatting) return;
  if (e.code === 'KeyW' || e.code === 'ArrowUp') moveF = true;
  if (e.code === 'KeyS' || e.code === 'ArrowDown') moveB = true;
  if (e.code === 'KeyA' || e.code === 'ArrowLeft') moveL = true;
  if (e.code === 'KeyD' || e.code === 'ArrowRight') moveR = true;
});

document.addEventListener('keyup', (e) => {
  if (e.code === 'KeyW' || e.code === 'ArrowUp') moveF = false;
  if (e.code === 'KeyS' || e.code === 'ArrowDown') moveB = false;
  if (e.code === 'KeyA' || e.code === 'ArrowLeft') moveL = false;
  if (e.code === 'KeyD' || e.code === 'ArrowRight') moveR = false;
});

document.addEventListener('mousemove', (e) => {
  if (blocker.style.display === 'none' && !isChatting) {
    player.angle += e.movementX * 0.004;
  }
});

window.addEventListener('resize', () => {
  w = window.innerWidth; h = window.innerHeight;
  canvas.width = w; canvas.height = h;
});

let lastTime = performance.now();
function animate() {
  requestAnimationFrame(
    animate
  );
  if (isGameOver) return;

  const now = performance.now();
  const delta = (now - lastTime) / 1000;
  lastTime = now;

  if (blocker.style.display === 'none') {
    let speed = 9 * delta;
    if (!isChatting) {
      if (moveF) {
        player.x += Math.sin(player.angle)*speed;
        player.z += Math.cos(player.angle)*speed;
      }
      if (moveB) {
        player.x -= Math.sin(player.angle)*speed;
        player.z -= Math.cos(player.angle)*speed;
      }
      if (moveL) {
        player.x += Math.cos(player.angle)*speed;
        player.z -= Math.sin(player.angle)*speed;
      }
      if (moveR) {
        player.x -= Math.cos(player.angle)*speed;
        player.z += Math.sin(player.angle)*speed;
      }
    }

    player.x = Math.max(-23, Math.min(23, player.x));
    player.z = Math.max(-23, Math.min(23, player.z));

    // 鍵キューブの移動＆跳ね返り処理
    for (let k of keys) {
      k.x += k.vx * delta;
      k.z += k.vz * delta;
      if (Math.abs(k.x) >= 23) k.vx *= -1;
      if (Math.abs(k.z) >= 23) k.vz *= -1;

      // プレイヤーが鍵に触れたらクイズ出題（解答中でない時）
      let kdx = player.x - k.x;
      let kdz = player.z - k.z;
      if (Math.sqrt(kdx*kdx + kdz*kdz) < 1.5 && !quizActive) {
        askQuiz();
      }
    }

    // チャット入力中でない時だけ2秒に1%（1秒に0.5%）暴走を進める
    if (corruption < 100 && !isChatting) {
      corruption += delta * 0.5;
      if (corruption >= 100) {
        corruption = 100;
        enemy.active = true;
      }
      rateBox.innerText = Math.floor(corruption);
    }

    // 出口クリア判定
    if (score >= 10) {
      let gdx = player.x - exitGate.x;
      let gdz = player.z - exitGate.z;
      if (Math.sqrt(gdx*gdx + gdz*gdz) < 1.5) {
        isGameOver = true;
        triggerWin();
        return;
      }
    }

    // 暴走度100%での敵追跡処理
    if (enemy.active) {
      warnBox.style.display = 'block';
      let eSpeed = 12 * delta; 
      let edx = player.x - enemy.x;
      let edz = player.z - enemy.z;
      let edist = Math.sqrt(edx*edx + edz*edz);
      
      if (edist > 0.1) {
        enemy.x += (edx / edist) * eSpeed;
        enemy.z += (edz / edist) * eSpeed;
      }

      if (edist < 1.3) {
        triggerGO();
        return;
      }
    }
  }

  ctx.fillStyle = '#000'; ctx.fillRect(0,0,w,h);
  ctx.fillStyle = '#111'; ctx.fillRect(0, h / 2, w, h / 2);

  const fov = Math.PI / 3; const numRays = 80; const rayStep = fov / numRays;

  for (let i=0; i<numRays; i++) {
    let rayAngle = player.angle - fov / 2 + i * rayStep;
    let distance = 0; let hit = false; let hitType = 0; let hitColor = '#fff';

    while (distance < 35 && !hit) {
      distance += 0.25;
      let checkX = player.x + Math.sin(rayAngle)*distance;
      let checkZ = player.z + Math.cos(rayAngle)*distance;

      if (enemy.active) {
        if (Math.abs(checkX - enemy.x) < enemy.size/2 && Math.abs(checkZ - enemy.z) < enemy.size/2) {
          hit = true; hitType = 2;
          hitColor = (Math.sin(performance.now()*0.03) > 0) ? '#f00' : '#200';
        }
      }

      if (!hit && score < 10) {
        for (let k of keys) {
          if (Math.abs(checkX - k.x) < 0.8 && Math.abs(checkZ - k.z) < 0.8) {
            hit = true; hitType = 3; hitColor = '#0ff';
          }
        }
      }

      if (!hit && score >= 10) {
        if (Math.abs(checkX - exitGate.x) < exitGate.size/2 && Math.abs(checkZ - exitGate.z) < exitGate.size/2) {
          hit = true; hitType = 4; hitColor = '#fff';
        }
      }

      if (!hit) {
        for (let p of pillars) {
          if (Math.abs(checkX - p.x) < p.size/2 && Math.abs(checkZ - p.z) < p.size/2) {
            hit = true; hitType = 1; hitColor = '#444';
          }
        }
      }

      if (!hit) {
        if (checkX >= 23.8) { hit = true; hitColor = '#800'; } 
        else if (checkX <= -23.8) { hit = true; hitColor = '#008'; } 
        else if (checkZ >= 23.8) { hit = true; hitColor = '#505'; } 
        else if (checkZ <= -23.8) { hit = true; hitColor = '#880'; } 
      }
    }

    let cDist = distance * Math.cos(rayAngle - player.angle);
    let wallH = Math.min(h, (h / cDist)*1.5);

    ctx.fillStyle = hitColor;
    let startX = (i * (w / numRays));
    let stepW = (w / numRays) + 1;
    ctx.fillRect(startX, (h - wallH) / 2, stepW, wallH);

    if (hitType === 2 && wallH > 15) {
      ctx.fillStyle = '#fff'; ctx.fillRect(startX, (h - 6) / 2, stepW, 6);
    }
  }
}

function triggerGO() {
  isGameOver = true;
  blocker.style.background = "#300";
  let hStr = "<h1>GAMEOVER</h1><p>デリートされました</p><br><p onclick='location.reload();' style='border:1px solid #fff;padding:10px;'>リトライ</p>";
  blocker.innerHTML = hStr; blocker.style.display = 'flex';
}

function triggerWin() {
  blocker.style.background = "#033";
  let wStr = "<h1>CLEAR</h1><p>全問正解し、無事脱出した！</p><br><p onclick='location.reload();' style='border:1px solid #fff;padding:10px;'>もう一度遊ぶ</p>";
  blocker.innerHTML = wStr; blocker.style.display = 'flex';
}

lastTime = performance.now();
animate();
</script>
</body>
</html>
