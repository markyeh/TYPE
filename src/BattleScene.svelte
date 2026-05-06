<script>
  import { fade } from 'svelte/transition';
  export let enemies;
  export let player;
  export let gameState;
  export let selectedMonsterId; // 新增 selectedMonsterId prop
  export let t;
  export let currentLanguage; // 接收 currentLanguage prop
  export let showComboDisplay; // 接收連擊數顯示狀態
  export let currentComboDisplayCount; // 接收當前連擊數
  export let currentWpm = 0; // 接收即時 WPM

  // 計算當前選中怪物的索引，用於定位選擇框
  $: selectedIndex = enemies.findIndex(e => e.id === selectedMonsterId);
  $: showFrame = selectedIndex !== -1 && (gameState === 'ACTION_SELECT' || gameState === 'BATTLE');
</script>

<div class="battle-scene">
  <div class="enemies-row">
    <!-- 平滑移動的選擇框 -->
    {#if showFrame}
      <div 
        class="selection-frame" 
        style="left: {((selectedIndex * 2 + 1) / (enemies.length * 2)) * 100}%;">
      </div>
    {/if}

    {#each enemies as enemy}
      <div 
        class="monster-wrapper" 
        class:dead={enemy.hp <= 0} 
        class:hit={enemy.isHit}
        class:attacking={enemy.isAttacking}
        class:target-success={enemy.isTargetSuccess}
        style="--tier-color: {enemy.wordType === 'white' ? '#fff' : (enemy.wordType === 'magic' ? '#3498db' : (enemy.wordType === 'rare' ? '#f1c40f' : '#9b59b6'))};">
        <div class="monster-sprite">
          {enemy.icon}
        </div>
        <div class="monster-name-display">
          <div class="name-race">{enemy.nameParts.race[currentLanguage]}</div>
          <div class="name-base">{enemy.nameParts.base[currentLanguage]}</div>
        </div>
        <div class="stat-row">
          {t('hp')}: {enemy.hp} / {enemy.maxHp}
          {#if enemy.lastDamage > 0}
            <span class="damage-pop">-{enemy.lastDamage}</span>
          {/if}
        </div>
        <div class="bar-container"><div class="hp-fill" style="width: {(enemy.hp/enemy.maxHp)*100}%"></div></div>
        <div class="stat-row">{t('mp')}: {enemy.mp} / {enemy.maxMp}</div>
        <div class="bar-container"><div class="mp-fill" style="width: {(enemy.mp/enemy.maxMp)*100}%"></div></div>
        <div class="stat-row">{t('atb')}: {Math.floor(enemy.atb)}%</div>
        <div class="bar-container atb"><div class="atb-fill" style="width: {enemy.atb}%"></div></div>
      </div>
    {/each}
  </div>

  <div class="player-section">
    <!-- 連擊顯示移出 player-visual-wrapper，避免受受傷震動動畫影響 -->
    <!-- 僅當 showComboDisplay 為真且連擊數達到 2 以上才顯示 -->
    {#if showComboDisplay && currentComboDisplayCount >= 2}
      <div class="combo-display">
        <div class="combo-number">{currentComboDisplayCount}</div>
        <div class="combo-text">HITS!</div>
        <div class="wpm-text">{currentWpm} WPM</div>
      </div>
    {/if}
    
    <div class="player-visual-wrapper" class:attacking={player.isAttacking} class:hit={player.isHit}>
      <div class="player-sprite">🛡️</div>
      <div class="player-tag">PLAYER</div> 
      <div class="stat-row">
        {t('hp')}: {player.hp} / {player.maxHp}
        {#if player.lastDamage > 0}
          <span class="damage-pop">-{player.lastDamage}</span>
        {/if}
      </div>
      <div class="bar-container"><div class="hp-fill" style="width: {(player.hp/player.maxHp)*100}%"></div></div>
      <div class="stat-row">{t('mp')}: {player.mp} / {player.maxMp}</div>
      <div class="bar-container"><div class="mp-fill" style="width: {(player.mp/player.maxMp)*100}%"></div></div>
      <div class="stat-row">{t('atb')}: {Math.floor(player.atb)}%</div>
      <div class="bar-container atb"><div class="atb-fill" style="width: {player.atb}%"></div></div>
    </div>
  </div>
</div>

<style>
  .battle-scene {
    flex: 3;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background: #000;
    perspective: 500px;
    position: relative;
  }
  .enemies-row {
    display: flex;
    width: 100%;
    justify-content: space-around;
    margin-bottom: 20px;
    position: relative; 
    align-items: center;
    min-height: 240px;
  }
  .monster-wrapper { text-align: center; transition: all 0.5s; }
  .monster-wrapper.dead { opacity: 0; transform: scale(0.5) translateY(100px); }
  .monster-wrapper.hit, .player-visual-wrapper.hit { animation: shake 0.15s infinite, flash 0.15s ease-out; }
  .monster-wrapper.attacking { animation: lunge 0.4s ease-out; z-index: 5; }
  .monster-wrapper.target-success { animation: target-flash 0.2s ease-out; }

  /* 獨立選擇框樣式 */
  .selection-frame {
    position: absolute;
    width: 160px;
    height: 230px;
    border: 2px solid yellow;
    box-shadow: 0 0 15px rgba(255, 255, 0, 0.5), inset 0 0 10px rgba(255, 255, 0, 0.3);
    top: 50%;
    transform: translate(-50%, -50%); /* 改為 -50% 達成真正的垂直置中 */
    transition: left 0.25s cubic-bezier(0.175, 0.885, 0.32, 1.275);
    pointer-events: none;
    z-index: 10;
    border-radius: 4px;
    background: rgba(255, 255, 0, 0.05);
  }

  .player-section {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .player-visual-wrapper { text-align: center; transition: transform 0.2s ease-out; }
  .player-visual-wrapper.attacking { transform: translateY(-40px) scale(1.1); }
  
  .player-sprite { font-size: 4rem; filter: drop-shadow(0 0 10px rgba(255,255,255,0.5)); margin-bottom: 5px; }
  .player-tag { font-size: 1.2rem; font-weight: bold; color: #fff; margin-bottom: 8px; text-transform: uppercase; }

  .monster-name-display { 
    font-weight: bold; 
    color: var(--tier-color, #fff); 
    margin-bottom: 8px; 
    min-height: 2.5em; /* 調整高度以容納類族與名稱 */
    display: flex; 
    flex-direction: column; /* 垂直排列 */
    align-items: center; 
    justify-content: center; 
    padding: 0 5px; 
    text-align: center;
    line-height: 1.1; /* 縮小行間距 */
  }
  .name-race { font-size: 0.7rem; opacity: 0.8; text-transform: uppercase; }
  .name-affix { 
    font-size: 0.65rem; 
    color: #aaa; /* 詞綴使用較淡顏色 */
    word-break: break-all; /* 避免過長詞綴衝出邊框 */
  }
  .name-base { font-size: 0.9rem; margin-top: 2px; }

  .letter.correct { color: #fff; text-shadow: 0 0 5px #fff; }

  .stat-row { text-align: left; width: 100px; margin: 0 auto; font-size: 0.7rem; }
  .monster-sprite { font-size: 3rem; margin-bottom: 5px; filter: drop-shadow(0 0 5px var(--tier-color, transparent)); }
  
  .bar-container { width: 100px; height: 4px; background: #000; border: 1px solid #fff; margin: 4px auto; }
  .hp-fill, .atb-fill, .mp-fill { height: 100%; background: #fff; transition: width 0.3s; }

  /* 傷害數值彈出樣式 */
  .damage-pop {
    color: #ff4444;
    font-weight: bold;
    margin-left: 5px;
    position: absolute;
    animation: damage-float 1s forwards;
    pointer-events: none;
  }

  /* 連擊數顯示樣式 */
  .combo-display {
    position: absolute;
    left: -140px; /* 移至玩家左側 */
    top: 20px;
    display: flex;
    flex-direction: column;
    align-items: flex-end; /* 向玩家方向對齊 */
    font-style: italic; /* 傾斜字體 */
    transform: skewX(-15deg); /* 模擬格鬥遊戲標題斜度 */
    z-index: 50;
    white-space: nowrap; /* 防止文字換行 */
    pointer-events: none;
    animation: combo-sf-in 0.15s ease-out forwards;
  }

  .combo-number {
    font-size: 4.5rem;
    line-height: 0.8;
    color: #ffcc00;
    font-weight: 900;
    text-shadow: 4px 4px 0px #800000, 6px 6px 10px rgba(0,0,0,0.5);
  }

  .combo-text {
    font-size: 1.2rem;
    color: #fff;
    font-weight: bold;
    letter-spacing: 2px;
    text-shadow: 2px 2px 0px #000;
  }

  .wpm-text {
    font-size: 1rem;
    color: #3498db;
    font-weight: bold;
    margin-top: 5px;
    text-shadow: 2px 2px 0px #000;
  }

  @keyframes combo-sf-in {
    0% { transform: translateX(-50px) skewX(-15deg) scale(1.5); opacity: 0; }
    100% { transform: translateX(0) skewX(-15deg) scale(1); opacity: 1; }
  }

  @keyframes shake {
    0%, 100% { transform: translate(0, 0); }
    25% { transform: translate(2px, 0); }
    75% { transform: translate(-2px, 0); }
  }
  @keyframes damage-float {
    0% { opacity: 1; transform: translateY(0); }
    100% { opacity: 0; transform: translateY(-20px); }
  }
  @keyframes flash {
    0%, 100% { filter: invert(0); }
    50% { filter: invert(1); }
  }
  @keyframes lunge {
    0%, 100% { transform: translateY(0); }
    30% { transform: translateY(50px) scale(1.2); }
  }
  @keyframes target-flash {
    0%, 100% { transform: scale(1); filter: brightness(1); }
    50% { transform: scale(1.1); filter: brightness(3); }
  }
</style>