<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';

const container = ref(null);
let scene, camera, renderer, particles, starGeo, stars, northStar;
let animationId;

onMounted(() => {
  initThree();
  animate();
  window.addEventListener('resize', onResize);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onResize);
  renderer.dispose();
});

function initThree() {
  scene = new THREE.Scene();
  // 添加一点深邃的蓝紫色雾气，模拟极光氛围
  scene.fog = new THREE.FogExp2(0x0b1026, 0.001);

  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 1000);
  camera.position.z = 1;
  camera.rotation.x = 1.16;
  camera.rotation.y = -0.12;
  camera.rotation.z = 0.27;

  renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  container.value.appendChild(renderer.domElement);

  createParticles();
  createNorthStar();
  createAmbientLight();
}

function createParticles() {
  // 创建流动的星尘粒子
  starGeo = new THREE.BufferGeometry();
  const count = 6000;
  const positions = new Float32Array(count * 3);
  const velocities = [];

  for(let i=0; i<count; i++) {
    positions[i*3] = (Math.random() - 0.5) * 600;
    positions[i*3+1] = (Math.random() - 0.5) * 600;
    positions[i*3+2] = (Math.random() - 0.5) * 600;
    velocities.push(0);
  }

  starGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));

  // 粒子材质 - 青色与金色的混合
  const sprite = new THREE.TextureLoader().load('https://threejs.org/examples/textures/sprites/disc.png');
  const starMaterial = new THREE.PointsMaterial({
    color: 0x00d2ff, // 北极星黛安娜的青色
    size: 0.7,
    map: sprite,
    transparent: true,
    opacity: 0.8,
    blending: THREE.AdditiveBlending
  });

  stars = new THREE.Points(starGeo, starMaterial);
  scene.add(stars);
  stars.userData = { velocities };
}

function createNorthStar() {
  // 核心北极星 - 使用发光材质
  const geometry = new THREE.SphereGeometry(2, 32, 32);
  const material = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    toneMapped: false
  });
  northStar = new THREE.Mesh(geometry, material);
  northStar.position.set(0, 30, -100); // 放在远处上方

  // 给北极星添加光晕
  const light = new THREE.PointLight(0x00ffff, 2, 500);
  light.position.set(0, 30, -80);
  scene.add(light);
  scene.add(northStar);
}

function createAmbientLight() {
  const ambient = new THREE.AmbientLight(0x404040);
  scene.add(ambient);
}

function animate() {
  const positions = starGeo.attributes.position.array;

  // 粒子流动动画 - 模拟极光流动感
  for(let i=0; i<6000; i++) {
    // 让粒子缓慢向下流动，类似雪花或能量流
    positions[i*3 + 1] -= 0.2;

    // 循环重置
    if (positions[i*3 + 1] < -200) {
      positions[i*3 + 1] = 200;
    }
  }

  starGeo.attributes.position.needsUpdate = true;
  stars.rotation.y += 0.0005; // 整体缓慢旋转

  // 北极星脉冲效果
  const time = Date.now() * 0.001;
  const scale = 1 + Math.sin(time * 2) * 0.1;
  northStar.scale.set(scale, scale, scale);

  renderer.render(scene, camera);
  animationId = requestAnimationFrame(animate);
}

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>

<template>
  <div ref="container" class="three-container"></div>
</template>

<style scoped>
.three-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1; /* 最底层 */
  pointer-events: none;
}
</style>