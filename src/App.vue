<template>
  <div class="fishing-game">
    <div class="game-header">
      <h1>🎣 Sporty Fishing</h1>
      <div class="balance">
        Balance: ${{ balance }}
      </div>
    </div>
    
    <div class="game-container">
      <div class="fishing-scene">
        <!-- 背景 - 替換成你的背景圖片 -->
        <!-- <div class="custom-background"></div> -->
        <div class="sky"></div>
        <div class="mountains"></div>
        
        <!-- 釣魚人 -->
        <div class="fisherman">
          <div class="person">
            <img src="/fisherman.png" alt="釣魚人" class="fisherman-img">
          </div>
          <div class="fishing-rod" 
               :class="{ 
                 casting: isFishing,
                 shaking: isFishing && escapeChance > 10
               }"
               :style="{ '--shake-intensity': shakeIntensity }">
            <!-- 選擇1: CSS釣魚竿 (目前) -->
            <div class="rod-line"></div>
            
            <!-- 選擇2: 圖片釣魚竿 (取消注釋來使用) -->
            <!-- <img src="@/assets/fishing-rod.png" alt="釣魚竿" class="rod-image"> -->
          </div>
        </div>
        
        <!-- 水面 -->
        <div class="water-surface">
          <div class="water-ripples" v-if="isFishing"></div>
          
          <!-- 釣魚線 -->
          <div class="fishing-line" 
               v-if="isFishing"
               :class="{ 
                 shaking: escapeChance > 20,
                 hooked: fishHooked 
               }"
               :style="{ 
                 '--line-shake': lineShakeAmount + 'px',
                 '--line-length': fishingLineLength + 'px'
               }">
            <div class="hook">🪝</div>
            
            <!-- 魚咬鉤效果 -->
            <div class="fish-bite" v-if="fishHooked" :class="{ struggling: fishStruggling }">
              🐟
            </div>
          </div>
        </div>
        
        <!-- 水底 -->
        <div class="underwater">
          <div class="fish-swimming" v-if="!fishHooked && isFishing">
            <div class="fish" v-for="i in 3" :key="i" :style="{ animationDelay: (i * 0.5) + 's' }">🐠</div>
          </div>
        </div>
        
        <!-- 遊戲狀態顯示 -->
        <div class="game-hud" v-if="isFishing">
          <div class="tension-meter">
            <div class="meter-label">Tension</div>
            <div class="meter-bar">
              <div class="meter-fill" :style="{ width: (escapeChance) + '%' }"></div>
            </div>
            <div class="meter-value">{{ escapeChance.toFixed(0) }}%</div>
          </div>
          
          <div class="multiplier-display">
            <div class="multiplier">{{ currentMultiplier.toFixed(2) }}x</div>
            <div class="potential-win">${{ (betAmount * currentMultiplier).toFixed(2) }}</div>
          </div>
          
          <div class="fishing-status">
            <div v-if="!fishHooked" class="waiting">Waiting for bite...</div>
            <div v-else class="hooked">Fish hooked! {{ fishStruggleTime.toFixed(1) }}s</div>
          </div>
        </div>
      </div>
      
      <div class="game-controls">
        <div class="bet-section" v-if="!isFishing">
          <label>Bet Amount:</label>
          <input 
            type="number" 
            v-model="betAmount" 
            :max="balance" 
            min="1"
            :disabled="isFishing"
          >
          <button 
            @click="startFishing" 
            :disabled="betAmount <= 0 || betAmount > balance"
            class="start-btn"
          >
            Start Fishing
          </button>
        </div>
        
        <div class="fishing-controls" v-if="isFishing">
          <button 
            @click="catchFish" 
            class="catch-btn"
            :disabled="!fishHooked"
            :class="{ disabled: !fishHooked, pulsing: fishHooked }"
          >
            <span v-if="!fishHooked">Waiting for bite...</span>
            <span v-else>Reel it in! (Win ${{ (betAmount * currentMultiplier).toFixed(2) }})</span>
          </button>
        </div>
      </div>
      
      <!-- 結果彈出視窗 -->
      <div class="result-overlay" v-if="gameResult" @click="closeResult">
        <div class="result-modal" :class="gameResult.type">
          <div class="result-icon">
            <span v-if="gameResult.type === 'win'">🎉</span>
            <span v-else-if="gameResult.type === 'lose'">😭</span>
            <span v-else>ℹ️</span>
          </div>
          <h3 class="result-title">
            {{ gameResult.type === 'win' ? 'Success!' : gameResult.type === 'lose' ? 'Failed!' : 'Info' }}
          </h3>
          <div class="result-message">
            {{ gameResult.message }}
          </div>
          <button class="result-close-btn" @click="closeResult">
            Continue Game
          </button>
        </div>
      </div>
      
      <!-- 版權聲明 -->
      <div class="copyright">
        © 2025 Evans. All rights reserved.
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'FishingGame',
  data() {
    return {
      balance: 1000,
      betAmount: 10,
      isFishing: false,
      fishingTime: 0,
      currentMultiplier: 1.0,
      escapeChance: 0,
      fishCaught: false,
      fishHooked: false,
      fishStruggling: false,
      fishStruggleTime: 0,
      gameResult: null,
      fishingInterval: null,
      hookTimeout: null,
      shakeIntensity: 0,
      lineShakeAmount: 0,
      fishingLineLength: 120
    }
  },
  methods: {
    startFishing() {
      if (this.betAmount > this.balance) return;
      
      this.balance -= this.betAmount;
      this.isFishing = true;
      this.fishingTime = 0;
      this.currentMultiplier = 1.0;
      this.escapeChance = 0;
      this.fishCaught = false;
      this.fishHooked = false;
      this.fishStruggling = false;
      this.fishStruggleTime = 0;
      this.gameResult = null;
      this.shakeIntensity = 0;
      this.lineShakeAmount = 0;
      
      // 隨機決定魚咬鉤時間 (3-8秒)
      const hookTime = 3000 + Math.random() * 5000;
      this.hookTimeout = setTimeout(() => {
        this.fishBitesHook();
      }, hookTime);
      
      this.fishingInterval = setInterval(() => {
        this.fishingTime += 0.1;
        
        if (this.fishHooked) {
          this.fishStruggleTime += 0.1;
          this.currentMultiplier = 1 + (this.fishStruggleTime * 0.3);
          this.escapeChance = Math.min(this.fishStruggleTime * 8, 95);
          
          // 更新抖動強度
          this.shakeIntensity = Math.min(this.escapeChance / 10, 8);
          this.lineShakeAmount = Math.min(this.escapeChance / 5, 15);
          
          // 魚掙扎動畫
          this.fishStruggling = Math.random() < 0.6;
          
          // 檢查逃脫 - 每0.1秒檢查一次
          const escapeRoll = Math.random() * 100;
          if (escapeRoll < this.escapeChance / 10) { // 每0.1秒有實際逃脫機率
            console.log('魚逃脫了！escapeChance:', this.escapeChance, 'roll:', escapeRoll);
            this.fishEscaped();
          }
        } else {
          // 如果魚還沒咬鉤，有機會不咬鉤就結束
          if (this.fishingTime > 15) { // 15秒後沒咬鉤就結束
            this.fishEscaped();
          }
        }
      }, 100);
    },
    
    fishBitesHook() {
      if (!this.isFishing) return;
      
      console.log('魚咬鉤了！fishHooked從', this.fishHooked, '變為 true');
      this.fishHooked = true;
      this.fishStruggleTime = 0;
    },
    
    catchFish() {
      if (!this.isFishing || !this.fishHooked) return;
      
      clearInterval(this.fishingInterval);
      clearTimeout(this.hookTimeout);
      this.fishCaught = true;
      
      const winAmount = this.betAmount * this.currentMultiplier;
      this.balance += winAmount;
      
      this.gameResult = {
        type: 'win',
        message: `Successfully caught a fish! Won $${winAmount.toFixed(2)}`
      };
      
      // 不自動關閉，等玩家點擊
    },
    
    fishEscaped() {
      if (!this.isFishing) return;
      
      clearInterval(this.fishingInterval);
      clearTimeout(this.hookTimeout);
      
      // 先保存魚的狀態用於顯示訊息
      const wasHooked = this.fishHooked;
      
      // 立即更新魚的狀態
      this.fishHooked = false;
      this.fishStruggling = false;
      
      const loseMessage = wasHooked 
        ? `The fish escaped! Lost $${this.betAmount}` 
        : `No fish bit the hook! Lost $${this.betAmount}`;
      
      this.gameResult = {
        type: 'lose',
        message: loseMessage
      };
      
      // 不自動關閉，等玩家點擊
    },
    
    closeResult() {
      this.gameResult = null;
      this.resetGame();
    },
    
    resetGame() {
      this.isFishing = false;
      this.fishingTime = 0;
      this.fishStruggleTime = 0;
      this.currentMultiplier = 1.0;
      this.escapeChance = 0;
      this.fishCaught = false;
      this.fishHooked = false;
      this.fishStruggling = false;
      this.gameResult = null;
      this.shakeIntensity = 0;
      this.lineShakeAmount = 0;
      
      if (this.balance <= 0) {
        this.balance = 1000;
        this.gameResult = {
          type: 'info',
          message: 'Insufficient balance, restarting with $1000!'
        };
      }
    }
  },
  
  beforeUnmount() {
    if (this.fishingInterval) {
      clearInterval(this.fishingInterval);
    }
    if (this.hookTimeout) {
      clearTimeout(this.hookTimeout);
    }
  }
}
</script>

