<script>
  import { fade } from 'svelte/transition';
  export let t;
  export let hotkeys = {};
  export let onClose;

  function formatKey(key) {
    if (Array.isArray(key)) return key.join(' / ');
    return key;
  }
</script>

<div class="helper-overlay" transition:fade={{ duration: 200 }} on:click|self={onClose}>
  <div class="helper-content">
    <div class="helper-header">
      <h2>{t('helpTitle')}</h2>
      <button class="close-btn" on:click={onClose}>×</button>
    </div>
    
    <div class="help-section">
      <div class="help-row"><span>{t('helpTarget')}:</span> <kbd>{formatKey(hotkeys.moveUp)}</kbd> <kbd>{formatKey(hotkeys.moveDown)}</kbd></div>
      <div class="help-row"><span>{t('helpAttack')}:</span> <kbd>{hotkeys.skillSlots?.join(' / ')}</kbd></div>
      <div class="help-row"><span>{t('hp')}/{t('mp')} Flask:</span> <kbd>{hotkeys.hpFlask}</kbd> / <kbd>{hotkeys.mpFlask}</kbd></div>
      <div class="help-row"><span>{t('burstSkillCycle')}:</span> <kbd>{hotkeys.burstSkillCycle}</kbd></div>
      <div class="help-row"><span>{t('helpLog')}:</span> <kbd>{hotkeys.toggleLog}</kbd></div>
      <div class="help-row"><span>{t('skills')}:</span> <kbd>{hotkeys.toggleSkills}</kbd></div>
      <div class="help-row"><span>{t('gamePaused')}:</span> <kbd>{hotkeys.pause}</kbd></div>
    </div>

    <button class="return-btn" on:click={onClose}>{t('helpClose')}</button>
  </div>
</div>

<style>
  .helper-overlay {
    position: absolute;
    top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(0, 0, 0, 0.85);
    display: flex; justify-content: center; align-items: center;
    z-index: 2000;
  }
  .helper-content {
    background: #000;
    border: 1px solid #f1c40f;
    padding: 30px;
    width: 400px;
    box-shadow: 0 0 20px rgba(241, 196, 15, 0.2);
  }
  .helper-header {
    display: flex; justify-content: space-between; align-items: center;
    border-bottom: 1px solid #333; margin-bottom: 20px; padding-bottom: 10px;
  }
  .helper-header h2 { margin: 0; color: #f1c40f; font-size: 1.2rem; }
  .close-btn { background: none; border: none; color: #fff; font-size: 1.5rem; cursor: pointer; }
  
  .help-section { display: flex; flex-direction: column; gap: 12px; margin-bottom: 25px; }
  .help-row { display: flex; justify-content: space-between; align-items: center; font-size: 0.9rem; }
  .help-row span { color: #aaa; }
  
  kbd {
    background: #222;
    border: 1px solid #555;
    border-radius: 3px;
    padding: 2px 6px;
    font-family: inherit;
    font-size: 0.8rem;
    color: #fff;
    box-shadow: 0 2px 0 #000;
  }

  .return-btn {
    width: 100%;
    padding: 10px;
    background: #000;
    color: #f1c40f;
    border: 1px solid #f1c40f;
    cursor: pointer;
    font-family: inherit;
  }
  .return-btn:hover { background: #f1c40f; color: #000; }
</style>