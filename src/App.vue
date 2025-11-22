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
          return { display, reading }
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

const handleKeyDown = (event) => {
  if (!isPlaying.value) {
    if (event.code === 'Space') {
      console.log('ゲームスタートなのです！')
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
      const firstChar = firstWordObj.reading[0]
      remainingPatterns.value = romajiMap[firstChar]
    }
    return
  }

  const nextInput = currentRomajiInput.value + event.key

  const nextPatterns = remainingPatterns.value.filter((pattern) => {
    return pattern.startsWith(nextInput)
  })

  if (nextPatterns.length > 0) {
    // 正しい入力
    totalKeystrokes.value++
    currentRomajiInput.value = nextInput
    remainingPatterns.value = nextPatterns

    userRomaji.value += event.key

    // 文字が完成したか
    if (remainingPatterns.value.includes(currentRomajiInput.value)) {
      currentCharIndex.value++
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

          const nextChar = nextWordObj.reading[0] // 判定は「読み」で
          remainingPatterns.value = romajiMap[nextChar]
        }
      } else {
        // 次の文字へ（単語の途中）
        const nextChar = currentWordObj.reading[currentCharIndex.value]
        remainingPatterns.value = romajiMap[nextChar]
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

  <div>タイプ数: {{ totalKeystrokes }} 回</div>
  <div>ミス数: {{ missCount }} 回</div>
  <div>タイム: {{ elapsedTime.toFixed(1) }} 秒</div>
</template>
