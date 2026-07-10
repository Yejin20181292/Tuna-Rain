<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue';

interface FallingItem {
  id: number;
  x: number;
  y: number;
  speed: number;
  width: number;
  height: number;
  type: 'normal' | 'golden' | 'bone';
  angle: number;
  spinSpeed: number;
}

interface ScorePopup {
  id: number;
  x: number;
  y: number;
  text: string;
  age: number;
  isGolden: boolean;
  isHeart?: boolean;
  isHazard?: boolean;
}

// Game states: TITLE, PLAYING, PAUSED, GAMEOVER
const gameState = ref<'TITLE' | 'PLAYING' | 'PAUSED' | 'GAMEOVER'>('TITLE');
const isLoading = ref(true);

const processedTunaSrc = ref('');
const processedMeatSrc = ref('');
const processedBoneSrc = ref('');

const score = ref(0);
const highScore = ref(0);
const lives = ref(10);

interface LeaderboardEntry {
  name: string;
  score: number;
  level: number;
  date: string;
}

const leaderboard = ref<LeaderboardEntry[]>([]);
const playerName = ref('');
const showNameInput = ref(false);
const submittedName = ref(false);
const newRankIndex = ref(-1);

const defaultLeaderboard: LeaderboardEntry[] = [
  { name: 'CAT_CATCH', score: 1200, level: 5, date: '2026-07-10' },
  { name: 'TUNA_KING', score: 1000, level: 5, date: '2026-07-10' },
  { name: 'CAN_OPENR', score: 850, level: 4, date: '2026-07-10' },
  { name: 'TUNA_LVER', score: 700, level: 3, date: '2026-07-10' },
  { name: 'SALMON_NO', score: 550, level: 3, date: '2026-07-10' },
  { name: 'ARCADE_FN', score: 400, level: 2, date: '2026-07-10' },
  { name: 'RAIN_DANC', score: 300, level: 2, date: '2026-07-10' },
  { name: 'FAST_HAND', score: 200, level: 1, date: '2026-07-10' },
  { name: 'TUNA_LITE', score: 100, level: 1, date: '2026-07-10' },
  { name: 'NOOB_PLAY', score: 50, level: 1, date: '2026-07-10' },
];

const submitScore = () => {
  if (submittedName.value) return;
  const nameToSave = playerName.value.trim().toUpperCase().slice(0, 8) || 'PLAYER';
  const entry: LeaderboardEntry = {
    name: nameToSave,
    score: score.value,
    level: currentLevel.value,
    date: new Date().toISOString().split('T')[0]
  };
  leaderboard.value.push(entry);
  leaderboard.value.sort((a, b) => b.score - a.score);
  leaderboard.value = leaderboard.value.slice(0, 10);
  localStorage.setItem('tuna_rain_leaderboard', JSON.stringify(leaderboard.value));
  
  newRankIndex.value = leaderboard.value.findIndex(e => e.score === entry.score && e.name === entry.name && e.level === entry.level);
  showNameInput.value = false;
  submittedName.value = true;
};

const offsetX = ref(0);
const rotation = ref(0);
const isTeleporting = ref(false);

const fallingItems = ref<FallingItem[]>([]);
const scorePopups = ref<ScorePopup[]>([]);

const currentLevel = ref(1);
const showLevelUp = ref(false);
let levelUpTimeout: number | null = null;

const triggerLevelUp = () => {
  showLevelUp.value = false;
  setTimeout(() => {
    showLevelUp.value = true;
    if (levelUpTimeout) clearTimeout(levelUpTimeout);
    levelUpTimeout = window.setTimeout(() => {
      showLevelUp.value = false;
    }, 2000);
  }, 50);
};

const keysPressed = {
  ArrowLeft: false,
  ArrowRight: false
};

let animationFrameId = 0;
let lastTime = 0;
let spawnTimer = 0;

const offsetXPx = computed(() => `${offsetX.value}px`);
const rotationDeg = computed(() => `${rotation.value}deg`);
const transitionStyle = computed(() => 
  isTeleporting.value ? 'none' : 'transform 0.1s linear' // Faster, tighter transition for frame-based controls
);

