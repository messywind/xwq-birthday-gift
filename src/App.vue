<script setup>
  import { ref, onMounted } from 'vue';
  import AuroraScene from './components/AuroraScene.vue';
  import GiftReveal from './components/GiftReveal.vue'; // 确保这个文件存在
  
  const stage = ref('INTRO');
  const bgAudio = ref(null);
  const isShaking = ref(false); // 控制屏幕抖动
  
  // --- 音效管理器 ---
  const sfxPlayers = {};
  // 预加载音效
  onMounted(() => {
      // 请确保 public 文件夹下有这4个音频文件
      sfxPlayers.whoosh = new Audio('/sfx-whoosh.mp3');
      sfxPlayers.whoosh.volume = 0.6;
      sfxPlayers.moonBurst = new Audio('/sfx-moon-burst.mp3');
      sfxPlayers.moonBurst.volume = 0.8;
      sfxPlayers.zedLaugh = new Audio('/sfx-zed-laugh.mp3');
      sfxPlayers.zedPop = new Audio('/sfx-zed-pop.mp3');
      sfxPlayers.zedPop.volume = 1.0;
  });
  
  // 提供给子组件调用的播放函数
  const handlePlaySfx = (sfxName) => {
      const player = sfxPlayers[sfxName];
      if (player) {
          player.currentTime = 0; // 每次从头播放
          player.play().catch(() => {}); // 忽略自动播放限制的报错
      }
  };
  // ----------------
  
  const memoryPhotos = ['/p1.png', '/p2.png', '/p3.png', '/p4.png', '/p5.png'];
  const blessingText = ["寒冰血脉见证此刻", "如北极星般指引方向", "如影流之主般掌控命运", "生日快乐，最强王者！"];
  
  const startSequence = () => {
    stage.value = 'ANIMATING';
    if (bgAudio.value) {
      bgAudio.value.volume = 0.4;
      bgAudio.value.play().catch(() => {});
    }
  };
  
  // 3D 动画大招爆炸时触发的回调
  const onSceneAnimationComplete = () => {
    // 触发剧烈的屏幕抖动
    isShaking.value = true;
    setTimeout(() => {
        isShaking.value = false;
        stage.value = 'CARD'; // 显示贺卡
    }, 800); // 抖动持续时间要配合爆炸音效的长度
  };
  
  const showGift = () => { stage.value = 'GIFT'; };
  </script>
  
  <template>
    <main class="main-container" :class="{ 'screen-shake': isShaking }">
      <audio ref="bgAudio" loop src="/bgm.mp3"></audio>
      
      <AuroraScene 
        :startTrigger="stage === 'ANIMATING'" 
        :photos="memoryPhotos"
        @animation-complete="onSceneAnimationComplete"
        @play-sfx="handlePlaySfx"
      />
  
      <div class="hero-wrapper" :class="{ 'faded': stage !== 'INTRO' && stage !== 'CARD' }">
          <div class="hero diana"></div>
          <div class="hero zed"></div>
      </div>
  
      <div class="ui-layer">
        <transition name="fade">
          <div v-if="stage === 'INTRO'" class="intro-box">
            <button class="hex-btn start-btn" @click="startSequence">
              <span>开启星界</span>
            </button>
          </div>
        </transition>
  
        <transition name="pop-in">
          <div v-if="stage === 'CARD'" class="card-glass">
              <div class="rune-header">✧ NORTH STAR ✧</div>
              <h1 class="summoner-name">TO: 召唤师</h1>
              <div class="text-content">
                  <p v-for="(t, i) in blessingText" :key="i" :style="{animationDelay: i*0.2 + 0.5 + 's'}">
                      {{ t }}
                  </p>
              </div>
              <div class="btn-row">
                  <button class="hex-btn" @click="showGift">查看掉落</button>
              </div>
          </div>
        </transition>
  
        <transition name="fade">
          <div v-if="stage === 'GIFT'" class="gift-wrapper">
              <GiftReveal />
          </div>
        </transition>
      </div>
    </main>
  </template>
  
  <style>
  /* ... 保持之前的样式不变 ... */
  body { margin: 0; background: #000; color: #fff; font-family: 'Cinzel', serif; overflow: hidden; }
  .main-container { position: relative; width: 100vw; height: 100vh; transition: transform 0.1s; }
  .ui-layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 10; pointer-events: none; display: flex; justify-content: center; align-items: center; }
  .ui-layer > * { pointer-events: auto; }
  .hero-wrapper { position: absolute; width: 100%; height: 100%; z-index: 2; transition: opacity 0.8s; pointer-events: none; }
  .hero-wrapper.faded { opacity: 0; }
  .hero { position: absolute; top: 0; bottom: 0; width: 50%; background-size: cover; mix-blend-mode: screen; opacity: 0.6; }
  .diana { left: 0; background-image: url('/diana.jpg'); mask-image: linear-gradient(to right, black, transparent); }
  .zed { right: 0; background-image: url('/zed.jpg'); mask-image: linear-gradient(to left, black, transparent); }
  
  /* --- 新增：剧烈屏幕抖动 --- */
  .screen-shake {
      animation: rough-shake 0.8s cubic-bezier(.36,.07,.19,.97) both;
      transform: translate3d(0, 0, 0);
      backface-visibility: hidden;
      perspective: 1000px;
  }
  @keyframes rough-shake {
    10%, 90% { transform: translate3d(-5px, 0, 0) rotate(-1deg); }
    20%, 80% { transform: translate3d(10px, 0, 0) rotate(2deg); }
    30%, 50%, 70% { transform: translate3d(-15px, 0, 0) rotate(-3deg) scale(1.02); filter: hue-rotate(30deg); } /* 抖动时带一点色偏和放大 */
    40%, 60% { transform: translate3d(15px, 0, 0) rotate(3deg) scale(1.02); }
  }
  
  /* --- 贺卡调整 --- */
  .card-glass { background: rgba(10, 20, 35, 0.8); border: 2px solid #00d2ff; padding: 3rem; text-align: center; backdrop-filter: blur(15px); box-shadow: 0 0 80px rgba(0, 210, 255, 0.4); border-radius: 8px; }
  .summoner-name { font-size: 2.8rem; background: linear-gradient(to right, #c8aa6e, #fff, #c8aa6e); -webkit-background-clip: text; -webkit-text-fill-color: transparent; margin: 1rem 0; text-shadow: 0 0 20px rgba(200, 170, 110, 0.5); }
  .text-content p { color: #bfefff; font-size: 1.2rem; margin: 0.6rem 0; opacity: 0; animation: textFadeIn 0.8s forwards; text-shadow: 0 0 5px #00d2ff; }
  .btn-row { margin-top: 2rem; }
  @keyframes textFadeIn { to { opacity: 1; } }
  .pop-in-enter-active { animation: popIn 0.6s cubic-bezier(0.34, 1.56, 0.64, 1); }
  @keyframes popIn { 0% { opacity: 0; transform: scale(0.8) translateY(50px); } 100% { opacity: 1; transform: scale(1) translateY(0); } }
  
  /* --- 按钮样式调整 --- */
  .hex-btn { background: linear-gradient(45deg, rgba(0,20,40,0.8), rgba(0,40,80,0.8)); border: 2px solid #c8aa6e; color: #c8aa6e; padding: 15px 40px; font-size: 1.3rem; cursor: pointer; transition: 0.3s; position: relative; overflow: hidden; letter-spacing: 3px; box-shadow: 0 0 20px rgba(200,170,110,0.2); }
  .hex-btn:hover { background: rgba(200, 170, 110, 0.3); box-shadow: 0 0 40px rgba(200, 170, 110, 0.6), inset 0 0 10px #c8aa6e; text-shadow: 0 0 10px #c8aa6e; }
  .fade-enter-active, .fade-leave-active { transition: opacity 0.5s; }
  .fade-enter-from, .fade-leave-to { opacity: 0; }
  </style>