<script>
    import { onMount, onDestroy } from 'svelte';
    import Header from './Header.svelte';
    import LogViewer from './LogViewer.svelte';
    import BattleScene from './BattleScene.svelte';
    import ModalOverlay from './ModalOverlay.svelte';
    import UIPanel from './UIPanel.svelte';
  
    // --- 遊戲資料定義 ---
    let gameConfig = {}; // 儲存遊戲配置參數
    let dictionaries = {};
    let monsterDb = null; // 儲存從 monster_db.json 讀取的資料
    let isLoaded = false;

    let gameState = 'BATTLE'; // BATTLE (ATB), ACTION_SELECT, TARGETING, BURST, GAME_OVER
    let previousState = 'BATTLE'; // 用於暫停後恢復
    let currentLanguage = 'en'; // 預設語言為英文
    let showLog = false; // 預設關閉日誌
    let gameScore = 0; // 當前地圖進度分數 (0-100)
    let isBossMode = false; // 是否進入 Boss 房
    let showComboDisplay = false; // 控制連擊數顯示 (這是 UI 狀態，非配置)
    let currentComboDisplayCount = 0; // 當前連擊數
    let showHelp = false; // 幫助說明顯示狀態
    let devMode = false; // Dev 模式（無敵狀態），從 config 載入，先給個預設值
    let logHistory = []; // 儲存歷史訊息
    let logContainer; // 用於控制捲動

    let gameStartTime = Date.now();
    let totalSuccessfulWords = 0;
    let finalResults = { time: 0, wpm: 0 };
    let bossTimer = null; // 用於追蹤 Boss 生成的定時器

    // 修改：支援多片段日誌，實現局部高亮
    function addLog(content, tier = 'system', highlight = null) {
      let segments = [];
      
      if (tier === 'system') {
        segments.push({ text: content, type: 'system' });
      } else if (highlight && content.includes(highlight)) {
        // 使用正規表達式切分字串，保留分隔符號（即 highlight 本身）
        const escapedHighlight = highlight.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        const parts = content.split(new RegExp(`(${escapedHighlight})`, 'g'));
        
        parts.forEach(part => {
          if (part === highlight) {
            segments.push({ text: part, type: tier }); // 怪物名稱使用階級顏色
          } else if (part) {
            segments.push({ text: part, type: 'normal' }); // 其他文字使用白色
          }
        });
      } else {
        segments.push({ text: content, type: tier === 'info' ? 'normal' : 'system' });
      }

      const newEntry = { segments, id: Date.now() + Math.random() };
      logHistory = [...logHistory, newEntry].slice(-50);
    }
    
    let player = {
      hp: 100, maxHp: 100, 
      mp: 50, maxMp: 100, // 初始值，將在 onMount 中被 gameConfig 覆寫
      atb: 0, speed: 0.8, // 初始值，將在 onMount 中被 gameConfig 覆寫
      isHit: false, isAttacking: false,
      selectedAction: null,
      lastDamage: 0
    };
  
    let enemies = [];
  
    let selectedMonsterId = null;
    let lastLoggedMonsterId = null;

    // 自動鎖定目標邏輯：
    // 當目前選中的怪物不存在、已死亡，或者怪物列表更新時，自動選擇 HP 最低的活著怪物
    $: if (isLoaded && enemies.length > 0) {
      const aliveEnemies = enemies.filter(e => e.hp > 0);
      const currentSelected = enemies.find(e => e.id === selectedMonsterId);
      if (aliveEnemies.length > 0 && (!currentSelected || currentSelected.hp <= 0)) {
        selectedMonsterId = aliveEnemies.reduce((prev, curr) => (prev.hp < curr.hp ? prev : curr)).id;
      }
    }

    // 當鎖定目標變動或進入行動選擇時，在 Log 顯示完整名稱
    $: if (gameState === 'ACTION_SELECT' && selectedMonsterId !== lastLoggedMonsterId) {
      const target = enemies.find(e => e.id === selectedMonsterId);
      if (target) {
        const fullName = target[`name_${currentLanguage}`] || target.name_en;
        addLog(t('targetLocked', fullName), target.wordType, fullName);
        lastLoggedMonsterId = selectedMonsterId;
      }
    } else if (gameState !== 'ACTION_SELECT') {
      lastLoggedMonsterId = null;
    }

    // Monkeytype 模式相關狀態
    let currentBurstWords = []; 
    let currentWordInput = ""; 
    let currentWordIndex = 0;
    let visibleWordsStartIndex = 0;
    let burstTimeLeft = 0;
    let burstBonusText = ""; // 新增：獎勵時間提示文字
    let burstMaxTime; // 從 config 載入
    let comboDisplayTimeout; // 用於控制連擊數顯示的定時器
    let comboCount = 0;
    let burstStartTime = 0;
    let currentWpm = 0;
    let currentTarget = null;
    let burstInterval;

    // --- 翻譯資料 ---
    const translations = {
        en: {
            gameLogInitializing: "Initializing word library...",
            gameLogLoaded: "Word library loaded! Waiting for ATB to fill...",
            gameLogError: "Error: Failed to load word library, please refresh the page.",
            playerTurn: "It's your turn! A:Attack, S:Skill, I:Item, B:Block, R:Run",
            enemyAttackLog: (enemyName, damage) => `${enemyName} attacks, dealing ${damage} damage!`,
            playerDefeated: "Defeated - Game Over",
            gamePaused: "Game Paused",
            continueGame: "Continue Game (ESC)",
            restartGame: "Restart Game",
            victory: "Victory!",
            clearTime: "Clear Time",
            avgWpm: "Average WPM",
            seconds: "s",
            mpNotEnough: (cost) => `Not enough MP! Skill requires ${cost} MP.`,
            skillActivated: "Skill activated! Combo damage increased by 2x!",
            targetingPrompt: "Enter the word above the monster to target...",
            devMode: "Dev Mode",
            burstPrompt: "TYPE:",
            burstPlaceholder: "Type fast!",
            comboEnd: (comboCount) => `Combo ended! Total ${comboCount} combos completed.`,
            targetLocked: (name) => `Target locked: ${name}`,
            enemyDefeated: (enemyName) => `${enemyName} was defeated!`,
            gameRestarted: "Game restarted!",
            commonWords: "Common Words",
            words: "words",
            attack: "Attack",
            skill: "Skill",
            item: "Item",
            block: "Block",
            run: "Run",
            hp: "HP",
            mp: "MP",
            atb: "ATB",
            helpTitle: "HOTKEYS / HELP",
            helpLog: "[0] Toggle Log Window",
            helpHelp: "[?] Toggle Help Overlay",
            helpEsc: "[Esc] Menu / Pause",
            helpTarget: "[H/L] or [Arrows] Select Target",
            helpAttack: "[A] Attack / [S] Skill (30MP)",
            helpItem: "[I] Item / [B] Block / [R] Run",
            helpClose: "Press [?] or [Esc] to return",
            skillPlaceholder: "You cast a skill!",
            blockAction: "You entered a defensive stance.",
            runAction: "Successfully escaped!",
            combo: "combo"
        },
        zh: {
            gameLogInitializing: "正在初始化單字庫...",
            gameLogLoaded: "單字庫載入完成！等待行動條充滿...",
            gameLogError: "錯誤：無法載入單字庫，請重新整理頁面。",
            playerTurn: "輪到你了！A:攻擊, S:技能, I:道具, B:防禦, R:逃跑",
            enemyAttackLog: (enemyName, damage) => `${enemyName} 發動攻擊，造成 ${damage} 點傷害！`,
            playerDefeated: "戰敗 - 遊戲結束",
            gamePaused: "遊戲暫停",
            continueGame: "繼續遊戲 (ESC)",
            restartGame: "重新開始遊戲",
            victory: "戰鬥勝利！",
            clearTime: "通關時間",
            avgWpm: "平均速度",
            seconds: "秒",
            mpNotEnough: (cost) => `MP 不足！發動技能需要 ${cost} 點 MP。`,
            skillActivated: "發動技能！連擊傷害提升為 2 倍！",
            targetingPrompt: "請輸入怪物頭上的單字來鎖定目標...",
            devMode: "開發者模式",
            burstPrompt: "輸入:",
            burstPlaceholder: "快打！",
            comboEnd: (comboCount) => `連擊結束！總共完成了 ${comboCount} 次連擊。`,
            targetLocked: (name) => `已鎖定目標：${name}`,
            enemyDefeated: (enemyName) => `${enemyName} 被擊倒了！`,
            gameRestarted: "遊戲重新開始！",
            commonWords: "常用單字",
            words: "單字",
            attack: "攻擊",
            skill: "技能",
            item: "道具",
            block: "防禦",
            run: "逃跑",
            hp: "HP",
            mp: "MP",
            atb: "ATB",
            helpTitle: "熱鍵 / 說明",
            helpLog: "[0] 開啟/關閉日誌視窗",
            helpHelp: "[?] 顯示/隱藏此說明",
            helpEsc: "[Esc] 開啟選單 / 暫停",
            helpTarget: "[H/L] 或 [方向鍵] 切換目標",
            helpAttack: "[A] 攻擊 / [S] 技能 (30MP)",
            helpItem: "[I] 道具 / [B] 防禦 / [R] 逃跑",
            helpClose: "按下 [?] 或 [Esc] 返回遊戲",
            skillPlaceholder: "你施放了一個技能！",
            blockAction: "你進入了防禦狀態。",
            runAction: "逃跑成功！",
            combo: "連擊"
        }
    };

    function t(key, ...args) {
        const text = translations[currentLanguage][key];
        if (typeof text === 'function') {
            return text(...args);
        }
        return text || `MISSING_TRANSLATION[${key}]`;
    }

    // Set initial gameLog based on default language
    addLog(t('gameLogInitializing'));
  
    onMount(async () => {
      await loadGameConfig(); // 確保 gameConfig 在其他依賴它的函數之前載入
      await loadAllDictionaries();
      
      // 初始化 devMode
      devMode = gameConfig.initialDevMode;

      // 使用 gameConfig 初始化玩家狀態
      player.hp = gameConfig.playerInitialHp;
      player.maxHp = gameConfig.playerInitialHp;
      player.mp = gameConfig.playerInitialMp;
      player.maxMp = gameConfig.playerInitialMp;
      player.speed = gameConfig.playerSpeed; // 從 config 載入玩家速度
      burstMaxTime = gameConfig.burstMaxTime; // 初始化 burstMaxTime
      player = player; // 強制觸發 player 物件的反應性更新
      spawnMonsters();
      const timer = setInterval(() => {
        if (isLoaded && gameState !== 'PAUSED' && gameState !== 'GAME_OVER') { // 暫停或結束時停止 ATB
          updateATB();
        }
      }, 50);
      return () => {
        clearInterval(timer);
        if (burstInterval) clearInterval(burstInterval);
      };
    });

    async function loadAllDictionaries() {
      try {
        const tiers = ['white', 'magic', 'rare', 'unique'];
        const fetchPromises = tiers.map(tier => {
          const path = gameConfig.wordDictionaries[tier]; // 從 gameConfig 讀取檔案路徑
          return fetch(path).then(res => {
            if (!res.ok) {
              throw new Error(`File not found: ${path} (HTTP ${res.status}). Please ensure the file exists in public/data/`);
            }
            return res.json();
          });
        });
        
        const results = await Promise.all(fetchPromises);
        
        tiers.forEach((tier, index) => {
          const data = results[index];
          let wordList = [];

          // 相容兩種格式：直接是陣列，或是包含 words 屬性的物件
          if (Array.isArray(data)) {
            wordList = data;
          } else if (data && Array.isArray(data.words)) {
            wordList = data.words;
          }

          if (wordList.length > 0) {
            dictionaries[tier] = wordList.map(w => w.toLowerCase());
          } else {
            dictionaries[tier] = [];
          }
        });

        // 載入怪物數據庫
        const monsterRes = await fetch('data/monster_db.json');
        if (monsterRes.ok) {
            monsterDb = await monsterRes.json();
        }

        isLoaded = true;
        addLog(t('gameLogLoaded'));
      } catch (error) {
        console.error("Failed to load dictionaries:", error);
        addLog(t('gameLogError'));
      }
    }
    
    async function loadGameConfig() {
      const res = await fetch('data/game_config.json');
      gameConfig = await res.json();
    }

    // 隨機決定怪物階級 (60% White, 30% Magic, 10% Rare)
    function getRandomTier() {
      const rand = Math.random();
      if (rand < 0.6) return 'white';
      if (rand < 0.9) return 'magic';
      return 'rare';
    }

    // 封裝單一怪物生成邏輯
    function generateMonster(id, tier) {
      const pool = dictionaries[tier] || ['error'];
      const randomWord = pool[Math.floor(Math.random() * pool.length)];

      const allAffixes = monsterDb ? [...monsterDb.affixes.prefixes, ...monsterDb.affixes.suffixes] : [];
      const tierConfig = gameConfig.monsterTiers ? gameConfig.monsterTiers[tier] : null;
      let name_en = tier.toUpperCase(), name_zh = tier.toUpperCase(), icon = tierConfig?.icon || '❓', nameParts = null;

      if (monsterDb) {
        const race = monsterDb.races[Math.floor(Math.random() * monsterDb.races.length)];
        const basePool = monsterDb.monster_base_names[race.id];
        const base = basePool[Math.floor(Math.random() * basePool.length)];
        let affixCount = gameConfig.monsterTiers[tier].affixCount; // 從 config 讀取詞綴數量
        const pickedAffixes = [...allAffixes].sort(() => 0.5 - Math.random()).slice(0, affixCount);
        
        const prefixEn = pickedAffixes.map(a => a.en).join(' ');
        const prefixZh = pickedAffixes.map(a => a.zh).join('');

        name_en = `${race.en}: ${prefixEn ? prefixEn + ' ' : ''}${base.en}`;
        name_zh = `${race.zh}: ${prefixZh}${base.zh}`;
        icon = base.icon;
        nameParts = {
          race: { en: race.en, zh: race.zh },
          affixes: { en: prefixEn, zh: prefixZh },
          base: { en: base.en, zh: base.zh }
        };
      }

      return {
        id, name_en, name_zh, word: randomWord, icon,
        hp: tierConfig?.hp || 100, maxHp: tierConfig?.hp || 100,
        mp: gameConfig.monsterInitialMp || 0, maxMp: gameConfig.monsterInitialMp || 0,
        atb: Math.random() * 30, speed: tierConfig?.speed || 0.5,
        wordType: tier, isAttacking: false, isTargetSuccess: false, lastDamage: 0,
        nameParts: nameParts || { race: {en:'', zh:''}, affixes: {en:'', zh:''}, base: {en:tier.toUpperCase(), zh:tier.toUpperCase()} }
      };
    }

    function spawnMonsters() {
      if (isBossMode) {
        enemies = [generateMonster(1, 'unique')];
      } else { // 確保場上怪物數量不超過最大值
        const currentEnemyCount = enemies.filter(e => e.hp > 0).length;
        const enemiesToSpawn = gameConfig.maxEnemies - currentEnemyCount;
        for (let i = 0; i < enemiesToSpawn; i++) {
          const newId = Math.max(...enemies.map(e => e.id), 0) + 1; // 確保 ID 唯一
          enemies = [...enemies, generateMonster(newId, getRandomTier())];
        }
      }
    }
  
    function updateATB() {
      // 玩家 ATB
      if (player.atb < 100) {
        const chargeSpeed = devMode ? player.speed * 5 : player.speed;
        player.atb += chargeSpeed;
        if (player.atb >= 100) {
          player.atb = 100;
          gameState = 'ACTION_SELECT'; // 進入行動選擇狀態
          addLog(t('playerTurn'), 'info');
        }
      }
  
      // 敵人 ATB
      enemies = enemies.map(e => {
        if (e.hp > 0 && e.atb < 100) {
          e.atb += e.speed;
          if (e.atb >= 100) enemyAttack(e); // 改為傳遞整個物件
        }
        return e;
      });
    }
  
    function enemyAttack(attacker) {
      const damage = gameConfig.enemyBaseDamage;
      // 根據當前語言選擇名稱
      const currentName = attacker[`name_${currentLanguage}`] || attacker.name_en;

      // 觸發怪物攻擊特效
      attacker.isAttacking = true;
      enemies = enemies;
      
      if (!devMode) {
        player.hp = Math.max(0, player.hp - damage);
      }

      // 觸發玩家受傷特效
      player.isHit = true;
      player.lastDamage = damage;
      setTimeout(() => {
        attacker.isAttacking = false;
        player.isHit = false;
        player.lastDamage = 0;
        enemies = enemies;
        player = player;
      }, 1000); // 延長時間讓傷害數值顯示

      attacker.atb = 0;
      addLog(t('enemyAttackLog', currentName, damage), attacker.wordType, currentName);
      if (player.hp <= 0) gameState = 'GAME_OVER';
    }
  
    function togglePause() {
      if (gameState === 'GAME_OVER') return;
      if (gameState === 'PAUSED') {
        gameState = previousState;
        showHelp = false;
      } else {
        previousState = gameState;
        gameState = 'PAUSED'; // 進入暫停狀態
      }
    }

    function restartGame() {
      if (bossTimer) clearTimeout(bossTimer); // 強制清除尚未觸發的 Boss 生成
      gameState = 'BATTLE'; // 重置遊戲狀態
      gameScore = 0;
      isBossMode = false;
      enemies = []; // 清空現有怪物，防止 ID 累加與排版錯誤
      player = {
        hp: gameConfig.playerInitialHp, maxHp: gameConfig.playerInitialHp, 
        mp: gameConfig.playerInitialMp, maxMp: gameConfig.playerInitialMp,
        atb: 0, speed: gameConfig.playerSpeed, isHit: false, // 使用 config 中的速度
        isAttacking: false,
        selectedAction: null,
        lastDamage: 0
      };
      spawnMonsters();
      addLog(t('gameRestarted'));
      gameStartTime = Date.now();
      totalSuccessfulWords = 0;
      finalResults = { time: 0, wpm: 0 };
      selectedMonsterId = null; // spawnMonsters會重新設定
      comboCount = 0;
      currentWpm = 0;
      if (burstInterval) clearInterval(burstInterval);
      currentWordInput = ""; // 重置輸入框內容
      currentWordIndex = 0; // 重置當前單字索引
      visibleWordsStartIndex = 0; // 重置可見單字起始索引
    }

    function clearLogs() {
      logHistory = [];
    }

    // --- 鍵盤監聽 ---
    function handleGlobalKeyDown(e) {
      // 切換日誌 (熱鍵 0)
      if (e.key === '0') {
        showLog = !showLog;
        return;
      }

      // 切換幫助 (熱鍵 ?)
      if (e.key === '?' || (e.key === '/' && e.shiftKey)) {
        showHelp = !showHelp;
        if (showHelp && gameState !== 'PAUSED') {
          previousState = gameState;
          gameState = 'PAUSED';
        } else if (!showHelp && gameState === 'PAUSED' && previousState !== 'PAUSED') {
          gameState = previousState;
        }
        return;
      }

      if (e.key === 'Escape') {
        togglePause();
        return;
      }

      if (gameState === 'PAUSED' || gameState === 'GAME_OVER') return;

      if (gameState === 'ACTION_SELECT') {
        const key = e.key.toUpperCase(); // M 改為 S
        const aliveEnemies = enemies.filter(e => e.hp > 0);

        // 處理選擇移動 (h/l 或 左右方向鍵)
        if (aliveEnemies.length > 0) {
          if (key === 'H' || key === 'ARROWLEFT') {
            e.preventDefault();
            const currentIndex = aliveEnemies.findIndex(m => m.id === selectedMonsterId);
            const newIndex = (currentIndex - 1 + aliveEnemies.length) % aliveEnemies.length;
            selectedMonsterId = aliveEnemies[newIndex].id;
            return;
          }
          if (key === 'L' || key === 'ARROWRIGHT') {
            e.preventDefault();
            const currentIndex = aliveEnemies.findIndex(m => m.id === selectedMonsterId);
            const newIndex = (currentIndex + 1) % aliveEnemies.length;
            selectedMonsterId = aliveEnemies[newIndex].id;
            return;
          }
        }

        if (['A', 'S', 'I', 'B', 'R'].includes(key)) {
          if (key === 'S' && player.mp < 30) {
            addLog(t('mpNotEnough', gameConfig.skillMpCost), 'info');
            return;
          }
          e.preventDefault();
          player.selectedAction = key;

          if (key === 'A' || key === 'S') {
            const target = aliveEnemies.find(m => m.id === selectedMonsterId);
            if (target) {
              target.isTargetSuccess = true;
              enemies = enemies;

              if (key === 'S') {
                player.mp -= gameConfig.skillMpCost;
                player = player;
                addLog(t('skillActivated'), 'info');
              }

              currentTarget = target;
              setTimeout(() => {
                target.isTargetSuccess = false;
                enemies = enemies;
                startBurst();
              }, 200);
            }
          } else {
            executeQuickAction(key);
          }
        }
      } else if (gameState === 'BURST') {
        if (e.key === ' ') {
          e.preventDefault();
          // 檢查當前輸入是否匹配目標單字 (去除空白後比對)
          if (currentWordInput.trim().toLowerCase() === currentBurstWords[currentWordIndex].toLowerCase()) {
            handleCorrectWord();
          }
        }
      }
    }
  
    // --- 動作處理 ---
    function executeQuickAction(action) {
      if (action === 'B') addLog(t('blockAction'), 'info');
      if (action === 'S') {
        addLog(t('skillPlaceholder'), 'info');
      }
      if (action === 'R') addLog(t('runAction'), 'info');
      resetTurn();
    }
  
    function startBurst() {
      gameState = 'BURST';
      comboCount = 0;
      currentComboDisplayCount = 0;
      showComboDisplay = true; // Burst Mode 開始即顯示
      burstStartTime = Date.now();
      currentWpm = 0;
      currentWordInput = "";
      currentWordIndex = 0;
      visibleWordsStartIndex = 0;
      burstTimeLeft = gameConfig.burstMaxTime;
      generateBurstWords();
      
      burstInterval = setInterval(() => {
        if (gameState !== 'PAUSED') { // 暫停時停止倒數
          burstTimeLeft -= 100;
          
          // 即時計算 WPM (Words Per Minute)
          const elapsedSeconds = (Date.now() - burstStartTime) / 1000;
          if (elapsedSeconds > 0.5) { // 稍微延遲後再顯示以避免初始數值過大
            currentWpm = Math.round((comboCount / elapsedSeconds) * 60);
          }
        }
        if (burstTimeLeft <= 0 || currentWordIndex >= currentBurstWords.length) {
          endBurst();
        }
      }, 100);
    }
  
    function generateBurstWords() {
      currentBurstWords = [];
      addBurstWords(gameConfig.typingConfig?.initialWordPoolSize || 50);
    }

    function addBurstWords(count) {
      const poolKey = currentTarget?.wordType;
      const pool = dictionaries[poolKey];
      if (!pool || pool.length === 0) {
        if (currentBurstWords.length === 0) currentBurstWords = ["error"];
        return;
      }
      
      const newWords = Array.from({ length: count }, () => 
        pool[Math.floor(Math.random() * pool.length)].toLowerCase()
      );
      currentBurstWords = [...currentBurstWords, ...newWords];
    }
  
    function handleCorrectWord() {
        // 更新連擊數顯示
        // --- 每次成功連擊都增加時間 ---
        const bonusTime = gameConfig.comboBonusTime;
        burstTimeLeft = Math.min(gameConfig.burstMaxTime, burstTimeLeft + bonusTime);
        burstBonusText = `+${(bonusTime / 1000).toFixed(1)}s`;
        setTimeout(() => { burstBonusText = ""; }, 800);
        // --- 每次成功連擊都增加時間 ---

        currentComboDisplayCount = comboCount + 1; // comboCount 還沒加，所以這裡加1
        // 移除原有的 setTimeout 邏輯，讓顯示狀態由 endBurst 統一關閉

        comboCount++;
        totalSuccessfulWords++;
        currentWordInput = "";
        
        // 觸發玩家攻擊特效
        player.isAttacking = true;
        setTimeout(() => {
          player.isAttacking = false;
          player = player;
        }, 200);

        // 立即計算傷害數值，以便後續視覺特效與傷害套用使用
        let multiplier = player.selectedAction === 'S' ? 2 : 1;
        if (devMode) multiplier *= 10; // Dev 模式傷害加成
        const damagePerWord = gameConfig.baseDamagePerWord * multiplier;

        // 觸發視覺特效
        currentTarget.isHit = true;
        currentTarget.lastDamage = damagePerWord;
        const hitTarget = currentTarget;
        setTimeout(() => {
          hitTarget.isHit = false;
          hitTarget.lastDamage = 0;
          enemies = enemies; // 確保狀態清除後能更新 UI
        }, 1000);

        // 套用傷害
        currentTarget.hp = Math.max(0, currentTarget.hp - damagePerWord);
        enemies = enemies; // 觸發 Svelte 反應式更新

        if (currentTarget.hp <= 0) {
          // 分數邏輯
          const scoreMap = gameConfig.scoreMap;
          const points = scoreMap[currentTarget.wordType] || 0;
          gameScore = Math.min(100, gameScore + points);

          // 檢查是否進入 Boss 房
          if (gameScore >= 100 && !isBossMode) {
            isBossMode = true;
            setTimeout(() => {
              enemies = [generateMonster(1, 'unique')];
              selectedMonsterId = enemies[0].id;
              addLog("--- BOSS ROOM ---", 'unique'); // Boss 訊息
            }, gameConfig.enemyRespawnDelay); // 等待最後一隻怪物的死亡動畫
          }
          // 擊敗 Boss 結算邏輯 (檢查是否真的擊敗了 Unique 階級的 Boss)
          else if (currentTarget.wordType === 'unique') {
            const endTime = Date.now();
            const totalSeconds = Math.max(1, (endTime - gameStartTime) / 1000);
            finalResults = {
              time: totalSeconds,
              wpm: Math.round((totalSuccessfulWords / (totalSeconds / 60)))
            };
            gameState = 'VICTORY';
            clearInterval(burstInterval); // 手動清除，不進入 resetTurn
            showComboDisplay = false;
            return;
          }
          // 非 Boss 階段的重生邏輯
          else if (!isBossMode) {
            const deadId = currentTarget.id;
            setTimeout(() => {
              const idx = enemies.findIndex(e => e.id === deadId);
              if (idx !== -1) { // 如果怪物位置還存在，則重生
                enemies[idx] = generateMonster(deadId, getRandomTier()); // 在原位重生
                enemies = enemies;
              }
            }, 1000);
          }

          const aliveEnemies = enemies.filter(e => e.hp > 0);
          if (aliveEnemies.length > 0) {
            // 如果還有怪物，自動切換到下一個目標（HP 最低者）
            currentTarget = aliveEnemies.reduce((prev, curr) => (prev.hp < curr.hp ? prev : curr));
            selectedMonsterId = currentTarget.id; // 確保黃色框立即移動到新目標
            moveToNextWord();
          } else {
            endBurst();
          }
        } else {
          moveToNextWord();
        }
    }

    function moveToNextWord() {
      currentWordIndex++;
      checkScroll();
      // 檢查單字量是否足夠，少於 10 個就自動補貨
      if (currentBurstWords.length - currentWordIndex < 10) {
        addBurstWords(20);
      }
    }

    function checkScroll() {
      const wordsPerLine = gameConfig.typingConfig?.wordsPerLine || 8;
      if (currentWordIndex > 0 && currentWordIndex % wordsPerLine === 0) {
        visibleWordsStartIndex = currentWordIndex;
      }
    }
  
    function endBurst() {
      clearInterval(burstInterval);
      // 根據當前語言選擇名稱
      const targetName = currentTarget[`name_${currentLanguage}`] || currentTarget.name_en;
      let logText = t('comboEnd', comboCount);
      
      if (currentTarget.hp <= 0) {
        logText += ` ${t('enemyDefeated', targetName)}`;
      }
      addLog(logText, currentTarget.wordType, targetName);
      showComboDisplay = false; // Burst Mode 真正結束時關閉顯示
      resetTurn();
    }
  
    function resetTurn() {
      player.atb = 0;
      player.selectedAction = null;
      currentTarget = null; // 清除當前目標
      currentWordInput = "";
      currentBurstWords = [];
      currentWordIndex = 0;
      currentWpm = 0;
      visibleWordsStartIndex = 0;
      // 安全起見，確保結束時關閉
      showComboDisplay = false; 
      if (comboDisplayTimeout) clearTimeout(comboDisplayTimeout);
      gameState = 'BATTLE';
    }
  </script>
  
  <svelte:window on:keydown={handleGlobalKeyDown} />
  
  <Header 
    {t} {devMode} {currentLanguage} 
    {gameScore}
    onToggleDev={() => devMode = !devMode} 
    onSetLanguage={(lang) => currentLanguage = lang} 
  />

  <div class="game-viewport">
    {#if showLog}
      <LogViewer {logHistory} {gameState} onClear={clearLogs} />
    {/if}

    <main class="game-container">
      <div class="game-main-content">
        <ModalOverlay 
          {gameState} {t} 
          onTogglePause={togglePause} 
          onRestart={restartGame}
          {finalResults}
          {showLog}
          {showHelp}
          onToggleLog={() => showLog = !showLog}
          onToggleHelp={() => showHelp = !showHelp}
        />

        <BattleScene {enemies} {player} {gameState} {selectedMonsterId} {t} {currentLanguage} {showComboDisplay} {currentComboDisplayCount} {currentWpm} />

        <UIPanel 
          {gameState} {t} 
          {burstTimeLeft} {burstMaxTime} 
          {burstBonusText} {comboCount}
          {currentBurstWords} bind:currentWordInput {currentWordIndex} 
          {visibleWordsStartIndex}
          wordsPerLine={gameConfig.typingConfig?.wordsPerLine || 8}
          linesToDisplay={gameConfig.typingConfig?.linesToDisplay || 2}
        />
      </div>
    </main>
  </div>
  
  <style>
    :global(body) {
      background-color: #000;
      color: #fff;
      font-family: 'Courier New', monospace;
      margin: 0;
      display: flex;
      flex-direction: column;
      height: 100vh;
      overflow: hidden;
    }
  
    .game-viewport {
      flex: 1;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 20px;
      box-sizing: border-box;
      gap: 20px; /* 增加日誌與遊戲間的間距 */
    }

    .game-container {
      width: 850px;
      height: 650px;
      background: #000;
      border: 1px solid #fff;
      display: flex;
      flex-direction: column;
      overflow: hidden;
      position: relative;
    }

    .game-main-content {
      flex: 1;
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }
  </style>
  