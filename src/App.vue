<template>
  <div class="fixed-height-status-area" v-if="gamePhase === 'inGame' || gamePhase === 'roomLobby'">
    <span :class="{ 'countdown-message': preRoundCountdown > 0 }" class="status-text-content">
      <span v-if="preRoundCountdown > 0">準備倒數：<span>{{ preRoundCountdown }}</span> 秒</span>
      <span v-else-if="roundInProgress" class="status-message">競標時間：<span class="time-display">{{
        roundElapsedTimeFormatted }}</span></span>
      <span v-else class="status-message">{{ messageText }}</span>
    </span>
    <transition name="fade">
      <div v-if="showWinnerDisplay" class="winner-announcement-overlay">
        <div class="winner-announcement-box">
          {{ winnerMessage }}
        </div>
      </div>
    </transition>
  </div>

  <div class="game-container">
    <h1>時間拍賣</h1>

    <div v-if="gamePhase === 'startScreen'" class="start-screen">
      <label for="playerName">你的名字：</label>
      <input type="text" id="playerName" v-model="playerName" placeholder="輸入你的名字" @keyup.enter="createRoom" />

      <label for="initialTime">起始時間 (秒):</label>
      <input type="number" id="initialTime" v-model.number="initialPlayerTime" placeholder="例如: 60" min="10"
        max="600" />
      <p class="input-hint">建議 10-600 秒</p>

      <label for="maxRounds">遊戲回合數:</label>
      <input type="number" id="maxRounds" v-model.number="customMaxRounds" placeholder="例如: 5 或 10" min="1" max="50" />
      <p class="input-hint">建議 1-50 回合</p>

      <button @click="createRoom" :disabled="!playerName.trim()">創建房間</button>

      <p class="separator">- 或 -</p>
      <input type="text" v-model="joinRoomId" placeholder="輸入房間ID" @keyup.enter="joinRoom" />
      <button @click="joinRoom" :disabled="!joinRoomId.trim() || !playerName.trim()">加入房間</button>
      <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
    </div>

    <div v-else-if="gamePhase === 'roomLobby' || gamePhase === 'inGame'" class="game-board">
      <p class="message">{{ messageText }}</p>
      <div class="top-info-row">
        <div class="header-info">
          <h2>房間ID: <span class="highlight">{{ roomId }}</span></h2>
        </div>
        <div class="header-info">
          <p>當前回合：<span class="highlight">{{ currentRound }}</span> / {{ maxRounds }}</p>
        </div>
      </div>

      <div class="main-game-content">
        <div class="players-status-section">
          <h3>玩家列表:</h3>
          <ul>
            <li v-for="player in players" :key="player.id">
              <span>{{ player.name }} <span v-if="myPlayer && player.id === myPlayer.id"> (你) </span></span>
              <span v-if="preRoundCountdown > 0" class="status-generic">
                準備中... </span>
              <span v-else-if="gamePhase === 'roomLobby'"
                :class="{ 'status-holding': player.isHoldingButton, 'status-not-holding': !player.isHoldingButton }">
                {{ player.isHoldingButton ? '按住中' : '未按住' }}
              </span>
              <span v-else-if="gamePhase === 'inGame'" class="status-generic">
                競標中
              </span>
              <span
                v-if="myPlayer && !player.isHoldingButton && player.id !== myPlayer.id && preRoundCountdown === 0 && !roundInProgress"
                class="prompt-message">
                請 {{ player.name }} 按住按鈕
              </span>
            </li>
          </ul>
        </div>

        <div class="player-info-section">
          <h2>你的狀態</h2>
          <div v-if="myPlayer" class="player-details">
            <p class="detail-item">
              <i class="icon fas fa-user"></i> 玩家名稱：<span class="highlight player-name">{{ myPlayer.name }}</span>
            </p>
            <p class="detail-item">
              <i class="icon fas fa-coins"></i> 你的代幣：<span class="highlight player-tokens">{{ myPlayer.tokens }}</span>
            </p>
          </div>
          <p v-else class="loading-message">載入玩家資訊...</p>
        </div>
      </div>

      <div class="bid-section-button-only">
        <button @mousedown="holdButton" @mouseup="releaseButton" @mouseleave="releaseButton"
          @touchstart.prevent="holdButton" @touchend.prevent="releaseButton"
          :disabled="!myPlayer || myPlayer.time <= 0 || gamePhase === 'gameOver'" class="hold-button">
          {{ preRoundCountdown > 0 ? "倒數中..." : myPlayer && myPlayer.isHoldingButton ? "競標中" : "按住準備競標" }}
        </button>
      </div>
      <button @click="leaveRoom" class="leave-button">離開房間</button>
    </div>

    <div v-else-if="gamePhase === 'gameOver'" class="game-over-screen">
      <h2>遊戲結束！</h2>
      <p class="final-message">{{ gameOverReason }}</p>

      <div v-if="finalGameWinner" class="final-winner-display">
        <h3 v-if="finalGameWinner.isTie">🏆 並列優勝者 🏆</h3>
        <h3 v-else>👑 最終優勝者 👑</h3>
        <p v-if="finalGameWinner.isTie" class="winner-name-large">{{ finalGameWinner.names }}</p>
        <p v-else class="winner-name-large">{{ finalGameWinner.name }}</p>
        <p v-if="finalGameWinner.isTie" class="winner-stats-large">
          代幣: {{ finalGameWinner.players[0].tokens }} 個，時間: {{ formatTime(finalGameWinner.players[0].time) }}
        </p>
        <p v-else class="winner-stats-large">
          獲得 {{ finalGameWinner.tokens }} 個代幣，剩餘時間 {{ formatTime(finalGameWinner.time) }}
        </p>
      </div>
      <div v-else class="final-winner-display">
        <h3>本場遊戲沒有明確的最終優勝者</h3>
        <p>（所有玩家代幣和時間均為零或沒有參與）</p>
      </div>

      <h3>所有玩家最終排名:</h3>
      <ul class="final-scores-list">
        <li v-for="(player, index) in players" :key="player.id"
          :class="{ 'is-final-winner': finalGameWinner && !finalGameWinner.isTie && player.id === finalGameWinner.id, 'is-cowinner': finalGameWinner && finalGameWinner.isTie && finalGameWinner.players.some(p => p.id === player.id) }">
          <span class="rank">{{ index + 1 }}.</span>
          <span class="player-name-final">{{ player.name }}</span>
          <span class="player-stats-final">代幣: {{ player.tokens }}, 時間: {{ formatTime(player.time) }}</span>
        </li>
      </ul>
      <button @click="restartGame">重新開始</button>
    </div>
  </div>

  <transition name="modal-fade">
    <div class="modal-overlay" v-if="showRulesModal">
      <div class="modal-content">
        <h2>時間拍賣遊戲規則</h2>
        <p>歡迎來到時間拍賣！這是一個獨特的競標遊戲，您的「時間」就是代幣。</p>
        <p>
          遊戲目標：在規定回合數內，獲得最多「代幣」的玩家獲勝！
        </p>
        <hr>
        <h3>基本玩法：</h3>
        <ul>
          <li>每回合開始前，所有玩家需同時「按住按鈕」進行準備。</li>
          <li>倒數結束後，競標開始，按住按鈕的玩家將開始消耗自己的「時間」。</li>
          <li>「時間」就是您的生命值和競標籌碼。時間歸零的玩家將被淘汰。</li>
          <li>您可以在任何時候放開按鈕，當您放手時，您本回合的按住時間將被記錄。</li>
          <li>當所有參與競標的玩家都放手後，本回合結束。</li>
        </ul>
        <h3>回合勝利條件：</h3>
        <ul>
          <li>本回合按住時間最長的玩家獲勝。</li>
          <li>獲勝者將獲得 **1 個遊戲代幣**，並扣除其本回合按住時間。</li>
          <li>如果多名玩家按住時間相同，則為並列獲勝，所有並列獲勝者都獲得代幣並扣除時間。</li>
          <li>如果本回合無人按住或無人堅持足夠時間，則無人獲勝，無人扣除時間，回合數照常推進。</li>
        </ul>
        <h3>遊戲結束與最終勝利：</h3>
        <ul>
          <li>當有玩家「時間」歸零，或遊戲達到設定的「總回合數」時，遊戲結束。</li>
          <li>遊戲結束時，**擁有最多「遊戲代幣」的玩家獲勝。**</li>
          <li>若代幣數量相同，則「剩餘時間」較多的玩家獲勝。</li>
          <li>代幣和時間都相同，則為並列優勝。</li>
        </ul>
        <button @click="closeRulesModal" class="modal-button">我明白了！</button>
      </div>
    </div>
  </transition>

  <transition name="modal-fade">
    <div class="modal-overlay" v-if="showRoundStatsModal">
      <div class="modal-content">
        <h2>第 {{ currentRound }} 回合統計</h2>
        <ul class="stats-list">
          <li v-for="player in players" :key="player.id">
            <span>{{ player.name }}</span>
            <span>代幣: {{ player.tokens }}</span>
          </li>
        </ul>
        <button @click="closeRoundStatsModal" class="modal-button">繼續遊戲</button>
      </div>
    </div>
  </transition>
