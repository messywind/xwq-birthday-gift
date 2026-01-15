<script setup>
import { ref, computed } from 'vue';
import NorthStarBg from './components/NorthStarBg.vue';
// 引入新组件
import PhotoCarousel from './components/PhotoCarousel.vue';
import GiftReveal from './components/GiftReveal.vue';

// 状态定义：'INTRO' | 'MAIN' | 'GIFT'
const viewState = ref('INTRO');
const bgAudio = ref(null);

const isIntro = computed(() => viewState.value === 'INTRO');
const isMain = computed(() => viewState.value === 'MAIN');
const isGift = computed(() => viewState.value === 'GIFT');
// 只要不是 Intro 状态，背景人物就应该显示
const showHeroes = computed(() => viewState.value !== 'INTRO');

// 祝福语
const blessingTitle = "TO: 召唤师";
const blessingText = [
  "在极寒的弗雷尔卓德星空下",
  "愿指引你的北极星永远闪耀",
  "祝你像黛安娜一样优雅坚韧",
  "生日快乐，传奇永不熄灭！"
];

const startExperience = () => {
  viewState.value = 'MAIN';
  if (bgAudio.value) {
    bgAudio.value.volume = 0.4;
    bgAudio.value.play().catch(e => console.log("Autoplay prevented", e));
  }
};

const showGift = () => {
  viewState.value = 'GIFT';
}
</script>

<template>
  <main class="main-stage">
    <audio ref="bgAudio" loop src="/bgm.mp3"></audio>
    <NorthStarBg />

    <div class="hero-layer diana" :class="{ 'active': showHeroes }"></div>
    <div class="hero-layer zed" :class="{ 'active': showHeroes }"></div>

    <transition name="fade">
      <div v-if="isIntro" class="screen-center intro-screen">
        <div class="hex-btn start-btn" @click="startExperience">
          <span>开启星界</span>
          <div class="hex-glow"></div>
        </div>
      </div>
    </transition>

    <transition name="fade-slide" mode="out-in">

      <div v-if="isMain" class="screen-center card-glass main-card-layout">
        <div class="content-inner">
          <div class="rune-icon">✧</div>
          <h1>{{ blessingTitle }}</h1>
          <div class="divider"></div>
          <div class="text-block">
            <p v-for="(line, index) in blessingText" :key="index" :style="{ animationDelay: `${index * 0.2}s` }">
              {{ line }}
            </p>
          </div>
        </div>

        <PhotoCarousel />

        <div class="hex-btn next-btn" @click="showGift">
          <span>查看神秘掉落</span>
        </div>
      </div>

      <div v-else-if="isGift" class="screen-center card-glass gift-card-layout">
        <GiftReveal />
      </div>

    </transition>
  </main>
</template>

<style>
/* ... (保留之前的全局样式: body, html, main-stage, hero-layer 等) ... */
body, html {
  margin: 0; padding: 0; overflow: hidden;
  background-color: #050810;
  font-family: 'Cinzel', serif;
  color: #fff;
}
.main-stage { position: relative; width: 100vw; height: 100vh; }
.hero-layer {
  position: absolute; top: 0; bottom: 0; width: 60%;
  background-size: cover; background-repeat: no-repeat;
  z-index: 2; opacity: 0;
  transition: opacity 2s ease-in-out, transform 10s ease-out;
  mix-blend-mode: screen;
  filter: drop-shadow(0 0 20px rgba(0, 255, 255, 0.3));
  pointer-events: none; /* 防止遮挡鼠标事件 */
}
.hero-layer.active { opacity: 0.6; transform: scale(1.05); }
.diana { left: -5%; background-image: url('/diana.jpg'); background-position: center top; mask-image: linear-gradient(to right, black 20%, transparent 100%); -webkit-mask-image: linear-gradient(to right, black 20%, transparent 100%); }
.zed { right: -5%; background-image: url('/zed.jpg'); background-position: center top; mask-image: linear-gradient(to left, black 20%, transparent 100%); -webkit-mask-image: linear-gradient(to left, black 20%, transparent 100%); }


