<template>
  <div class="interactive-container">
    <div class="header" v-if="!isMobile">
      <h2>数字人交互</h2>
      <p class="description">与数字人进行实时对话和交互</p>
    </div>
    <div class="content">
      <div class="avatar-render">
        <div :id="containerId" class="sdk-container" />
        <div v-show="appState.ui.subTitleText" class="subtitle">{{ appState.ui.subTitleText }}</div>
        <div v-show="appState.asr.isListening" class="voice-animation">
          <img :src="siriIcon" alt="语音输入" />
        </div>
        <div class="control-panel">
          <div class="control-row">
            <button class="ctrl btn-green" @click="handleControl('speak')">说话</button>
            <button class="ctrl btn-green" @click="handleControl('think')">思考</button>
            <button class="ctrl btn-green" @click="handleControl('listen')">听</button>
            <button class="ctrl btn-green" @click="handleControl('idle')">空闲</button>
            <button class="ctrl btn-green" @click="handleControl('interactive_idle')">空闲互动</button>
            <button class="ctrl btn-green" @click="handleControl('interrupt')">中断</button>
          </div>
          <div class="control-row">
            <button class="ctrl btn-gray" @click="handleControl('offline')">断线</button>
            <button class="ctrl btn-gray" @click="handleControl('online')">在线</button>
            <button class="ctrl btn-gray" @click="handleControl('destroy')">销毁</button>
          </div>
        </div>
        <div class="input-area">
          <div class="input-container">
            <textarea v-model="appState.ui.text" class="input-text" :placeholder="isMobile ? '消息' : '请输入消息...'" rows="1" @keydown.enter.exact.prevent="handleSendMessage" />
            <button v-if="!appState.ui.text.trim()" @click="handleVoiceInput" :disabled="!appState.avatar.connected" class="btn btn-voice" :class="{ 'active': appState.asr.isListening }">
              <svg viewBox="0 0 24 24" class="icon">
                <path d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3z"/>
                <path d="M17 11c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"/>
              </svg>
            </button>
            <button v-else @click="handleSendMessage" :disabled="!appState.avatar.connected" class="btn btn-send">
              <svg viewBox="0 0 24 24" class="icon">
                <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"/>
              </svg>
            </button>
          </div>
        </div>
        <div v-if="!appState.avatar.connected" class="loading-placeholder">
          <div class="loading-text">-- 正在连接 --</div>
        </div>
      </div>
      <div class="panel" :class="{ 'panel-collapsed': !isPanelOpen }" v-if="!isMobile">
        <div class="panel-header">
          <h3>设置</h3>
          <button class="panel-toggle" @click="togglePanel">
            <svg v-if="isPanelOpen" viewBox="0 0 24 24" class="icon"><path d="M7.41 15.41L12 10.83l4.59 4.58L18 14l-6-6-6 6z"/></svg>
            <svg v-else viewBox="0 0 24 24" class="icon"><path d="M7.41 8.59L12 13.17l4.59-4.58L18 10l-6 6-6-6 1.41-1.41z"/></svg>
          </button>
        </div>
        <div v-show="isPanelOpen" class="panel-content"><Info /></div>
      </div>
      <button v-if="isMobile" class="floating-settings-btn" @click="togglePanel" :class="{ 'active': isPanelOpen }">
        <svg v-if="!isPanelOpen" viewBox="0 0 24 24" class="icon">
          <path d="M19.14,12.94c0.04-0.3,0.06-0.61,0.06-0.94c0-0.32-0.02-0.64-0.07-0.94l2.03-1.58c0.18-0.14,0.23-0.41,0.12-0.61 l-1.92-3.32c-0.12-0.22-0.37-0.29-0.59-0.22l-2.39,0.96c-0.5-0.38-1.03-0.7-1.62-0.94L14.4,2.81c-0.04-0.24-0.24-0.41-0.48-0.41 h-3.84c-0.24,0-0.43,0.17-0.47,0.41L9.25,5.35C8.66,5.59,8.12,5.92,7.63,6.29L5.24,5.33c-0.22-0.08-0.47,0-0.59,0.22L2.74,8.87 C2.62,9.08,2.66,9.34,2.86,9.48l2.03,1.58C4.84,11.36,4.82,11.69,4.82,12s0.02,0.64,0.07,0.94l-2.03,1.58 c-0.18,0.14-0.23,0.41-0.12,0.61l1.92,3.32c0.12,0.22,0.37,0.29,0.59,0.22l2.39-0.96c0.5,0.38,1.03,0.7,1.62,0.94l0.36,2.54 c0.05,0.24,0.24,0.41,0.48,0.41h3.84c0.24,0,0.44-0.17,0.47-0.41l0.36-2.54c0.59-0.24,1.13-0.56,1.62-0.94l2.39,0.96 c0.22,0.08,0.47,0,0.59-0.22l1.92-3.32c0.12-0.22,0.07-0.47-0.12-0.61L19.14,12.94z M12,15.6c-1.98,0-3.6-1.62-3.6-3.6 s1.62-3.6,3.6-3.6s3.6,1.62,3.6,3.6S13.98,15.6,12,15.6z"/>
        </svg>
        <svg v-else viewBox="0 0 24 24" class="icon"><path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"/></svg>
      </button>
      <div v-if="isMobile" v-show="isPanelOpen" class="mobile-panel-overlay" @click="togglePanel">
        <div class="mobile-panel" @click.stop>
          <div class="mobile-panel-header">
            <h3>设置</h3>
            <button class="mobile-panel-close" @click="togglePanel">
              <svg viewBox="0 0 24 24" class="icon"><path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z"/></svg>
            </button>
          </div>
          <div class="mobile-panel-content"><Info /></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { inject, computed, ref, shallowRef, onMounted, onUnmounted } from "vue";