</template>

<script>
import { io } from 'socket.io-client';

export default {
  data() {
    return {
      socket: null,
      playerName: '',
      roomId: '',
      joinRoomId: '',
      gamePhase: 'startScreen', // 'startScreen', 'roomLobby', 'inGame', 'gameOver'
      players: [],
      myPlayer: null, // 儲存當前玩家的詳細資訊
      messageText: '等待所有玩家加入並按住按鈕...', // 頂部主要訊息顯示
      errorMessage: '', // 錯誤訊息顯示

      preRoundCountdown: 0, // 賽前倒數計時
      roundInProgress: false, // 競標是否進行中 (碼表是否在跑)
      roundElapsedTime: 0, // 競標已進行的時間
      currentRound: 0, // 當前回合數
      maxRounds: 19, // 遊戲總回合數 (會被自定義值覆蓋)

      // 新增：自定義起始時間和回合數
      initialPlayerTime: 60, // 預設起始時間 (秒)
      customMaxRounds: 5,   // 預設遊戲回合數

      winnerMessage: '', // 單回合獲勝者公告內容
      showWinnerDisplay: false, // 控制單回合獲勝者公告顯示
      winnerDisplayTimeout: null, // 清除單回合獲勝者公告計時器

      gameOverReason: '', // 遊戲結束原因
      finalGameWinner: null, // 最終遊戲優勝者資訊 (可以是單一對象或並列贏家對象)

      // --- 新增 Modal 相關數據 ---
      showRulesModal: false, // 控制規則 Modal 顯示
      showRoundStatsModal: false, // 控制回合統計 Modal 顯示
    };
  },
  computed: {
    // 格式化競標時間顯示
    roundElapsedTimeFormatted() {
      const minutes = Math.floor(this.roundElapsedTime / 60);
      const seconds = this.roundElapsedTime % 60;
      return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
    },
  },
  methods: {
    // 連接 WebSocket
    connectSocket() {
      // 嘗試從環境變數獲取後端 URL，否則使用預設本地地址
      this.socket = io(import.meta.env.VITE_BACKEND_URL || 'http://localhost:3000');

      // 監聽連接成功事件
      this.socket.on('connect', () => {
        console.log('Connected to server');
        this.errorMessage = ''; // 清除任何連接錯誤訊息
      });

      // 監聽斷開連接事件
      this.socket.on('disconnect', () => {
        console.log('Disconnected from server');
        this.errorMessage = '與伺服器斷開連接。';
        // 當斷開連接時，重置所有遊戲數據並返回開始畫面
        this.resetGameData();
        this.gamePhase = 'startScreen';
      });

      // 監聽玩家狀態更新事件
      this.socket.on('playerStatusUpdate', (data) => {
        this.players = data.players; // 更新玩家列表
        this.currentRound = data.currentRound; // 更新當前回合數
        this.maxRounds = data.maxRounds; // 更新總回合數
        // 根據遊戲狀態切換前端顯示階段
        if (data.gameStatus === 'inRound' || data.gameStatus === 'preCountdown') {
          this.gamePhase = 'inGame';
        } else if (data.gameStatus === 'waiting' || data.gameStatus === 'roundEnded') {
          this.gamePhase = 'roomLobby';
        } else if (data.gameStatus === 'gameOver') {
          this.gamePhase = 'gameOver';
        }
        // 更新當前玩家資訊
        this.myPlayer = this.players.find(p => p.id === this.socket.id) || null;
        console.log('Player status updated:', this.players);
      });

      // 監聽通用訊息事件
      this.socket.on('message', (message) => {
        this.messageText = message; // 顯示伺服器發送的訊息
        console.log('Server message:', message);
      });

      // 監聽賽前倒數更新事件
      this.socket.on('preRoundCountdownUpdate', (countdown) => {
        this.preRoundCountdown = countdown; // 更新倒數計時
        this.roundInProgress = false; // 倒數期間競標尚未開始
        console.log('Pre-round countdown:', countdown);
      });

      // 監聽回合開始事件
      this.socket.on('roundStarting', () => {
        this.roundInProgress = true; // 競標開始
        this.preRoundCountdown = 0; // 清除倒數
        this.roundElapsedTime = 0; // 重置競標計時器
        this.messageText = '競標開始....';
        console.log('Round starting!');
      });

      // 監聽競標時間更新事件
      this.socket.on('roundTimerUpdate', (elapsedTime) => {
        this.roundElapsedTime = elapsedTime; // 更新競標已進行時間
      });

      // 監聽單回合獲勝者公告事件
      this.socket.on('roundWinnerAnnounced', (data) => {
        this.winnerMessage = data.message; // 顯示獲勝者訊息
        this.showWinnerDisplay = true; // 顯示獲勝者公告
        // 清除現有計時器（如果存在），避免重複觸發
        if (this.winnerDisplayTimeout) {
          clearTimeout(this.winnerDisplayTimeout);
        }
        // 設定計時器，3秒後隱藏獲勝者公告
        this.winnerDisplayTimeout = setTimeout(() => {
          this.showWinnerDisplay = false;
          this.winnerMessage = '';
        }, 3000);
        console.log('Winner announced:', data);
      });

      // 監聽回合結束事件 (通知進入下一個準備階段)
      this.socket.on('roundEnded', (data) => {
        this.roundInProgress = false; // 競標結束
        this.preRoundCountdown = 0; // 清除倒數
        this.messageText = data.message;
        this.players = data.updatedPlayers;
        this.currentRound = data.currentRound;
        this.maxRounds = data.maxRounds; // 確保從正確的屬性獲取總回合數
        this.myPlayer = this.players.find(p => p.id === this.socket.id) || null;
        console.log('Round ended:', data.message);
      });

      // 監聽遊戲結束事件
      this.socket.on('gameOver', (data) => {
        this.gamePhase = 'gameOver'; // 設定遊戲階段為結束
        this.gameOverReason = data.reason; // 遊戲結束原因
        this.players = data.finalPlayers; // 所有玩家的最終狀態 (後端已排序)
        this.finalGameWinner = data.finalWinner; // 最終優勝者資訊 (可能包含 isTie)
        this.messageText = data.reason;
        console.log('Game Over:', data.reason);
        console.log('Final Players:', data.finalPlayers);
        console.log('Final Winner:', data.finalWinner);
      });

      // --- 新增：監聽回合統計 Modal 事件 ---
      this.socket.on('showRoundStatsModal', () => {
        this.showRoundStatsModal = true;
      });
    },

    // 創建房間
    createRoom() {
      if (!this.playerName.trim()) {
        this.errorMessage = '請輸入你的名字。';
        return;
      }
      // 驗證自定義輸入值
      if (typeof this.initialPlayerTime !== 'number' || this.initialPlayerTime < 10 || this.initialPlayerTime > 600 || isNaN(this.initialPlayerTime)) {
        this.errorMessage = '請輸入有效的起始時間 (10-600 秒)。';
        return;
      }
      if (typeof this.customMaxRounds !== 'number' || this.customMaxRounds < 1 || this.customMaxRounds > 50 || isNaN(this.customMaxRounds)) {
        this.errorMessage = '請輸入有效的遊戲回合數 (1-50 回合)。';
        return;
      }

      this.errorMessage = ''; // 清除之前的錯誤訊息
      // 如果未連接或已斷開，則重新連接 Socket
      if (!this.socket || !this.socket.connected) {
        this.connectSocket();
      }
      // 發送包含自定義值的數據對象到後端
      this.socket.emit('createRoom', {
        playerName: this.playerName,
        initialTime: this.initialPlayerTime,
        maxRounds: this.customMaxRounds
      }, (response) => { // 後端回調函數
        if (response.success) {
          this.roomId = response.roomId;
          this.myPlayer = response.player;
          this.gamePhase = 'roomLobby'; // 切換到大廳階段
          this.messageText = '房間已創建，等待其他玩家加入...';
          console.log('Room created:', this.roomId);
          this.checkAndShowRulesModal(); // 創建房間後檢查並顯示規則 Modal
        } else {
          this.errorMessage = response.message; // 顯示後端返回的錯誤訊息
        }
      });
    },

    // 加入房間
    joinRoom() {
      if (!this.joinRoomId.trim() || !this.playerName.trim()) {
        this.errorMessage = '請輸入你的名字和房間ID。';
        return;
      }
      this.errorMessage = ''; // 清除錯誤訊息
      if (!this.socket || !this.socket.connected) {
        this.connectSocket();
      }
      this.socket.emit('joinRoom', this.joinRoomId, this.playerName, (response) => { // 後端回調函數
        if (response.success) {
          this.roomId = response.roomId;
          this.myPlayer = response.player;
          this.gamePhase = 'roomLobby'; // 切換到大廳階段
          this.messageText = '已加入房間。';
          console.log('Joined room:', this.roomId);
          this.checkAndShowRulesModal(); // 加入房間後檢查並顯示規則 Modal
        } else {
          this.errorMessage = response.message; // 顯示後端返回的錯誤訊息
        }
      });
    },

    // 離開房間
    leaveRoom() {
      if (this.socket && this.socket.connected) {
        this.socket.emit('leaveRoom'); // 通知後端離開房間
      }
      // 立即切換到開始畫面並重置本地數據，提供更好的用戶體驗
      this.gamePhase = 'startScreen';
      this.resetGameData();
    },

    // 按住按鈕
    holdButton() {
      // 檢查玩家是否存在或時間是否用盡
      if (!this.myPlayer || this.myPlayer.time <= 0) {
        if (this.myPlayer && this.myPlayer.time <= 0) {
          this.messageText = '你的時間已用盡，無法按住！';
        }
        return;
      }

      // 只有在未按住、競標未開始、且倒數未開始時才能按住
      if (!this.myPlayer.isHoldingButton && !this.roundInProgress && this.preRoundCountdown === 0) {
        this.socket.emit('playerHolding'); // 通知後端玩家開始按住
        console.log('Holding button...');
      } else {
        console.log('Cannot hold button. Conditions:', {
          myPlayerTime: this.myPlayer.time,
          isHoldingButton: this.myPlayer.isHoldingButton,
          roundInProgress: this.roundInProgress,
          preRoundCountdown: this.preRoundCountdown
        });
      }
    },

    // 放開按鈕
    releaseButton() {
      // 只有在按住狀態下才能放開
      if (this.myPlayer && this.myPlayer.isHoldingButton) {
        this.socket.emit('playerReleased'); // 通知後端玩家放開按鈕
        console.log('Released button.');
      }
    },

    // 時間格式化函數 (秒 -> 分:秒)
    formatTime(seconds) {
      if (typeof seconds !== 'number' || isNaN(seconds) || seconds < 0) {
        return '00:00'; // 處理無效輸入
      }
      const min = Math.floor(seconds / 60);
      const sec = Math.floor(seconds % 60);
      return `${min.toString().padStart(2, '0')}:${sec.toString().padStart(2, '0')}`;
    },

    // 重新開始遊戲 (回到開始畫面並重置所有數據)
    restartGame() {
      this.resetGameData();
      this.gamePhase = 'startScreen';
    },

    // 重置所有遊戲數據到初始狀態
    resetGameData() {
      this.roomId = '';
      this.joinRoomId = '';
      this.players = [];
      this.myPlayer = null;
      this.messageText = '等待所有玩家加入並按住按鈕...';
      this.errorMessage = '';
      this.preRoundCountdown = 0;
      this.roundInProgress = false;
      this.roundElapsedTime = 0;
      this.currentRound = 0;
      this.maxRounds = 19;
      this.winnerMessage = '';
      this.showWinnerDisplay = false;
      if (this.winnerDisplayTimeout) {
        clearTimeout(this.winnerDisplayTimeout);
        this.winnerDisplayTimeout = null;
      }
      // 重置自定義設置為預設值
      this.initialPlayerTime = 60;
      this.customMaxRounds = 5;
      this.gameOverReason = '';
      this.finalGameWinner = null;

      // 重置 Modal 狀態
      this.showRulesModal = false;
      this.showRoundStatsModal = false;
    },

    // --- 新增 Modal 相關方法 ---
    // 檢查並顯示規則 Modal
    checkAndShowRulesModal() {
      // 使用 localStorage 檢查玩家是否已經看過規則
      const hasSeenRules = localStorage.getItem('hasSeenTimeAuctionRules');
      if (!hasSeenRules) {
        this.showRulesModal = true;
      }
    },
    // 關閉規則 Modal 並標記為已看過
    closeRulesModal() {
      this.showRulesModal = false;
      localStorage.setItem('hasSeenTimeAuctionRules', 'true'); // 標記為已看過
    },
    // 關閉回合統計 Modal
    closeRoundStatsModal() {
      this.showRoundStatsModal = false;
    }
  },
  // 組件掛載後立即連接 WebSocket
  mounted() {
    this.connectSocket();
  },
  // 組件銷毀前斷開 WebSocket 連接並清理計時器
  beforeUnmount() {
    if (this.socket) {
      this.socket.disconnect();
    }
    if (this.winnerDisplayTimeout) {
      clearTimeout(this.winnerDisplayTimeout);
    }
  }
};
</script>