// Adaptive Chrome Keying to remove image backgrounds
const processImage = (src: string): Promise<string> => {
  return new Promise((resolve) => {
    const img = new Image();
    img.src = src;
    img.onload = () => {
      const canvas = document.createElement('canvas');
      canvas.width = img.width;
      canvas.height = img.height;
      const ctx = canvas.getContext('2d');
      if (!ctx) {
        resolve(src);
        return;
      }
      
      ctx.drawImage(img, 0, 0);
      const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const data = imgData.data;
      const width = canvas.width;
      const height = canvas.height;
      
      const getIdx = (x: number, y: number) => (y * width + x) * 4;
      const refR = data[0];
      const refG = data[1];
      const refB = data[2];
      
      const visited = new Uint8Array(width * height);
      const queue: [number, number][] = [];
      const corners = [[0, 0], [width - 1, 0], [0, height - 1], [width - 1, height - 1]];
      
      for (const [cx, cy] of corners) {
        const idx = cy * width + cx;
        if (!visited[idx]) {
          visited[idx] = 1;
          queue.push([cx, cy]);
        }
      }
      
      while (queue.length > 0) {
        const [x, y] = queue.shift()!;
        const idx = getIdx(x, y);
        const r = data[idx];
        const g = data[idx+1];
        const b = data[idx+2];
        
        const colorDist = Math.abs(r - refR) + Math.abs(g - refG) + Math.abs(b - refB);
        if (colorDist < 45) {
          data[idx+3] = 0; // Make transparent
          
          const neighbors = [[x - 1, y], [x + 1, y], [x, y - 1], [x, y + 1]];
          for (const [nx, ny] of neighbors) {
            if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
              const nidx = ny * width + nx;
              if (!visited[nidx]) {
                visited[nidx] = 1;
                queue.push([nx, ny]);
              }
            }
          }
        }
      }
      
      ctx.putImageData(imgData, 0, 0);
      resolve(canvas.toDataURL('image/png'));
    };
  });
};

const resumeGame = () => {
  if (gameState.value !== 'PAUSED') return;
  gameState.value = 'PLAYING';
  lastTime = 0;
  animationFrameId = requestAnimationFrame(gameLoop);
};

const handleKeyDown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    if (gameState.value === 'PLAYING') {
      gameState.value = 'PAUSED';
      cancelAnimationFrame(animationFrameId);
      keysPressed.ArrowLeft = false;
      keysPressed.ArrowRight = false;
    } else if (gameState.value === 'PAUSED') {
      resumeGame();
    }
    return;
  }
  
  if (gameState.value !== 'PLAYING') return;
  if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
    event.preventDefault();
    keysPressed[event.key] = true;
  }
};

const handleKeyUp = (event: KeyboardEvent) => {
  if (event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
    keysPressed[event.key] = false;
  }
};

const spawnItem = () => {
  const rand = Math.random();
  let type: 'normal' | 'golden' | 'bone' = 'normal';
  let width = 85;
  let height = 60;
  
  if (rand < 0.25) { // 25% chance of spawning hazard fish bone
    type = 'bone';
    width = 65;
    height = 37; // slightly smaller than tuna chunk
  } else if (rand < 0.3625) { // 11.25% chance of spawning golden tuna
    type = 'golden';
    width = 85;
    height = 60;
  } else { // 63.75% chance of spawning normal tuna
    type = 'normal';
    width = 85;
    height = 60;
  }
  
  const x = Math.random() * (window.innerWidth - width);
  const y = -height;
  
  // Base speed scales up slowly by level (max Level 5)
  const baseSpeed = Math.random() * 2 + 3.5;
  const speed = baseSpeed + (currentLevel.value - 1) * 0.9;
  
  const angle = Math.random() * 360;
  const spinSpeed = (Math.random() - 0.5) * 4;
  
  fallingItems.value.push({
    id: Date.now() + Math.random(),
    x,
    y,
    width,
    height,
    speed,
    type,
    angle,
    spinSpeed
  });
};

const catchItem = (item: FallingItem) => {
  if (item.type === 'bone') {
    loseLife(); // Lose 1 life on catching a fish bone hazard
    
    // Spawn floating damage popup (-1 ❤️)
    scorePopups.value.push({
      id: Date.now() + Math.random(),
      x: item.x + item.width / 2,
      y: item.y,
      text: '-1 ❤️',
      age: 0,
      isGolden: false,
      isHazard: true
    });
    return;
  }
  
  const points = item.type === 'golden' ? 50 : 10;
  score.value += points;
  
  // Check for Level Up (every 400 points, max Level 5)
  const newLevel = Math.min(5, Math.floor(score.value / 400) + 1);
  if (newLevel > currentLevel.value) {
    currentLevel.value = newLevel;
    triggerLevelUp();
  }
  
  if (score.value > highScore.value) {
    highScore.value = score.value;
    localStorage.setItem('tuna_rain_highscore', highScore.value.toString());
  }
  
  if (item.type === 'golden') {
    lives.value = Math.min(10, lives.value + 1); // Increase lives by 1, cap at 10
    
    // Spawn score popup +50
    scorePopups.value.push({
      id: Date.now() + Math.random(),
      x: item.x + item.width / 2 - 22,
      y: item.y,
      text: '+50',
      age: 0,
      isGolden: true
    });
    
    // Spawn life recovery popup (+1 ❤️)
    scorePopups.value.push({
      id: Date.now() + Math.random(),
      x: item.x + item.width / 2 + 22,
      y: item.y,
      text: '+1 ❤️',
      age: 0,
      isGolden: false,
      isHeart: true
    });
  } else {
    // Normal chunk
    scorePopups.value.push({
      id: Date.now() + Math.random(),
      x: item.x + item.width / 2,
      y: item.y,
      text: `+${points}`,
      age: 0,
      isGolden: false
    });
  }
};