import { avatarService } from "../services/avatar";
import { useAsr } from "../composables/useAsr";
import type { AppState, AppStore, AsrProvider } from "../types";
import siriIcon from "../assets/siri.png";
import Info from "./ConfigPanel.vue";

// 示例文本常量
const DEFAULT_SPEAK_TEXT = `<speak>
    <ue4event>
        <type>walk</type>
        <data>
            <target>K</target>
        </data>
    </ue4event>
汉皇重色思倾国，御宇多年求不得。杨家有女初长成，养在深闺人未识。天生丽质难自弃，一朝选在君王侧。回眸一笑百媚生，六宫粉黛无颜色。春寒赐浴华清池，温泉水滑洗凝脂。侍儿扶起娇无力，始是新承恩泽时。云鬓花颜金步摇，芙蓉帐暖度春宵。春宵苦短日高起，从此君王不早朝。承欢侍宴无闲暇，春从春游夜专夜。后宫佳丽三千人，三千宠爱在一身。金屋妆成娇侍夜，玉楼宴罢醉和春。姊妹弟兄皆列土，可怜光彩生门户。遂令天下父母心，不重生男重生女。骊宫高处入青云，仙乐风飘处处闻。缓歌谩舞凝丝竹，尽日君王看不足。渔阳鼙鼓动地来，惊破霓裳羽衣曲。九重城阙烟尘生，千乘万骑西南行。翠华摇摇行复止，西出都门百余里。六军不发无奈何，宛转蛾眉马前死。花钿委地无人收，翠翘金雀玉搔头。君王掩面救不得，回看血泪相和流。黄埃散漫风萧索，云栈萦纡登剑阁。峨嵋山下少人行，旌旗无光日色薄。蜀江水碧蜀山青，圣主朝朝暮暮情。行宫见月伤心色，夜雨闻铃肠断声。天旋地转回龙驭，到此踌躇不能去。马嵬坡下泥土中，不见玉颜空死处。君臣相顾尽沾衣，东望都门信马归。归来池苑皆依旧，太液芙蓉未央柳。芙蓉如面柳如眉，对此如何不泪垂。春风桃李花开日，秋雨梧桐叶落时。西宫南内多秋草，落叶满阶红不扫。梨园弟子白发新，椒房阿监青娥老。夕殿萤飞思悄然，孤灯挑尽未成眠。迟迟钟鼓初长夜，耿耿星河欲曙天。鸳鸯瓦冷霜华重，翡翠衾寒谁与共。悠悠生死别经年，魂魄不曾来入梦。临邛道士鸿都客，能以精诚致魂魄。为感君王辗转思，遂教方士殷勤觅。排空驭气奔如电，升天入地求之遍。上穷碧落下黄泉，两处茫茫皆不见。忽闻海上有仙山，山在虚无缥渺间。楼阁玲珑五云起，其中绰约多仙子。中有一人字太真，雪肤花貌参差是。金阙西厢叩玉扃，转教小玉报双成。闻道汉家天子使，九华帐里梦魂惊。揽衣推枕起徘徊，珠箔银屏迤逦开。云鬓半偏新睡觉，花冠不整下堂来。风吹仙袂飘飘举，犹似霓裳羽衣舞。玉容寂寞泪阑干，梨花一枝春带雨。含情凝睇谢君王，一别音容两渺茫。昭阳殿里恩爱绝，蓬莱宫中日月长。回头下望人寰处，不见长安见尘雾。惟将旧物表深情，钿合金钗寄将去。钗留一股合一扇，钗擘黄金合分钿。但教心似金钿坚，天上人间会相见。临别殷勤重寄词，词中有誓两心知。七月七日长生殿，夜半无人私语时。在天愿作比翼鸟，在地愿为连理枝。天长地久有时尽，此恨绵绵无绝期。
</speak>`;

