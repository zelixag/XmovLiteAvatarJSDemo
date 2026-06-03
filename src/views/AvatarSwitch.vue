<template>
  <div class="avatar-switch-container">
    <div class="header">
      <h2>实时换人演示</h2>
      <p class="description">选择多个数字人进行展示，通过切换实现"换人"效果</p>
    </div>

    <div class="main-content">
      <!-- 左侧：控制面板 -->
      <div class="left-panel">
        <!-- 说话控制 -->
        <div class="speak-panel">
          <h3>说话控制</h3>
          <div class="control-group">
            <label>说话内容：</label>
            <textarea
              v-model="speakText"
              class="speak-textarea"
              placeholder="输入要说的内容..."
              rows="5"
            ></textarea>
          </div>
          <div class="control-group">
            <label>音量：</label>
            <input
              type="range"
              v-model.number="volume"
              min="0"
              max="1"
              step="0.1"
              @input="setVolume"
            />
            <span>{{ volume }}</span>
          </div>
          <button
            class="btn btn-primary full-width"
            @click="speakToCurrentAvatar"
            :disabled="!currentAvatar || currentAvatar.loading || !currentAvatar.instance"
          >
            说话
          </button>
        </div>

        <!-- 状态信息 -->
        <div class="status-info">
          <h3>状态信息</h3>
          <div class="status-list">
            <div
              v-for="(avatar, index) in avatars"
              :key="index"
              class="status-item"
            >
              <strong>{{ avatar.name }}:</strong>
              <span>状态: {{ getStatusText(avatar.status) }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 中间：数字人显示窗口 -->
      <div class="center-panel">
        <div class="avatar-display-wrapper">
          <div 
            v-for="(avatar, index) in avatars" 
            :key="avatar.name" 
            class="avatar-display" 
            :id="'avatar-display-' + index"
            :class="{ 
              'current-avatar': currentAvatar && currentAvatar.name === avatar.name,
              'avatar-online': avatar.status === 'online' || avatar.status === 'visible',
              'avatar-invisible': avatar.status === 'invisible'
            }"
            :style="{
              display: avatar.instance ? 'block' : 'none',
              zIndex: avatar.status === 'visible' ? 10 : (avatar.status === 'invisible' ? 1 : 0),
              visibility: avatar.status === 'invisible' ? 'hidden' : 'visible'
            }"
          >
            <div v-if="!currentAvatar || currentAvatar.loading" class="loading-indicator">
              <div class="spinner"></div>
              <span>{{ currentAvatar?.loading ? '加载中...' : '请选择一个数字人' }}</span>
            </div>
          </div>
          <div class="current-avatar-info" v-if="currentAvatar && !currentAvatar.loading && !currentAvatar.error">
            <span class="avatar-name">{{ currentAvatar.name }}</span>
            <span class="avatar-status">{{ getStatusText(currentAvatar.status) }}</span>
          </div>
        </div>
      </div>

      <!-- 右侧：数字人列表 -->
      <div class="right-panel">
        <div class="avatar-list-header">
          <h3>数字人列表</h3>
        </div>
        <div class="avatar-cards-container">
          <div
            v-for="(avatar, index) in avatars"
            :key="index"
            class="avatar-card"
          >
            <div class="avatar-card-image">
              <div class="avatar-image-placeholder">
                <span>{{ avatar.name.charAt(0) }}</span>
              </div>
              <div 
                v-if="avatar.instance" 
                class="avatar-status-badge"
                :class="{
                  'status-online': avatar.status === 'online' || avatar.status === 'visible' || avatar.status === 'disconnected',
                  'status-connected': avatar.status === 'invisible'
                }"
              >
                <span v-if="avatar.status === 'online' || avatar.status === 'visible' || avatar.status === 'disconnected'">✓ 在线</span>
                <span v-else-if="avatar.status === 'invisible'">已连接,隐身</span>
              </div>
            </div>
            
            <div class="avatar-card-content">
              <h4 class="avatar-card-title">{{ avatar.name }}</h4>
            </div>
            
            <div class="avatar-card-actions">
              <template v-if="!avatar.instance">
                <button
                  class="btn-card btn-card-primary"
                  @click="connectAvatar(index)"
                  :disabled="avatar.loading"
                >
                  连接
                </button>
              </template>
              <template v-else>
                <template v-if="avatar.status === 'online' || avatar.status === 'visible'">
                  <button
                    class="btn-card btn-card-white"
                    @click="disableAvatar(index)"
                    :disabled="avatar.loading"
                  >
                    隐身
                  </button>
                </template>
                <template v-else-if="avatar.status === 'invisible'">
                  <button
                    class="btn-card btn-card-primary"
                    @click="enableAvatar(index)"
                    :disabled="avatar.loading"
                  >
                    显示
                  </button>
                </template>
                <template v-else>
                  <!-- 有 instance 但 status 未同步（如 SDK 传了数字或未触发 onStatusChange）：按在线处理 -->
                  <button
                    class="btn-card btn-card-white"
                    @click="disableAvatar(index)"
                    :disabled="avatar.loading"
                  >
                    隐身
                  </button>
                </template>
                <button
                  class="btn-card btn-card-danger"
                  @click="disconnectAvatar(index)"
                  :disabled="avatar.loading"
                >
                  断开
                </button>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, shallowRef, computed, onUnmounted, inject, nextTick } from 'vue'
