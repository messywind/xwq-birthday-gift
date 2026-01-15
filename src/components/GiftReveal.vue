<script setup>
// 不需要额外的逻辑，只是展示
</script>

<template>
  <div class="gift-container">
    <h2 class="gift-title">生日专属赠礼</h2>

    <div class="skin-frame">
      <img src="/gift-skin.jpg" alt="Skin Gift" class="skin-img">

      <div class="frame-overlay"></div>
      <div class="frame-corner top-left"></div>
      <div class="frame-corner top-right"></div>
      <div class="frame-corner bottom-left"></div>
      <div class="frame-corner bottom-right"></div>
    </div>

    <div class="action-area">
      <p class="gift-tip">检测到来自弗雷尔卓德的礼物信标</p>
      <div class="login-btn">
        → 立即上号领取 ←
      </div>
    </div>
  </div>
</template>

<style scoped>
.gift-container {
  text-align: center;
  color: #c8aa6e;
}

.gift-title {
  font-size: 2rem;
  margin-bottom: 1.5rem;
  text-shadow: 0 0 10px rgba(200, 170, 110, 0.5);
  background: linear-gradient(to bottom, #f0e6d2, #c8aa6e);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

/* --- 皮肤框核心样式 --- */
.skin-frame {
  position: relative;
  width: 300px;
  height: 450px; /* 竖版加载图比例 */
  margin: 0 auto;
  /* LoL 风格的深色金边底座 */
  background: #050810;
  border: 3px solid #c8aa6e;
  box-shadow:
      0 0 30px rgba(0, 210, 255, 0.4), /* 青色外发光 */
      inset 0 0 30px rgba(0, 0, 0, 0.8);
  overflow: hidden;
}

.skin-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  /* 稍微放大一点并做一点缓慢移动的效果 */
  animation: panImage 20s infinite alternate ease-in-out;
}

/* 内发光遮罩 */
.frame-overlay {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  box-shadow: inset 0 0 50px rgba(200, 170, 110, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.1);
  pointer-events: none;
}

/* --- 装饰性边角 (模拟 Hextech 螺丝) --- */
.frame-corner {
  position: absolute;
  width: 15px;
  height: 15px;
  background: #c8aa6e;
  box-shadow: 0 0 5px #00d2ff;
}
.top-left { top: -5px; left: -5px; }
.top-right { top: -5px; right: -5px; }
.bottom-left { bottom: -5px; left: -5px; }
.bottom-right { bottom: -5px; right: -5px; }

/* --- 底部操作区 --- */
.action-area {
  margin-top: 2rem;
}

.gift-tip {
  color: #00d2ff;
  font-size: 0.9rem;
  margin-bottom: 1rem;
  letter-spacing: 1px;
}

.login-btn {
  display: inline-block;
  padding: 12px 30px;
  background: linear-gradient(45deg, #001428, #002846);
  border: 2px solid #c8aa6e;
  color: #f0e6d2;
  font-size: 1.2rem;
  font-weight: bold;
  letter-spacing: 2px;
  cursor: pointer;
  box-shadow: 0 0 15px rgba(200, 170, 110, 0.3);
  transition: all 0.3s;
  animation: pulseBtn 2s infinite;
}

.login-btn:hover {
  background: #c8aa6e;
  color: #050810;
  box-shadow: 0 0 30px rgba(200, 170, 110, 0.8);
}

@keyframes panImage {
  from { transform: scale(1.1) translateY(0); }
  to { transform: scale(1.1) translateY(-20px); }
}

@keyframes pulseBtn {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); box-shadow: 0 0 25px rgba(0, 210, 255, 0.6); }
}
</style>