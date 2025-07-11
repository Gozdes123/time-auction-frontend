<template>
  <div class="fixed-height-status-area" v-if="gamePhase === 'inGame' || gamePhase === 'roomLobby'">
    <span :class="{ 'countdown-message': preRoundCountdown > 0 }" class="status-text-content">
      <span v-if="preRoundCountdown > 0">準備倒數：<span class="highlight">{{ preRoundCountdown }}</span> 秒</span>
      <span v-else-if="roundInProgress">競標時間：<span class="time-display">{{ roundElapsedTimeFormatted }}</span></span>
      <span v-else class="initial-status-message">等待所有玩家按住按鈕...</span>
    </span>
  </div>

  <div class="game-container">
    <h1>時間拍賣</h1>

    <div v-if="gamePhase === 'startScreen'" class="start-screen">
      <label for="playerName">你的名字：</label>
      <input type="text" id="playerName" v-model="playerName" placeholder="輸入你的名字" @keyup.enter="handleCreateRoom" />
      <button @click="handleCreateRoom" :disabled="!playerName.trim()">創建房間</button>
      <p class="separator">- 或 -</p>
      <input type="text" v-model="roomIdInput" placeholder="輸入房間ID" @keyup.enter="handleJoinRoom" />
      <button @click="handleJoinRoom" :disabled="!playerName.trim() || !roomIdInput.trim()">加入房間</button>
      <p v-if="message" class="error-message">{{ message }}</p>
    </div>

    <div v-else-if="gamePhase === 'roomLobby' || gamePhase === 'inGame'" class="game-board">
      <p class="message">{{ message }}</p>

      <div class="header-info">
        <h2>房間ID: <span class="highlight">{{ roomId }}</span></h2>
        <p>當前回合：<span class="highlight">{{ currentRound }}</span> / {{ maxRounds }}</p>
      </div>

      <div class="players-status-section">
        <h3>玩家列表:</h3>
        <ul>
          <li v-for="p in players" :key="p.id">
            <span>{{ p.name }} <span v-if="p.id === player.id"> (你) </span></span>
            <span :class="{ 'status-holding': p.isHoldingButton, 'status-not-holding': !p.isHoldingButton }">
              {{ p.isHoldingButton ? '按住中' : '未按住' }}
            </span>
            <span v-if="!p.isHoldingButton && p.id !== player.id && preRoundCountdown === 0 && !roundInProgress"
              class="prompt-message">
              請 {{ p.name }} 按住按鈕
            </span>
          </li>
        </ul>
      </div>

      <div class="player-info-section">
        <h2>你的狀態</h2>
        <p>玩家名稱：<span class="highlight">{{ player.name }}</span></p>
        <p>你的代幣：<span class="highlight">{{ player.tokens }}</span></p>
      </div>

      <div class="bid-section-button-only">
        <button @mousedown="handleButtonDown" @mouseup="handleButtonUp" @mouseleave="handleButtonUp"
          @touchstart.prevent="handleButtonDown" @touchend.prevent="handleButtonUp"
          :disabled="!player || player.time <= 0 || gamePhase === 'gameOver'" class="hold-button">
          {{ preRoundCountdown > 0 ? "倒數中..." : isHoldingButton ? "堅持中..." : "按住準備競標" }}
        </button>
      </div>
      <button @click="leaveRoom" class="leave-button">離開房間</button>
    </div>

    <div v-else-if="gamePhase === 'gameOver'" class="game-over-screen">
      <h2>遊戲結束！</h2>
      <p class="final-message">{{ message }}</p>
      <h3>最終分數：</h3>
      <ul>
        <li v-for="p in players" :key="p.id">
          {{ p.name }}: {{ p.tokens }} 代幣, {{ formatTime(p.time) }} 時間
        </li>
      </ul>
      <button @click="resetGame">重新開始</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted } from "vue";
import { io } from 'socket.io-client';

// --- Socket.IO 連接 ---
const socket = io('http://localhost:3000');

socket.on('connect', () => {
  console.log('已連接到伺服器！Socket ID:', socket.id);
});

socket.on('disconnect', () => {
  console.log('已斷開與伺服器的連接。');
  gamePhase.value = 'startScreen';
  message.value = '與伺服器斷開連接！請重新整理或嘗試重新加入。';
  clearInterval(preRoundInterval);
  clearInterval(myHoldTimerInterval);
  clearInterval(roundTimerInterval);
  virtualPlayerReleaseTimeouts.forEach(timeout => clearTimeout(timeout));
});