<style scoped>
/* 調整 game-container 寬度使其更大 */
.game-container {
  font-family: "Arial", sans-serif;
  max-width: 800px;
  /* 從 700px 增加到 800px，利用更多寬度 */
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
  width: 100%;
}

h1 {
  color: #333;
  margin-bottom: 15px;
  font-size: 2.2em;
  /* 稍微放大標題 */
}

/* 開始畫面 */
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
  max-width: 600px;
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

/* 新增的輸入提示樣式 */
.start-screen .input-hint {
  font-size: 0.9em;
  color: #777;
  margin-top: -8px;
  /* 向上微調 */
  margin-bottom: 5px;
  /* 增加與下一個元素的間距 */
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
  /* 統一背景 */
  padding: 20px;
  /* 增加內邊距 */
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.message {
  font-weight: bold;
  color: #e44d26;
  margin-bottom: 10px;
  /* 減少間距 */
  font-size: 1.2em;
  min-height: 1.2em;
  /* 確保即使無內容也佔據高度 */
}

/* 頂部倒數/碼表區 */
.fixed-height-status-area {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 15px;
  margin: 10px auto 0;
  max-width: 800px;
  /* 與新的 game-container 寬度一致 */
  text-align: center;
  height: 80px;
  /* 調整高度以適應新內容大小 */
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  position: relative;
  overflow: hidden;
}

.status-text-content {
  font-size: 2.2em;
  font-weight: bold;
  color: #333;
  display: flex;
  justify-content: center;
  align-items: baseline;
  background-color: #e0f2f7;
  border-radius: 6px;
  border: 1px solid #b3e5fc;
  /* **調整以容納最長文本，確保穩定和居中** */
  margin: auto;
}

.status-text-content .highlight {
  font-size: 1.2em;
}

.countdown-message {
  color: #f44336;
  padding: 18px;
}

.status-message {
  font-size: 0.9em;
  color: #6c757d;
  padding: 18px;
}

/* 獲勝者公告樣式 */
.winner-announcement-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.85);
  /* 更深的半透明背景 */
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  border-radius: 8px;
}

