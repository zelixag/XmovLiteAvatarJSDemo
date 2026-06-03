<template>
  <div class="walk-container">
    <div class="header">
      <h2>行走功能演示</h2>
      <p class="description">配置行走点位，控制数字人在场景中行走</p>
    </div>

    <div class="main-content">
      <!-- 左侧：数字人显示区域 -->
      <div class="left-panel">
        <div class="avatar-display-wrapper">
          <div id="walk-avatar" class="avatar-display"></div>
          <div v-if="loading" class="loading-overlay">
            <div class="progress-panel">
              <div class="progress-label">加载中</div>
              <div class="progress-bar-container">
                <div class="progress-bar" :style="{ width: progress + '%' }"></div>
              </div>
              <div class="progress-percentage">{{ progress }}%</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧：控制面板 -->
      <div class="right-panel">
        <!-- 连接 -->
        <div class="section">
          <div class="section-row">
            <button class="btn" @click="connectAvatar" :disabled="loading || !!avatarInstance">
              {{ loading ? '连接中...' : avatarInstance ? '已连接' : '连接' }}
            </button>
            <button class="btn" @click="disconnectAvatar" :disabled="!avatarInstance">断开</button>
            <button class="btn btn-sm" @click="showParams = !showParams" :class="{ active: showParams }">参数</button>
          </div>
          <div v-if="showParams" class="params-panel">
            <div class="param-item"><input v-model="appId" type="text" placeholder="App ID" /></div>
            <div class="param-item"><input v-model="appSecret" type="text" placeholder="App Secret" /></div>
            <div class="param-item"><input v-model="gatewayServer" type="text" placeholder="Gateway" /></div>
          </div>
          <div v-if="avatarInstance" class="status-row">
            <span class="status-dot connected"></span>已连接
            <span v-if="walkState" class="status-divider">|</span>
            <span v-if="walkState" class="status-dot" :class="walkState === 'walk_start' ? 'walking' : 'idle'"></span>
            <span v-if="walkState">{{ walkState === 'walk_start' ? '行走中' : '空闲' }}</span>
          </div>
        </div>

        <!-- 点位 -->
        <div class="section">
          <div class="position-grid">
            <button v-for="point in pointNames" :key="point"
                    :class="['pos-btn', { active: currentPosition === point }]"
                    @click="walkToPoint(point)"
                    :disabled="!avatarInstance || walkState === 'walk_start'">
              {{ point }}
            </button>
          </div>
        </div>

        <!-- 说话 -->
        <div class="section">
          <textarea v-model="speakText" class="speak-textarea" placeholder="说话内容（SSML）..." rows="3"></textarea>
          <button class="btn full-width" @click="speak" :disabled="!avatarInstance || walkState === 'walk_start'">说话</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, shallowRef, onUnmounted, inject } from 'vue'
import { appStore } from '../stores/app'
import type { AppState } from '../types'

const appStateInjected = inject<AppState>('appState')!
const appStoreInjected = appStore

const appId = ref(appStateInjected?.avatar.appId || '')
const appSecret = ref(appStateInjected?.avatar.appSecret || '')
const gatewayServer = ref('https://nebula-agent.xingyun3d.com/user/v1/ttsa/session')

// 独立的数字人实例（不共享）- 使用 shallowRef 优化性能
const avatarInstance = shallowRef<any>(null)
const loading = ref(false)
const progress = ref(0)
const walkState = ref('')
const currentPosition = ref('F')
const showParams = ref(false)
const pointNames = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K'] as const

const speakText = ref(`<speak>
    <ue4event>
        <type>walk</type>
        <data>
            <target>K</target>
        </data>
    </ue4event>
霜降已过，初冬将至。虽然天气渐冷，但荣成市旅游市场依然"热"力十足。频繁冲上热搜，在这背后是荣成市依托本地特色，精准深耕客源市场，优化营销策略，吸引更多游客"奔荣而来"探访"诗与远方"。
</speak>`)