const appState = inject<AppState>("appState")!;
const appStore = inject<AppStore>("appStore")!;
const containerId = computed(() => avatarService.getContainerId());
const isMobile = ref(window.innerWidth < 1024);
const isPanelOpen = ref(false);

function togglePanel() {
  isPanelOpen.value = !isPanelOpen.value;
}

function updateIsMobile() {
  isMobile.value = window.innerWidth < 1024;
  if (!isMobile.value) isPanelOpen.value = false;
}

function handleSendMessage() {
  if (!appState.ui.text.trim()) return;
  appStore.sendMessage();
  appState.ui.text = "";
}

function handleControl(action: string) {
  const instance = appState.avatar.instance;
  if (!instance) {
    if (action === 'speak') {
      alert("请先连接数字人");
    }
    return;
  }
  
  switch (action) {
    case "speak": {
      const speakText = appState.ui.text.trim() || DEFAULT_SPEAK_TEXT;
      try {
        // 直接调用 speak API: instance.speak(ssml, isStart, isEnd)
        instance.speak(speakText, true, true);
        // 清空输入框
        appState.ui.text = "";
      } catch (e) {
        alert('说话失败: ' + (e as Error).message);
      }
      break;
    }
    case "think":
      instance.think?.();
      break;
    case "listen":
      handleVoiceInput();
      break;
    case "idle":
      instance.idle?.() || instance.interactiveidle?.() || appStore.interrupt();
      break;
    case "interactive_idle":
      instance.interactiveidle?.() || appStore.interrupt();
      break;
    case "interrupt":
      appStore.interrupt();
      break;
    case "offline":
      appStore.disconnectAvatar();
      break;
    case "online":
      appStore.connectAvatar();
      break;
    case "destroy":
      instance.destroy?.();
      appStore.disconnectAvatar();
      break;
  }
}

