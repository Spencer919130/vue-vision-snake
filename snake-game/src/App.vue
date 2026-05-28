<template>
  <div class="game-container">
    <div class="header-section">
      <h1>Vision Snake 🐍 <span class="badge" :class="{ 'frenzy-mode': isFrenzy }">{{ isFrenzy ? '暴走衝刺 👊' : '滑軌導航' }}</span></h1>
      <p class="status-bar">狀態：<span :style="{ color: isReady ? '#00ffcc' : '#ff4444' }">{{ statusMessage }}</span></p>
      
      <div class="dashboard">
        <div class="dash-item">鎖定位置 X: <span class="dash-val">{{ handX.toFixed(0) }}</span>, Y: <span class="dash-val">{{ handY.toFixed(0) }}</span></div>
        <div class="dash-item">目前狀態: <span class="dash-val" style="color: #ff3366;">{{ handGestureText }}</span></div>
        <div class="dash-item">蛇前進方向: <span class="dash-val" style="color: #00ffcc;">{{ currentDirection }}</span></div>
      </div>

      <div class="score-board">
        <span>得分: <strong class="score-text">{{ score }}</strong></span>
        <span>最高紀錄: <strong>{{ highScore }}</strong></span>
      </div>
    </div>
    
    <div class="view-group">
      <div class="video-box" :class="{ 'frenzy-border': isFrenzy }">
        <p>視訊偵測 (食指導航 / 握拳暴走)</p>
        <video ref="videoElement" autoplay playsinline width="240" height="160"></video>
      </div>

      <div class="canvas-box">
        <canvas ref="gameCanvas" width="600" height="400" class="game-board" :class="{ 'frenzy-shake': isFrenzy }"></canvas>
        
        <div v-if="gameOver" class="game-over-overlay">
          <h2>GAME OVER</h2>
          <p>最終得分: {{ score }}</p>
          <button @click="resetGame" class="restart-btn">再次挑戰</button>
        </div>
      </div>
    </div>

    <div class="instructions">
      <h3>🎮 核心進化玩法：食指十字滑軌導航</h3>
      <ul>
        <li><strong>軌道滑行控制：</strong> 伸出單手，用<strong>食指指尖</strong>在鏡頭前移動。畫面中央有兩條固定的滑軌虛線，定位點會被<strong>強迫鎖定在 X 軸或 Y 軸上</strong>滑動，大幅簡化操作！</li>
        <li><strong>轉向判定：</strong> 將食指推到畫面上方或下方（鎖定在縱軸），蛇便會上下轉向；將食指移到畫面左方或右方（鎖定在橫軸），蛇便會左右轉向。</li>
        <li><strong>✊ 握拳衝刺：</strong> 隨時將手掌<strong>用力握拳</strong>（伸出手指為 0），即可進入「暴走模式」，此時速度與得分皆翻倍，但身體每 0.6 秒會強迫自動變長！</li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const gameCanvas = ref(null);
const videoElement = ref(null);
const statusMessage = ref('正在初始化食指滑軌導航系統...');
const isReady = ref(false);

// 遊戲狀態
const score = ref(0);
const highScore = ref(0);
const gameOver = ref(false);
const isFrenzy = ref(false);

// 食指鎖定座標與狀態
const handX = ref(300);
const handY = ref(200);
const handGestureText = ref('未偵測');

// 蛇與食物
const snakeBody = ref([
  { x: 300, y: 200 },
  { x: 280, y: 200 },
  { x: 260, y: 200 }
]);
const segmentSize = 20;
const food = ref({ x: 0, y: 0 });

const currentDirection = ref('RIGHT');
let nextDirection = 'RIGHT';

let gameIntervalId = null;
let frenzyTimerId = null;

const generateFood = () => {
  const maxX = (600 / segmentSize) - 1;
  const maxY = (400 / segmentSize) - 1;
  food.value = {
    x: Math.floor(Math.random() * maxX) * segmentSize,
    y: Math.floor(Math.random() * maxY) * segmentSize
  };
};