import type { AppState, AppStore } from '../types'

// 注入全局状态（可选使用）
const appState = inject<AppState>('appState')
const appStore = inject<AppStore>('appStore')

interface AvatarInstance {
  name: string
  appId: string
  appSecret: string
  instance: any | null
  loading: boolean
  error: string | null
  status: 'disconnected' | 'online' | 'visible' | 'invisible'
}

// 环境配置接口（借鉴参考项目）
interface EnvironmentConfig {
  gatewayServer: string
  appId: string
  appSecret: string
  avatars: Omit<AvatarInstance, 'instance' | 'loading' | 'error' | 'status' | 'config'>[]
}

// 环境配置对象（借鉴参考项目）
const envConfigs: Record<string, EnvironmentConfig> = {
  test: {
    gatewayServer: 'https://nebula-agent.xingyun3d.com/user/v1/ttsa/session',
    // 优先使用全局状态中的配置，否则使用默认值
    appId: appState?.avatar.appId || '',
    appSecret: appState?.avatar.appSecret || '',
    avatars: [
      {
        name: '数字人1',
        appId: appState?.avatar.appId || '',
        appSecret: appState?.avatar.appSecret || ''
      },
      {
        name: '数字人2',
        appId: appState?.avatar.appId || '',
        appSecret: appState?.avatar.appSecret || ''
      }
    ]
  }
}

// 根据环境模式选择配置（默认使用 test 环境）
const currentEnv = 'test'
const currentConfig = envConfigs[currentEnv]

// 数字人列表（从环境配置中初始化）- 使用 shallowRef 优化性能
const avatars = shallowRef<AvatarInstance[]>(
  currentConfig.avatars.map(avatar => ({
    ...avatar,
    instance: null,
    loading: false,
    error: null,
    status: 'disconnected' as const
  }))
)

// SDK 配置（从环境配置中获取，借鉴参考项目）
const appId = ref(currentConfig.appId)
const appSecret = ref(currentConfig.appSecret)
const gatewayServer = ref(currentConfig.gatewayServer)

// 响应式数据
const selectedIndex = ref(-1)
const speakText = ref('你好，这是一段测试语音')
const volume = ref(1)

const currentAvatar = computed(() => {
  if (selectedIndex.value >= 0 && selectedIndex.value < avatars.value.length) {
    return avatars.value[selectedIndex.value]
  }
  return null
})

// 获取状态文本
function getStatusText(status: string): string {
  const statusMap: Record<string, string> = {
    disconnected: '未连接',
    online: '在线',
    visible: '可见',
    invisible: '隐身'
  }
  return statusMap[status] || status
}

// 计算已连接的数字人数量
const connectedCount = computed(() => {
  return avatars.value.filter(avatar => avatar.instance !== null).length
})