const loseLife = () => {
  lives.value -= 1;
  if (lives.value <= 0) {
    gameState.value = 'GAMEOVER';
    cancelAnimationFrame(animationFrameId);
    
    // Check if score enters TOP 10
    submittedName.value = false;
    playerName.value = '';
    newRankIndex.value = -1;
    
    const isTop10 = leaderboard.value.length < 10 || score.value > leaderboard.value[leaderboard.value.length - 1]?.score;
    if (isTop10) {
      showNameInput.value = true;
    } else {
      showNameInput.value = false;
    }
  }
};

const gameLoop = (timestamp: number) => {
  if (gameState.value !== 'PLAYING') return;
  
  if (!lastTime) lastTime = timestamp;
  const deltaTime = timestamp - lastTime;
  lastTime = timestamp;
  
  // 1. Move Player
  const moveSpeed = 25; // pixels per frame
  if (keysPressed.ArrowLeft) {
    offsetX.value -= moveSpeed;
    rotation.value = -12;
  } else if (keysPressed.ArrowRight) {
    offsetX.value += moveSpeed;
    rotation.value = 12;
  } else {
    rotation.value = rotation.value * 0.8;
    if (Math.abs(rotation.value) < 0.1) rotation.value = 0;
  }
  
  // Wrap-around bounds
  const wrapBoundary = window.innerWidth / 2 + 100;
  if (offsetX.value > wrapBoundary) {
    isTeleporting.value = true;
    offsetX.value = -wrapBoundary;
    setTimeout(() => { isTeleporting.value = false; }, 50);
  } else if (offsetX.value < -wrapBoundary) {
    isTeleporting.value = true;
    offsetX.value = wrapBoundary;
    setTimeout(() => { isTeleporting.value = false; }, 50);
  }
  
  // 2. Spawning Chunks
  spawnTimer += deltaTime;
  const spawnInterval = Math.max(300, 1000 - (currentLevel.value - 1) * 120);
  if (spawnTimer >= spawnInterval) {
    spawnTimer = 0;
    spawnItem();
  }
  
  // 3. Update & Collision Check
  const canYTop = window.innerHeight - 38 - 135; // Top of the can
  const canYBottom = window.innerHeight - 38;    // Bottom of the can
  const canWidth = 190;
  const canXLeft = window.innerWidth / 2 + offsetX.value - canWidth / 2;
  const canXRight = window.innerWidth / 2 + offsetX.value + canWidth / 2;
  
  for (let i = fallingItems.value.length - 1; i >= 0; i--) {
    const item = fallingItems.value[i];
    item.y += item.speed;
    item.angle += item.spinSpeed;
    
    const itemBottom = item.y + item.height;
    
    // Check overlap with the mouth/rim of the can
    if (
      itemBottom >= canYTop &&
      item.y <= canYBottom &&
      item.x + item.width / 2 >= canXLeft &&
      item.x + item.width / 2 <= canXRight
    ) {
      catchItem(item);
      fallingItems.value.splice(i, 1);
    } else if (item.y > window.innerHeight) {
      if (item.type === 'normal') {
        loseLife();
      }
      fallingItems.value.splice(i, 1);
    }
  }
  
  // 4. Update Score Popups
  for (let i = scorePopups.value.length - 1; i >= 0; i--) {
    const popup = scorePopups.value[i];
    popup.y -= 1.8;
    popup.age += deltaTime;
    if (popup.age > 800) {
      scorePopups.value.splice(i, 1);
    }
  }
  
  animationFrameId = requestAnimationFrame(gameLoop);
};

