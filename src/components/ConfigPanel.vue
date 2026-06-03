<template>
  <div class="config-panel">
    <!-- 虚拟人配置 -->
    <section class="config-section">
      <h3 class="section-title">虚拟人 SDK 配置</h3>

      <div class="form-group">
        <label>应用 APP ID：</label>
        <input
          v-model="appState.avatar.appId"
          type="text"
          placeholder="请输入 APP ID"
        />
      </div>

      <div class="form-group">
        <label>应用 APP Secret</label>
        <input
          v-model="appState.avatar.appSecret"
          type="text"
          placeholder="请输入 APP Secret"
        />
      </div>
    </section>

    <!-- ASR配置 -->
    <section class="config-section">
      <div class="section-title">
        <h3>语音识别配置</h3>
        <button
          @click="handleTestAsrConnection"
          :disabled="isTestingAsr"
          class="btn-connect-status"
          :class="{ connected: asrConnected }"
        >
          {{ isTestingAsr ? "测试中..." : asrConnected ? "已连接" : "连接" }}
        </button>
      </div>

      <div class="form-group">
        <label>ASR 服务商</label>
        <select v-model="appState.asr.provider" @change="handleProviderChange">
          <option value="tx">腾讯云</option>
          <option value="xmov">xmovASR</option>
          <!-- <option value="doubao">豆包ASR</option> -->
          <option value="xunfei">讯飞ASR</option>
        </select>
      </div>

      <div class="form-group">
        <label>ASR App ID</label>
        <input
          v-model="appState.asr.appId"
          type="text"
          placeholder="请输入 ASR App ID"
        />
      </div>

      <div class="form-group" v-if="needsSecretId">
        <label>ASR Secret ID</label>
        <input
          v-model="appState.asr.secretId"
          type="text"
          placeholder="请输入 Secret ID"
        />
      </div>

      <div class="form-group">
        <label>ASR Secret Key</label>
        <input
          v-model="appState.asr.secretKey"
          type="text"
          placeholder="请输入 Secret Key"
        />
      </div>
    </section>

    <!-- LLM配置 -->
    <section class="config-section">
      <div class="section-title">
        <h3 class="">大语言模型配置</h3>
        <button
          @click="handleTestLlmConnection"
          :disabled="isTesting"
          class="btn-connect-status"
          :class="{ connected: llmConnected }"
        >
          {{ isTesting ? "测试中..." : llmConnected ? "已连接" : "连接" }}
        </button>
      </div>

      <div class="form-group">
        <label>模型选择</label>
        <select v-model="appState.llm.model" @change="handleModelChange">
          <option
            v-for="model in supportedModels"
            :key="model.value"
            :value="model.value"
          >
            {{ model.value }}
          </option>
        </select>
      </div>

      <div class="form-group">
        <label>API Key</label>
        <input
          v-model="appState.llm.apiKey"
          type="password"
          placeholder="请输入 API Key"
        />
      </div>

      <!-- 当模型为coze时显示bot_id输入 -->
      <div v-if="appState.llm.model === 'coze'" class="form-group">
        <label>Bot ID</label>
        <input
          v-model="appState.llm.botId"
          type="text"
          placeholder="请输入 Bot ID"
          required
        />
      </div>
    </section>

    <!-- 控制按钮 -->
    <section class="control-section">
      <div class="button-group">
        <button
          @click="handleConnect"
          :disabled="isConnecting || allConnected"
          class="btn btn-primary"
        >
          {{
            isConnecting
              ? "连接中..."
              : allConnected
                ? "已连接"
                : "连接"
          }}
        </button>

        <button
          @click="handleDisconnect"
          :disabled="!appState.avatar.connected && !asrConnected && !llmConnected"
          class="btn btn-secondary"
        >
          断开
        </button>
      </div>
    </section>

    <!-- 消息交互 -->
    <section class="message-section">
      <h3 class="section-title">消息交互</h3>

      <div class="form-group">
        <label>输入消息</label>
        <textarea
          v-model="appState.ui.text"
          rows="4"
          placeholder="请输入您的消息..."
        />
      </div>

      <div class="button-group">
        <button
          @click="handleVoiceInput"
          :disabled="!appState.avatar.connected || appState.asr.isListening"
          class="btn btn-voice"
        >
          {{ appState.asr.isListening ? "正在听..." : "语音输入" }}
        </button>

        <button
          @click="handleSendMessage"
          :disabled="
            !appState.avatar.connected || !appState.ui.text.trim() || isSending
          "
          class="btn btn-primary"
        >
          {{ isSending ? "发送中..." : "发送" }}
        </button>

        <button
          @click="handleInterrupt"
          :disabled="!appState.avatar.connected"
          class="btn btn-interrupt"
        >
          打断
        </button>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { inject, ref, computed, onMounted, shallowRef } from "vue";
