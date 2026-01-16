<script setup>
import { onMounted, onBeforeUnmount, ref, watch } from 'vue';
import * as THREE from 'three';
import gsap from 'gsap';

const props = defineProps({
  startTrigger: Boolean,
  photos: Array
});

const emit = defineEmits(['animation-complete']);

const container = ref(null);
let scene, camera, renderer;
let auroraMesh, starField, photoGroup;
let snowSystem, zedParticles, shadowClonesGroup;
let animationId;
let time = 0;
let zedTexture, snowTexture;

// --- Web Audio API (去掉了定格音效) ---
const AudioContext = window.AudioContext || window.webkitAudioContext;
const audioCtx = new AudioContext();

function createReverb(duration = 1.5, decay = 2.0, reverse = false) {
  const sampleRate = audioCtx.sampleRate;
  const length = sampleRate * duration;
  const impulse = audioCtx.createBuffer(2, length, sampleRate);
  const left = impulse.getChannelData(0);
  const right = impulse.getChannelData(1);
  for (let i = 0; i < length; i++) {
    let n = reverse ? length - i : i;
    left[i] = (Math.random() * 2 - 1) * Math.pow(1 - n / length, decay);
    right[i] = (Math.random() * 2 - 1) * Math.pow(1 - n / length, decay);
  }
  const convolver = audioCtx.createConvolver();
  convolver.buffer = impulse;
  return convolver;
}

const masterReverb = createReverb();
masterReverb.connect(audioCtx.destination);

const SoundSynth = {
  // 1. 风声 (保留)
  playWhoosh: () => {
    if (audioCtx.state === 'suspended') audioCtx.resume();
    const t = audioCtx.currentTime;
    const duration = 0.3;
    const bufferSize = audioCtx.sampleRate * duration;
    const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
    const data = buffer.getChannelData(0);
    let lastOut = 0;
    for (let i = 0; i < bufferSize; i++) {
      const white = Math.random() * 2 - 1;
      data[i] = (lastOut + (0.02 * white)) / 1.02;
      lastOut = data[i];
      data[i] *= 3.5;
    }
    const noise = audioCtx.createBufferSource();
    noise.buffer = buffer;
    const filter = audioCtx.createBiquadFilter();
    filter.type = 'highpass';
    filter.frequency.setValueAtTime(800, t);
    filter.frequency.linearRampToValueAtTime(2000, t + duration);
    const gain = audioCtx.createGain();
    gain.gain.setValueAtTime(0, t);
    gain.gain.linearRampToValueAtTime(0.3, t + 0.1);
    gain.gain.exponentialRampToValueAtTime(0.01, t + duration);
    noise.connect(filter);
    filter.connect(gain);
    gain.connect(audioCtx.destination);
    noise.start(t);
  },
  // (去掉了 playMoonBurst - 定格音效)

  // 2. 劫的爆炸 (保留)
  playZedPop: () => {
    if (audioCtx.state === 'suspended') audioCtx.resume();
    const t = audioCtx.currentTime;
    // 冲击层
    const bufferSize = audioCtx.sampleRate * 0.1;
    const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
    const noise = audioCtx.createBufferSource();
    noise.buffer = buffer;
    const noiseFilter = audioCtx.createBiquadFilter();
    noiseFilter.type = 'lowpass';
    noiseFilter.frequency.value = 3000;
    const noiseGain = audioCtx.createGain();
    noiseGain.gain.setValueAtTime(0.8, t);
    noiseGain.gain.exponentialRampToValueAtTime(0.01, t + 0.1);
    noise.connect(noiseFilter);
    noiseFilter.connect(noiseGain);
    noiseGain.connect(masterReverb);
    noiseGain.connect(audioCtx.destination);
    noise.start(t);
    // 能量层
    const osc = audioCtx.createOscillator();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(800, t);
    osc.frequency.exponentialRampToValueAtTime(50, t + 0.4);
    const oscGain = audioCtx.createGain();
    oscGain.gain.setValueAtTime(0.5, t);
    oscGain.gain.exponentialRampToValueAtTime(0.01, t + 0.4);
    osc.connect(oscGain);
    oscGain.connect(masterReverb);
    oscGain.connect(audioCtx.destination);
    osc.start(t);
    osc.stop(t + 0.5);
  }
};