const startGame = () => {
  score.value = 0;
  lives.value = 10;
  offsetX.value = 0;
  rotation.value = 0;
  fallingItems.value = [];
  scorePopups.value = [];
  keysPressed.ArrowLeft = false;
  keysPressed.ArrowRight = false;
  lastTime = 0;
  spawnTimer = 0;
  
  currentLevel.value = 1;
  triggerLevelUp(); // Trigger "LEVEL 1" banner at start
  
  gameState.value = 'PLAYING';
  animationFrameId = requestAnimationFrame(gameLoop);
};

onMounted(async () => {
  processedTunaSrc.value = await processImage('/tuna-can.jpg');
  processedMeatSrc.value = await processImage('/tuna-meat.png');
  processedBoneSrc.value = await processImage('/fish-bone.png');
  isLoading.value = false;
  
  const savedHighScore = localStorage.getItem('tuna_rain_highscore');
  if (savedHighScore) {
    highScore.value = parseInt(savedHighScore, 10);
  }

  const savedLeaderboard = localStorage.getItem('tuna_rain_leaderboard');
  if (savedLeaderboard) {
    leaderboard.value = JSON.parse(savedLeaderboard);
  } else {
    leaderboard.value = [...defaultLeaderboard];
    localStorage.setItem('tuna_rain_leaderboard', JSON.stringify(defaultLeaderboard));
  }
  
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('keyup', handleKeyUp);
});

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId);
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
});
</script>

