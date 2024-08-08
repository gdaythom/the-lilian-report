<script setup>
import { useRoute } from 'vue-router'
const route = useRoute();
</script>
<template>
  <div class="about">
    <h1>Tier</h1>
    {{ route.params.tier }}
  </div>
</template>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>