/* --- 新增/修改的样式 --- */

/* 通用居中容器 */
.screen-center {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  z-index: 10;
}

/* Intro 特定样式 */
.intro-screen {
  width: 100%; height: 100%;
  display: flex; justify-content: center; align-items: center;
  background: rgba(5, 8, 16, 0.8);
  backdrop-filter: blur(5px);
}

/* 通用海克斯按钮样式 */
.hex-btn {
  position: relative;
  padding: 15px 40px;
  border: 1px solid #c8aa6e;
  color: #c8aa6e;
  font-size: 1.2rem;
  cursor: pointer;
  overflow: hidden;
  transition: all 0.3s;
  letter-spacing: 2px;
  background: rgba(0, 20, 40, 0.6);
  box-shadow: 0 0 15px rgba(200, 170, 110, 0.2);
  text-align: center;
}
.hex-btn:hover {
  background: rgba(200, 170, 110, 0.2);
  box-shadow: 0 0 25px rgba(200, 170, 110, 0.6);
  text-shadow: 0 0 8px #c8aa6e;
}
.start-btn { font-size: 1.5rem; }
.next-btn { margin-top: 2rem; width: fit-content; margin-left: auto; margin-right: auto;}


/* 玻璃卡片容器修改 */
.card-glass {
  width: auto; /* 宽度自适应内容 */
  max-width: 90%;
  max-height: 90vh; /* 防止过高 */
  overflow-y: auto; /* 如果内容太多允许滚动 */
  padding: 2rem 3rem;
  background: rgba(10, 20, 40, 0.5);
  border: 1px solid rgba(0, 210, 255, 0.3);
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.5), inset 0 0 20px rgba(0, 210, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 4px;
  /* 隐藏滚动条但保留功能 */
  scrollbar-width: none;
  -ms-overflow-style: none;
}
.card-glass::-webkit-scrollbar { display: none; }

/* 主界面和礼物界面的特定布局微调 */
.main-card-layout { display: flex; flex-direction: column; align-items: center; }
.gift-card-layout { padding: 3rem; }


/* 内容样式 */
.content-inner { text-align: center; margin-bottom: 1rem; }
.content-inner h1 {
  font-size: 1.8rem; margin-bottom: 0.5rem;
  background: linear-gradient(to bottom, #f0e6d2, #c8aa6e);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent;
}
.divider { height: 2px; background: linear-gradient(90deg, transparent, #00d2ff, transparent); margin: 1rem auto; width: 60%; }
.text-block p {
  font-size: 1rem; line-height: 1.6; color: #e0faff; margin: 0.3rem 0;
  opacity: 0; animation: fadeSlideUp 0.8s forwards;
}
.rune-icon { font-size: 1.2rem; color: #00d2ff; animation: pulse 3s infinite; }

/* --- 动画定义 --- */
@keyframes fadeSlideUp {
  from { opacity: 0; transform: translateY(15px); }
  to { opacity: 1; transform: translateY(0); }
}
@keyframes pulse {
  0%, 100% { opacity: 0.5; text-shadow: 0 0 5px #00d2ff; }
  50% { opacity: 1; text-shadow: 0 0 20px #00d2ff; }
}

/* Vue Transitions */
/* Fade (用于 Intro) */
.fade-enter-active, .fade-leave-active { transition: opacity 0.8s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

/* Fade-Slide (用于主界面和礼物界面切换) */
.fade-slide-enter-active, .fade-slide-leave-active { transition: all 0.6s cubic-bezier(0.4, 0, 0.2, 1); }
.fade-slide-enter-from { opacity: 0; transform: translate(-50%, -40%) scale(0.95); } /* 进入时稍微有点缩放和位移 */
.fade-slide-leave-to { opacity: 0; transform: translate(-50%, -60%) scale(1.05); } /* 离开时向上飘并放大一点 */
</style>