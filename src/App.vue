<script setup lang="ts"></script>

<script setup> 
import { ref, onMounted } from 'vue';
const question = ref('Now Loading...'); 
let txt = "./wordlist.txt";
let wordList = [];

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
</script>

<template>
  <div id="question-display">
      {{ question }}
  </div>
  <br>

  入力欄<br>
  <input type="text" id="user-input">
</template>

<style scoped></style>