import { useAsr } from "../composables/useAsr";
import { SUPPORTED_LLM_MODELS ,LLM_CONFIG} from "../constants";
import { avatarState } from "../stores/app";
import { llmService } from "../services/llm";
import { loadConfig, getAsrConfig, getLlmConfig } from "../utils/config-loader";
import type { AppState, AppStore, AsrProvider } from "../types";

// 注入全局状态和方法
const appState = inject<AppState>("appState")!;
const appStore = inject<AppStore>("appStore")!;

// 组件状态
const isConnecting = ref(false);
const isSending = ref(false);
const isTesting = ref(false);
const isTestingAsr = ref(false);
const asrConnected = ref(false);
const llmConnected = ref(false);
const supportedModels = SUPPORTED_LLM_MODELS;
// 配置缓存 - 使用 shallowRef 优化性能
const configCache = shallowRef<any>(null);

// 计算属性：三项全部连接成功
const allConnected = computed(() => appState.avatar.connected && asrConnected.value && llmConnected.value);

// 计算属性：虚拟人是否正在说话
const isSpeaking = computed(() => avatarState.value === "speak");

// 计算属性：是否需要显示 Secret ID 字段
const needsSecretId = computed(() => {
  return !["xmov", "xunfei"].includes(appState.asr.provider);
});

// ASR Hook - 使用computed确保配置更新时重新创建
const asrConfig = computed(() => ({
  provider: appState.asr.provider as AsrProvider,
  appId: appState.asr.appId,
  secretId: appState.asr.secretId,
  secretKey: appState.asr.secretKey,
}));

// 初始化ASR hook（用于停止功能）
const { stop: stopAsr } = useAsr(asrConfig.value);

// 组件挂载时加载配置
onMounted(() => {
  initConfig();
});

async function initConfig() {
  // 加载配置文件
  const config = await loadConfig();
  if (!config) return;
  
  configCache.value = config;
  console.log("配置加载成功:", config);
  
  // 虚拟人配置
  appState.avatar.appId = config.avatar.appId;
  appState.avatar.appSecret = config.avatar.appSecret;
  appState.avatar.gatewayServer = config.avatar.gatewayServer || '';

  // ASR配置 - 默认使用腾讯云
  const defaultAsrConfig = config.asr.tx;
  appState.asr.appId = defaultAsrConfig.appId;
  appState.asr.secretId = defaultAsrConfig.secretId || "";
  appState.asr.secretKey = defaultAsrConfig.secretKey;

  // LLM配置 - 默认使用豆包配置
  const defaultLlmConfig = config.llm[LLM_CONFIG.DEFAULT_MODEL];
  appState.llm.apiKey = defaultLlmConfig.apiKey;
  // 只有coze模型需要botId
  appState.llm.botId = appState.llm.model === "coze" ? (defaultLlmConfig.botId || "") : "";
}

// 事件处理函数
async function handleConnect() {
  if (isConnecting.value) return;

  isConnecting.value = true;
  asrConnected.value = false;
  llmConnected.value = false;
  const results: string[] = [];

  // 1. 连接数字人
  if (appState.avatar.appId && appState.avatar.appSecret) {
    try {
      await appStore.connectAvatar();
      results.push('数字人: 连接成功');
    } catch (error: any) {
      results.push(`数字人: 连接失败 - ${error?.message || error}`);
    }
  } else {
    results.push('数字人: 配置不完整，跳过');
  }

  // 2. 测试ASR
  if (appState.asr.appId && appState.asr.secretKey) {
    try {
      const { testConnection } = useAsr({
        provider: appState.asr.provider as AsrProvider,
        appId: appState.asr.appId,
        secretId: appState.asr.secretId,
        secretKey: appState.asr.secretKey,
      });
      await testConnection();
      asrConnected.value = true;
      results.push('ASR: 连接成功');
    } catch (error: any) {
      results.push(`ASR: 连接失败 - ${error?.message || error}`);
    }
  } else {
    results.push('ASR: 配置不完整，跳过');
  }

  // 3. 测试LLM
  if (appState.llm.apiKey) {
    try {
      const findConfig = SUPPORTED_LLM_MODELS.find((item) => item.value === appState.llm.model) || { label: 'openai', baseURL: '' };
      await llmService.testConnection({
        provider: findConfig?.label as any,
        model: appState.llm.model,
        apiKey: appState.llm.apiKey,
        baseURL: findConfig?.baseURL || '',
        botId: appState.llm.botId,
      });
      llmConnected.value = true;
      results.push('LLM: 连接成功');
    } catch (error: any) {
      results.push(`LLM: 连接失败 - ${error?.message || error}`);
    }
  } else {
    results.push('LLM: 配置不完整，跳过');
  }

  isConnecting.value = false;
  alert(results.join('\n'));
}