<template>
  <div class="fullscreen-bg">
    <!-- Game HUD -->
    <div v-if="gameState === 'PLAYING'" class="game-hud">
      <div class="score-container">
        <span class="label">SCORE</span>
        <span class="value">{{ score }}</span>
      </div>
      <div class="lives-container">
        <div 
          v-for="life in 10" 
          :key="life" 
          class="heart" 
          :class="{ lost: life > lives }"
        >
          ❤️
        </div>
      </div>
    </div>

    <!-- Falling Items Layer -->
    <div v-if="gameState === 'PLAYING' && processedMeatSrc && processedBoneSrc" class="falling-layer">
      <div 
        v-for="item in fallingItems" 
        :key="item.id" 
        class="falling-chunk"
        :class="item.type"
        :style="{
          left: `${item.x}px`,
          top: `${item.y}px`,
          width: `${item.width}px`,
          height: `${item.height}px`,
          transform: `rotate(${item.angle}deg)`
        }"
      >
        <img v-if="item.type === 'bone'" :src="processedBoneSrc" alt="Fish Bone" />
        <img v-else :src="processedMeatSrc" alt="Tuna Chunk" />
      </div>
    </div>

    <!-- Level Up Announcement Banner -->
    <div v-if="showLevelUp && gameState === 'PLAYING'" class="level-up-overlay">
      <div class="level-up-banner">
        <h2 class="level-up-title">LEVEL UP!</h2>
        <div class="level-up-number">LEVEL {{ currentLevel }}</div>
      </div>
    </div>

    <!-- Score Popups Layer -->
    <div v-if="gameState === 'PLAYING'" class="popups-layer">
      <div 
        v-for="popup in scorePopups" 
        :key="popup.id" 
        class="floating-score"
        :class="{ golden: popup.isGolden, heart: popup.isHeart, hazard: popup.isHazard }"
        :style="{
          left: `${popup.x}px`,
          top: `${popup.y}px`
        }"
      >
        {{ popup.text }}
      </div>
    </div>

    <!-- Conveyor Belt bottom base -->
    <div class="conveyor-container">
      <img src="/conveyor.png" alt="Conveyor Belt" class="conveyor-belt" />
      
      <!-- Interactive Player Tuna Can -->
      <div v-if="processedTunaSrc && gameState === 'PLAYING'" class="cans-container">
        <div class="tuna-can-static">
          <img :src="processedTunaSrc" alt="Tuna Can" />
        </div>
      </div>
    </div>

    <!-- Screen Overlays -->
    <div v-if="isLoading" class="game-overlay">
      <div class="loading-box">
        <div class="spinner"></div>
        <p>공장 정밀 가동 중... 에셋 세척 중...</p>
      </div>
    </div>

    <div v-else-if="gameState === 'TITLE'" class="game-overlay">
      <div class="glass-menu">
        <h1 class="glow-title">참치를 담아라!</h1>
        <p class="tagline">참치 살코기 빗속에서 참치캔을 조작해 살코기를 골라 받아내세요!</p>
        
        <div class="highscore-badge">
          🏆 최 고 점 수 : <span class="highlight">{{ highScore }}</span>
        </div>

        <div class="how-to-play">
          <h3>🕹️ 조작 방식 및 규칙</h3>
          <ul>
            <li>⌨️ <b>좌우 방향키(←, →)</b>를 눌러 참치캔을 레일 위로 움직집니다.</li>
            <li>🐟 <b>하늘에서 비처럼 내리는 참치 살코기</b>를 참치캔 안으로 받으세요 (+10).</li>
            <li>✨ 희귀한 <b>황금 참치 살코기</b>는 엄청난 보너스를 제공합니다 (+50).</li>
            <li>☠️ <b>생선 가시</b>를 참치캔으로 받으면 <b>라이프(❤️)가 1 줄어드니</b> 피하세요!</li>
            <li>💔 일반 살코기를 바닥에 떨어뜨리면 <b>라이프(❤️)가 1 줄어듭니다</b> (총 10개).</li>
            <li>🌀 화면 끝으로 넘어가면 <b>반대편 화면 끝으로 이어지는 루프(Wrap)</b> 효과 적용!</li>
          </ul>
        </div>

        <button class="menu-btn" @click="startGame">게임 시작</button>
      </div>
    </div>

    <!-- Pause Screen Overlay -->
    <div v-else-if="gameState === 'PAUSED'" class="game-overlay">
      <div class="glass-menu pause-box animate-fadeInUp">
        <h1 class="glow-title pause">PAUSED</h1>
        <p class="tagline">일시정지 중입니다</p>
        
        <div class="pause-details">
          <div class="score-results">
            <div class="score-row">
              <span class="label">현재 점수</span>
              <span class="val pulse-glow-cyan">{{ score }}</span>
            </div>
            <div class="score-row">
              <span class="label">현재 레벨</span>
              <span class="val level-color">LEVEL {{ currentLevel }}</span>
            </div>
          </div>
          <p class="pause-desc">ESC 키를 다시 누르거나 아래 버튼을 클릭하여 계속 진행하세요.</p>
        </div>

        <button class="menu-btn resume" @click="resumeGame">이어하기</button>
      </div>
    </div>

    <div v-else-if="gameState === 'GAMEOVER'" class="game-overlay">
      <div class="glass-menu gameover-box">
        <h1 class="glow-title gameover">GAME OVER</h1>
        <p class="tagline">더 이상 살코기를 담을 칸이 없습니다!</p>

        <!-- New Rank Nickname Input Form -->
        <div v-if="showNameInput" class="name-input-box animate-fadeInUp">
          <h3>🎉 TOP 10 랭킹 진입! 🎉</h3>
          <p>닉네임을 입력해 전광판에 기록하세요 (최대 8자)</p>
          <div class="input-row">
            <input 
              v-model="playerName" 
              type="text" 
              maxlength="8" 
              placeholder="NICKNAME" 
              @keyup.enter="submitScore" 
              class="nickname-input"
            />
            <button class="menu-btn submit-btn" @click="submitScore">등록</button>
          </div>
        </div>

        <div class="score-results">
          <div class="score-row">
            <span class="label">최종 획득 점수</span>
            <span class="val pulse-glow">{{ score }}</span>
          </div>
          <div class="score-row">
            <span class="label">종료 시 레벨</span>
            <span class="val level-color">LEVEL {{ currentLevel }}</span>
          </div>
          <div class="score-row">
            <span class="label">개인 최고 기록</span>
            <span class="val">{{ highScore }}</span>
          </div>
        </div>

        <!-- Leaderboard display -->
        <div class="leaderboard-box">
          <h3>🏆 실시간 TOP 10 랭킹</h3>
          <div class="leaderboard-table">
            <div class="table-header">
              <span class="col-rank">순위</span>
              <span class="col-name">닉네임</span>
              <span class="col-level">레벨</span>
              <span class="col-score">점수</span>
            </div>
            <div class="table-body">
              <div 
                v-for="(entry, index) in leaderboard" 
                :key="index" 
                class="table-row"
                :class="{ 
                  highlight: index === newRankIndex, 
                  rank1: index === 0, 
                  rank2: index === 1, 
                  rank3: index === 2 
                }"
              >
                <span class="col-rank">
                  <template v-if="index === 0">🥇</template>
                  <template v-else-if="index === 1">🥈</template>
                  <template v-else-if="index === 2">🥉</template>
                  <template v-else>{{ index + 1 }}</template>
                </span>
                <span class="col-name">{{ entry.name }}</span>
                <span class="col-level">Lv.{{ entry.level }}</span>
                <span class="col-score">{{ entry.score }}</span>
              </div>
            </div>
          </div>
        </div>

        <button class="menu-btn retry" @click="startGame">다시 도전</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fullscreen-bg {
  position: relative;
  width: 100vw;
  height: 100vh;
  background-image: url('/factory-bg.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  overflow: hidden;
  font-family: 'Inter', 'Outfit', sans-serif;
  user-select: none;
}

/* Game HUD */
.game-hud {
  position: absolute;
  top: 25px;
  left: 0;
  width: 100%;
  padding: 0 40px;
  box-sizing: border-box;
  display: flex;
  justify-content: space-between;
  align-items: center;
  z-index: 100;
  pointer-events: none;
}

.score-container {
  display: flex;
  flex-direction: column;
  background: rgba(15, 23, 42, 0.65);
  border: 1px solid rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  padding: 8px 24px;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.score-container .label {
  font-size: 0.75rem;
  font-weight: 700;
  color: #a5f3fc;
  letter-spacing: 2px;
}

.score-container .value {
  font-size: 1.8rem;
  font-weight: 900;
  color: #fff;
  text-shadow: 0 0 10px rgba(6, 182, 212, 0.5);
}

.lives-container {
  display: flex;
  gap: 12px;
  background: rgba(15, 23, 42, 0.65);
  border: 1px solid rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  padding: 12px 20px;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.heart {
  font-size: 1.5rem;
  transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.heart.lost {
  opacity: 0.2;
  filter: grayscale(1);
  transform: scale(0.8);
}

/* Conveyor and Player Container */
.conveyor-container {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 120px;
  display: flex;
  align-items: flex-end;
  z-index: 10;
}

.conveyor-belt {
  width: 100%;
  height: 100%;
  object-fit: fill;
  filter: drop-shadow(0 -8px 12px rgba(0, 0, 0, 0.4));
}

.cans-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: visible;
  pointer-events: none;
}

.tuna-can-static {
  position: absolute;
  bottom: 38px;
  left: 50%;
  transform: translate(calc(-50% + v-bind(offsetXPx)), 0) rotate(v-bind(rotationDeg));
  height: 135px;
  transition: v-bind(transitionStyle);
  transform-origin: bottom center;
}

.tuna-can-static img {
  height: 100%;
  width: auto;
  object-fit: contain;
  filter: drop-shadow(4px 8px 8px rgba(0, 0, 0, 0.45));
}

/* Falling Items Layer */
.falling-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 5;
}

.falling-chunk {
  position: absolute;
  pointer-events: none;
}

.falling-chunk img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  filter: drop-shadow(3px 4px 4px rgba(0, 0, 0, 0.3));
}

.falling-chunk.golden img {
  filter: hue-rotate(52deg) brightness(1.6) saturate(1.8) drop-shadow(0 0 12px #eab308);
}

.falling-chunk.bone img {
  filter: drop-shadow(0 0 8px rgba(239, 68, 68, 0.6));
}

/* Score Popups Layer */
.popups-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 20;
}

.floating-score {
  position: absolute;
  font-size: 1.8rem;
  font-weight: 900;
  color: #22c55e;
  text-shadow: 0 0 8px rgba(34, 197, 94, 0.8);
  transform: translate(-50%, -100%);
  animation: floatUp 0.8s cubic-bezier(0.19, 1, 0.22, 1) forwards;
}

.floating-score.golden {
  color: #f59e0b;
  text-shadow: 0 0 12px rgba(245, 158, 11, 0.9);
  font-size: 2.2rem;
}

.floating-score.heart {
  color: #f43f5e;
  text-shadow: 0 0 10px rgba(244, 63, 94, 0.9);
  font-size: 1.8rem;
}

@keyframes floatUp {
  0% {
    opacity: 1;
    transform: translate(-50%, -30px) scale(0.8);
  }
  20% {
    opacity: 1;
    transform: translate(-50%, -60px) scale(1.2);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -120px) scale(1.0);
  }
}

