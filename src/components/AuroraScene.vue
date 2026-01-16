<script setup>
  import { onMounted, onBeforeUnmount, ref, watch } from 'vue';
  import * as THREE from 'three';
  import gsap from 'gsap';
  
  const props = defineProps({
    startTrigger: Boolean,
    photos: Array
  });
  
  // 增加一个播放音效的事件通知
  const emit = defineEmits(['animation-complete', 'play-sfx']);
  
  const container = ref(null);
  let scene, camera, renderer;
  let auroraMesh, starField, photoGroup;
  let moonParticles, zedParticles, shadowClonesGroup; // 新增特效组
  let animationId;
  let time = 0;
  let zedTexture; // 劫的贴图
  
  // --- Shader (保持之前的极光 Shader，效果很好) ---
  const vertexShader = `varying vec2 vUv; void main() { vUv = uv; vec4 mvPosition = modelViewMatrix * vec4(position, 1.0); gl_Position = projectionMatrix * mvPosition; }`;
  const fragmentShader = `uniform float uTime; varying vec2 vUv; float random(vec2 st) { return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123); } float noise(vec2 st) { vec2 i = floor(st); vec2 f = fract(st); float a = random(i); float b = random(i + vec2(1.0, 0.0)); float c = random(i + vec2(0.0, 1.0)); float d = random(i + vec2(1.0, 1.0)); vec2 u = f * f * (3.0 - 2.0 * f); return mix(a, b, u.x) + (c - a)* u.y * (1.0 - u.x) + (d - b) * u.x * u.y; } void main() { vec2 uv = vUv; float wave = sin(uv.x * 10.0 + uTime * 0.5) * 0.1; float n = noise(uv * vec2(5.0, 20.0) - vec2(0, uTime * 0.8)); float strength = smoothstep(0.0, 0.8, n) * smoothstep(1.0, 0.6, uv.y + wave); vec3 color1 = vec3(0.0, 0.8, 0.9); vec3 color2 = vec3(0.6, 0.0, 0.8); vec3 finalColor = mix(color1, color2, uv.y + n * 0.2); gl_FragColor = vec4(finalColor, strength * 0.6 * (1.0 - uv.y)); }`;
  
  function init() {
    scene = new THREE.Scene();
    // 初始雾气：深蓝黑色
    scene.fog = new THREE.FogExp2(0x050815, 0.001);
  
    camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 2000);
    // 初始相机位置，稍微仰拍
    camera.position.set(0, -10, 20);
    camera.lookAt(0, 10, -100);
  
    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    container.value.appendChild(renderer.domElement);
  
    const texLoader = new THREE.TextureLoader();
    zedTexture = texLoader.load('/zed-sprite.png'); // 加载劫的素材
  
    createAurora();
    createStarField(); // 更密集的星空
    createPhotoStream();
    createMoonParticles(); // 皎月爆发粒子池
    createZedParticles();  // 劫爆发粒子池
    createShadowClones();  // 影分身池
    
    animate();
  }
  
  // --- 创建各种元素 ---
  function createAurora() {
      const geometry = new THREE.PlaneGeometry(100, 50, 64, 64);
      // ... (弯曲几何体的代码保持不变)
      const count = geometry.attributes.position.count;
      for(let i = 0; i < count; i++){
          const x = geometry.attributes.position.getX(i);
          const z = geometry.attributes.position.getZ(i);
          geometry.attributes.position.setZ(z - Math.pow(x, 2) * 0.02);
      }
      const material = new THREE.ShaderMaterial({ uniforms: { uTime: { value: 0 } }, vertexShader, fragmentShader, transparent: true, side: THREE.DoubleSide, depthWrite: false, blending: THREE.AdditiveBlending });
      auroraMesh = new THREE.Mesh(geometry, material);
      auroraMesh.position.set(0, 20, -150);
      auroraMesh.scale.set(2,2,2);
      scene.add(auroraMesh);
  }
  
  function createStarField() {
    const geometry = new THREE.BufferGeometry();
    const count = 8000; // 增加粒子数量，制造速度线感
    const positions = new Float32Array(count * 3);
    for(let i=0; i<count; i++) {
      positions[i*3] = (Math.random() - 0.5) * 400;
      positions[i*3+1] = (Math.random() - 0.5) * 400;
      positions[i*3+2] = Math.random() * -600; // 深度拉得更远
    }
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    const material = new THREE.PointsMaterial({ color: 0xa0e0ff, size: 0.3, transparent: true, opacity: 0.6, blending: THREE.AdditiveBlending });
    starField = new THREE.Points(geometry, material);
    scene.add(starField);
  }
  
  function createPhotoStream() {
    photoGroup = new THREE.Group();
    const loader = new THREE.TextureLoader();
    // 调整照片的初始位置，让它们散布在一条长长的隧道里
    props.photos.forEach((url, i) => {
      const tex = loader.load(url);
      // 保持宽高比，假设照片是竖版的
      const geometry = new THREE.PlaneGeometry(8, 12); 
      const material = new THREE.MeshBasicMaterial({ map: tex, transparent: true, opacity: 0, side: THREE.DoubleSide });
      const mesh = new THREE.Mesh(geometry, material);
      // 螺旋分布在 Z 轴上
      const angle = i * 1.5; 
      const radius = 15;
      mesh.position.set(Math.cos(angle) * radius, Math.sin(angle) * radius + 5, -100 - (i * 80));
      // 让照片稍微倾斜，增加动感
      mesh.rotation.set(Math.random()*0.5, Math.random()*0.5, Math.random()*0.2);
      photoGroup.add(mesh);
    });
    scene.add(photoGroup);
  }
  
  // 创建一个通用的粒子爆炸系统
  function createExplosionSystem(color, count, size) {
      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(count * 3);
      const velocities = new Float32Array(count * 3);
      for (let i = 0; i < count; i++) {
          positions[i*3] = 0; positions[i*3+1] = 0; positions[i*3+2] = 0;
          // 随机向四周爆发的速度向量
          const theta = Math.random() * Math.PI * 2;
          const phi = Math.acos(Math.random() * 2 - 1);
          const speed = Math.random() * 2 + 1;
          velocities[i*3] = speed * Math.sin(phi) * Math.cos(theta);
          velocities[i*3+1] = speed * Math.sin(phi) * Math.sin(theta);
          velocities[i*3+2] = speed * Math.cos(phi);
      }
      geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));
      const material = new THREE.PointsMaterial({ color: color, size: size, transparent: true, opacity: 0, blending: THREE.AdditiveBlending, depthWrite: false });
      return new THREE.Points(geometry, material);
  }
  
  function createMoonParticles() {
      moonParticles = createExplosionSystem(0x00d2ff, 500, 0.8); // 青色皎月粒子
      scene.add(moonParticles);
  }
  function createZedParticles() {
      zedParticles = createExplosionSystem(0xff004c, 2000, 1.2); // 红色大量劫粒子
      scene.add(zedParticles);
  }
  
  function createShadowClones() {
      shadowClonesGroup = new THREE.Group();
      shadowClonesGroup.visible = false;
      const material = new THREE.SpriteMaterial({ map: zedTexture, color: 0x6600cc, transparent: true, opacity: 0.5, blending: THREE.AdditiveBlending });
      for(let i=0; i<3; i++) {
          const sprite = new THREE.Sprite(material);
          sprite.scale.set(20, 20, 1);
          shadowClonesGroup.add(sprite);
      }
      scene.add(shadowClonesGroup);
  }
  
  
  // --- 触发特效函数 ---
  function triggerParticles(system, position) {
      system.material.opacity = 1;
      system.position.copy(position);
      const positions = system.geometry.attributes.position.array;
      for(let i=0; i<positions.length; i++) positions[i] = 0; // 重置位置到中心
      system.geometry.attributes.position.needsUpdate = true;
      // 粒子爆发动画
      gsap.to(system.material, { opacity: 0, duration: 1.5, ease: "power2.out" });
  }
  
  
  // --- 核心动画序列：电影级运镜 ---
  watch(() => props.startTrigger, async (val) => {
    if (val) {
      // 使用 async/await 来构建清晰的步骤
      await playCinematicSequence();
    }
  });
  
  async function playCinematicSequence() {
    // --- 0. 蓄力阶段 ---
    gsap.to(camera, { fov: 110, duration: 1.5, ease: "power2.inOut" });
    
    // 【修复点 1】：分开处理 density 和 color，绝对不能直接传字符串给 scene.fog
    gsap.to(scene.fog, { density: 0.005, duration: 1.5 });
    // 我们直接对 r, g, b 属性进行动画，从当前颜色变到黑色 (0,0,0)
    gsap.to(scene.fog.color, { r: 0, g: 0, b: 0, duration: 1.5 });

    await gsap.to(camera.position, { z: 50, duration: 1.5 });

    // --- 1. 循环播放照片回忆流 ---
    for (let i = 0; i < photoGroup.children.length; i++) {
        const photo = photoGroup.children[i];
        
        // A. 极速冲刺
        emit('play-sfx', 'whoosh');
        gsap.to(photo.material, { opacity: 1, duration: 0.3 });
        await gsap.to(camera.position, { 
            x: photo.position.x * 0.8,
            y: photo.position.y,
            z: photo.position.z + 30,
            duration: 0.7, 
            ease: "expo.inOut"
        });

        // B. “定格”瞬间
        emit('play-sfx', 'moon-burst');
        triggerParticles(moonParticles, photo.position);
        
        gsap.to(camera.position, { z: photo.position.z + 10, duration: 1.5, ease: "linear" });
        gsap.to(photo.rotation, { y: photo.rotation.y + 0.2, duration: 1.5 });
        await gsap.to(photo.material, { opacity: 0.5, duration: 1.5 });

        // C. 准备下一次冲刺
        if (i < photoGroup.children.length - 1) {
             gsap.fromTo(camera, { fov: 120 }, { fov: 100, duration: 0.5 });
        }
    }

    // --- 2. 终局：劫的瞬狱影杀阵 ---
    emit('play-sfx', 'zed-laugh');
    
    gsap.to(camera.position, { z: -180, duration: 1, ease: "power2.in" });
    
    shadowClonesGroup.visible = true;
    shadowClonesGroup.children.forEach((clone, i) => {
        const angle = (i / 3) * Math.PI * 2;
        clone.position.set(Math.cos(angle)*50, Math.sin(angle)*50, -250);
        gsap.to(clone.position, { x: 0, y: 0, z: -200, duration: 0.8, ease: "back.in(1.5)" });
        gsap.to(clone.material, { opacity: 1, duration: 0.5 });
    });
    
    await gsap.delayedCall(0.8, () => {}).then();
    
    shadowClonesGroup.visible = false;
    
    emit('play-sfx', 'zed-pop');
    triggerParticles(zedParticles, new THREE.Vector3(0, 0, -200));
    
    // 【修复点 2】：大招闪光特效，使用 .call() 避免破坏对象结构
    const flash = gsap.timeline();
    
    // 瞬间红光
    flash.call(() => {
        scene.fog.color.setHex(0xff0000); 
        scene.fog.density = 0.05;
    })
    .to({}, { duration: 0.05 }) // 保持 0.05秒
    // 瞬间全黑
    .call(() => {
        scene.fog.color.setHex(0x000000);
        scene.fog.density = 1;
    })
    // 缓慢恢复
    .to(scene.fog, { 
        density: 0.001, // 回到原来的稀薄雾气
        duration: 2, 
        ease: "power2.out", 
        delay: 0.5 
    }, "recover")
    .to(scene.fog.color, { 
        // 这里的颜色对应 0x050815 (深蓝黑)
        r: 0.02, 
        g: 0.03, 
        b: 0.08,
        duration: 2, 
        ease: "power2.out",
        delay: 0.5 
    }, "recover");
         
    emit('animation-complete');

    gsap.to(camera.position, { z: 20, duration: 2, delay: 0.5, ease: "expo.out" });
    gsap.to(camera, { fov: 70, duration: 2, delay: 0.5 });
}
  
  
  function animate() {
    time += 0.01;
    if (auroraMesh) auroraMesh.material.uniforms.uTime.value = time;
    
    // 星空高速后退，制造速度感
    if (starField) {
        starField.position.z += 2; 
        if(starField.position.z > 200) starField.position.z = 0;
    }
  
    // 更新粒子系统
    [moonParticles, zedParticles].forEach(system => {
        if(system && system.material.opacity > 0) {
          const positions = system.geometry.attributes.position.array;
          const velocities = system.geometry.attributes.velocity.array;
          for(let i=0; i<positions.length; i+=3) {
              positions[i] += velocities[i];
              positions[i+1] += velocities[i+1];
              positions[i+2] += velocities[i+2];
              velocities[i+1] -= 0.02; // 模拟重力下落
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
    // 清理内存...
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
  .scene-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 1; }
  </style>