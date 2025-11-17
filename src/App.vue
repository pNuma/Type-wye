<script setup lang="ts"></script>

<script setup> 
import { ref, onMounted } from 'vue';
import { romajiMap } from './romajiMap.js';
const question = ref('Now Loading...'); 
let txt = "./wordlist.txt";
let wordList = [];

const handleInput = (e) => {
  const inputText = e.target.value; 
  
  console.log("今、入力された文字:", inputText);
  }

onMounted(() => {
  fetch(txt)
    .then(response => {
      return response.text(); 
    })
    .then(data => {
      wordList = data.split(/\n/);
      question.value = wordList[0];
    })
    .catch(error => {
      console.error('ファイルが読み込めません:', error);
      question.value = 'お題が見つかりません';
    });
});


// romajiMap を使って、カタカナ文字列をローマ字候補の配列に変換
const convertRomaji = (word) => {
  const result = [];
  
  for (let i = 0; i < word.length; i++) {
    const char = word[i]; 
    
    const candidates = romajiMap[char];
    
    if (candidates) {
      result.push(candidates);
    } else {
      result.push([char]);
    }
  }
  
  return result;
};

</script>

<template>
  <div id="question-display">
    {{ question }} 
  </div>
  
  <br>
  
  入力欄<br>
  <input 
    type="text" 
    id="user-input"
    @input="handleInput" 
  >
</template>