/* Glassmorphic Overlays */
.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(8, 13, 28, 0.55);
  backdrop-filter: blur(12px);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 200;
}

.loading-box {
  text-align: center;
  color: #fff;
  font-weight: 600;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(6, 182, 212, 0.25);
  border-top: 4px solid #06b6d4;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 16px auto;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.glass-menu {
  background: rgba(15, 23, 42, 0.72);
  border: 1px solid rgba(255, 255, 255, 0.12);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.45);
  border-radius: 24px;
  padding: 40px;
  width: 90%;
  max-width: 500px;
  text-align: center;
  color: #fff;
  animation: fadeInUp 0.5s cubic-bezier(0.23, 1, 0.32, 1) forwards;
}

.glow-title {
  font-size: 3rem;
  font-weight: 900;
  letter-spacing: 4px;
  color: #fff;
  text-shadow: 0 0 12px #06b6d4, 0 0 25px #06b6d4, 0 0 40px #0284c7;
  margin-top: 0;
  margin-bottom: 8px;
}

.glow-title.gameover {
  text-shadow: 0 0 12px #ef4444, 0 0 25px #ef4444, 0 0 40px #b91c1c;
}

.tagline {
  color: #cbd5e1;
  font-size: 0.95rem;
  margin-bottom: 24px;
  line-height: 1.5;
}

