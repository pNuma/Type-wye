<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { romajiMap } from './romajiMap.js'

// 状態変数たち
const question = ref('Now Loading...')
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
const totalKeystrokes = ref(0)
const missCount = ref(0)
const elapsedTime = ref(0)
let timerId = null
let startTime = 0

onMounted(() => {
  fetch(txt)
    .then((response) => response.text())
    .then((data) => {
      // データの読み込みと加工（漢字対応）
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

// タイマー機能
const startTimer = () => {
  startTime = Date.now()
  timerId = setInterval(() => {
    const now = Date.now()
    elapsedTime.value = (now - startTime) / 1000
  }, 10)
}

const smallKana = ['ァ', 'ィ', 'ゥ', 'ェ', 'ォ', 'ッ', 'ャ', 'ュ', 'ョ', 'ヮ', 'ヵ', 'ヶ']

//カタカナ文と現在位置から、打てるパターンを計算する関数
const getNextPatterns = (reading, index) => {
  if (index >= reading.length) return []

  const patterns = []
  const char1 = reading[index]
  const char2 = reading[index + 1]

  // 拗音
  if (char2 && smallKana.includes(char2)) {
    const compound = char1 + char2

    if (romajiMap[compound]) {
      romajiMap[compound].forEach((r) => {
        patterns.push({ val: r, len: 2 }) // 2文字進む
      })
    }
  }

  //促音
  if (char1 === 'ッ' && char2) {
    if (romajiMap[char2]) {
      const nextRomajiCandidates = romajiMap[char2]

      // 次の文字の子音を見る
      nextRomajiCandidates.forEach((nextRomaji) => {
        const consonant = nextRomaji[0]

        if (!['a', 'i', 'u', 'e', 'o', 'n'].includes(consonant)) {
          patterns.push({ val: consonant, len: 1 })
        }
      })
    }
  }

  //ンの入力
  else if (char1 === 'ン') {
    patterns.push({ val: 'nn', len: 1 })
    patterns.push({ val: 'xn', len: 1 })

    if (char2 && romajiMap[char2]) {
      romajiMap[char2].forEach((nextRomaji) => {
        const nextHead = nextRomaji[0]

        if (!['a', 'i', 'u', 'e', 'o', 'n', 'y'].includes(nextHead)) {
          patterns.push({ val: 'n' + nextRomaji, len: 2 })
        }
      })
    }
  }

  // 通常の1文字
  if (romajiMap[char1]) {
    romajiMap[char1].forEach((r) => {
      patterns.push({ val: r, len: 1 }) // 1文字進む
    })
  } else {
    // 記号など
    patterns.push({ val: char1, len: 1 })
  }

  return patterns
}

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

// ゲームの状態 ('start' | 'playing' | 'result')
const gameState = ref('start')

// 理論値の合計
const totalOptimalKeystrokes = ref(0)

const handleKeyDown = (event) => {
  if (gameState.value === 'start') {
    if (event.code === 'Space') {
      gameState.value = 'playing'
      startTimer()

      // 初期化
      currentWordIndex.value = 0
      currentCharIndex.value = 0
      currentRomajiInput.value = ''
      userRomaji.value = ''
      totalKeystrokes.value = 0
      missCount.value = 0
      elapsedTime.value = 0
      totalOptimalKeystrokes.value = 0

      // 最初の単語セット
      const firstWordObj = wordList[0]
      question.value = firstWordObj.display
      questionReading.value = firstWordObj.reading
      remainingPatterns.value = getNextPatterns(firstWordObj.reading, 0)
    }
    return
  }

  // 2. リザルト画面のとき（リトライ機能）
  if (gameState.value === 'result') {
    if (event.code === 'Space') {
      // もう一度スタート画面に戻るのです
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
    // 正しい入力
    totalKeystrokes.value++
    currentRomajiInput.value = nextInput
    remainingPatterns.value = nextPatterns

    userRomaji.value += event.key

    const matched = remainingPatterns.value.find((p) => p.val === currentRomajiInput.value)

    // 文字が完成したか
    if (matched) {
      currentCharIndex.value += matched.len
      currentRomajiInput.value = ''

      const currentWordObj = wordList[currentWordIndex.value] // 現在の単語オブジェクト

      // 単語が完成したか
      if (currentCharIndex.value >= currentWordObj.reading.length) {
        console.log('次の単語へ')
        // 単語の理論値を合計に足す
        totalOptimalKeystrokes.value += currentWordObj.optimal

        currentWordIndex.value++

        if (currentWordIndex.value >= wordList.length) {
          // ゲーム終了！
          clearInterval(timerId)
          gameState.value = 'result' // リザルト画面へ！
        } else {
          // 次の単語へ
          userRomaji.value = ''
          currentCharIndex.value = 0

          const nextWordObj = wordList[currentWordIndex.value]
          question.value = nextWordObj.display // 漢字
          questionReading.value = nextWordObj.reading // ふりがな

          remainingPatterns.value = getNextPatterns(nextWordObj.reading, 0)
        }
      } else {
        // 次の文字へ（単語の途中）
        remainingPatterns.value = getNextPatterns(currentWordObj.reading, currentCharIndex.value)
      }
    }
  } else {
    // ミスタイプ
    missCount.value++
  }
}
</script>

<template>
  <div class="container">
    <div v-if="gameState === 'start'" class="screen start-screen">
      <h1 class="title">しらないタイピング</h1>
      <p class="blink">Press Space to Start</p>
    </div>

    <div v-else-if="gameState === 'playing'" class="screen play-screen">
      <div class="hud">
        <span>Time: {{ elapsedTime.toFixed(1) }}</span>
        <span>Score: {{ totalKeystrokes }}</span>
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
          <span class="value">{{ elapsedTime.toFixed(2) }}秒</span>
        </div>
        <div class="result-item">
          <span class="label">あなたの入力数</span>
          <span class="value">{{ totalKeystrokes }}打</span>
        </div>
        <div class="result-item">
          <span class="label">理論上の最短</span>
          <span class="value">{{ totalOptimalKeystrokes }}打</span>
        </div>
        <div class="result-item">
          <span class="label">無駄打ち</span>
          <span class="value" :class="{ good: totalKeystrokes - totalOptimalKeystrokes <= 0 }">
            +{{ totalKeystrokes - totalOptimalKeystrokes }}
          </span>
        </div>
        <div class="result-item">
          <span class="label">ミスタイプ</span>
          <span class="value" :class="{ bad: missCount > 0 }">{{ missCount }}回</span>
        </div>
      </div>

      <p class="blink retry-msg">Press Space to Retry</p>
    </div>
  </div>
</template>

<style scoped>
/* 全体の箱：画面の真ん中に持ってくるのです */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #222; /* 黒に近いグレー */
  color: #fff;
  font-family: 'Courier New', monospace; /* プログラミングっぽいフォント */
  text-align: center;
}

/* 各画面共通 */
.screen {
  width: 100%;
  max-width: 800px;
}

/* タイトル */
.title {
  font-size: 4rem;
  margin-bottom: 2rem;
  color: #00ffcc; /* サイバーな緑 */
  text-shadow: 0 0 10px #00ffcc;
}

/* 点滅アニメーション */
.blink {
  animation: blink 1s infinite;
  font-size: 1.5rem;
  margin-top: 2rem;
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
  border-bottom: 1px solid #555;
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
  color: deepskyblue;
  min-height: 3rem; /* 文字がなくてもガタつかないように */
  text-shadow: 0 0 5px deepskyblue;
}

/* リザルト画面 */
.result-box {
  background: #333;
  padding: 2rem;
  border-radius: 10px;
  margin: 2rem auto;
  width: 80%;
}
.result-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 1rem;
  font-size: 1.5rem;
  border-bottom: 1px dashed #555;
}
.good {
  color: #00ffcc;
}
.bad {
  color: #ff3366;
}
.retry-msg {
  color: #aaa;
}
</style>
