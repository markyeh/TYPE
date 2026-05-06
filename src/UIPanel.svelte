<script>
  export let gameState;
  export let t;
  export let burstTimeLeft;
  export let burstMaxTime;
  export let comboCount;
  export let currentBurstWord;
  export let burstInput;
  export let burstBonusText; // 接收獎勵文字
</script>

<div class="ui-panel">
  <div class="input-area">
    {#if gameState === 'ACTION_SELECT'}
      <div class="action-menu">
        <span class="key-hint">A</span> {t('attack')}
        <span class="key-hint">S</span> {t('skill')}
        <span class="key-hint">I</span> {t('item')}
        <span class="key-hint">B</span> {t('block')}
        <span class="key-hint">R</span> {t('run')}
      </div>
    {:else if gameState === 'BURST'}
      <div class="burst-container">
        <div class="timer-bar" style="width: {(burstTimeLeft/burstMaxTime)*100}%"></div>
        <div class="timer-text">
          {(burstTimeLeft / 1000).toFixed(1)}s
          {#if burstBonusText}
            <span class="bonus-hint">{burstBonusText}</span>
          {/if}
        </div>
        <div class="burst-word">
          {t('burstPrompt')}
          <span class="word-display">
            {#each currentBurstWord.split('') as char, i}
              <span class="letter" 
                class:correct={burstInput[i]?.toLowerCase() === char} 
                class:incorrect={burstInput[i] !== undefined && burstInput[i]?.toLowerCase() !== char} 
                class:pending={burstInput[i] === undefined}>{char}</span>
            {/each}
          </span>
        </div>
      </div>
    {/if}
  </div>
</div>

<style>
  .ui-panel {
    height: 180px; 
    display: flex; 
    flex-direction: column;
    flex-shrink: 0;
    padding: 10px 20px; gap: 10px; background: rgba(255,255,255,0.05); box-sizing: border-box;
  }
  .input-area { flex: 1; display: flex; justify-content: center; align-items: center; }
  .targeting-status { font-size: 1.2rem; color: #fff; animation: pulse 1.5s infinite; }
  @keyframes pulse { 0% { opacity: 0.5; } 50% { opacity: 1; } 100% { opacity: 0.5; } }
  .action-menu { font-size: 1.2rem; }
  .key-hint { background: #fff; color: #000; padding: 2px 6px; margin-right: 5px; }
  .burst-container { text-align: center; width: 100%; }
  .timer-bar { height: 4px; background: #fff; margin-bottom: 5px; transition: width 0.1s linear; }
  .timer-text { font-size: 0.9rem; color: #fff; margin-bottom: 2px; font-weight: bold; }
  .bonus-hint { color: #f1c40f; margin-left: 8px; animation: floatUp 0.8s ease-out forwards; position: absolute; }
  @keyframes floatUp { 0% { opacity: 1; transform: translateY(0); } 100% { opacity: 0; transform: translateY(-20px); } }
  .burst-word .word-display { font-size: 1.6rem; font-weight: bold; }
  .letter.correct { color: #fff; text-decoration: underline; }
  .letter.incorrect { color: #ff4d4d; text-decoration: underline; font-weight: 900; }
  .letter.pending { color: #666; }
  .combo { font-size: 1rem; color: #fff; }
</style>