// 判定伸出的手指數量（物理幾何距離法）
const countExtendedFingers = (landmarks) => {
  const palmCenter = landmarks[9]; // 中指根部
  const fingerTips = [8, 12, 16, 20];
  let count = 0;

  fingerTips.forEach(tipId => {
    const tip = landmarks[tipId];
    const distance = Math.sqrt(Math.pow(tip.x - palmCenter.x, 2) + Math.pow(tip.y - palmCenter.y, 2));
    if (distance > 0.18) count++;
  });

  const thumbTip = landmarks[4];
  const indexBase = landmarks[5];
  const thumbDistance = Math.sqrt(Math.pow(thumbTip.x - indexBase.x, 2) + Math.pow(thumbTip.y - indexBase.y, 2));
  if (thumbDistance > 0.13) count++;

  return count;
};

// 解析食指絕對象限與中央軌道鎖定邏輯
const processAbsoluteNavigation = (multiHandLandmarks) => {
  if (!multiHandLandmarks || multiHandLandmarks.length === 0) {
    handGestureText.value = "未偵測到手";
    statusMessage.value = "⚠️ 請將手放進鏡頭內！";
    return;
  }

  statusMessage.value = "🟢 軌道搖桿連線成功！";
  const landmarks = multiHandLandmarks[0];
  
  // 取得食指指尖 (Landmark 8) 的原始映射座標
  const rawX = (1 - landmarks[8].x) * 600;
  const rawY = landmarks[8].y * 400;

  // 檢查是否握拳觸發暴走
  const fingers = countExtendedFingers(landmarks);
  
  if (fingers === 0) {
    handGestureText.value = "✊ 握拳 (暴走衝刺)";
    if (!isFrenzy.value) {
      isFrenzy.value = true;
      clearInterval(frenzyTimerId);
      frenzyTimerId = setInterval(() => {
        if (!gameOver.value) {
          const tail = { ...snakeBody.value[snakeBody.value.length - 1] };
          snakeBody.value.push(tail);
        }
      }, 600);
    }
    return; // 暴走期間鎖定當前方向，防止握拳瞬間產生的抖動誤判轉向
  } else {
    handGestureText.value = `☝️ 食指導航 (${fingers} 指伸出)`;
    if (isFrenzy.value) {
      isFrenzy.value = false;
      clearInterval(frenzyTimerId);
    }
  }

  // 🎯 十字軌道物理鎖定核心演算法
  const distToVerticalAxis = Math.abs(rawX - 300);   // 離中央縱軸距離
  const distToHorizontalAxis = Math.abs(rawY - 200); // 離中央橫軸距離

  if (distToVerticalAxis < distToHorizontalAxis) {
    // 離縱軸比較近 -> 強制把 X 軸吸附在中央 300 線上，小圓圈只能上下滑動
    handX.value = 300;
    handY.value = rawY;
  } else {
    // 離橫軸比較近 -> 強制把 Y 軸吸附在中央 200 線上，小圓圈只能左右滑動
    handX.value = rawX;
    handY.value = 200;
  }

  // 🗺️ 依據吸附在軌道上的相對位置下達轉向命令
  const dx = handX.value - 300;
  const dy = handY.value - 200;

  // 中央死區 (Dead-zone)，手在十字路口中心點附近時不轉向
  if (Math.abs(dx) < 30 && Math.abs(dy) < 30) return;

  if (handX.value !== 300) {
    // 圓圈鎖定在橫軸上 (控制左右轉)
    if (dx > 0) {
      if (currentDirection.value !== 'LEFT') nextDirection = 'RIGHT';
    } else {
      if (currentDirection.value !== 'RIGHT') nextDirection = 'LEFT';
    }
  } else {
    // 圓圈鎖定在縱軸上 (控制上下轉)
    if (dy > 0) {
      if (currentDirection.value !== 'UP') nextDirection = 'DOWN';
    } else {
      if (currentDirection.value !== 'DOWN') nextDirection = 'UP';
    }
  }
};

// 獨立時脈的核心物理引擎
const gameTick = () => {
  if (gameOver.value) return;

  currentDirection.value = nextDirection;
  const head = { ...snakeBody.value[0] };

  if (currentDirection.value === 'LEFT') head.x -= segmentSize;
  if (currentDirection.value === 'RIGHT') head.x += segmentSize;
  if (currentDirection.value === 'UP') head.y -= segmentSize;
  if (currentDirection.value === 'DOWN') head.y += segmentSize;

  if (head.x < 0 || head.x >= 600 || head.y < 0 || head.y >= 400) return endGame();

  for (let i = 1; i < snakeBody.value.length; i++) {
    if (head.x === snakeBody.value[i].x && head.y === snakeBody.value[i].y) return endGame();
  }

  snakeBody.value.unshift(head);

  if (head.x === food.value.x && head.y === food.value.y) {
    score.value += isFrenzy.value ? 30 : 10;
    generateFood();
  } else {
    snakeBody.value.pop();
  }

  drawGame();
};

