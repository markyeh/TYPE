<script>
  import { onMount, afterUpdate } from 'svelte';
  export let gameState;
  export let t;
  export let burstTimeLeft;
  export let burstMaxTime;
  export let burstBonusText; // 接收獎勵文字
  
  // Monkeytype 相關 props
  export let currentBurstWords = [];
  export let currentWordInput = "";
  export let currentWordIndex = 0;
  export let visibleWordsStartIndex = 0;
  export let wordsPerLine = 8;
  export let linesToDisplay = 2;
  export let comboCount;

  let inputEl;

  // 當進入 BURST 模式或更新時，強制聚焦輸入框
  afterUpdate(() => {
    if (gameState === 'BURST' && inputEl && document.activeElement !== inputEl) {
      inputEl.focus();
    }
  });
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
        <div class="word-display-container" style="--lines-to-display: {linesToDisplay}; --line-height-val: 1.3em; --row-gap: 0.3em;">
          <div class="word-lines" style="transform: translateY(calc(-1 * (var(--line-height-val) + var(--row-gap)) * {Math.floor(currentWordIndex / wordsPerLine)}))">
            {#each currentBurstWords as word, i (i)}
              <span class="word-item" 
                class:typed={i < currentWordIndex}
                class:current={i === currentWordIndex}
                class:pending={i > currentWordIndex}
              >
                {#if i === currentWordIndex}
                  {#each word.split('') as char, charIndex (charIndex)}
                    {#if charIndex === currentWordInput.length}
                      <span class="caret">|</span>
                    {/if}<span class="letter" 
                      class:correct={currentWordInput[charIndex]?.toLowerCase() === char} 
                      class:incorrect={currentWordInput[charIndex] !== undefined && currentWordInput[charIndex]?.toLowerCase() !== char} 
                      class:pending={currentWordInput[charIndex] === undefined}>{char}</span>{/each}
                  {#if currentWordInput.length >= word.length}
                    <span class="caret">|</span>
                  {/if}
                {:else}
                  {word}
                {/if}
                {#if i < currentBurstWords.length - 1 && (i + 1) % wordsPerLine !== 0}
                  <span class="word-space"> </span>
                {/if}
              </span>
            {/each}
          </div>
          <input 
            type="text" 
            class="hidden-input" 
            bind:this={inputEl}
            bind:value={currentWordInput} 
            autocomplete="off" 
            spellcheck="false" 
            maxlength={currentBurstWords[currentWordIndex]?.length || 0}
            disabled={gameState !== 'BURST'}
            on:keydown={(e) => { if(e.key === ' ') e.preventDefault(); }}
          />
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
  .letter.correct { color: #fff; }
  .letter.incorrect { color: #ff4d4d; font-weight: 900; }
  .letter.pending { color: #666; }
  .combo { font-size: 1rem; color: #fff; }

  .word-display-container {
    position: relative;
    height: calc((var(--line-height-val) * var(--lines-to-display)) + (var(--row-gap) * (var(--lines-to-display) - 1)) + 4px);
    overflow: hidden;
    font-size: 1.5rem;
    line-height: var(--line-height-val);
    margin-top: 10px;
    background: rgba(0,0,0,0.3);
    padding: 2px 15px;
    border-radius: 4px;
    box-sizing: border-box;
  }
  .word-lines {
    transition: transform 0.25s cubic-bezier(0.25, 1, 0.5, 1);
    display: flex;
    flex-wrap: wrap;
    gap: var(--row-gap) 0.6em;
    will-change: transform;
  }
  .word-item { 
    color: #666; 
    display: inline-flex;
    align-items: center;
  }
  .word-item.typed { color: #444; }
  .word-item.current { color: #fff; }
  .word-item.pending { color: #666; }

  .caret {
    color: #f1c40f;
    display: inline-block;
    width: 0;
    overflow: visible;
    position: relative;
    left: -1px;
    font-weight: normal;
    animation: caretBlink 1s infinite;
    pointer-events: none;
  }

  @keyframes caretBlink {
    0%, 100% { opacity: 1; }
    50% { opacity: 0; }
  }
  
  .hidden-input {
    position: absolute;
    opacity: 0;
    pointer-events: none;
    top: 0;
    left: 0;
  }
</style>