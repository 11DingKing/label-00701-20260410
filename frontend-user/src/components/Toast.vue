<script setup>
defineProps({
  show: Boolean,
  message: String
})
</script>

<template>
  <Transition name="toast">
    <div class="toast" v-if="show">
      {{ message }}
    </div>
  </Transition>
</template>
