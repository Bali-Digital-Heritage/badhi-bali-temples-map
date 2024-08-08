<template>
  <div class="container">
    <div class="card" v-for="layer in layers" :key="layer.id">
      <header class="card-header">
        <p class="card-header-title">{{ layer.name }}</p>
        <button class="card-header-icon" aria-label="more options">
          <span class="icon">
            <i :class="{'fa': true, 'fa-eye': layer.isactive, 'fa-eye-slash': !layer.isactive}" @click="toggleLayer(layer)" aria-hidden="true"></i>
          </span>
          
        </button>
      </header>
      <div class="card-content" v-show="layer.isactive">
        <div class="content">
            {{ layer.description }}            
           <div class="legend mt-3" v-html="layer.legend"></div>
        </div>
      </div>
      
    </div>
  </div>
</template>

<script setup>
import { defineEmits } from 'vue';

defineProps({
    layers:[]
})
const emit = defineEmits(['togglelayer']);


const toggleLayer = (layer)=>{
    layer.isactive = !layer.isactive;
    emit('togglelayer', layer); // Emit the event with the updated layer
}

</script>

<style lang="scss" scoped></style>