<style scoped>
.fishing-game {
  background: white;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.2);
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 2px solid #f0f0f0;
}

.game-header h1 {
  color: #e11d48;
  font-size: 2rem;
  text-shadow: 2px 2px 4px rgba(225, 29, 72, 0.3);
}

.balance {
  font-size: 1.5rem;
  font-weight: bold;
  color: #e11d48;
}

.game-container {
  display: flex;
  flex-direction: column;
  gap: 30px;
}

/* 釣魚場景 */
.fishing-scene {
  position: relative;
  height: 500px;
  border-radius: 15px;
  overflow: hidden;
  background: linear-gradient(to bottom, #87ceeb 0%, #98d8e8 30%, #4682b4 70%, #2c5d7a 100%);
}

/* 自定義背景圖片 - 取消注释並修改路徑 */
/*
.custom-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: url('@/assets/background.jpg');
  background-size: cover;
  background-position: center;
  z-index: 1;
}
*/

/* 天空背景 */
.sky {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 150px;
  background: linear-gradient(to bottom, #87ceeb 0%, #b0e0e6 100%);
}

/* 山脈背景 */
.mountains {
  position: absolute;
  top: 100px;
  left: 0;
  right: 0;
  height: 80px;
  background: linear-gradient(45deg, #696969, #a9a9a9, #696969);
  clip-path: polygon(0% 100%, 20% 40%, 40% 60%, 60% 20%, 80% 50%, 100% 30%, 100% 100%);
}

/* 釣魚人 */
.fisherman {
  position: absolute;
  top: 50px;
  left: 50px;
  z-index: 10;
}

.person {
  font-size: 3rem;
  margin-bottom: 10px;
}

/* 自定義釣魚人圖片 */
.fisherman-img {
  width: 150px;           /* 調整寬度 */
  height: 150px;          /* 調整高度 */
  object-fit: contain;    /* 保持比例 */
  filter: drop-shadow(2px 2px 4px rgba(0,0,0,0.3));
}

/* 自定義釣魚竿圖片 - 取消注释來使用 */
/*
.rod-image {
  width: 150px;
  height: auto;
  object-fit: contain;
  transform-origin: 10px center;
  filter: drop-shadow(2px 2px 4px rgba(0,0,0,0.3));
}

.fishing-rod.casting .rod-image {
  transform: rotate(25deg);
}

.fishing-rod.shaking .rod-image {
  animation: rod-shake-image 0.3s ease-in-out infinite;
}

@keyframes rod-shake-image {
  0%, 100% { transform: rotate(25deg); }
  25% { transform: rotate(calc(25deg + var(--shake-intensity, 0) * 1deg)); }
  75% { transform: rotate(calc(25deg - var(--shake-intensity, 0) * 1deg)); }
}
*/

.fishing-rod {
  position: relative;
  width: 150px;
  height: 4px;
  background: #8b4513;
  border-radius: 2px;
  transform-origin: 10px center;
  transform: rotate(35deg);
  transition: transform 0.5s ease;
}

.fishing-rod.casting {
  transform: rotate(25deg);
}

.fishing-rod.shaking {
  animation: rod-shake 0.3s ease-in-out infinite;
}

.rod-line {
  position: absolute;
  right: 0;
  top: 50%;
  width: 2px;
  height: 200px;
  background: #333;
  transform-origin: top;
}

@keyframes rod-shake {
  0%, 100% { transform: rotate(25deg); }
  25% { transform: rotate(calc(25deg + var(--shake-intensity, 0) * 1deg)); }
  75% { transform: rotate(calc(25deg - var(--shake-intensity, 0) * 1deg)); }
}

/* 水面 */
.water-surface {
  position: absolute;
  top: 180px;
  left: 0;
  right: 0;
  height: 320px;
  background: linear-gradient(to bottom, rgba(135, 206, 235, 0.8) 0%, rgba(70, 130, 180, 0.9) 100%);
}

.water-ripples {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 100px;
  height: 100px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.6);
  animation: ripples 2s ease-out infinite;
}

@keyframes ripples {
  0% {
    width: 20px;
    height: 20px;
    opacity: 1;
  }
  100% {
    width: 100px;
    height: 100px;
    opacity: 0;
  }
}

/* 釣魚線 */
.fishing-line {
  position: absolute;
  top: -20px;
  left: 50%;
  transform: translateX(-50%);
  width: 2px;
  height: var(--line-length, 120px);
  background: #333;
  transform-origin: top;
}

.fishing-line.shaking {
  animation: line-shake 0.2s ease-in-out infinite;
}

.fishing-line.hooked {
  animation: line-tension 0.5s ease-in-out infinite;
}

@keyframes line-shake {
  0%, 100% { transform: translateX(-50%); }
  50% { transform: translateX(calc(-50% + var(--line-shake, 5px))); }
}

@keyframes line-tension {
  0%, 100% { transform: translateX(-50%) scaleY(1); }
  50% { transform: translateX(calc(-50% + 3px)) scaleY(1.1); }
}

.hook {
  position: absolute;
  bottom: -15px;
  left: -8px;
  font-size: 1.5rem;
}

/* 魚咬鉤效果 */
.fish-bite {
  position: absolute;
  bottom: -25px;
  left: -15px;
  font-size: 2rem;
  z-index: 5;
}

.fish-bite.struggling {
  animation: fish-struggle 0.3s ease-in-out;
}

@keyframes fish-struggle {
  0%, 100% { transform: rotate(0deg); }
  25% { transform: rotate(-15deg) translateY(-5px); }
  75% { transform: rotate(15deg) translateY(5px); }
}

/* 水底游泳的魚 */
.underwater {
  position: absolute;
  bottom: 50px;
  left: 0;
  right: 0;
  height: 150px;
}

.fish-swimming {
  position: relative;
  width: 100%;
  height: 100%;
}

.fish {
  position: absolute;
  font-size: 1.5rem;
  animation: swimming 4s ease-in-out infinite;
}

.fish:nth-child(1) {
  top: 20px;
  animation-duration: 3s;
}

.fish:nth-child(2) {
  top: 60px;
  animation-duration: 4s;
}

.fish:nth-child(3) {
  top: 100px;
  animation-duration: 5s;
}

@keyframes swimming {
  0% {
    left: -50px;
    transform: scaleX(1);
  }
  50% {
    left: 50%;
    transform: scaleX(1);
  }
  100% {
    left: calc(100% + 50px);
    transform: scaleX(-1);
  }
}

/* 遊戲HUD */
.game-hud {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 20px;
  border-radius: 10px;
  min-width: 200px;
}

.tension-meter {
  margin-bottom: 15px;
}

.meter-label {
  font-size: 0.9rem;
  margin-bottom: 5px;
}

.meter-bar {
  width: 100%;
  height: 20px;
  background: #333;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.meter-fill {
  height: 100%;
  background: linear-gradient(90deg, #4ade80 0%, #fbbf24 50%, #ef4444 100%);
  transition: width 0.3s ease;
}

.meter-value {
  text-align: center;
  font-weight: bold;
  margin-top: 5px;
}

.multiplier-display {
  text-align: center;
  margin-bottom: 15px;
}

.multiplier {
  font-size: 1.5rem;
  font-weight: bold;
  color: #e11d48;
}

.potential-win {
  font-size: 1.1rem;
  color: #4ade80;
}

.fishing-status {
  text-align: center;
  font-size: 0.9rem;
}

.waiting {
  color: #94a3b8;
}

.hooked {
  color: #e11d48;
  font-weight: bold;
  animation: pulse 1s ease-in-out infinite;
}

/* 控制按鈕 */
.game-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.bet-section {
  display: flex;
  align-items: center;
  gap: 15px;
  flex-wrap: wrap;
}

.bet-section label {
  font-weight: bold;
  color: #2d3748;
}

.bet-section input {
  padding: 10px;
  border: 2px solid #cbd5e0;
  border-radius: 5px;
  font-size: 1rem;
  width: 120px;
}

.start-btn, .catch-btn {
  padding: 15px 30px;
  font-size: 1.1rem;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.start-btn {
  background: #e11d48;
  color: white;
}

.start-btn:hover:not(:disabled) {
  background: #be185d;
  transform: translateY(-2px);
}

.start-btn:disabled {
  background: #a0aec0;
  cursor: not-allowed;
}

.catch-btn {
  background: #e11d48;
  color: white;
  font-size: 1.2rem;
  padding: 20px 40px;
  min-width: 300px;
}

.catch-btn.disabled {
  background: #a0aec0;
  cursor: not-allowed;
}

.catch-btn.pulsing {
  animation: pulse 1s ease-in-out infinite;
}

.catch-btn:hover:not(.disabled) {
  background: #be185d;
  transform: scale(1.05);
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.02); }
}

/* 結果彈出視窗 */
.result-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeInOverlay 0.3s ease-out;
}

@keyframes fadeInOverlay {
  from { opacity: 0; }
  to { opacity: 1; }
}

.result-modal {
  background: white;
  border-radius: 20px;
  padding: 40px;
  text-align: center;
  min-width: 350px;
  max-width: 90vw;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: popIn 0.4s cubic-bezier(0.68, -0.55, 0.265, 1.55);
  position: relative;
}

@keyframes popIn {
  from { 
    opacity: 0; 
    transform: scale(0.5) translateY(-50px); 
  }
  to { 
    opacity: 1; 
    transform: scale(1) translateY(0); 
  }
}

.result-icon {
  font-size: 4rem;
  margin-bottom: 20px;
  animation: bounceIcon 0.6s ease-out 0.2s both;
}

@keyframes bounceIcon {
  from { transform: scale(0); }
  50% { transform: scale(1.2); }
  to { transform: scale(1); }
}

.result-title {
  font-size: 2rem;
  margin-bottom: 15px;
  font-weight: bold;
}

.result-modal.win .result-title {
  color: #e11d48;
}

.result-modal.lose .result-title {
  color: #e11d48;
}

.result-modal.info .result-title {
  color: #e11d48;
}

.result-message {
  font-size: 1.3rem;
  margin-bottom: 30px;
  line-height: 1.5;
  color: #4a5568;
}

.result-close-btn {
  background: linear-gradient(135deg, #e11d48 0%, #be185d 100%);
  color: white;
  border: none;
  padding: 15px 40px;
  font-size: 1.1rem;
  font-weight: bold;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(225, 29, 72, 0.4);
}

.result-close-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(225, 29, 72, 0.6);
}

.result-close-btn:active {
  transform: translateY(0);
}

/* 不同類型的邊框顏色 */
.result-modal.win {
  border-top: 5px solid #e11d48;
}

.result-modal.lose {
  border-top: 5px solid #e11d48;
}

.result-modal.info {
  border-top: 5px solid #e11d48;
}

/* 版權聲明 */
.copyright {
  text-align: center;
  margin-top: 30px;
  padding: 15px;
  color: #6b7280;
  font-size: 0.9rem;
  border-top: 1px solid #e5e7eb;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 0 0 15px 15px;
}

/* 響應式設計 */
@media (max-width: 768px) {
  .fishing-game {
    padding: 20px;
  }
  
  .game-header {
    flex-direction: column;
    gap: 15px;
    text-align: center;
  }
  
  .fishing-scene {
    height: 400px;
  }
  
  .fisherman {
    left: 20px;
    top: 30px;
  }
  
  .person {
    font-size: 2rem;
  }
  
  .fishing-rod {
    width: 100px;
  }
  
  .game-hud {
    position: static;
    margin-top: 20px;
    margin-bottom: 20px;
  }
  
  .bet-section {
    justify-content: center;
  }
  
  .catch-btn {
    min-width: 250px;
    font-size: 1rem;
  }
}
</style>