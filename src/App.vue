<template>
  <div class="app">
    <h1>YOLO v8 标注工具</h1>
    <div class="main-content">
      <ImageAnnotator />
    </div>
  </div>
</template>

<script>
import ImageAnnotator from './components/ImageAnnotator.vue'

export default {
  name: 'App',
  components: {
    ImageAnnotator
  }
}
</script>

<style>
.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.main-content {
  margin-top: 2rem;
}
</style> 