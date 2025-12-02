<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { romajiMap } from './romajiMap.js'

// 状態変数たち
const question = ref('Now Loading...')
const showRomajiTable = ref(false)
const questionReading = ref('') // 読み仮名用
const userRomaji = ref('')
let txt = './wordlist.txt'
let wordList = []

// ゲーム進行用
const currentWordIndex = ref(0)
const currentCharIndex = ref(0)
const remainingPatterns = ref([])
const currentRomajiInput = ref('')

// リザルト用
const totaltype = ref(0)
const correcttype = ref(0)
const missCount = ref(0)
const elapsedTime = ref(0)
let timerId = null
let startTime = 0

// 理論値の合計
const totalOptimalKeystrokes = ref(0)
// ゲームの状態 ('start' | 'playing' | 'result')
const gameState = ref('start')

const smallKana = ['ァ', 'ィ', 'ゥ', 'ェ', 'ォ', 'ッ', 'ャ', 'ュ', 'ョ', 'ヮ', 'ヵ', 'ヶ']

// --- ロジック関数群 ---

// 読み仮名から、理論上の最短タイプ数を計算する関数(DP)
const calculateOptimalKeystrokes = (reading) => {
  const dp = new Array(reading.length + 1).fill(Infinity)
  dp[0] = 0
  for (let i = 0; i < reading.length; i++) {
    if (dp[i] === Infinity) continue
    const patterns = getNextPatterns(reading, i)
    patterns.forEach((pattern) => {
      const nextIndex = i + pattern.len
      const strokes = pattern.val.length
      if (dp[i] + strokes < dp[nextIndex]) {
        dp[nextIndex] = dp[i] + strokes
      }
    })
  }
  return dp[reading.length]
}

// カタカナ文と現在位置から、打てるパターンを計算する関数
const getNextPatterns = (reading, index) => {
  if (index >= reading.length) return []
  const patterns = []
  const char1 = reading[index]
  const char2 = reading[index + 1]

  // A. 拗音
  if (char2 && smallKana.includes(char2)) {
    const compound = char1 + char2
    if (romajiMap[compound]) {
      romajiMap[compound].forEach((r) => {
        patterns.push({ val: r, len: 2 })
      })
    }
  }

  // B. 促音（っ）
  if (char1 === 'ッ' && char2) {
    if (romajiMap[char2]) {
      const nextRomajiCandidates = romajiMap[char2]
      nextRomajiCandidates.forEach((nextRomaji) => {
        const consonant = nextRomaji[0]
        if (!['a', 'i', 'u', 'e', 'o', 'n'].includes(consonant)) {
          patterns.push({ val: consonant, len: 1 })
        }
      })
    }
  }

  // C. 撥音（ん）
  else if (char1 === 'ン') {
    patterns.push({ val: 'nn', len: 1 })
    patterns.push({ val: 'xn', len: 1 })

    if (char2 && romajiMap[char2]) {
      romajiMap[char2].forEach((nextRomaji) => {
        const nextHead = nextRomaji[0]
        // 母音、な行、や行 以外なら合体（持ち越し）
        if (!['a', 'i', 'u', 'e', 'o', 'n', 'y'].includes(nextHead)) {
          patterns.push({ val: 'n' + nextRomaji, len: 2 })
        }
      })
    }
  }

  // D. 通常の1文字
  else if (romajiMap[char1]) {
    romajiMap[char1].forEach((r) => {
      patterns.push({ val: r, len: 1 })
    })
  } else {
    patterns.push({ val: char1, len: 1 })
  }

  return patterns
}

// --- 初期化とイベントハンドラ ---

onMounted(() => {
  fetch(txt)
    .then((response) => response.text())
    .then((data) => {
      const list = data
        .split(/\n/)
        .map((word) => word.trim())
        .filter((word) => word.length > 0)
        .map((line) => {
          const parts = line.split(',')
          const display = parts[0]
          const reading = parts[1] || parts[0]
          const optimal = calculateOptimalKeystrokes(reading)
          return { display, reading, optimal }
        })
      wordList = list
      question.value = 'Press Space to Start'
    })
    .catch((error) => {
      console.error('ファイルが読み込めません:', error)
      question.value = 'お題が見つかりません'
    })

  window.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
})

const startTimer = () => {
  startTime = Date.now()
  timerId = setInterval(() => {
    const now = Date.now()
    elapsedTime.value = (now - startTime) / 1000
  }, 10)
}

// --- メインロジック ---