// --- 遊戲狀態變數 ---
const gamePhase = ref('startScreen'); // 'startScreen', 'roomLobby', 'inGame', 'gameOver'
const playerName = ref("");
const roomIdInput = ref("");
const roomId = ref("");

const player = ref(null); // 當前玩家自己
const players = ref([]); // 房間內所有玩家

const currentRound = ref(0);
const maxRounds = 19;

const message = ref("");

// --- 競標相關狀態 (部分仍由前端維護，部分將由後端同步) ---
const preRoundCountdown = ref(0);
let preRoundInterval = null; // 本地計時器，顯示倒數

const isHoldingButton = ref(false); // 你個人是否正在按住按鈕 (本地UI狀態)
let myHoldTimerInterval = null;

const roundElapsedTime = ref(0);
let roundTimerInterval = null;

// 虛擬玩家邏輯已移除，但為防止參考錯誤，保留聲明
const virtualPlayers = ref([]);
let virtualPlayerReleaseTimeouts = [];

const roundInProgress = ref(false);

// --- 計算屬性 ---
const formatTime = (seconds) => {
  if (seconds < 0) seconds = 0;
  const minutes = Math.floor(seconds / 60);
  const remainingSeconds = seconds % 60;
  const formattedMinutes = String(minutes).padStart(2, "0");
  const formattedSeconds = String(remainingSeconds).padStart(2, "0");
  return `${formattedMinutes}:${formattedSeconds}`;
};

const roundElapsedTimeFormatted = computed(() => {
  return formatTime(roundElapsedTime.value);
});

// --- 生命週期鉤子：組件銷毀時清理定時器 ---
onUnmounted(() => {
  clearInterval(preRoundInterval);
  clearInterval(myHoldTimerInterval);
  clearInterval(roundTimerInterval);
  virtualPlayerReleaseTimeouts.forEach((timeout) => clearTimeout(timeout));
  if (socket) {
    socket.disconnect();
  }
});

// --- 房間管理與遊戲啟動 ---
function handleCreateRoom() {
  if (!playerName.value.trim()) { message.value = "請輸入你的名字！"; return; }
  socket.emit('createRoom', playerName.value.trim(), (response) => {
    if (response.success) {
      roomId.value = response.roomId;
      player.value = response.player;
      gamePhase.value = 'roomLobby'; // 進入房間大廳
      message.value = `房間 ${response.roomId} 已創建！等待其他玩家加入。`;
    } else {
      message.value = response.message || '創建房間失敗！';
    }
  });
}

function handleJoinRoom() {
  if (!playerName.value.trim() || !roomIdInput.value.trim()) { message.value = "請輸入你的名字和房間ID！"; return; }
  socket.emit('joinRoom', roomIdInput.value.trim(), playerName.value.trim(), (response) => {
    if (response.success) {
      roomId.value = response.roomId;
      player.value = response.player;
      gamePhase.value = 'roomLobby'; // 進入房間大廳
      message.value = `已加入房間 ${response.roomId}！`;
    } else {
      message.value = response.message || '加入房間失敗！';
    }
  });
}

function leaveRoom() {
  if (roomId.value) {
    socket.emit('leaveRoom'); // 後端會從 socket.roomId 獲取房間ID
    resetGameStates(); // 重置所有遊戲相關狀態
    gamePhase.value = 'startScreen'; // 回到開始畫面
    message.value = '已離開房間。';
  }
}

// --- 玩家按鈕操作 ---
function handleButtonDown() {
  if (!player.value || !roomId.value) return;
  if (!isHoldingButton.value) { // 避免重複發送
    isHoldingButton.value = true;
    socket.emit('playerHolding'); // 通知後端我按住了
  }
}

function handleButtonUp() {
  if (!player.value || !roomId.value) return;
  if (isHoldingButton.value) { // 避免重複發送
    isHoldingButton.value = false;
    socket.emit('playerReleased'); // 通知後端我放開了
  }
}

// --- Socket.IO 事件監聽 (接收後端廣播) ---

// 接收所有玩家狀態更新 (包括 isHoldingButton)
socket.on('playerStatusUpdate', (updatedPlayers) => {
  players.value = updatedPlayers;
  // 更新我自己的 player 物件，確保本地狀態與後端同步
  if (player.value) {
    const me = updatedPlayers.find(p => p.id === socket.id);
    if (me) {
      player.value = me;
      // isHoldingButton.value = me.isHoldingButton; // 不直接同步，讓本地按鈕狀態由 mouseup/down 控制
    }
  }
});