// 初始化单个数字人（参考 MultiAvatarDemo 的实现方式）
async function initAvatar(index: number, useInvisibleMode: boolean = false) {
  // 始终通过 avatars.value[index] 访问，确保响应式
  const avatar = avatars.value[index]
  if (avatar.instance) {
    console.log(`${avatar.name} 已经初始化`)
    return
  }

  if (!window.XmovAvatar) {
    alert('SDK未加载，请稍候再试')
    return
  }

  // 检查连接数量限制（最多允许三个数字人连接）
  if (connectedCount.value >= 3) {
    alert('最多允许三个数字人连接，请先断开其他数字人再建立连接')
    return
  }

  avatar.loading = true
  avatar.error = null

  try {
    const containerId = '#avatar-display-' + index
    const container = document.querySelector(containerId)
    if (!container) {
      avatar.error = '容器不存在'
      avatar.loading = false
      return
    }

    // 使用环境配置中的 gatewayServer（借鉴参考项目）
    // 创建 XmovAvatar 实例（使用环境配置中的 gatewayServer 和 appId/appSecret）
    avatar.instance = new window.XmovAvatar({
      containerId: containerId,
      appId: avatar.appId || appId.value,
      appSecret: avatar.appSecret || appSecret.value,
      gatewayServer: gatewayServer.value,
      enableLogger: true,
      enableDebugger: false,
      onStatusChange: (status: any) => {
        // 通过 avatars.value[index] 更新，确保 Vue 响应式生效
        if (index >= avatars.value.length) {
          console.warn(`索引 ${index} 无效，跳过状态更新`)
          return
        }
        const target = avatars.value[index]
        if (!target || !target.instance) {
          console.warn(`实例已销毁，跳过状态更新`)
          return
        }

        console.log(`${target.name} Status:`, status)

        // SDK 可能传枚举数字：online=0, visible=5, invisible=6
        const isOnline = status === 'online' || status === 0
        const isVisible = status === 'visible' || status === 5
        const isInvisible = status === 'invisible' || status === 6
        if (isOnline) {
          selectedIndex.value = index
          target.status = 'online'
        } else if (isVisible) {
          target.status = 'visible'
        } else if (isInvisible) {
          target.status = 'invisible'
        }
        target.loading = false
      },
      onRenderChange: (state: any) => {
        if (index >= avatars.value.length) return
        const target = avatars.value[index]
        if (!target || !target.instance) return
        console.log(`${target.name} Render State:`, state)
        if (state === 'rendering' && (target.status === 'online' || target.status === 'visible')) {
          target.status = 'visible'
        }
      },
      onMessage: (error: any) => {
        // 检查索引有效性
        if (index >= avatars.value.length || !avatar || !avatar.instance) {
          return
        }
        console.log(`${avatar.name} Message:`, error)
        if (error && error.code && error.code !== 0) {
          // 使用 setTimeout 延迟更新 error
          setTimeout(() => {
            if (index < avatars.value.length && avatars.value[index] && avatars.value[index].instance) {
              avatars.value[index].error = error.message || '未知错误'
            }
          }, 0)
        }
      }
    })

    // 初始化SDK（如果支持 initModel 参数，使用隐身模式）
    const initOptions: any = {
      onDownloadProgress: (progress: number) => {
        // 检查索引有效性
        if (index >= avatars.value.length || !avatar || !avatar.instance) {
          console.warn(`索引 ${index} 无效或实例已销毁，跳过进度更新`)
          return
        }
        console.log(`${avatar.name} 加载进度: ${progress}%`)
        if (progress >= 100 && index < avatars.value.length) {
          const target = avatars.value[index]
          if (target && target.instance) {
            target.status = 'online'
            target.loading = false
          }
        }
      }
    }
    
    // 如果支持 initModel，且需要隐身模式，则设置
    // 注意：需要根据实际 SDK API 调整
    if (useInvisibleMode && typeof avatar.instance.init === 'function') {
      // 某些 SDK 版本可能支持 initModel 参数
      // initOptions.initModel = 'invisible' // 根据实际 API 调整
    }
    
    await avatar.instance.init(initOptions)
  } catch (error: any) {
    // 错误处理时也要检查索引有效性
    if (index < avatars.value.length && avatar) {
      console.error(`初始化 ${avatar.name} 失败:`, error)
      avatar.error = error.message || '初始化失败'
      avatar.loading = false
    } else {
      console.error(`初始化失败: 索引 ${index} 无效`, error)
    }
    
    // 如果数字人1连接失败，自动尝试连接数字人2
    if (index === 0 && avatars.value.length > 1) {
      console.log(`${avatar.name} 连接失败，自动尝试连接数字人2...`)
      setTimeout(() => {
        if (avatars.value[1].status === 'disconnected' && !avatars.value[1].loading) {
          connectAvatar(1)
        }
      }, 1000)
    }
  }
}

// 连接数字人（参考 MultiAvatarDemo 的实现方式）
async function connectAvatar(index: number) {
  const avatar = avatars.value[index]
  if (avatar.instance) {
    return // 已经连接
  }

  // 检查连接数量
  const currentConnectedCount = connectedCount.value
  
  // 如果目前没有连接（连接数量为0），正常连接
  // 如果已经有连接展示在容器里面，其他的连接都选择隐身模式连接
  const useInvisibleMode = currentConnectedCount > 0
  
  await initAvatar(index, useInvisibleMode)
}