// --- Shader (极光) ---
const vertexShader = `varying vec2 vUv; void main() { vUv = uv; vec4 mvPosition = modelViewMatrix * vec4(position, 1.0); gl_Position = projectionMatrix * mvPosition; }`;
const fragmentShader = `
uniform float uTime;
varying vec2 vUv;
float random(vec2 st) { return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123); }
float noise(vec2 st) { vec2 i = floor(st); vec2 f = fract(st); float a = random(i); float b = random(i + vec2(1.0, 0.0)); float c = random(i + vec2(0.0, 1.0)); float d = random(i + vec2(1.0, 1.0)); vec2 u = f * f * (3.0 - 2.0 * f); return mix(a, b, u.x) + (c - a)* u.y * (1.0 - u.x) + (d - b) * u.x * u.y; }
void main() {
    vec2 uv = vUv;
    vec2 scaledUV = uv * vec2(20.0, 10.0);
    float wave = sin(uv.x * 15.0 + uTime * 0.4) * 0.08;
    float n = noise(scaledUV - vec2(0, uTime * 0.6));
    float strength = smoothstep(0.1, 0.9, n) * smoothstep(1.0, 0.3, uv.y + wave * 2.0);
    vec3 color1 = vec3(0.0, 0.8, 0.9);
    vec3 color2 = vec3(0.6, 0.0, 0.8);
    vec3 finalColor = mix(color1, color2, uv.y + n * 0.3);
    gl_FragColor = vec4(finalColor, strength * 0.8 * smoothstep(0.0, 0.2, uv.y));
}
`;

function init() {
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x050815, 0.001);

  camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 4000);
  camera.position.set(0, -10, 20);
  camera.lookAt(0, 10, -100);

  renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.capabilities.logarithmicDepthBuffer = true;
  container.value.appendChild(renderer.domElement);

  const texLoader = new THREE.TextureLoader();
  // 【修改】劫的影分身贴图：使用在线烟雾/暗影图，确保一定能显示
  zedTexture = texLoader.load('/zed-sprite.png');
  // 如果你有本地图片，请改回 '/zed-sprite.png'，但必须确保图片是透明底的白色或亮色图案，因为我们稍后会给它着色

  snowTexture = texLoader.load('/snowflake.png');

  createAurora();
  createStarField();
  createPhotoStream();
  createSnowSystem();
  createZedParticles();
  createShadowClones();

  animate();
}

function createAurora() {
  const geometry = new THREE.PlaneGeometry(4000, 2000, 128, 64);
  const material = new THREE.ShaderMaterial({
    uniforms: { uTime: { value: 0 } },
    vertexShader, fragmentShader,
    transparent: true, side: THREE.DoubleSide, depthWrite: false, blending: THREE.AdditiveBlending
  });
  auroraMesh = new THREE.Mesh(geometry, material);
  auroraMesh.position.set(0, 200, -1000);
  scene.add(auroraMesh);
}

function createStarField() {
  const geometry = new THREE.BufferGeometry();
  const count = 10000;
  const positions = new Float32Array(count * 3);
  for(let i=0; i<count; i++) {
    positions[i*3] = (Math.random() - 0.5) * 2000;
    positions[i*3+1] = (Math.random() - 0.5) * 1000;
    positions[i*3+2] = Math.random() * -1500;
  }
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const material = new THREE.PointsMaterial({
    color: 0xa0e0ff, size: 0.5, transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending, fog: true
  });
  starField = new THREE.Points(geometry, material);
  scene.add(starField);
}

function createPhotoStream() {
  photoGroup = new THREE.Group();
  const loader = new THREE.TextureLoader();
  const targetHeight = 12;

  props.photos.forEach((url, i) => {
    const geometry = new THREE.PlaneGeometry(1, 1);
    const material = new THREE.MeshBasicMaterial({ transparent: true, opacity: 0, side: THREE.DoubleSide });
    const mesh = new THREE.Mesh(geometry, material);

    loader.load(url, (texture) => {
      mesh.material.map = texture;
      mesh.material.needsUpdate = true;
      const img = texture.image;
      const aspect = img.width / img.height;
      mesh.scale.set(targetHeight * aspect, targetHeight, 1);
    });

    const angle = i * 1.5;
    const radius = 18;
    mesh.position.set(Math.cos(angle) * radius, Math.sin(angle) * radius + 5, -150 - (i * 100));
    mesh.rotation.set(Math.random()*0.3, Math.random()*0.3, Math.random()*0.1);
    photoGroup.add(mesh);
  });
  scene.add(photoGroup);
}

