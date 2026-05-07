<script>
  export let t;
  export let devMode;
  export let currentLanguage;
  export let onToggleDev;
  export let onSetLanguage;
</script>

<header class="game-header">
  <div class="game-title">Path of Typing</div>

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