const handleKeyDown = (event) => {
  if (gameState.value === 'start') {
    if (event.code === 'Tab') {
      event.preventDefault()
      showRomajiTable.value = !showRomajiTable.value
      return
    }
    if (event.code === 'Space') {
      showRomajiTable.value = false
      gameState.value = 'playing'
      startTimer()

      // 初期化
      currentWordIndex.value = 0
      currentCharIndex.value = 0
      currentRomajiInput.value = ''
      userRomaji.value = ''
      totaltype.value = 0
      correcttype.value = 0
      missCount.value = 0
      elapsedTime.value = 0
      totalOptimalKeystrokes.value = 0

      const firstWordObj = wordList[0]
      question.value = firstWordObj.display
      questionReading.value = firstWordObj.reading
      remainingPatterns.value = getNextPatterns(firstWordObj.reading, 0)
    }
    return
  }

  if (gameState.value === 'result') {
    if (event.code === 'Space') {
      gameState.value = 'start'
      question.value = 'Press Space to Start'
    }
    return
  }

  const nextInput = currentRomajiInput.value + event.key
  const nextPatterns = remainingPatterns.value.filter((pattern) => {
    return pattern.val.startsWith(nextInput)
  })

  if (nextPatterns.length > 0) {
    // 正解
    correcttype.value++
    totaltype.value++
    currentRomajiInput.value = nextInput
    remainingPatterns.value = nextPatterns
    userRomaji.value += event.key

    const matched = remainingPatterns.value.find((p) => p.val === currentRomajiInput.value)

    if (matched) {
      currentCharIndex.value += matched.len
      currentRomajiInput.value = ''

      const currentWordObj = wordList[currentWordIndex.value]

      if (currentCharIndex.value >= currentWordObj.reading.length) {
        // 単語完了
        totalOptimalKeystrokes.value += currentWordObj.optimal
        currentWordIndex.value++

        if (currentWordIndex.value >= wordList.length) {
          // ゲーム終了
          clearInterval(timerId)
          gameState.value = 'result'
        } else {
          // 次の単語へ
          userRomaji.value = ''
          currentCharIndex.value = 0
          const nextWordObj = wordList[currentWordIndex.value]
          question.value = nextWordObj.display
          questionReading.value = nextWordObj.reading
          remainingPatterns.value = getNextPatterns(nextWordObj.reading, 0)
        }
      } else {
        // 次の文字へ
        remainingPatterns.value = getNextPatterns(currentWordObj.reading, currentCharIndex.value)
      }
    }
  } else {
    // ミスタイプ
    totaltype.value++
    missCount.value++
  }
}
</script>

<template>
  <div class="container">
    <div v-if="gameState === 'start'" class="screen start-screen">
      <h1 class="title">Siranaiタイピング</h1>

      <div v-if="!showRomajiTable">
        <p class="blink">Press Space to Start</p>
        <p style="font-size: 1rem; color: #888; margin-top: 1rem">(Press Tab to see Romaji Map)</p>
      </div>

      <div v-else class="romaji-table-overlay">
        <h2>対応表 (Romaji Map)</h2>
        <div class="table-grid">
          <div v-for="(patterns, kana) in romajiMap" :key="kana" class="table-item">
            <div class="kana">{{ kana }}</div>
            <div class="romaji">{{ patterns.join(', ') }}</div>
          </div>
        </div>
        <p style="margin-top: 1rem; color: #aaa">Press Tab to Close</p>
      </div>
    </div>

    <div v-else-if="gameState === 'playing'" class="screen play-screen">
      <div class="hud">
        <span>Time: {{ elapsedTime.toFixed(1) }}</span>
        <span>TotalType: {{ totaltype }}</span>
      </div>

      <div id="question-display">
        <div class="reading">{{ questionReading }}</div>
        <div class="kanji">{{ question }}</div>
        <div class="input-feedback">{{ userRomaji }}</div>
      </div>
    </div>

    <div v-else-if="gameState === 'result'" class="screen result-screen">
      <h2>Result</h2>

      <div class="result-box">
        <div class="result-item">
          <span class="label">タイム</span>
          <span class="value">{{ elapsedTime.toFixed(2) }} 秒</span>
        </div>

        <hr style="border: 0; border-top: 2px dashed #ff8f3f; margin: 20px 0; opacity: 0.5" />

        <div class="result-item">
          <span class="label">総タイプ数</span>
          <span class="value">{{ totaltype }} 打</span>
        </div>

        <div class="result-item">
          <span class="label">正タイプ率</span>
          <span
            class="value"
            :class="{
              good: totaltype > 0 && correcttype === totaltype,
              bad: totaltype === 0 || correcttype !== totaltype,
            }"
          >
            {{ totaltype > 0 ? Math.round((correcttype / totaltype) * 100) : 0 }} %
          </span>
        </div>

        <hr style="border: 0; border-top: 2px dashed #ff8f3f; margin: 20px 0; opacity: 0.5" />

        <div class="result-item">
          <span class="label">有効入力</span>
          <span class="value">{{ correcttype }} 打</span>
        </div>

        <div class="result-item">
          <span class="label">最短入力</span>
          <span class="value">{{ totalOptimalKeystrokes }} 打</span>
        </div>

        <div class="result-item">
          <span class="label">ロス</span>
          <span
            class="value"
            :class="{
              good: correcttype - totalOptimalKeystrokes <= 0,
              bad: correcttype - totalOptimalKeystrokes > 0,
            }"
          >
            {{
              correcttype - totalOptimalKeystrokes <= 0
                ? 'Perfect!!'
                : '+' + (correcttype - totalOptimalKeystrokes) + ' 打'
            }}
          </span>
        </div>
      </div>

      <p class="blink retry-msg">Press Space to Retry</p>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Zen+Maru+Gothic:wght@500&display=swap');