async function connectAvatar() {
  if (!appId.value || !appSecret.value) {
    alert('请先配置 AppId 和 AppSecret')
    return
  }

  if (avatarInstance.value) return

  loading.value = true
  progress.value = 0

  try {
    const containerId = 'walk-avatar'
    if (!document.querySelector(`#${containerId}`)) {
      alert('容器不存在')
      loading.value = false
      return
    }

    // 创建独立的数字人实例
    avatarInstance.value = await appStoreInjected.createAvatarInstance({
      containerId,
      appId: appId.value,
      appSecret: appSecret.value,
      gatewayServer: gatewayServer.value,
      useInvisibleMode: false,
      onStatusChange: (status: any) => {
        console.log('Walk Status:', status)
      },
      onWalkStateChange: (state: string) => {
        walkState.value = state
        if (state === 'walk_end') {
          // 行走结束后可以重新启用按钮
        }
      },
      onDownloadProgress: (prog: number) => {
        progress.value = prog
        if (prog === 100) {
          loading.value = false
        }
      }
    })
    
    currentPosition.value = 'F'
    loading.value = false
    progress.value = 100
    console.log('数字人连接成功')
  } catch (e: any) {
    alert('连接失败: ' + (e.message || '未知错误'))
    loading.value = false
    avatarInstance.value = null
  }
}

async function disconnectAvatar() {
  if (avatarInstance.value) {
    try {
      await avatarInstance.value.destroy()
    } catch (e) {
      console.error('断开失败:', e)
    }
    avatarInstance.value = null
  }
  walkState.value = ''
  currentPosition.value = 'F'
  progress.value = 0
}

function walkToPoint(pointName: string) {
  if (!avatarInstance.value || walkState.value === 'walk_start') return;
  
  const speakContent = `<speak>
    <ue4event>
        <type>walk</type>
        <data>
            <target>${pointName}</target>
        </data>
    </ue4event>
    正在移动到点位${pointName}
</speak>`;
  
  try {
    avatarInstance.value.speak(speakContent, true, true);
    currentPosition.value = pointName;
    console.log(`数字人开始向点位 ${pointName} 移动`);
  } catch (e) {
    console.error('行走控制失败:', e);
    alert('行走控制失败，请稍后重试');
  }
}

function speak() {
  if (!avatarInstance.value || !speakText.value.trim()) {
    alert(avatarInstance.value ? '请输入内容' : '请先连接');
    return;
  }
  try {
    avatarInstance.value.speak(speakText.value, true, true);
  } catch (e) {
    alert('说话失败: ' + (e as Error).message);
  }
}

onUnmounted(() => {
  if (avatarInstance.value) {
    avatarInstance.value.destroy().catch(console.error)
    avatarInstance.value = null
  }
})
</script>

<style scoped>
.walk-container { width: 100vw; height: 100vh; display: flex; flex-direction: column; background: var(--bg-primary); }
.header { padding: var(--space-md); background: var(--bg-primary); border-bottom: 1px solid var(--border-color); }
.header h2 { margin: 0 0 4px 0; font-family: var(--font-heading); font-weight: 400; font-size: 24px; letter-spacing: -0.01em; }
.description { margin: 0; color: var(--text-muted); font-size: 13px; }
.main-content { flex: 1; display: flex; gap: 0; padding: var(--space-md); overflow: hidden; }
.left-panel { flex: 1; background: transparent; overflow: hidden; position: relative; border: 1px solid var(--border-color); }
.right-panel { width: 300px; background: var(--bg-primary); border-left: 1px solid var(--border-color); padding: var(--space-md); overflow-y: auto; display: flex; flex-direction: column; gap: 20px; }
.avatar-display-wrapper { width: 100%; height: 100%; position: relative; background: transparent; }
.avatar-display { width: 100%; height: 100%; position: relative; }
.avatar-display > div { width: 100%; height: 100%; position: absolute; top: 0; left: 0; }
.avatar-display :deep(canvas), .avatar-display :deep(video) { width: 100%; height: 100%; object-fit: contain; display: block; }
.loading-overlay { position: absolute; inset: 0; background: rgba(0,0,0,0.6); display: flex; align-items: center; justify-content: center; z-index: 100; }
.progress-panel { background: var(--bg-primary); padding: var(--space-md); text-align: center; min-width: 240px; border: 1px solid var(--border-color); }
.progress-label { font-size: 11px; color: var(--text-muted); margin-bottom: 8px; letter-spacing: 0.08em; text-transform: uppercase; }
.progress-bar-container { width: 100%; height: 3px; background: var(--border-color); overflow: hidden; margin-bottom: 8px; }
.progress-bar { height: 100%; background: var(--text-primary); transition: width 0.3s; }
.progress-percentage { font-size: 14px; color: var(--text-primary); font-family: var(--font-mono); }

