<script setup lang="ts"></script>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { romajiMap } from './romajiMap.js'
const question = ref('Now Loading...')
let txt = './wordlist.txt'
let wordList = []

const currentWordIndex = ref(0)
const currentCharIndex = ref(0)
const remainingPatterns = ref([])
const currentRomajiInput = ref('')
const totalKeystrokes = ref(0)

const handleInput = (e) => {
  const inputText = e.target.value

  console.log('今、入力された文字:', inputText)
}

onMounted(() => {
  fetch(txt)
    .then((response) => {
      return response.text()
    })
    .then((data) => {
      const list = data
        .split(/\n/)
        .map((word) => word.trim())
        .filter((word) => word.length > 0)
      wordList = list
      question.value = wordList[0]

      currentWordIndex.value = 0
      currentCharIndex.value = 0
      currentRomajiInput.value = ''

      const firstChar = wordList[0][0]

      if (romajiMap[firstChar]) {
        remainingPatterns.value = romajiMap[firstChar]
      } else {
        // マップにない文字（記号とか）なら、その文字そのものを正解とする
        remainingPatterns.value = [firstChar]
      }

      console.log('1文字目の正解パターン:', remainingPatterns.value)
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

// romajiMap を使って、カタカナ文字列をローマ字候補の配列に変換
const convertRomaji = (word) => {
  const result = []

  for (let i = 0; i < word.length; i++) {
    const char = word[i]

    const candidates = romajiMap[char]

    if (candidates) {
      result.push(candidates)
    } else {
      result.push([char])
    }
  }

  return result
}

const handleKeyDown = (event) => {
  console.log('押されたキー:', event.key)

  const nextInput = currentRomajiInput.value + event.key

  const nextPatterns = remainingPatterns.value.filter((pattern) => {
    return pattern.startsWith(nextInput)
  })

  if (nextPatterns.length > 0) {
    console.log('次の候補:', nextPatterns)
    totalKeystrokes.value++

    currentRomajiInput.value = nextInput
    remainingPatterns.value = nextPatterns

    //正解処理
    if (remainingPatterns.value.includes(currentRomajiInput.value)) {
      currentCharIndex.value++
      currentRomajiInput.value = ''

      // 単語が完成したかチェック
      if (currentCharIndex.value >= wordList[currentWordIndex.value].length) {
        console.log('次の単語へ')
        currentWordIndex.value++
        if (currentWordIndex.value >= wordList.length) {
          console.log('おしまい')
          question.value = 'おしまい！お疲れ様でした！'
        } else {
          currentCharIndex.value = 0
          const nextWord = wordList[currentWordIndex.value]
          question.value = nextWord
          const nextChar = nextWord[0]
          remainingPatterns.value = romajiMap[nextChar]

          console.log('次の単語をセット:', nextWord)
        }
      } else {
        const currentWord = wordList[currentWordIndex.value]
        const nextChar = currentWord[currentCharIndex.value]

        remainingPatterns.value = romajiMap[nextChar]
        console.log('次の文字の正解パターンをセット:', remainingPatterns.value)
      }
    }
  } else {
    console.log('違う')
  }
}
</script>

<template>
  <div id="question-display">
    {{ question }}
  </div>

  <br />

  <div>タイプ数: {{ totalKeystrokes }} 回</div>
</template>