// 隐身数字人（进入隐身模式）
async function disableAvatar(index: number) {
  const avatar = avatars.value[index]
  if (!avatar.instance || (avatar.status !== 'online' && avatar.status !== 'visible')) {
    return
  }

  if (avatar.loading) {
    return
  }

  try {
    avatar.instance.switchInvisibleMode()
    console.log(`${avatar.name} 已隐身（进入隐身模式）`)
  } catch (error) {
    console.error(`隐身 ${avatar.name} 失败:`, error)
    avatar.loading = false
  }
}

// 在线数字人（从隐身模式转为渲染）
async function enableAvatar(index: number) {
  const avatar = avatars.value[index]
  if (!avatar.instance) {
    return
  }

  if (avatar.loading) {
    return
  }

  try {
    if (avatar.status === 'invisible') {
      // 如果之前有选中的数字人，将其设置为隐身模式
      if (selectedIndex.value >= 0 && selectedIndex.value !== index) {
        const previousAvatar = avatars.value[selectedIndex.value]
        if (previousAvatar.instance && (previousAvatar.status === 'online' || previousAvatar.status === 'visible')) {
          previousAvatar.instance.switchInvisibleMode()
        }
      }
      
      avatar.instance.switchInvisibleMode()
      selectedIndex.value = index
      console.log(`${avatar.name} 已在线（退出隐身模式）`)
    }
  } catch (error) {
    console.error(`在线 ${avatar.name} 失败:`, error)
    avatar.loading = false
  }
}

// 断开数字人连接（参考 MultiAvatarDemo 的实现方式）
async function disconnectAvatar(index: number) {
  const avatar = avatars.value[index]
  if (!avatar.instance) {
    return
  }

  try {
    // 先保存实例引用，因为 destroy 后 instance 会被设置为 null
    const instance = avatar.instance
    
    // 等待 SDK 完全销毁（包括 DOM 清理）
    await instance.destroy()
    
    // 等待多个事件循环，确保所有异步操作完成
    await nextTick()
    await new Promise(resolve => requestAnimationFrame(() => {
      setTimeout(() => {
        requestAnimationFrame(() => {
          setTimeout(resolve, 0)
        })
      }, 0)
    }))
    
    // 现在安全地更新响应式数据
    if (index < avatars.value.length && avatars.value[index] === avatar) {
      // 先清除选中状态（如果适用），避免触发其他响应式更新
      if (selectedIndex.value === index) {
        selectedIndex.value = -1
        // 等待选中状态更新完成
        await nextTick()
      }
      
      // 然后更新 avatar 状态
      avatar.instance = null
      avatar.status = 'disconnected'
      avatar.error = null
      avatar.loading = false
      
      // 再次等待 Vue 完成更新
      await nextTick()
    }
    
    console.log(`已断开 ${avatar.name}`)
  } catch (error) {
    console.error(`断开 ${avatar.name} 失败:`, error)
    // 即使出错，也要清理状态
    if (index < avatars.value.length && avatars.value[index] === avatar) {
      avatar.instance = null
      avatar.status = 'disconnected'
      avatar.error = null
      avatar.loading = false
    }
  }
}

// 对当前选中的数字人说话
function speakToCurrentAvatar() {
  if (!currentAvatar.value || !currentAvatar.value.instance || currentAvatar.value.loading) {
    return
  }
  currentAvatar.value.instance.speak(speakText.value, true, true)
}

// 设置音量
function setVolume() {
  avatars.value.forEach(avatar => {
    if (avatar.instance) {
      avatar.instance.setVolume(volume.value)
    }
  })
}

// 组件卸载时清理
onUnmounted(() => {
  avatars.value.forEach(avatar => {
    if (avatar.instance) {
      try {
        avatar.instance.stop()
        avatar.instance.destroy()
      } catch (error) {
        console.error('清理失败:', error)
      }
    }
  })
})
</script>