socket.on('preRoundCountdownUpdate', (count) => {
  preRoundCountdown.value = count;
  message.value = `所有玩家已按住！倒數：${count} 秒`;
  gamePhase.value = 'inGame'; // 進入遊戲階段，顯示倒數
  roundInProgress.value = false;
  roundElapsedTime.value = 0; // 倒數時競標時間清零
});

socket.on('roundStarting', () => {
  message.value = '競標開始！堅持住！';
  roundInProgress.value = true;
  preRoundCountdown.value = 0;
  // 本地碼表將由 roundTimerUpdate 驅動
});

socket.on('roundTimerUpdate', (elapsedTime) => {
  roundElapsedTime.value = elapsedTime;
});

socket.on('roundEnded', (data) => {
  message.value = data.message;
  roundInProgress.value = false;
  players.value = data.updatedPlayers;
  if (player.value) {
    const me = data.updatedPlayers.find(p => p.id === socket.id);
    if (me) player.value = me;
  }
  // 後端會控制下一回合的啟動
});

socket.on('gameOver', (data) => {
  gamePhase.value = 'gameOver';
  message.value = data.reason;
  players.value = data.finalPlayers;
});

socket.on('errorMessage', (msg) => {
  message.value = msg;
});

// --- 遊戲狀態重置 ---
function resetGameStates() { // 獨立的內部重置函數
  playerName.value = '';
  roomIdInput.value = '';
  roomId.value = '';
  player.value = null;
  players.value = [];
  currentRound.value = 0;
  message.value = '';
  preRoundCountdown.value = 0;
  isHoldingButton.value = false;
  roundElapsedTime.value = 0;
  roundInProgress.value = false;
  clearInterval(preRoundInterval);
  clearInterval(myHoldTimerInterval);
  clearInterval(roundTimerInterval);
  virtualPlayerReleaseTimeouts.forEach(timeout => clearTimeout(timeout));
}

function resetGame() { // 暴露給外部的重置按鈕
  resetGameStates();
  gamePhase.value = 'startScreen';
}
</script>

<style scoped>
/* 調整 game-container 寬度使其更大 */
.game-container {
  font-family: "Arial", sans-serif;
  max-width: 700px;
  /* 從 600px 增加到 700px */
  margin: 20px auto;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  background-color: #f9f9f9;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 15px;
}

h1 {
  color: #333;
  margin-bottom: 15px;
  font-size: 2em;
  /* 稍微放大標題 */
}

/* 調整 start-screen 寬度 */
.start-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  /* 增加間距 */
  padding: 25px;
  /* 增加內邊距 */
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  max-width: 500px;
  /* 增加最大寬度 */
  margin: 0 auto;
  /* 居中 */
}

.start-screen label {
  font-weight: bold;
  font-size: 1.1em;
  margin-bottom: 8px;
}

.start-screen input {
  padding: 10px;
  /* 增加輸入框大小 */
  border-radius: 5px;
  border: 1px solid #ddd;
  width: 90%;
  /* 佔用更多寬度 */
  max-width: 300px;
  /* 限制最大寬度 */
  text-align: center;
  font-size: 1em;
}