function createSnowSystem() {
  const particleCount = 1500;
  const geometry = new THREE.BufferGeometry();
  const positions = new Float32Array(particleCount * 3);
  const velocities = new Float32Array(particleCount * 3);

  for (let i = 0; i < particleCount; i++) {
    positions[i * 3] = (Math.random() - 0.5) * 100;
    positions[i * 3 + 1] = (Math.random() - 0.5) * 100;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 100;
    velocities[i * 3] = (Math.random() - 0.5) * 0.2;
    velocities[i * 3 + 1] = (Math.random() - 0.5) * 0.2 - 0.05;
    velocities[i * 3 + 2] = (Math.random() - 0.5) * 0.2;
  }
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));
  const material = new THREE.PointsMaterial({
    size: 2.5,
    map: snowTexture,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    transparent: true,
    opacity: 0,
    color: 0xa0e0ff
  });
  snowSystem = new THREE.Points(geometry, material);
  scene.add(snowSystem);
}

function showSnowAt(position) {
  snowSystem.position.copy(position);
  gsap.to(snowSystem.material, { opacity: 0.8, duration: 1, ease: "power2.out" });
}

function hideSnow() {
  gsap.to(snowSystem.material, { opacity: 0, duration: 0.5, ease: "power2.in" });
}

function createExplosionSystem(color, count, size) {
  const geometry = new THREE.BufferGeometry();
  const positions = new Float32Array(count * 3);
  const velocities = new Float32Array(count * 3);
  for (let i = 0; i < count; i++) {
    positions[i*3] = 0; positions[i*3+1] = 0; positions[i*3+2] = 0;
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(Math.random() * 2 - 1);
    const speed = Math.random() * 3 + 1;
    velocities[i*3] = speed * Math.sin(phi) * Math.cos(theta);
    velocities[i*3+1] = speed * Math.sin(phi) * Math.sin(theta);
    velocities[i*3+2] = speed * Math.cos(phi);
  }
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));
  const material = new THREE.PointsMaterial({ color: color, size: size, transparent: true, opacity: 0, blending: THREE.AdditiveBlending, depthWrite: false });
  return new THREE.Points(geometry, material);
}

function createZedParticles() {
  zedParticles = createExplosionSystem(0xff004c, 2500, 1.5);
  scene.add(zedParticles);
}

// 【关键修改】影分身创建逻辑
function createShadowClones() {
  shadowClonesGroup = new THREE.Group();
  shadowClonesGroup.visible = false;

  // 使用深紫色着色，确保即使贴图是白色的也能显示为暗影
  const material = new THREE.SpriteMaterial({
    map: zedTexture,
    color: 0x8800ff, // 暗紫色
    transparent: true,
    opacity: 0, // 初始透明
    blending: THREE.AdditiveBlending
  });

  for(let i=0; i<3; i++) {
    const sprite = new THREE.Sprite(material);
    // 【关键】加大尺寸，确保看得见
    sprite.scale.set(40, 40, 1);
    shadowClonesGroup.add(sprite);
  }
  scene.add(shadowClonesGroup);
}

function triggerParticles(system, position) {
  system.material.opacity = 1;
  system.position.copy(position);
  const positions = system.geometry.attributes.position.array;
  for(let i=0; i<positions.length; i++) positions[i] = 0;
  system.geometry.attributes.position.needsUpdate = true;
  gsap.to(system.material, { opacity: 0, duration: 1.5, ease: "power2.out" });
}

watch(() => props.startTrigger, async (val) => {
  if (val) {
    await playCinematicSequence();
  }
});