function handleDisconnect() {
  console.log("开始断开所有连接...");
  // 1. 断开虚拟人连接
  appStore.disconnectAvatar();
  // 2. 断开ASR连接
  if (appState.asr.isListening) {
    stopAsr();
    appStore.stopVoiceInput();
  }
  asrConnected.value = false;
  // 3. 断开LLM连接
  llmService.disconnect();
  llmConnected.value = false;
  console.log("所有连接已断开");
}

function handleVoiceInput() {
  if (appState.asr.isListening) {
    stopAsr();
    // stopAsrWithConfig();
    appStore.stopVoiceInput();
    return;
  }

  // 验证ASR配置
  const { appId, secretId, secretKey, provider } = appState.asr;
  if (!appId || !secretKey) {
    alert("请先配置ASR信息（App ID、Secret ID、Secret Key）");
    return;
  }

  // 创建新的ASR实例（使用当前配置）
  const { start: startAsrWithConfig, stop: stopAsrWithConfig } = useAsr({
    provider: provider as AsrProvider,
    appId: appState.asr.appId,
    secretId: appState.asr.secretId,
    secretKey: appState.asr.secretKey,
  });

  // 用于防止重复发送的标志
  const hasAutoSent = ref(false);

  const handleAsrFinished = async (text: string) => {
    appState.ui.text = text;
    stopAsrWithConfig();
    appStore.stopVoiceInput();

    // 自动发送给大模型（防止重复发送）
    if (text.trim() && !hasAutoSent.value) {
      hasAutoSent.value = true;
      try {
        isSending.value = true;
        await appStore.sendMessage();
      } catch (error) {
        console.error("自动发送消息失败:", error);
        alert("自动发送消息失败");
      } finally {
        isSending.value = false;
        hasAutoSent.value = false;
      }
    }
  };

  const handleAsrError = (error: any) => {
    console.error("语音识别错误:", error);
    stopAsrWithConfig();
    appStore.stopVoiceInput();
    hasAutoSent.value = false;
  };

  appStore.startVoiceInput({
    onFinished: (text: string) => {
      appState.ui.text = text;
      stopAsrWithConfig();
      appStore.stopVoiceInput();
    },
    onError: handleAsrError,
  });

  startAsrWithConfig({
    onFinished: handleAsrFinished,
    onError: handleAsrError,
  });
}

async function handleSendMessage() {
  if (isSending.value || !appState.ui.text.trim()) return;

  isSending.value = true;
  try {
    await appStore.sendMessage();
  } catch (error) {
    console.error("发送消息失败:", error);
    alert("发送消息失败");
  } finally {
    isSending.value = false;
  }
}

function handleInterrupt() {
  if (!appState.avatar.connected) return;

  appStore.interrupt();
}

// 测试LLM连接
async function handleTestLlmConnection() {
  if (isTesting.value) return;

  const { apiKey, model } = appState.llm;
  if (!apiKey) {
    alert("请先输入API Key");
    return;
  }

  isTesting.value = true;
  try {
    const findConfig = SUPPORTED_LLM_MODELS.find((item) => item.value === model) || {label:'openai',baseURL:''};
    await llmService.testConnection({
      provider: findConfig?.label,
      model,
      apiKey,
      baseURL: findConfig?.baseURL,
      botId: appState.llm.botId,
    });
    llmConnected.value = true;
    // 单个测试连接成功不弹窗，按钮状态会变化
  } catch (error) {
    console.error("LLM连接测试失败:", error);
    alert("请检测LLM连接信息是否正确");
  } finally {
    isTesting.value = false;
  }
}

// 测试ASR连接
async function handleTestAsrConnection() {
  if (isTestingAsr.value) return;

  const { appId, secretId, secretKey, provider } = appState.asr;
  if (!appId || !secretKey) {
    alert("请先填写完整的ASR配置信息");
    return;
  }

  isTestingAsr.value = true;
  try {
    // 创建新的ASR实例进行测试
    const { testConnection } = useAsr({
      provider: provider as AsrProvider,
      appId,
      secretId,
      secretKey,
    });

    await testConnection();
    asrConnected.value = true;
    // 单个测试连接成功不弹窗，按钮状态会变化
  } catch (error) {
    console.error("ASR连接测试失败:", error);
    alert("请检测ASR连接信息是否正确");
  } finally {
    isTestingAsr.value = false;
  }
}
// 重置ASR配置 - 从配置文件加载对应服务商的密钥
function handleProviderChange() {
  const config = configCache.value;
  if (!config) {
    appState.asr.appId = "";
    appState.asr.secretId = "";
    appState.asr.secretKey = "";
    return;
  }

  const provider = appState.asr.provider;
  // 从配置中获取对应服务商的密钥
  const asrConfig = getAsrConfig(config, provider);

  // 更新到全局状态
  appState.asr.appId = asrConfig.appId;
  appState.asr.secretId = asrConfig.secretId || "";
  appState.asr.secretKey = asrConfig.secretKey;
}