// 繪製 Canvas 畫面
const drawGame = () => {
  if (!gameCanvas.value) return;
  const ctx = gameCanvas.value.getContext('2d');

  ctx.fillStyle = isFrenzy.value ? '#220505' : '#111111';
  ctx.fillRect(0, 0, 600, 400);

  // 1. 畫背景的十字滑軌虛線（固定在正中央 300, 200）
  ctx.strokeStyle = isFrenzy.value ? '#4a1515' : '#252525';
  ctx.lineWidth = 2; // 稍微加粗讓軌道更明顯
  ctx.setLineDash([6, 6]);
  ctx.beginPath(); ctx.moveTo(300, 0); ctx.lineTo(300, 400); ctx.stroke();
  ctx.beginPath(); ctx.moveTo(0, 200); ctx.lineTo(600, 200); ctx.stroke();
  ctx.setLineDash([]); // 還原實線

  // 2. 🎯 優化視覺：只畫被強制「鎖定」在軌道上的孤單小圓圈
  if (isReady.value) {
    ctx.fillStyle = isFrenzy.value ? '#ff0055' : '#00ffcc';
    ctx.shadowBlur = 12;
    ctx.shadowColor = isFrenzy.value ? '#ff0055' : '#00ffcc';
    
    ctx.beginPath();
    // 繪製鎖定後的 handX 與 handY 座標
    ctx.arc(handX.value, handY.value, 8, 0, Math.PI * 2);
    ctx.fill();
    ctx.shadowBlur = 0; // 解除發光避免影響效能
  }

  // 3. 畫食物
  ctx.fillStyle = '#ff3366';
  ctx.fillRect(food.value.x + 2, food.value.y + 2, segmentSize - 4, segmentSize - 4);

  // 4. 畫蛇身
  snakeBody.value.forEach((seg, i) => {
    if (i === 0) {
      ctx.fillStyle = isFrenzy.value ? '#ff0055' : '#00ffcc';
    } else {
      ctx.fillStyle = isFrenzy.value ? '#aa0033' : '#00aa99';
    }
    ctx.fillRect(seg.x, seg.y, segmentSize, segmentSize);
  });
};

let currentSpeed = 140;
const manageGameLoop = () => {
  clearInterval(gameIntervalId);
  gameIntervalId = setInterval(() => {
    gameTick();
    const targetSpeed = isFrenzy.value ? 70 : 140;
    if (currentSpeed !== targetSpeed) {
      currentSpeed = targetSpeed;
      manageGameLoop();
    }
  }, currentSpeed);
};

const endGame = () => {
  gameOver.value = true;
  if (score.value > highScore.value) highScore.value = score.value;
  clearInterval(gameIntervalId);
  clearInterval(frenzyTimerId);
};

const resetGame = () => {
  snakeBody.value = [{ x: 300, y: 200 }, { x: 280, y: 200 }, { x: 260, y: 200 }];
  score.value = 0;
  gameOver.value = false;
  isFrenzy.value = false;
  currentDirection.value = 'RIGHT';
  nextDirection = 'RIGHT';
  generateFood();
  manageGameLoop();
};

onMounted(() => {
  generateFood();
  
  const MyHands = window.Hands;
  const MyCamera = window.Camera;

  if (MyHands && MyCamera) {
    const hands = new MyHands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
    hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
    
    hands.onResults((results) => {
      if (results.multiHandLandmarks) {
        processAbsoluteNavigation(results.multiHandLandmarks);
      }
    });

    if (videoElement.value) {
      const camera = new MyCamera(videoElement.value, {
        onFrame: async () => { await hands.send({ image: videoElement.value }); },
        width: 240,
        height: 160
      });
      camera.start().then(() => {
        isReady.value = true;
        manageGameLoop();
      });
    }
  }
});
</script>