<style scoped>
.avatar-switch-container { width: 100vw; height: 100vh; display: flex; flex-direction: column; background: var(--bg-primary); }
.header { padding: var(--space-md); background: var(--bg-primary); border-bottom: 1px solid var(--border-color); display: flex; align-items: center; gap: 20px; }
.header h2 { margin: 0; font-family: var(--font-heading); font-weight: 400; font-size: 24px; letter-spacing: -0.01em; }
.description { margin: 0; color: var(--text-muted); font-size: 13px; }
.back-btn { margin-left: auto; padding: 8px 16px; border: 1px solid var(--text-primary); background: transparent; color: var(--text-primary); cursor: pointer; font-size: 13px; letter-spacing: 0.04em; transition: all 0.15s ease; }
.back-btn:hover { background: var(--text-primary); color: var(--bg-primary); }
.main-content { flex: 1; display: flex; gap: 0; padding: var(--space-md); overflow: hidden; }
.left-panel { width: 280px; background: var(--bg-primary); border-right: 1px solid var(--border-color); padding: var(--space-md); overflow-y: auto; }
.center-panel { flex: 1; background: #000; overflow: hidden; position: relative; display: flex; align-items: center; justify-content: center; padding: 20px; border: 1px solid var(--border-color); }
.right-panel { width: 280px; background: var(--bg-primary); border-left: 1px solid var(--border-color); padding: var(--space-md); overflow-y: auto; }

.avatar-display-wrapper { position: relative; height: 640px; width: 360px; text-align: center; margin: 0 auto; }
.avatar-display { width: 100%; height: 100%; background: var(--bg-secondary); position: absolute !important; top: 0; left: 0; overflow: hidden; border: 1px solid var(--border-color); }
.avatar-display > div { width: 100%; height: 100%; position: absolute; top: 0; left: 0; }
.avatar-display :deep(canvas), .avatar-display :deep(video) { width: 100%; height: 100%; object-fit: contain; display: block; }
.avatar-display.avatar-online { z-index: 10; visibility: visible; }
.avatar-display.avatar-invisible { z-index: 1; visibility: hidden; pointer-events: none; }
.loading-indicator { display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100%; gap: 10px; }
.spinner { width: 32px; height: 32px; border: 2px solid var(--border-color); border-top: 2px solid var(--text-primary); border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
.current-avatar-info { position: absolute; bottom: 20px; left: 20px; padding: 8px 16px; border: 1px solid rgba(255,255,255,0.3); color: white; }
.avatar-name { font-weight: 400; margin-right: 10px; font-family: var(--font-heading); letter-spacing: 0.02em; }
.speak-panel, .status-info { margin-bottom: var(--space-md); padding-bottom: var(--space-md); border-bottom: 1px solid var(--border-color-light); }
.control-group { margin-bottom: 12px; }
.control-group label { display: block; margin-bottom: 4px; font-size: 11px; font-weight: 400; letter-spacing: 0.06em; color: var(--text-muted); }
.speak-textarea { width: 100%; padding: 8px 0; border: none; border-bottom: 1px solid var(--border-color); font-size: 14px; resize: vertical; background: transparent; color: var(--text-primary); }
.speak-textarea:focus { outline: none; border-bottom-color: var(--text-primary); border-bottom-width: 2px; }

.btn { width: auto; min-width: auto; white-space: nowrap; padding: 10px 20px; border: 1px solid var(--text-primary); cursor: pointer; font-size: 13px; font-weight: 400; letter-spacing: 0.04em; transition: all 0.15s ease; background: transparent; color: var(--text-primary); }
.btn:hover:not(:disabled) { background: var(--text-primary); color: var(--bg-primary); }
.btn:disabled { opacity: 0.3; cursor: not-allowed; }
.full-width { width: 100%; }
.avatar-card { background: var(--bg-primary); padding: 12px; margin-bottom: 12px; border: 1px solid var(--border-color); }
.avatar-card-image { position: relative; width: 100%; height: 100px; background: var(--bg-secondary); display: flex; align-items: center; justify-content: center; margin-bottom: 10px; }
.avatar-image-placeholder { font-size: 36px; color: var(--text-muted); font-family: var(--font-heading); }
.avatar-status-badge { position: absolute; top: 5px; right: 5px; padding: 2px 8px; font-size: 10px; letter-spacing: 0.06em; border: 1px solid; }
.status-online { color: var(--accent); border-color: var(--accent); background: transparent; }
.status-connected { color: var(--text-primary); border-color: var(--text-primary); background: transparent; }
.avatar-card-title { margin: 0 0 8px 0; font-size: 14px; font-weight: 400; }
.btn-card { padding: 6px 12px; border: 1px solid var(--text-primary); cursor: pointer; font-size: 11px; margin-right: 5px; margin-bottom: 5px; transition: all 0.15s ease; font-weight: 400; letter-spacing: 0.04em; background: transparent; color: var(--text-primary); }
.btn-card:hover:not(:disabled) { background: var(--text-primary); color: var(--bg-primary); }
.btn-card:disabled { opacity: 0.3; cursor: not-allowed; }
.status-list { display: flex; flex-direction: column; gap: 8px; }
.status-item { padding: 10px; border: 1px solid var(--border-color); display: flex; flex-direction: column; gap: 5px; background: transparent; }
</style>