.start-screen button {
  padding: 12px 25px;
  /* 增加按鈕大小 */
  font-size: 1.1em;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.start-screen button:hover:not(:disabled) {
  background-color: #45a049;
}

.start-screen button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.start-screen .separator {
  margin: 15px 0;
  /* 增加間距 */
  font-weight: bold;
  color: #555;
  font-size: 1.1em;
}

.error-message {
  color: red;
  margin-top: 15px;
  /* 增加間距 */
  font-weight: bold;
  font-size: 1.1em;
}

/* 遊戲板 (包含房間大廳和遊戲中) */
.game-board {
  display: flex;
  flex-direction: column;
  gap: 15px;
  background-color: #fff;
  padding: 20px;
  /* 增加內邊距 */
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.message {
  font-weight: bold;
  color: #e44d26;
  margin-bottom: 10px;
  font-size: 1.2em;
  /* 稍微放大 */
}

/* 頂部倒數/碼表區 (獨立於 game-container) */
.fixed-height-status-area {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 15px;
  margin: 10px auto 0;
  max-width: 700px;
  /* 與新的 game-container 寬度一致 */
  text-align: center;
  height: 80px;
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* 倒數和碼表的統一內容顯示容器 */
.status-text-content {
  font-size: 2.2em;
  /* 調整字體大小 */
  font-weight: bold;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: baseline;
  background-color: #e0f2f7;
  padding: 10px 20px;
  /* 調整內邊距 */
  border-radius: 6px;
  border: 1px solid #b3e5fc;
  width: 400px;
  /* 調整寬度以容納更長的文本，並確保穩定 */
  margin: auto;
}

.status-text-content .highlight {
  font-size: 1.2em;
}

.countdown-message {
  color: #f44336;
}

.initial-status-message {
  font-size: 0.9em;
  /* 調整字體大小 */
  color: #6c757d;
}

/* 玩家資訊區塊 (頂部房間ID和回合) */
.header-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  /* 垂直居中 */
  background-color: #f0f0f0;
  padding: 12px 20px;
  /* 增加內邊距 */
  border-radius: 8px;
  font-size: 1em;
  /* 稍微放大 */
}

.header-info h2 {
  margin: 0;
  border-bottom: none;
  padding-bottom: 0;
  font-size: 1.3em;
  /* 稍微放大 */
}

.header-info p {
  margin: 0;
  font-weight: bold;
  font-size: 1.1em;
  /* 稍微放大 */
}


/* 玩家列表區塊 */
.players-status-section {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 20px;
  /* 增加內邊距 */
  text-align: left;
}

.players-status-section h3 {
  color: #555;
  margin-top: 0;
  margin-bottom: 12px;
  /* 增加間距 */
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  font-size: 1.2em;
  /* 稍微放大 */
}

.players-status-section ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.players-status-section li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  /* 增加列表項間距 */
  border-bottom: 1px dashed #eee;
  font-size: 1em;
  /* 稍微放大 */
}

.players-status-section li:last-child {
  border-bottom: none;
}

.status-holding {
  color: #28a745;
  font-weight: bold;
}

.status-not-holding {
  color: #6c757d;
}

.prompt-message {
  color: #ffc107;
  font-weight: bold;
  margin-left: 10px;
  font-size: 0.9em;
  /* 調整字體大小 */
}


/* 你的狀態區塊 */
.player-info-section {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 20px;
  /* 增加內邊距 */
  text-align: left;
}

.player-info-section h2 {
  color: #555;
  margin-top: 0;
  margin-bottom: 12px;
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  font-size: 1.2em;
}

.player-info-section p {
  margin: 8px 0;
  /* 增加段落間距 */
  font-size: 1em;
  /* 稍微放大 */
}


/* 按鈕區塊 */
.bid-section-button-only {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 20px;
  /* 增加內邊距 */
  text-align: center;
}

.hold-button {
  width: 250px;
  /* 進一步放大按鈕 */
  height: 90px;
  /* 進一步放大按鈕 */
  font-size: 1.7em;
  /* 進一步放大字體 */
  font-weight: bold;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 12px;
  /* 稍微圓角 */
  cursor: pointer;
  box-shadow: 0 5px 10px rgba(0, 0, 0, 0.25);
  /* 稍微增加陰影 */
  transition: background-color 0.1s ease-out, transform 0.1s ease-out;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.hold-button:active {
  background-color: #1976d2;
  transform: translateY(3px);
  /* 按下時位移多一點 */
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.25);
}

.hold-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
  box-shadow: none;
}

.leave-button {
  background-color: #dc3545;
  color: white;
  padding: 10px 20px;
  /* 稍微放大 */
  border-radius: 5px;
  border: none;
  cursor: pointer;
  font-size: 1em;
  /* 稍微放大 */
  margin-top: 15px;
  /* 增加間距 */
}

/* 遊戲結束畫面 */
.game-over-screen {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 10px;
  padding: 25px;
  /* 增加內邊距 */
  margin-top: 20px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.game-over-screen h2 {
  color: #dc3545;
  margin-bottom: 15px;
  font-size: 1.8em;
}

.game-over-screen h3 {
  color: #555;
  margin-top: 20px;
  margin-bottom: 12px;
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  font-size: 1.2em;
}

.game-over-screen ul {
  list-style: none;
  padding: 0;
  margin-bottom: 20px;
}

.game-over-screen li {
  padding: 8px 0;
  font-size: 1em;
}

.game-over-screen button {
  padding: 12px 25px;
  font-size: 1.1em;
}
</style>