<script setup>
  import { defineEmits } from 'vue'

  const emit = defineEmits(['fileLoaded'])

  const loadDataFromFile = (value) => {
    const file = value.target.files[0];
    const reader = new FileReader();
    reader.onload = e => emit('fileLoaded', e.target.result);
    reader.readAsText(file);
  }
</script>

<template>
  <label class="text-reader">
    <input type="file" @change="loadDataFromFile" accept=".csv">
  </label>
</template>