.section { padding-bottom: 16px; border-bottom: 1px solid var(--border-color-light); }
.section:last-child { border-bottom: none; padding-bottom: 0; }
.section-row { display: flex; gap: 6px; }
.section-row .btn { flex: 1; }
.section-row .btn-sm { flex: 0 0 auto; padding: 8px 14px; font-size: 12px; }
.section-row .btn-sm.active { background: var(--text-primary); color: var(--bg-primary); }

.btn { padding: 8px 16px; border: 1px solid var(--text-primary); cursor: pointer; font-size: 13px; transition: all 0.15s ease; background: transparent; color: var(--text-primary); letter-spacing: 0.04em; }
.btn:hover:not(:disabled) { background: var(--text-primary); color: var(--bg-primary); }
.btn:disabled { opacity: 0.3; cursor: not-allowed; }
.full-width { width: 100%; margin-top: 8px; }

.params-panel { margin-top: 10px; display: flex; flex-direction: column; gap: 8px; }
.param-item input { width: 100%; padding: 6px 0; border: none; border-bottom: 1px solid var(--border-color); font-size: 13px; background: transparent; color: var(--text-primary); }
.param-item input:focus { outline: none; border-bottom-color: var(--text-primary); border-bottom-width: 2px; }
.param-item input::placeholder { color: var(--text-muted); }

.status-row { margin-top: 10px; font-size: 12px; color: var(--text-muted); display: flex; align-items: center; gap: 6px; letter-spacing: 0.04em; }
.status-dot { width: 6px; height: 6px; border-radius: 50%; background: var(--text-muted); flex-shrink: 0; }
.status-dot.connected { background: var(--accent); }
.status-dot.walking { background: var(--text-primary); }
.status-dot.idle { background: var(--text-muted); }
.status-divider { color: var(--border-color); }

.position-grid { display: grid; grid-template-columns: repeat(6, 1fr); gap: 4px; }
.pos-btn { width: 100%; aspect-ratio: 1; border: 1px solid var(--border-color); background: transparent; color: var(--text-secondary); cursor: pointer; font-size: 11px; font-family: var(--font-mono); transition: all 0.15s ease; padding: 0; display: flex; align-items: center; justify-content: center; }
.pos-btn:hover:not(:disabled) { border-color: var(--text-primary); color: var(--text-primary); }
.pos-btn.active { background: var(--text-primary); color: var(--bg-primary); border-color: var(--text-primary); }
.pos-btn:disabled { opacity: 0.3; cursor: not-allowed; }

.speak-textarea { width: 100%; padding: 6px 0; border: none; border-bottom: 1px solid var(--border-color); font-family: var(--font-body); font-size: 13px; box-sizing: border-box; resize: vertical; background: transparent; color: var(--text-primary); margin-bottom: 4px; }
.speak-textarea:focus { outline: none; border-bottom-color: var(--text-primary); border-bottom-width: 2px; }
.speak-textarea::placeholder { color: var(--text-muted); }
</style>
