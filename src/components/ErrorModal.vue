<template>
  <Teleport to="body">
    <div v-if="visible" class="error-overlay" @click.self="close">
      <div class="error-modal">
        <div class="error-header">
          <span class="error-icon">!</span>
          <span class="error-title">错误</span>
        </div>
        <div class="error-body">{{ message }}</div>
        <div class="error-footer">
          <button class="error-btn" @click="close">确定</button>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { errorMessage } from '../stores/app'

const visible = computed(() => !!errorMessage.value)
const message = computed(() => errorMessage.value)

function close() {
  errorMessage.value = ''
}
</script>

<style scoped>
.error-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}
.error-modal {
  background: #fff;
  border-radius: 12px;
  width: 360px;
  max-width: 90vw;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}
.error-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 20px 24px 0;
}
.error-icon {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: #e74c3c;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 16px;
  flex-shrink: 0;
}
.error-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
}
.error-body {
  padding: 16px 24px;
  font-size: 14px;
  color: #666;
  line-height: 1.6;
  word-break: break-all;
}
.error-footer {
  padding: 0 24px 20px;
  display: flex;
  justify-content: flex-end;
}
.error-btn {
  padding: 8px 32px;
  border: none;
  border-radius: 6px;
  background: #e74c3c;
  color: #fff;
  font-size: 14px;
  cursor: pointer;
}
.error-btn:hover {
  background: #c0392b;
}
</style>
