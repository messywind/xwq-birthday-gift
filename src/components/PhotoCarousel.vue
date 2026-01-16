<script setup>
// 这里假设有5张照片。你可以根据实际情况修改数组。
const photos = [
  '/p1.png',
  '/p2.png',
  '/p3.png',
  '/p4.png',
  '/p5.png',
  '/p6.png',
  '/p7.png'
];
</script>

<template>
  <div class="carousel-container">
    <div class="carousel">
      <div
          v-for="(photo, index) in photos"
          :key="index"
          class="carousel__item"
          :style="{ '--i': index, '--total': photos.length }"
      >
        <img :src="photo" alt="memory" loading="lazy" />
        <div class="photo-border"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.carousel-container {
  perspective: 1000px; /* 3D 透视深 */
  width: 300px;
  height: 200px;
  margin: 40px auto 20px;
  position: relative;
}

.carousel {
  width: 100%;
  height: 100%;
  position: absolute;
  transform-style: preserve-3d; /* 关键：保留子元素的3D空间 */
  animation: rotate 20s infinite linear; /* 持续旋转动画 */
}

.carousel:hover {
  animation-play-state: paused; /* 鼠标悬停时暂停 */
}

.carousel__item {
  position: absolute;
  width: 180px;
  height: 240px;
  left: 50%;
  top: 50%;
  /* 计算每个图片的位置：先旋转到对应角度，再向外推 Z 轴 */
  transform: translate(-50%, -50%) rotateY(calc(var(--i) * (360deg / var(--total)))) translateZ(250px);
  backface-visibility: hidden;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 5px 15px rgba(0,0,0,0.5);
  transition: transform 0.3s;
}

/* 给照片加一个海克斯风格的边框 */
.photo-border {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  border: 2px solid rgba(0, 210, 255, 0.5);
  box-shadow: inset 0 0 15px rgba(0, 210, 255, 0.3);
  pointer-events: none;
  z-index: 2;
}

.carousel__item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  filter: brightness(0.8); /* 稍微压暗一点，突出氛围 */
  transition: filter 0.3s;
}

.carousel__item:hover img {
  filter: brightness(1.1);
}

@keyframes rotate {
  from { transform: rotateY(0deg); }
  to { transform: rotateY(360deg); }
}
</style>