// 重置LLM配置 - 从配置文件加载对应模型的密钥
function handleModelChange() {
  const config = configCache.value;
  if (!config) {
    appState.llm.apiKey = "";
    appState.llm.botId = "";
    return;
  }
  
  const model = appState.llm.model;
  // 从配置中获取对应模型的密钥
  const llmConfig = getLlmConfig(config, model);

  // 更新到全局状态
  appState.llm.apiKey = llmConfig.apiKey;
  // 只有coze模型需要botId
  appState.llm.botId = model === "coze" ? (llmConfig.botId || "") : "";
}
</script>

<style scoped>
.config-panel {
  width: 420px;
  max-height: 100vh;
  overflow-y: auto;
  padding: var(--space-md);
  background: var(--bg-primary);
  border-left: 1px solid var(--border-color);
  display: flex;
  flex-direction: column;
  gap: var(--space-lg);
  position: absolute;
  right: 0;
  top: 0;
  height: 100%;
  z-index: 999;
}

.config-section,
.control-section,
.message-section {
  border: none;
  border-bottom: 1px solid var(--border-color-light);
  padding: 0 0 var(--space-md) 0;
  background: transparent;
}

.config-section:last-child,
.control-section:last-child,
.message-section:last-child {
  border-bottom: none;
}

.section-title {
  margin: 0 0 var(--space-md) 0;
  font-size: 11px;
  font-weight: 400;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--text-muted);
  border-bottom: none;
  padding-bottom: 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: var(--space-sm);
}
.section-title h3 {
  font-size: 11px;
  font-weight: 400;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--text-muted);
}

.form-group {
  margin-bottom: var(--space-md);
}

.form-group:last-child {
  margin-bottom: 0;
}

label {
  display: block;
  margin-bottom: 4px;
  font-size: 12px;
  font-weight: 400;
  letter-spacing: 0.04em;
  color: var(--text-secondary);
}

input,
select,
textarea {
  width: 100%;
  padding: 8px 0;
  border: none;
  border-bottom: 1px solid var(--border-color);
  border-radius: 0;
  font-size: 14px;
  background: transparent;
  color: var(--text-primary);
  transition: border-color 0.15s ease;
}

input:focus,
select:focus,
textarea:focus {
  outline: none;
  border-bottom-color: var(--text-primary);
  border-bottom-width: 2px;
  box-shadow: none;
}

textarea {
  resize: vertical;
  min-height: 80px;
}

select {
  appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 24 24' fill='%239a9a9a' xmlns='http://www.w3.org/2000/svg'%3E%3Cpath d='M7 10l5 5 5-5z'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 4px center;
  background-size: 16px;
  padding-right: 24px;
}

.button-group {
  display: flex;
  gap: var(--space-sm);
  justify-content: flex-end;
}

.btn {
  flex: 1;
  padding: 10px 20px;
  border: 1px solid var(--text-primary);
  border-radius: 0;
  font-size: 13px;
  font-weight: 400;
  letter-spacing: 0.04em;
  cursor: pointer;
  transition: all 0.15s ease;
  background: transparent;
  color: var(--text-primary);
}

.btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.btn-primary:hover:not(:disabled) {
  background: var(--text-primary);
  color: var(--bg-primary);
}

.btn-secondary:hover:not(:disabled) {
  background: var(--text-primary);
  color: var(--bg-primary);
}

.btn-voice:hover:not(:disabled) {
  background: var(--text-primary);
  color: var(--bg-primary);
}

.btn-interrupt:hover:not(:disabled) {
  background: var(--text-primary);
  color: var(--bg-primary);
}

.btn-connect-status {
  padding: 4px 12px;
  border: 1px solid var(--text-primary);
  border-radius: 0;
  font-size: 11px;
  font-weight: 400;
  letter-spacing: 0.06em;
  cursor: pointer;
  transition: all 0.15s ease;
  background: transparent;
  color: var(--text-primary);
  flex: none;
  width: auto;
  min-width: 56px;
  white-space: nowrap;
  margin-left: auto;
}

.btn-connect-status:hover:not(:disabled) {
  background: var(--text-primary);
  color: var(--bg-primary);
}

.btn-connect-status.connected {
  border-color: var(--accent);
  background: var(--accent);
  color: #fff;
}

.btn-connect-status.connected:hover:not(:disabled) {
  background: var(--accent-muted);
  border-color: var(--accent-muted);
}

.btn-connect-status:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
</style>