/* 全体 */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;

  background-color: #ecfff5;
  background-image: radial-gradient(#23b3cd 1px, transparent 1px);
  background-size: 20px 20px;

  color: #ff8f3f;
  font-family: 'Zen Maru Gothic', sans-serif;
  text-align: center;
}

/* 各画面共通 */
.screen {
  background: #ffffffe6;
  padding: 40px;
  border-radius: 20px;
  box-shadow: 0 10px 25px #23b3cd26;
  width: 90%;
  max-width: 700px;
  backdrop-filter: blur(5px);
}

/* タイトル */
.title {
  font-size: 3.5rem;
  margin-bottom: 1.5rem;
  color: #ff8f3f;
  text-shadow:
    1px 1px 0px #ffffff,
    2px 2px 0px #23b3cd;
}

/* 点滅アニメーション */
.blink {
  animation: blink 1s infinite;
  font-size: 1.5rem;
  margin-top: 2rem;
  color: #23b3cd;
}
@keyframes blink {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}

/* プレイ画面 */
.hud {
  display: flex;
  justify-content: space-between;
  font-size: 1.2rem;
  margin-bottom: 3rem;
  border-bottom: 2px dashed #ff8f3f;
  padding-bottom: 10px;
}
.reading {
  font-size: 1.5rem;
  color: #aaa;
  margin-bottom: 0.5rem;
}
.kanji {
  font-size: 4rem;
  font-weight: bold;
  margin-bottom: 1rem;
}
.input-feedback {
  font-size: 2.5rem;
  font-weight: bold;
  color: #23b3cd;
  min-height: 3rem;
}

/* リザルト画面 */
.result-box {
  background: #23b4cd31;
  padding: 2rem;
  border-radius: 10px;
  margin: 2rem auto;
  width: 40%;
}
.result-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 1rem;
  font-size: 1.5rem;
}
.good {
  color: #00c73c;
}
.bad {
  color: #ff3366;
}
.retry-msg {
  color: #23b3cd;
}

/* ローマ字表の枠組み */
.romaji-table-overlay {
  background: #ffffffea;
  border: 3px solid #ff8f3f;
  border-radius: 15px;
  padding: 20px;
  max-height: 60vh;
  overflow-y: auto;
  width: 90%;
  margin: 0 auto;
  box-shadow: 0 0 15px #5d4037d7;
  color: #5d4037;

  /* 画面の上に表示する */
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 100;
}

/* グリッドレイアウト */
.table-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 10px;
  text-align: center;
}

.table-item {
  background: #ecfff5;
  padding: 8px;
  border-radius: 8px;
  border: 1px solid #23b3cd;
}

.kana {
  color: #ff8f3f;
  font-weight: bold;
  font-size: 1.2rem;
}

.romaji {
  font-size: 0.9rem;
  color: #23b3cd;
  font-family: sans-serif;
}

/*スクロールバー*/
.romaji-table-overlay::-webkit-scrollbar {
  width: 10px;
}
.romaji-table-overlay::-webkit-scrollbar-track {
  background: #fff;
  border-radius: 5px;
}
.romaji-table-overlay::-webkit-scrollbar-thumb {
  background: #ff8f3f;
  border-radius: 5px;
}
</style>