function handleVoiceInput() {
  if (appState.asr.isListening) {
    appStore.stopVoiceInput();
    return;
  }
  if (!appState.asr.appId || !appState.asr.secretKey) {
    alert("请先配置ASR信息");
    return;
  }
  const { start } = useAsr({
    provider: appState.asr.provider as AsrProvider,
    appId: appState.asr.appId,
    secretId: appState.asr.secretId,
    secretKey: appState.asr.secretKey,
  });
  appStore.startVoiceInput({ onFinished: () => {}, onError: () => {} });
  start({
    onFinished: (text: string) => {
      appState.ui.text = text;
      appStore.stopVoiceInput();
      appStore.sendMessage();
      appState.ui.text = "";
    },
    onError: () => {
      appStore.stopVoiceInput();
    },
  });
}

onMounted(() => {
  window.addEventListener("resize", updateIsMobile);
});

onUnmounted(() => {
  window.removeEventListener("resize", updateIsMobile);
  appState.avatar.instance?.destroy();
  appState.avatar.instance = null;
});
</script>

<style scoped>
.interactive-container { width: 100vw; height: 100vh; display: flex; flex-direction: column; background: var(--bg-primary); }
.content { flex: 1; display: flex; gap: 0; padding: var(--space-md); overflow: hidden; }
.avatar-render { flex: 1; background: transparent; overflow: hidden; position: relative; border: 1px solid var(--border-color); }
.header { padding: var(--space-md); background: var(--bg-primary); border-bottom: 1px solid var(--border-color); }
.header h2 { margin: 0; font-size: 24px; font-family: var(--font-heading); font-weight: 400; letter-spacing: -0.01em; }
.description { margin: 4px 0 0 0; color: var(--text-muted); font-size: 13px; }
.sdk-container { height: 100%; }
.sdk-container canvas { object-fit: cover !important; width: 100% !important; }
.sdk-container :deep(canvas), .sdk-container :deep(video) { display: block; width: 100%; height: 100%; object-fit: contain; }
@media (min-width: 1025px) { .sdk-container { aspect-ratio: 16/9 !important; margin: 0 auto; } }
.subtitle, .voice-animation, .loading-placeholder { position: absolute; left: 50%; transform: translateX(-50%); }
.subtitle { bottom: 200px; width: 375px; max-width: 90%; word-break: break-word; text-align: center; font-size: 18px; color: #fff; padding: 8px 16px; z-index: 100; font-family: var(--font-heading); font-style: italic; letter-spacing: 0.02em; background: rgba(0,0,0,0.45); border-radius: 8px; }
.voice-animation { top: 75%; width: 360px; max-width: 90%; z-index: 101; }
.voice-animation > img { width: 100%; height: auto; }
.loading-placeholder { display: flex; justify-content: center; align-items: center; top: 0; left: 0; right: 0; bottom: 0; transform: none; z-index: 100; background: rgba(0,0,0,0.3); }
.loading-text { font-size: 16px; color: rgba(255,255,255,0.6); font-family: var(--font-mono); letter-spacing: 0.1em; }
.input-area { position: absolute; bottom: 0; left: 0; right: 0; z-index: 102; padding: 8px 12px 12px; background: linear-gradient(to top, rgba(0,0,0,0.45) 0%, transparent 100%); }
.input-container { display: flex; align-items: center; gap: 8px; background: rgba(255,255,255,0.95); border-radius: 24px; padding: 4px 4px 4px 16px; margin: 0 auto; max-width: 600px; }
.input-text { flex: 1; border: none; outline: none; background: transparent; color: var(--text-primary); font-size: 15px; line-height: 1.4; resize: none; max-height: 100px; overflow-y: auto; padding: 8px 0; font-family: var(--font-body); }
.input-text::placeholder { color: var(--text-muted); }
.btn, .ctrl, .panel-toggle, .mobile-panel-close { cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.15s ease; }
.btn { width: 38px; height: 38px; border: none; border-radius: 50%; background: transparent; color: var(--text-muted); padding: 0; flex-shrink: 0; }
.btn:disabled { opacity: 0.3; cursor: not-allowed; }
.btn:hover:not(:disabled) { color: var(--text-primary); background: rgba(0,0,0,0.06); }
.btn-voice.active { background: var(--accent); color: #fff; animation: pulse 2s ease-in-out infinite; }
.btn-send { color: var(--accent); }
.btn-send:hover:not(:disabled) { color: #fff; background: var(--accent); }
.ctrl { flex: 1; min-width: 70px; height: 34px; font-size: 12px; letter-spacing: 0.06em; border: 1px solid rgba(255,255,255,0.5); background: transparent; color: rgba(255,255,255,0.85); padding: 0 12px; }
.ctrl:hover:not(:disabled) { background: rgba(255,255,255,0.2); color: #fff; border-color: rgba(255,255,255,0.8); }
.control-panel { position: absolute; bottom: 100px; left: 50%; transform: translateX(-50%); width: calc(100% - 48px); max-width: 600px; z-index: 150; display: flex; flex-direction: column; gap: 6px; padding: 8px; border-radius: 12px; background: rgba(0,0,0,0.35); backdrop-filter: blur(4px); }
.control-row { display: flex; flex-wrap: wrap; gap: 6px; }
.icon { width: 18px; height: 18px; fill: currentColor; }
.panel { width: 480px; background: var(--bg-primary); border-left: 1px solid var(--border-color); overflow-y: auto; transition: all 0.2s ease; display: flex; flex-direction: column; flex-shrink: 0; }
.panel-header, .mobile-panel-header { display: flex; justify-content: space-between; align-items: center; flex-shrink: 0; border-bottom: 1px solid var(--border-color); padding: 20px var(--space-md); }
.panel-header h3, .mobile-panel-header h3 { font-family: var(--font-heading); font-size: 20px; font-weight: 400; color: var(--text-primary); letter-spacing: -0.01em; }
.panel-toggle, .mobile-panel-close { width: 32px; height: 32px; border: 1px solid var(--border-color); background: transparent; color: var(--text-muted); padding: 0; }
.panel-toggle:hover, .mobile-panel-close:hover { border-color: var(--text-primary); color: var(--text-primary); background: transparent; }
.panel-toggle:active, .floating-settings-btn:active { transform: scale(0.95); }
.panel-content, .mobile-panel-content { flex: 1; overflow-y: auto; }
.panel-collapsed { width: auto; }
.panel-collapsed .panel-content { display: none; }
.panel-collapsed .panel-header { border-bottom: none; }
.floating-settings-btn { position: fixed; top: 16px; right: 16px; width: 44px; height: 44px; border: 1px solid var(--text-primary); background: var(--bg-primary); color: var(--text-primary); padding: 0; z-index: 10000; transition: all 0.15s ease; }
.floating-settings-btn:hover { background: var(--text-primary); color: var(--bg-primary); }
.floating-settings-btn.active { border-color: var(--accent); background: var(--accent); color: #fff; }
.mobile-panel-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.3); z-index: 9999; display: flex; align-items: flex-end; animation: fadeIn 0.2s ease; }
.mobile-panel { width: 100%; max-height: 80vh; background: var(--bg-primary); display: flex; flex-direction: column; animation: slideUp 0.2s ease; border-top: 1px solid var(--border-color); }
.mobile-panel-header { padding: 16px var(--space-md); }
@keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.8; } }
@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
@keyframes slideUp { from { transform: translateY(100%); } to { transform: translateY(0); } }
@media (max-width: 1023px) { .header { display: none; } .content { flex-direction: column; padding: 0; gap: 0; height: 100%; } .avatar-render { border: none; } .panel { display: none; } .subtitle { bottom: 160px; } }
@media (max-width: 768px) { .input-area { padding: 6px 8px; } .input-container { border-radius: 22px; padding: 2px 2px 2px 14px; } .input-text { font-size: 14px; padding: 6px 0; max-height: 80px; } .btn { width: 36px; height: 36px; } .icon { width: 20px; height: 20px; } .control-panel { bottom: 70px; width: calc(100% - 16px); } .ctrl { font-size: 11px; height: 30px; } }
</style>