.highscore-badge {
  display: inline-block;
  background: rgba(234, 179, 8, 0.15);
  border: 1px solid rgba(234, 179, 8, 0.3);
  color: #fef08a;
  padding: 6px 18px;
  border-radius: 20px;
  font-weight: 700;
  font-size: 0.95rem;
  margin-bottom: 24px;
  text-shadow: 0 0 5px rgba(234, 179, 8, 0.3);
}

.how-to-play {
  text-align: left;
  background: rgba(15, 23, 42, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 16px;
  padding: 20px 24px;
  margin-bottom: 30px;
}

.how-to-play h3 {
  margin-top: 0;
  font-size: 1.05rem;
  color: #67e8f9;
  border-bottom: 1px solid rgba(103, 232, 249, 0.15);
  padding-bottom: 8px;
}

.how-to-play ul {
  padding-left: 0;
  margin: 0;
  list-style: none;
}

.how-to-play li {
  font-size: 0.85rem;
  color: #e2e8f0;
  margin-bottom: 10px;
  line-height: 1.45;
}

.menu-btn {
  background: linear-gradient(135deg, #06b6d4 0%, #3b82f6 100%);
  border: none;
  border-radius: 14px;
  padding: 16px;
  width: 100%;
  color: #fff;
  font-size: 1.25rem;
  font-weight: 800;
  cursor: pointer;
  box-shadow: 0 6px 20px rgba(6, 182, 212, 0.35);
  transition: all 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.menu-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(6, 182, 212, 0.5);
  background: linear-gradient(135deg, #22d3ee 0%, #60a5fa 100%);
}

.menu-btn.retry {
  background: linear-gradient(135deg, #ef4444 0%, #dc2626 100%);
  box-shadow: 0 6px 20px rgba(239, 68, 68, 0.35);
}

.menu-btn.retry:hover {
  background: linear-gradient(135deg, #f87171 0%, #ef4444 100%);
  box-shadow: 0 8px 25px rgba(239, 68, 68, 0.5);
}

/* Gameover statistics */
.score-results {
  background: rgba(15, 23, 42, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 16px;
  padding: 20px 24px;
  margin-bottom: 30px;
}

.score-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.score-row:last-child {
  margin-bottom: 0;
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  padding-top: 12px;
}

.score-row .label {
  font-size: 0.9rem;
  color: #94a3b8;
}

.score-row .val {
  font-size: 1.6rem;
  font-weight: 900;
}

.pulse-glow {
  color: #ef4444;
  text-shadow: 0 0 10px rgba(239, 68, 68, 0.5);
  animation: pulse 1s infinite alternate;
}

.pulse-glow-cyan {
  color: #67e8f9;
  text-shadow: 0 0 10px rgba(103, 232, 249, 0.6);
  animation: pulse 1s infinite alternate;
}

.pause-desc {
  font-size: 0.85rem;
  color: #94a3b8;
  margin-top: 15px;
  text-align: center;
  line-height: 1.4;
}

.pause-box {
  width: 380px !important;
  max-width: 90vw;
}

.glow-title.pause {
  text-shadow: 0 0 10px #06b6d4, 0 0 20px #06b6d4, 0 0 35px #0891b2;
}

.level-color {
  color: #67e8f9;
  text-shadow: 0 0 10px rgba(103, 232, 249, 0.6);
}

.floating-score.hazard {
  color: #ef4444;
  text-shadow: 0 0 10px rgba(239, 68, 68, 0.9), 0 0 20px rgba(239, 68, 68, 0.4);
  font-size: 2.2rem;
}

/* Leaderboard & Name input styling */
.gameover-box {
  width: 480px !important;
  max-width: 90vw;
}

.leaderboard-box {
  margin: 15px 0;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 12px;
  padding: 12px;
}

.leaderboard-box h3 {
  margin: 0 0 10px 0;
  font-size: 1.1rem;
  color: #67e8f9;
  text-shadow: 0 0 8px rgba(103, 232, 249, 0.4);
  text-align: center;
}

.leaderboard-table {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.table-header {
  display: flex;
  font-size: 0.8rem;
  color: #94a3b8;
  padding: 4px 8px;
  font-weight: 700;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.table-body {
  max-height: 180px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 3px;
  padding-right: 4px;
}

/* Custom Scrollbar for scoreboard list */
.table-body::-webkit-scrollbar {
  width: 4px;
}
.table-body::-webkit-scrollbar-track {
  background: transparent;
}
.table-body::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 2px;
}

.table-row {
  display: flex;
  align-items: center;
  font-size: 0.9rem;
  padding: 5px 8px;
  background: rgba(255, 255, 255, 0.01);
  border-radius: 6px;
  border: 1px solid transparent;
  transition: all 0.3s ease;
}

.table-row.rank1 {
  background: rgba(234, 179, 8, 0.08);
  border-color: rgba(234, 179, 8, 0.2);
  font-weight: 800;
  color: #fef08a;
}

.table-row.rank2 {
  background: rgba(226, 232, 240, 0.06);
  border-color: rgba(226, 232, 240, 0.15);
  font-weight: 700;
  color: #e2e8f0;
}

.table-row.rank3 {
  background: rgba(180, 83, 9, 0.06);
  border-color: rgba(180, 83, 9, 0.15);
  font-weight: 700;
  color: #ffedd5;
}

.table-row.highlight {
  animation: scoreRowHighlight 1s infinite alternate;
  border-color: #67e8f9;
}

@keyframes scoreRowHighlight {
  0% { background: rgba(103, 232, 249, 0.05); }
  100% { background: rgba(103, 232, 249, 0.25); }
}

.col-rank { width: 15%; text-align: center; }
.col-name { width: 35%; text-align: left; }
.col-level { width: 25%; text-align: center; color: #94a3b8; }
.col-score { width: 25%; text-align: right; font-weight: 900; }

.table-row.rank1 .col-score { color: #f59e0b; text-shadow: 0 0 8px rgba(245, 158, 11, 0.5); }

/* Name input styling */
.name-input-box {
  background: rgba(103, 232, 249, 0.05);
  border: 1px solid rgba(103, 232, 249, 0.2);
  border-radius: 12px;
  padding: 12px;
  margin-bottom: 12px;
  text-align: center;
  animation: pulseInput 1.5s infinite alternate;
}

@keyframes pulseInput {
  0% { border-color: rgba(103, 232, 249, 0.2); }
  100% { border-color: rgba(103, 232, 249, 0.5); }
}

.name-input-box h3 {
  margin: 0 0 4px 0;
  font-size: 1.1rem;
  color: #67e8f9;
}

.name-input-box p {
  margin: 0 0 10px 0;
  font-size: 0.8rem;
  color: #e2e8f0;
}

.input-row {
  display: flex;
  gap: 8px;
  justify-content: center;
}

.nickname-input {
  background: rgba(0, 0, 0, 0.5);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: #fff;
  padding: 8px 12px;
  font-size: 1rem;
  font-weight: 900;
  text-transform: uppercase;
  text-align: center;
  width: 140px;
  letter-spacing: 2px;
  outline: none;
  transition: all 0.3s ease;
}

.nickname-input:focus {
  border-color: #67e8f9;
  box-shadow: 0 0 8px rgba(103, 232, 249, 0.4);
}

.submit-btn {
  padding: 8px 16px !important;
  font-size: 0.9rem !important;
  margin: 0 !important;
}

.animate-fadeInUp {
  animation: fadeInUp 0.4s ease-out forwards;
}

@keyframes pulse {
  0% { transform: scale(1); }
  100% { transform: scale(1.05); }
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
/* Level Up Banner overlay and styles */
.level-up-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 150;
}

.level-up-banner {
  position: absolute;
  top: 42%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
  pointer-events: none;
  animation: bannerEnter 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards,
             bannerExit 0.5s ease-in 1.5s forwards;
}

.level-up-title {
  font-size: 2.2rem;
  font-weight: 900;
  color: #67e8f9;
  text-shadow: 0 0 10px #06b6d4, 0 0 20px #06b6d4, 0 0 35px #0891b2;
  margin: 0;
  letter-spacing: 4px;
}

.level-up-number {
  font-size: 4rem;
  font-weight: 950;
  color: #fff;
  text-shadow: 0 0 15px rgba(255, 255, 255, 0.8), 0 0 30px #3b82f6, 0 0 45px #1d4ed8;
  margin-top: 5px;
  letter-spacing: 2px;
}

@keyframes bannerEnter {
  0% {
    opacity: 0;
    transform: translate(-50%, -30%) scale(0.6);
  }
  100% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}

@keyframes bannerExit {
  0% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -70%) scale(0.85);
  }
}
</style>
