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
const isPlaying = ref(false) // ゲーム中フラグ

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
    const compound = char1 + char2 //例： "シ"+"ャ"="シャ"

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

  // --- パターンB: 通常の1文字 ---
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

const handleKeyDown = (event) => {
  if (!isPlaying.value) {
    if (event.code === 'Space') {
      console.log('ゲームスタート')
      isPlaying.value = true
      startTimer()

      // 初期化
      currentWordIndex.value = 0
      currentCharIndex.value = 0
      currentRomajiInput.value = ''
      userRomaji.value = ''
      totalKeystrokes.value = 0
      missCount.value = 0
      elapsedTime.value = 0

      // 最初の単語をセット
      const firstWordObj = wordList[0]
      question.value = firstWordObj.display
      questionReading.value = firstWordObj.reading

      // 最初の文字のパターン
      remainingPatterns.value = getNextPatterns(firstWordObj.reading, 0)
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
        currentWordIndex.value++

        // ゲーム終了チェック
        if (currentWordIndex.value >= wordList.length) {
          question.value = 'おしまい！お疲れ様でした！'
          questionReading.value = '' // 読み仮名は消す
          userRomaji.value = ''
          clearInterval(timerId)
          isPlaying.value = false // ゲーム終了状態
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
  <div id="question-display">
    <div style="font-size: 1.5rem; color: gray">{{ questionReading }}</div>
    <div style="font-size: 3rem">{{ question }}</div>
  </div>

  <div style="color: deepskyblue; font-weight: bold; font-size: 2rem; min-height: 3rem">
    {{ userRomaji }}
  </div>

  <br />
  <div style="color: gray; font-size: 0.8rem">
    最適打鍵数: {{ wordList[currentWordIndex]?.optimal }} 回
  </div>
  <div>タイプ数: {{ totalKeystrokes }} 回</div>
  <div>ミス数: {{ missCount }} 回</div>
  <div>タイム: {{ elapsedTime.toFixed(1) }} 秒</div>
</template>
