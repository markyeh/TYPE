<script>
  export let t;
  export let devMode;
  export let currentLanguage;
  export let gameScore;
  export let onToggleDev;
  export let onSetLanguage;
</script>

<header class="game-header">
  <div class="game-title">TYPE: Keyboard Adventurer</div>

  <div class="progress-section">
    <div class="progress-text">STAGE PROGRESS: {gameScore}%</div>
    <div class="progress-bar-container">
      <div class="progress-fill" style="width: {gameScore}%"></div>
      {#if gameScore >= 100}
        <div class="boss-ready">BOSS READY</div>
      {/if}
    </div>
  </div>

  <div class="header-controls">
    <button class="dev-toggle" class:active={devMode} on:click={onToggleDev}>
      {t('devMode')}: {devMode ? 'ON' : 'OFF'}
    </button>
    <button on:click={() => onSetLanguage('en')} class:active={currentLanguage === 'en'}>EN</button>
    <button on:click={() => onSetLanguage('zh')} class:active={currentLanguage === 'zh'}>ZH</button>
  </div>
</header>

<style>
  .game-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 20px;
    border-bottom: 1px solid #fff;
    background: #000;
    width: 100%;
    box-sizing: border-box;
    z-index: 10;
  }

  .game-title {
    font-size: 0.9rem;
    font-weight: bold;
    flex: 1;
  }

  .progress-section {
    flex: 2;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 0 20px;
  }

  .progress-text {
    font-size: 0.6rem;
    color: #888;
    margin-bottom: 4px;
    letter-spacing: 1px;
  }

  .progress-bar-container {
    width: 100%;
    max-width: 300px;
    height: 6px;
    background: #222;
    border: 1px solid #444;
    position: relative;
    overflow: hidden;
  }

  .progress-fill {
    height: 100%;
    background: linear-gradient(90deg, #3498db, #f1c40f);
    transition: width 0.4s cubic-bezier(0.4, 0, 0.2, 1);
    box-shadow: 0 0 10px rgba(241, 196, 15, 0.3);
  }

  .boss-ready {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(155, 89, 182, 0.6);
    color: #fff;
    font-size: 0.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    animation: blink 0.8s infinite;
  }

  @keyframes blink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.3; }
  }

  .header-controls {
    display: flex;
    gap: 10px;
    flex: 1;
    justify-content: flex-end;
  }

  .header-controls button {
    background: #000; color: #fff; border: 1px solid #fff; padding: 3px 8px;
    cursor: pointer; font-family: inherit;
    font-size: 0.7rem;
    transition: all 0.2s;
  }
  .header-controls button.active { background: #fff; color: #000; }
</style>