.winner-announcement-box {
  background-color: #ffeb3b;
  /* 亮黃色背景 */
  color: #333;
  /* 深色文字 */
  padding: 15px 30px;
  border-radius: 8px;
  font-size: 1.8em;
  /* 放大字體 */
  font-weight: bold;
  text-align: center;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

/* 過渡效果 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}


/* 佈局調整：將房間資訊和玩家個人狀態並排 */
.top-info-row {
  display: flex;
  justify-content: space-between;
  gap: 15px;
  /* 兩個區塊之間的間距 */
  flex-wrap: wrap;
  /* 小螢幕下允許換行 */
}

.header-info {
  flex: 1;
  /* 讓它們均分空間 */
  min-width: 280px;
  /* 最小寬度，防止過度壓縮 */
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 15px 20px;
  /* 調整內邊距 */
  text-align: left;
}

.header-info h2,
.header-info p {
  color: #555;
  margin-top: 0;
  margin-bottom: 10px;
  border-bottom: 1px solid #eee;
  padding-bottom: 5px;
  font-size: 1.2em;
}

/* 玩家資訊區塊 */
.player-info-section {
  background-color: #f7fafd;
  /* 保持輕微的背景色 */
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 20px;
  /* 增加內邊距 */
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
  /* 保持內陰影 */
  text-align: left;
  /* 左對齊文本 */
  display: flex;
  /* 使用 flexbox 佈局 */
  flex-direction: column;
  /* 垂直排列內容 */
  gap: 15px;
  /* 增加區塊內元素間的間距 */
  flex: 1;
  min-width: 280px;
}

.player-info-section h2 {
  color: #627d98;
  /* 保持標題顏色 */
  border-bottom: 1px solid #e2e8f0;
  /* 保持底線 */
  padding-bottom: 8px;
  margin-bottom: 15px;
  /* 增加標題與內容的間距 */
  font-weight: 600;
  font-size: 1.3em;
}

.player-details {
  display: flex;
  flex-direction: column;
  gap: 10px;
  /* 每個資訊項目之間的間距 */
}

.detail-item {
  display: flex;
  /* 讓圖標和文字在同一行 */
  align-items: center;
  /* 垂直居中對齊 */
  padding: 8px 10px;
  /* 每個資訊項目的內邊距 */
  background-color: #ffffff;
  /* 項目背景色 */
  border-radius: 6px;
  /* 圓角 */
  border: 1px solid #eef2f6;
  /* 輕微邊框 */
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  /* 輕微陰影 */
  font-size: 1.1em;
  /* 調整字體大小 */
  color: #334e68;
  /* 主文字顏色 */
}

.detail-item .icon {
  margin-right: 10px;
  /* 圖標與文字的間距 */
  font-size: 1.2em;
  /* 圖標大小 */
  color: #2196f3;
  /* 圖標顏色，使用主題的強調色 */
  min-width: 25px;
  /* 確保圖標有最小寬度，防止文本擠壓 */
  text-align: center;
}

.detail-item .highlight {
  font-weight: 700;
  /* 強調數字和名稱 */
  color: #333;
  /* 可以選擇更深的顏色來強調 */
  margin-left: 5px;
  /* 強調內容與文字間的間距 */
}

.player-name {
  color: #1976d2;
  /* 玩家名稱可以特別顏色 */
}

.player-time {
  color: #4CAF50;
  /* 時間使用綠色 */
}

.player-tokens {
  color: #ff9800;
  /* 代幣使用橙色 */
}

.loading-message {
  text-align: center;
  font-style: italic;
  color: #627d98;
  padding: 10px;
}


/* 玩家列表區塊 */
.players-status-section {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 20px;
  text-align: left;
  flex: 1;
  min-width: 300px;
}

.players-status-section h3 {
  color: #555;
  margin-top: 0;
  margin-bottom: 12px;
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  font-size: 1.2em;
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
  border-bottom: 1px dashed #eee;
  font-size: 1em;
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
}

.status-generic {
  color: #888;
  /* 灰色表示通用狀態 */
  font-style: italic;
}


/* 按鈕區塊 */
.bid-section-button-only {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 20px;
  text-align: center;
}

.hold-button {
  width: 250px;
  height: 90px;
  font-size: 1.7em;
  font-weight: bold;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  box-shadow: 0 5px 10px rgba(0, 0, 0, 0.25);
  transition: background-color 0.1s ease-out, transform 0.1s ease-out;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.hold-button:active {
  background-color: #1976d2;
  transform: translateY(3px);
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
  border-radius: 5px;
  border: none;
  cursor: pointer;
  font-size: 1em;
  margin-top: 15px;
}

/* 遊戲結束畫面 */
.game-over-screen {
  background-color: #fff;
  border: 1px solid #eee;
  border-radius: 10px;
  padding: 25px;
  margin-top: 20px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.game-over-screen h2 {
  color: #dc3545;
  margin-bottom: 10px;
  font-size: 2.2em;
  font-weight: 700;
}

.final-message {
  font-size: 1.3em;
  color: #555;
  margin-bottom: 15px;
  text-align: center;
}

.final-winner-display {
  background-color: #ffffe0;
  border: 2px solid #ffd700;
  border-radius: 10px;
  padding: 15px 30px;
  margin-top: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  text-align: center;
  width: 80%;
  max-width: 400px;
}

.final-winner-display h3 {
  font-size: 1.6em;
  color: #daa520;
  margin-bottom: 10px;
  border-bottom: none;
  padding-bottom: 0;
}

.winner-name-large {
  font-size: 2em;
  font-weight: bold;
  color: #4CAF50;
  margin-bottom: 5px;
}

.winner-stats-large {
  font-size: 1.5em;
  font-weight: 600;
  color: #ff9800;
}

.game-over-screen h3 {
  color: #555;
  margin-top: 25px;
  margin-bottom: 15px;
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
  font-size: 1.4em;
  width: 100%;
}

.final-scores-list {
  list-style: none;
  padding: 0;
  margin-bottom: 25px;
  width: 90%;
  max-width: 500px;
  background-color: #f0f4f8;
  border-radius: 8px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
}

.final-scores-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 15px;
  border-bottom: 1px dashed #c0c0c0;
  font-size: 1.1em;
  color: #333;
}

.final-scores-list li:last-child {
  border-bottom: none;
}

.final-scores-list li .rank {
  font-weight: bold;
  color: #666;
  margin-right: 10px;
  min-width: 25px;
  text-align: left;
}

.final-scores-list li .player-name-final {
  font-weight: bold;
  color: #334e68;
  flex-grow: 1;
  text-align: left;
}

.final-scores-list li .player-stats-final {
  color: #627d98;
  font-size: 0.95em;
  min-width: 150px;
  text-align: right;
}

.final-scores-list li.is-final-winner {
  background-color: #e6ffe6;
  border: 1px solid #4CAF50;
  font-weight: bold;
  box-shadow: 0 0 5px rgba(76, 175, 80, 0.3);
}

.final-scores-list li.is-cowinner {
  background-color: #e0f2f7;
  border: 1px solid #2196f3;
  font-weight: bold;
  box-shadow: 0 0 5px rgba(33, 150, 243, 0.3);
}


.game-over-screen button {
  padding: 14px 30px;
  font-size: 1.2em;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.3s, transform 0.2s;
}

.game-over-screen button:hover {
  background-color: #1976d2;
  transform: translateY(-2px);
}

/* 響應式調整 */
@media (max-width: 600px) {
  .game-over-screen h2 {
    font-size: 1.8em;
  }

  .final-winner-display {
    width: 95%;
    padding: 10px 15px;
  }

  .winner-name-large {
    font-size: 1.8em;
  }

  .winner-stats-large {
    font-size: 1.2em;
  }

  .final-scores-list {
    width: 95%;
  }

  .final-scores-list li {
    flex-direction: column;
    align-items: flex-start;
    padding: 10px;
  }

  .final-scores-list li .player-stats-final {
    margin-top: 5px;
    font-size: 0.9em;
  }

  .final-scores-list li .rank {
    margin-right: 0;
    width: 100%;
    text-align: left;
  }

  .final-scores-list li .player-name-final {
    width: 100%;
    text-align: left;
  }

  .final-scores-list li .player-stats-final {
    width: 100%;
    text-align: left;
    min-width: unset;
  }
}


/* Modal 相關樣式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  /* 確保在最上層 */
}

.modal-content {
  background-color: #fff;
  padding: 30px;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
  max-width: 600px;
  width: 90%;
  text-align: left;
  max-height: 80vh;
  /* 限制最大高度 */
  overflow-y: auto;
  /* 超出部分滾動 */
}

.modal-content h2 {
  color: #2196f3;
  margin-top: 0;
  margin-bottom: 20px;
  font-size: 1.8em;
  text-align: center;
}

.modal-content h3 {
  color: #333;
  margin-top: 15px;
  margin-bottom: 10px;
  font-size: 1.3em;
  border-bottom: 1px dashed #ddd;
  padding-bottom: 5px;
}

.modal-content p {
  line-height: 1.6;
  margin-bottom: 10px;
  color: #555;
}

.modal-content hr {
  border: 0;
  border-top: 1px solid #eee;
  margin: 20px 0;
}

.modal-content ul {
  list-style: disc;
  margin-left: 20px;
  margin-bottom: 15px;
}

.modal-content ul li {
  margin-bottom: 5px;
  line-height: 1.5;
  color: #555;
}

.modal-button {
  display: block;
  width: 100%;
  padding: 12px 20px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1.1em;
  margin-top: 25px;
  transition: background-color 0.3s;
}

.modal-button:hover {
  background-color: #45a049;
}

/* 回合統計 Modal 特有樣式 */
.stats-list {
  list-style: none;
  padding: 0;
  margin-top: 15px;
}

.stats-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 0;
  border-bottom: 1px dashed #eee;
  font-size: 1.1em;
  color: #333;
}

.stats-list li:last-child {
  border-bottom: none;
}

.stats-list li span:first-child {
  font-weight: bold;
  color: #334e68;
}

.stats-list li span:last-child {
  color: #ff9800;
  font-weight: bold;
}

/* Modal 過渡動畫 */
.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.3s ease;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
  opacity: 0;
}

.modal-fade-enter-active .modal-content,
.modal-fade-leave-active .modal-content {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

.modal-fade-enter-from .modal-content,
.modal-fade-leave-to .modal-content {
  transform: translateY(-20px);
  opacity: 0;
}
</style>