async function playCinematicSequence() {
  // 0. 蓄力
  gsap.to(camera, { fov: 110, duration: 1.5, ease: "power2.inOut" });
  gsap.to(scene.fog, { density: 0.0025, duration: 1.5 });
  gsap.to(scene.fog.color, { r: 0.05, g: 0.05, b: 0.2, duration: 1.5 });
  await gsap.to(camera.position, { z: 50, duration: 1.5 });

  // 1. 照片流循环
  for (let i = 0; i < photoGroup.children.length; i++) {
    const photo = photoGroup.children[i];

    // A. 冲刺
    hideSnow();
    SoundSynth.playWhoosh();
    gsap.to(photo.material, { opacity: 1, duration: 0.3 });
    await gsap.to(camera.position, {
      x: photo.position.x * 0.8,
      y: photo.position.y,
      z: photo.position.z + 40,
      duration: 0.7,
      ease: "expo.inOut"
    });

    // B. 定格 (去掉了声音)
    // SoundSynth.playMoonBurst(); <--- 已注释
    showSnowAt(photo.position);

    gsap.to(camera.position, { z: photo.position.z + 10, duration: 1.5, ease: "linear" });
    gsap.to(photo.rotation, { x: 0, y: 0, z: 0, duration: 1.5 });

    await gsap.to({}, { duration: 1.5 });

    gsap.to(photo.material, { opacity: 0.5, duration: 0.5 });
    hideSnow();
    await gsap.to({}, { duration: 0.5 });

    if (i < photoGroup.children.length - 1) {
      gsap.fromTo(camera, { fov: 120 }, { fov: 100, duration: 0.5 });
    }
  }

  // 2. 终局：劫的瞬狱影杀阵
  // 【修改】缩短距离，确保分身在雾气范围内可见
  gsap.to(camera.position, { z: -250, duration: 1, ease: "power2.in" });

  shadowClonesGroup.visible = true;
  shadowClonesGroup.children.forEach((clone, i) => {
    const angle = (i / 3) * Math.PI * 2;
    // 起始位置
    clone.position.set(Math.cos(angle)*80, Math.sin(angle)*80, -350);
    clone.material.opacity = 0;

    // 汇聚到相机前方 (-300)
    gsap.to(clone.position, { x: 0, y: 0, z: -300, duration: 0.8, ease: "back.in(1.5)" });
    // 确保不透明度变为1，显示出来
    gsap.to(clone.material, { opacity: 0.8, duration: 0.5 });
  });

  await gsap.delayedCall(0.8, () => {}).then();

  shadowClonesGroup.visible = false;
  SoundSynth.playZedPop();
  // 爆炸粒子位置也拉近
  triggerParticles(zedParticles, new THREE.Vector3(0, 0, -300));

  const flash = gsap.timeline();
  flash.call(() => { scene.fog.color.setHex(0xff0000); scene.fog.density = 0.05; })
      .to({}, { duration: 0.05 })
      .call(() => { scene.fog.color.setHex(0x000000); scene.fog.density = 1; })
      .to(scene.fog, { density: 0.001, duration: 2, ease: "power2.out", delay: 0.5 }, "recover")
      .to(scene.fog.color, { r: 0.02, g: 0.03, b: 0.08, duration: 2, ease: "power2.out", delay: 0.5 }, "recover");

  emit('animation-complete');
  gsap.to(camera.position, { z: 20, duration: 2, delay: 0.5, ease: "expo.out" });
  gsap.to(camera, { fov: 70, duration: 2, delay: 0.5 });
}

function animate() {
  time += 0.01;
  if (auroraMesh) auroraMesh.material.uniforms.uTime.value = time;

  if (starField) {
    starField.position.z += 3;
    const positions = starField.geometry.attributes.position.array;
    for(let i=2; i<positions.length; i+=3) {
      positions[i] += 3;
      if(positions[i] > 100) positions[i] = -1500;
    }
    starField.geometry.attributes.position.needsUpdate = true;
  }

  if (snowSystem && snowSystem.material.opacity > 0) {
    const positions = snowSystem.geometry.attributes.position.array;
    const velocities = snowSystem.geometry.attributes.velocity.array;
    const range = 50;
    for(let i = 0; i < positions.length; i += 3) {
      positions[i] += velocities[i];
      positions[i+1] += velocities[i+1];
      positions[i+2] += velocities[i+2];
      if (positions[i] > range) positions[i] -= range * 2;
      if (positions[i] < -range) positions[i] += range * 2;
      if (positions[i+1] > range) positions[i+1] -= range * 2;
      if (positions[i+1] < -range) positions[i+1] += range * 2;
      if (positions[i+2] > range) positions[i+2] -= range * 2;
      if (positions[i+2] < -range) positions[i+2] += range * 2;
    }
    snowSystem.geometry.attributes.position.needsUpdate = true;
    snowSystem.rotation.y += 0.002;
    snowSystem.rotation.z += 0.001;
  }

  [zedParticles].forEach(system => {
    if(system && system.material.opacity > 0) {
      const positions = system.geometry.attributes.position.array;
      const velocities = system.geometry.attributes.velocity.array;
      for(let i=0; i<positions.length; i+=3) {
        positions[i] += velocities[i];
        positions[i+1] += velocities[i+1];
        positions[i+2] += velocities[i+2];
        velocities[i+1] -= 0.03;
      }
      system.geometry.attributes.position.needsUpdate = true;
    }
  });

  camera.updateProjectionMatrix();
  renderer.render(scene, camera);
  animationId = requestAnimationFrame(animate);
}

onMounted(() => {
  init();
  window.addEventListener('resize', onResize);
});
onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onResize);
  if (renderer) renderer.dispose();
  if (audioCtx) audioCtx.close();
});

function onResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>

<template>
  <div ref="container" class="scene-container"></div>
</template>

<style scoped>
.scene-container {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  z-index: 1; background: #050815;
}
</style>