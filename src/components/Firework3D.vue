<template>
  <div class="firework-container" ref="container"></div>
</template>

<script setup>
import Stats from 'three/addons/libs/stats.module.js';
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';


// 初始化 stats
// 初始化全局stats实例
let statsInstance = null;

// 场景、相机和渲染器
const container = ref(null);
let scene, camera, renderer, controls;

// 粒子系统相关变量
const shells = [];
const stars = [];
const sparks = [];
const instancedSystems = []; // 添加实例化系统数组
const PI_2 = Math.PI * 2;
const PI_HALF = Math.PI * 0.5;
  const GRAVITY = 0.025; // 略微增加重力常数，加快上升和下落的速度

const keys = {
  w: false,
  a: false,
  s: false,
  d: false,
  space: false,
  shift: false
};

const velocity = new THREE.Vector3();
const acceleration = 30.0;
const damping = 10.0;
const moveSpeed = 50.0;

// 更真实的焰火颜色配置
const COLOR_NAMES = [
  'Red', 'Orange', 'Gold', 'Yellow', 'Green',
  'Cyan', 'Blue', 'Purple', 'Pink', 'White', 'Silver',
  'VividPink', 'DeepBlue', 'Emerald', 'Peach', 'Magenta'
];

const colors = {
  // 暖色系
  Red: 0xff2233,        // 更鲜艳的红色
  Orange: 0xff7700,      // 橙色
  Gold: 0xffbf00,        // 金色
  Yellow: 0xffee44,      // 黄色
  Peach: 0xffaa88,       // 桃色
  
  // 冷色系
  Green: 0x22ff66,       // 更鲜艳的翠绿色
  Emerald: 0x00cc88,     // 祖母绿
  Cyan: 0x00ffdd,        // 青色
  Blue: 0x3377ff,        // 亮蓝色
  DeepBlue: 0x0055ff,    // 深蓝色
  Purple: 0x9944ff,      // 紫色
  
  // 鲜艳系
  Pink: 0xff66ee,        // 粉色
  VividPink: 0xff1177,   // 艳粉色
  Magenta: 0xff00cc,     // 洋红色

  // 特殊效果
  White: 0xffffff,       // 纯白色
  Silver: 0xc0c0c0       // 银色
};

// 粒子材质
let starMaterial, sparkMaterial, cometMaterial;

// 初始化 3D 场景
function initScene() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x051525); // 0.3亮度的微微蓝色

  // 相机设置
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 10000);
  camera.position.z = 600; // 将z位置从150增加到600
  camera.position.y = 200; // 将y位置从100增加到200
  camera.position.x = 400; // 将x位置从50增加到400
  camera.lookAt(0, 100, 0); // 调整视角，向下看以便能看到地面和烟花

  // 渲染器设置 - 优化渲染器配置
  renderer = new THREE.WebGLRenderer({ 
    antialias: window.devicePixelRatio <= 1.5, // 只在低分辨率下开启抗锯齿
    powerPreference: 'high-performance',
    precision: 'mediump' // 使用中等精度着色器提高性能
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(1.5, window.devicePixelRatio)); // 限制最大像素比
  container.value.appendChild(renderer.domElement);

  // 添加指针锁定控制器
  controls = new PointerLockControls(camera, renderer.domElement);

  // 点击页面后启用指针锁定
  document.addEventListener('click', () => {
    controls.lock();
  });

  // 添加模拟月亮的光源
  const moonLight = new THREE.PointLight(0xffffff, 0.8); // 白色光，强度0.8
  moonLight.position.set(1000, 3000, 1000); // 设置在很高且远的位置
  moonLight.distance = 8000; // 光照范围
  moonLight.decay = 0.1; // 减小衰减率，使光线传播更远
  scene.add(moonLight);
  
  // 为月光添加一个淡蓝色的环境光，增强夜空氛围
  const moonAmbientLight = new THREE.HemisphereLight(0xcceeff, 0x222233, 0.3);
  scene.add(moonAmbientLight);

  // 添加月亮模型
  const moonGeometry = new THREE.SphereGeometry(100, 32, 32); // 月亮半径100
  const moonMaterial = new THREE.MeshPhongMaterial({
    color: 0xffffee, // 淡黄白色
    emissive: 0xffffee,
    emissiveIntensity: 1.0,
    shininess: 100,
    transparent: true,
    opacity: 0.9
  });
  const moon = new THREE.Mesh(moonGeometry, moonMaterial);
  moon.position.copy(moonLight.position); // 与光源位置一致
  scene.add(moon);
  
  // 给月亮添加发光效果
  const moonGlow = new THREE.Sprite(new THREE.SpriteMaterial({
    map: createParticleTexture(),
    color: 0xffffdd,
    transparent: true,
    blending: THREE.AdditiveBlending,
    opacity: 0.6
  }));
  moonGlow.scale.set(300, 300, 1);
  moonGlow.position.copy(moonLight.position);
  scene.add(moonGlow);

  document.addEventListener('keydown', (e) => {
  switch (e.code) {
    case 'KeyW': keys.w = true; break;
    case 'KeyA': keys.a = true; break;
    case 'KeyS': keys.s = true; break;
    case 'KeyD': keys.d = true; break;
    case 'Space': keys.space = true; break;
    case 'ShiftLeft':
    case 'ShiftRight': keys.shift = true; break;
  }
});
document.addEventListener('keyup', (e) => {
  switch (e.code) {
    case 'KeyW': keys.w = false; break;
    case 'KeyA': keys.a = false; break;
    case 'KeyS': keys.s = false; break;
    case 'KeyD': keys.d = false; break;
    case 'Space': keys.space = false; break;
    case 'ShiftLeft':
    case 'ShiftRight': keys.shift = false; break;
  }
});



  // 确保在动画循环中调用updateCameraPosition
  function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
  }
  animate();

  document.addEventListener('keyup', (e) => {
    if (e.key in keys) keys[e.key] = false;
  });

  // 添加环境光
  const ambientLight = new THREE.AmbientLight(0x334455, 0.3);
  scene.add(ambientLight);

  // 添加地面
  addGround();

  // 创建粒子材质
  createParticleMaterials();

}
function updateCameraPosition(deltaTime) {
  const direction = new THREE.Vector3();
  camera.getWorldDirection(direction);
  direction.y = 0;
  direction.normalize();

  const right = new THREE.Vector3();
  right.crossVectors(direction, new THREE.Vector3(0, 1, 0)).normalize();
  const up = new THREE.Vector3(0, 1, 0);

  const input = new THREE.Vector3();
  if (keys.w) input.add(direction);
  if (keys.s) input.addScaledVector(direction, -1);
  if (keys.a) input.addScaledVector(right, -1);
  if (keys.d) input.add(right);
  if (keys.space) input.add(up);
  if (keys.shift) input.addScaledVector(up, -1);

  if (input.lengthSq() > 0) {
    input.normalize();
    velocity.addScaledVector(input, acceleration * deltaTime);
  }

  velocity.multiplyScalar(1 - damping * deltaTime);

  const moveDelta = velocity.clone().multiplyScalar(moveSpeed * deltaTime);
  controls.object.position.add(moveDelta);
}


// 创建粒子材质
function createParticleMaterials() {
  // 星星粒子材质
  const starTexture = createParticleTexture();
  starMaterial = new THREE.PointsMaterial({
    size: 50.0, // 调整至50.0
    map: starTexture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    vertexColors: true
  });
  starMaterial.userData.originalSize = 50.0;

  // 火花粒子材质
  const sparkTexture = createParticleTexture(0.8);
  sparkMaterial = new THREE.PointsMaterial({
    size: 40.0, // 调整至40.0
    map: sparkTexture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    vertexColors: true
  });

  // 彗星尾部材质
  const cometTexture = createParticleTexture(0.6);
  cometMaterial = new THREE.PointsMaterial({
    size: 10.0, // 保持彗星尾部尺寸
    map: cometTexture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    vertexColors: true
  });

  // 存储原始大小用于后续动画
  starMaterial.userData.originalSize = 1.6;
  sparkMaterial.userData.originalSize = 0.6;
  cometMaterial.userData.originalSize = 2.0;
}

// 创建粒子纹理 - 降低分辨率
function createParticleTexture(power = 1.0) {
  const canvas = document.createElement('canvas');
  canvas.width = 64; // 从128降为64
  canvas.height = 64; // 从128降为64
  const ctx = canvas.getContext('2d');

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 中心点
  const centerX = canvas.width / 2;
  const centerY = canvas.height / 2;
  const radius = canvas.width / 2;

  // 创建更精细的径向渐变，颜色使用更类似焰火的暖色调
  const gradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, radius);

  // 按照要求调整渐变区域分布
  // 中心区域 (15%)
  gradient.addColorStop(0, 'rgba(255, 255, 255, 1)'); // 中心为纯白
  gradient.addColorStop(0.15, 'rgba(255, 250, 240, 0.98)'); // 更亮的暖白色
  
  // 中间区域 (60%)
  gradient.addColorStop(0.25, 'rgba(255, 235, 180, 0.95)'); // 更亮的火焰橙黄色
  gradient.addColorStop(0.5, 'rgba(255, 200, 140, 0.80)'); // 更亮的橙色
  gradient.addColorStop(0.75, 'rgba(220, 150, 100, 0.50)'); // 更亮的烟火余烬色
  
  // 边缘区域 (25%)
  gradient.addColorStop(0.85, 'rgba(210, 120, 70, 0.30)'); // 更明显的边缘
  gradient.addColorStop(1, 'rgba(200, 100, 60, 0)'); // 完全透明的边缘

  // 填充渐变
  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.fill();

  // 添加焰火特有的增强辉光效果
  // 内部热核 - 始终添加增强辉光
  const coreGradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, radius * 0.4);
  coreGradient.addColorStop(0, 'rgba(255, 255, 240, 0.98)');
  coreGradient.addColorStop(1, 'rgba(255, 220, 120, 0)');

  ctx.globalCompositeOperation = 'lighter'; // 使用加亮模式增强亮度
  ctx.fillStyle = coreGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius * 0.3, 0, Math.PI * 2);
  ctx.fill();

  // 外部辉光
  const glowGradient = ctx.createRadialGradient(centerX, centerY, radius * 0.5, centerX, centerY, radius);
  glowGradient.addColorStop(0, 'rgba(255, 220, 140, 0.15)');
  glowGradient.addColorStop(0.5, 'rgba(255, 160, 80, 0.35)'); // 更明显的火焰色辉光
  glowGradient.addColorStop(1, 'rgba(220, 100, 50, 0.1)');

  ctx.globalCompositeOperation = 'screen'; // 使用叠加模式增强亮度
  ctx.fillStyle = glowGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.fill();

  // 增强边缘发光效果
  const edgeGlowGradient = ctx.createRadialGradient(centerX, centerY, radius * 0.75, centerX, centerY, radius);
  edgeGlowGradient.addColorStop(0, 'rgba(255, 180, 100, 0.4)');
  edgeGlowGradient.addColorStop(1, 'rgba(240, 120, 60, 0)');
  
  ctx.fillStyle = edgeGlowGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.fill();
  
  ctx.globalCompositeOperation = 'source-over';

  // 创建并返回纹理
  const texture = new THREE.Texture(canvas);
  texture.needsUpdate = true;
  return texture;
}

// 创建彩色粒子纹理 - 降低分辨率
function createColoredParticleTexture(color, power = 1.0) {
  const canvas = document.createElement('canvas');
  canvas.width = 64; // 从128降为64
  canvas.height = 64; // 从128降为64
  const ctx = canvas.getContext('2d');

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 中心点
  const centerX = canvas.width / 2;
  const centerY = canvas.height / 2;
  const radius = canvas.width / 2;

  // 将传入的颜色转换为RGB
  const r = ((color >> 16) & 255) / 255;
  const g = ((color >> 8) & 255) / 255;
  const b = (color & 255) / 255;

  // 确定颜色是暖色还是冷色
  const isWarm = r > b; // 如果红色分量大于蓝色分量，视为暖色

  // 创建径向渐变
  const gradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, radius);

  if (isWarm) {
    // 暖色系粒子 (红、橙、黄等) - 按照要求调整区域分布
    // 中心区域 (15%)
    gradient.addColorStop(0, `rgba(255, 255, 255, 1)`); // 中心为白色
    gradient.addColorStop(0.15, `rgba(255, ${Math.floor(Math.min(255, g * 255 * 1.3))}, ${Math.floor(Math.min(255, b * 255 * 1.3))}, 0.98)`);
    
    // 中间区域 (60%)
    gradient.addColorStop(0.25, `rgba(${Math.floor(Math.min(255, r * 255 * 1.3))}, ${Math.floor(Math.min(255, g * 220 * 1.3))}, ${Math.floor(Math.min(255, b * 170 * 1.3))}, 0.85)`);
    gradient.addColorStop(0.5, `rgba(${Math.floor(Math.min(255, r * 240 * 1.3))}, ${Math.floor(Math.min(255, g * 180 * 1.3))}, ${Math.floor(Math.min(255, b * 140 * 1.3))}, 0.7)`);
    gradient.addColorStop(0.75, `rgba(${Math.floor(Math.min(255, r * 220 * 1.3))}, ${Math.floor(Math.min(255, g * 140 * 1.3))}, ${Math.floor(Math.min(255, b * 80 * 1.3))}, 0.5)`);
    
    // 边缘区域 (25%)
    gradient.addColorStop(0.85, `rgba(${Math.floor(Math.min(255, r * 180 * 1.3))}, ${Math.floor(Math.min(255, g * 100 * 1.3))}, ${Math.floor(Math.min(255, b * 50 * 1.3))}, 0.25)`);
    gradient.addColorStop(1, `rgba(${Math.floor(Math.min(255, r * 150 * 1.3))}, ${Math.floor(Math.min(255, g * 70 * 1.3))}, ${Math.floor(Math.min(255, b * 30 * 1.3))}, 0)`);
  } else {
    // 冷色系粒子 (蓝、绿、紫等) - 按照要求调整区域分布
    // 中心区域 (15%)
    gradient.addColorStop(0, `rgba(255, 255, 255, 1)`); // 中心为白色
    gradient.addColorStop(0.15, `rgba(${Math.floor(Math.min(255, (r * 200 + 55) * 1.3))}, ${Math.floor(Math.min(255, (g * 200 + 55) * 1.3))}, 255, 0.98)`);
    
    // 中间区域 (60%)
    gradient.addColorStop(0.25, `rgba(${Math.floor(Math.min(255, r * 200 * 1.3))}, ${Math.floor(Math.min(255, g * 220 * 1.3))}, 255, 0.85)`);
    gradient.addColorStop(0.5, `rgba(${Math.floor(Math.min(255, r * 180 * 1.3))}, ${Math.floor(Math.min(255, g * 200 * 1.3))}, ${Math.floor(Math.min(255, b * 240 * 1.3))}, 0.7)`);
    gradient.addColorStop(0.75, `rgba(${Math.floor(Math.min(255, r * 130 * 1.3))}, ${Math.floor(Math.min(255, g * 180 * 1.3))}, ${Math.floor(Math.min(255, b * 220 * 1.3))}, 0.5)`);
    
    // 边缘区域 (25%)
    gradient.addColorStop(0.85, `rgba(${Math.floor(Math.min(255, r * 100 * 1.3))}, ${Math.floor(Math.min(255, g * 140 * 1.3))}, ${Math.floor(Math.min(255, b * 180 * 1.3))}, 0.25)`);
    gradient.addColorStop(1, `rgba(${Math.floor(Math.min(255, r * 70 * 1.3))}, ${Math.floor(Math.min(255, g * 100 * 1.3))}, ${Math.floor(Math.min(255, b * 150 * 1.3))}, 0)`);
  }

  // 填充渐变
  ctx.fillStyle = gradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.fill();

  // 增强内核辉光效果
  const coreGradient = ctx.createRadialGradient(centerX, centerY, 0, centerX, centerY, radius * 0.4);
  const coreColor = isWarm ?
    `rgba(255, ${Math.floor(Math.min(255, (220 + g * 35) * 1.5))}, ${Math.floor(Math.min(255, (170 + b * 60) * 1.5))}, 0.95)` :
    `rgba(${Math.floor(Math.min(255, (200 + r * 55) * 1.5))}, ${Math.floor(Math.min(255, (220 + g * 35) * 1.5))}, 255, 0.95)`;

  coreGradient.addColorStop(0, coreColor);
  coreGradient.addColorStop(1, `rgba(${Math.floor(Math.min(255, r * 255 * 1.5))}, ${Math.floor(Math.min(255, g * 255 * 1.5))}, ${Math.floor(Math.min(255, b * 255 * 1.5))}, 0)`);

  ctx.globalCompositeOperation = 'lighter';
  ctx.fillStyle = coreGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius * 0.3, 0, Math.PI * 2);
  ctx.fill();
  
  // 增强边缘发光效果
  const edgeGlowGradient = ctx.createRadialGradient(centerX, centerY, radius * 0.75, centerX, centerY, radius);
  const edgeColor = isWarm ?
    `rgba(${Math.floor(r * 255 * 1.3)}, ${Math.floor(g * 180 * 1.3)}, ${Math.floor(b * 100 * 1.3)}, 0.4)` :
    `rgba(${Math.floor(r * 120 * 1.3)}, ${Math.floor(g * 150 * 1.3)}, ${Math.floor(b * 255 * 1.3)}, 0.4)`;
    
  edgeGlowGradient.addColorStop(0, edgeColor);
  edgeGlowGradient.addColorStop(1, `rgba(${Math.floor(r * 255)}, ${Math.floor(g * 255)}, ${Math.floor(b * 255)}, 0)`);
  
  ctx.fillStyle = edgeGlowGradient;
  ctx.beginPath();
  ctx.arc(centerX, centerY, radius, 0, Math.PI * 2);
  ctx.fill();
  
  ctx.globalCompositeOperation = 'source-over';

  // 创建并返回纹理
  const texture = new THREE.Texture(canvas);
  texture.needsUpdate = true;
  return texture;
}

// 添加地面
function addGround() {
  // 创建主地面 - 扩大地面尺寸
  const geometry = new THREE.PlaneGeometry(2000, 2000);
  const textureLoader = new THREE.TextureLoader();
  const grassTexture = textureLoader.load('/grass.jpg');
  // 设置贴图重复
  grassTexture.wrapS = THREE.RepeatWrapping;
  grassTexture.wrapT = THREE.RepeatWrapping;
  grassTexture.repeat.set(40, 40); // 增加重复次数以适应更大的地面
  const material = new THREE.MeshBasicMaterial({
    blending: THREE.AdditiveBlending, // 使用加法混合以增强光照效果
    map: grassTexture,
    side: THREE.DoubleSide,
    transparent: true,
    opacity: 0.6
  });
  const ground = new THREE.Mesh(geometry, material);
  ground.rotation.x = Math.PI / 2;
  ground.position.y = -20; // 降低地面位置
  scene.add(ground);

  // 移除网格辅助和坐标轴辅助
  // const gridHelper = new THREE.GridHelper(500, 50, 0x888888, 0x444444);
  // gridHelper.position.y = -10;
  // scene.add(gridHelper);

  // const axesHelper = new THREE.AxesHelper(50);
  // axesHelper.position.y = -10;
  // scene.add(axesHelper);
}

// 随机颜色
function randomColor() {
  const name = COLOR_NAMES[Math.floor(Math.random() * COLOR_NAMES.length)];
  return { name, value: colors[name] };
}

// 创建彗星
function createComet(x, y, z, color, angle, velocity, life) {
  // 创建头部球体
  const headGeometry = new THREE.BufferGeometry();
  const headPositions = new Float32Array(3);
  const headColors = new Float32Array(3);
  const headSizes = new Float32Array(1);
  
  // 设置头部
  headPositions[0] = x;
  headPositions[1] = y;
  headPositions[2] = z;
  
  const colorObj = new THREE.Color(color);
  headColors[0] = Math.min(1.0, colorObj.r * 1.5);
  headColors[1] = Math.min(1.0, colorObj.g * 1.5);
  headColors[2] = Math.min(1.0, colorObj.b * 1.5);
  headSizes[0] = 40.0; // 头部大小
  
  headGeometry.setAttribute('position', new THREE.BufferAttribute(headPositions, 3));
  headGeometry.setAttribute('color', new THREE.BufferAttribute(headColors, 3));
  headGeometry.setAttribute('size', new THREE.BufferAttribute(headSizes, 1));
  
  // 使用自定义着色器材质实现头部效果
  const headMaterial = new THREE.ShaderMaterial({
    uniforms: {
      pointTexture: { value: createParticleTexture(0.9) }
    },
    vertexShader: `
      attribute float size;
      attribute vec3 color;
      varying vec3 vColor;
      
      void main() {
        vColor = color;
        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
        gl_PointSize = size * (300.0 / -mvPosition.z);
        gl_Position = projectionMatrix * mvPosition;
      }
    `,
    fragmentShader: `
      uniform sampler2D pointTexture;
      varying vec3 vColor;
      
      void main() {
        gl_FragColor = vec4(vColor, 1.0) * texture2D(pointTexture, gl_PointCoord);
      }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending
  });
  
  const head = new THREE.Points(headGeometry, headMaterial);
  scene.add(head);
  
  // 为彗星添加Z轴方向的随机偏移
  const zAngle = (Math.random() - 0.5) * 0.2;
  
  // 扩展彗星头部对象，添加物理属性
  head.speedX = Math.sin(angle) * velocity;
  head.speedY = Math.cos(angle) * velocity * 0.525; // 降低65%的速度以减少上升高度
  head.speedZ = Math.sin(zAngle) * velocity;
  head.life = life;
  head.positions = headPositions;
  head.positionAttribute = headGeometry.getAttribute('position');
  head.color = color;
  head.heavy = true; // 重物体，受空气阻力影响小
  head.mass = 1.0; // 彗星的质量
  head.radius = 1.5; // 彗星头部半径，用于粒子生成
  
  // 创建尾部粒子系统
  const tailParticles = [];
  const maxTailParticles = 300; // 增加最大尾部粒子数量，从200增加到300
  
  // 创建尾部粒子几何体
  const tailGeometry = new THREE.BufferGeometry();
  const tailPositions = new Float32Array(maxTailParticles * 3);
  const tailColors = new Float32Array(maxTailParticles * 3);
  const tailSizes = new Float32Array(maxTailParticles);
  const tailOpacities = new Float32Array(maxTailParticles);
  
  // 初始化尾部粒子属性
  for (let i = 0; i < maxTailParticles; i++) {
    const idx = i * 3;
    tailPositions[idx] = x;
    tailPositions[idx + 1] = y;
    tailPositions[idx + 2] = z;
    
    tailColors[idx] = colorObj.r;
    tailColors[idx + 1] = colorObj.g;
    tailColors[idx + 2] = colorObj.b;
    
    tailSizes[i] = 0;
    tailOpacities[i] = 0; // 初始不可见
  }
  
  tailGeometry.setAttribute('position', new THREE.BufferAttribute(tailPositions, 3));
  tailGeometry.setAttribute('color', new THREE.BufferAttribute(tailColors, 3));
  tailGeometry.setAttribute('size', new THREE.BufferAttribute(tailSizes, 1));
  tailGeometry.setAttribute('opacity', new THREE.BufferAttribute(tailOpacities, 1));
  
  // 使用自定义着色器材质实现尾部效果
  const tailMaterial = new THREE.ShaderMaterial({
    uniforms: {
      pointTexture: { value: createParticleTexture(0.7) }
    },
    vertexShader: `
      attribute float size;
      attribute vec3 color;
      attribute float opacity;
      varying vec3 vColor;
      varying float vOpacity;
      
      void main() {
        vColor = color;
        vOpacity = opacity;
        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
        gl_PointSize = size * (300.0 / -mvPosition.z);
        gl_Position = projectionMatrix * mvPosition;
      }
    `,
    fragmentShader: `
      uniform sampler2D pointTexture;
      varying vec3 vColor;
      varying float vOpacity;
      
      void main() {
        gl_FragColor = vec4(vColor, vOpacity) * texture2D(pointTexture, gl_PointCoord);
      }
    `,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending
  });
  
  const tail = new THREE.Points(tailGeometry, tailMaterial);
  scene.add(tail);
  
  // 粒子系统属性
  head.tail = tail;
  head.tailParticles = tailParticles;
  head.maxTailParticles = maxTailParticles;
  head.tailPositions = tailPositions;
  head.tailColors = tailColors;
  head.tailSizes = tailSizes;
  head.tailOpacities = tailOpacities;
  head.tailPositionAttribute = tailGeometry.getAttribute('position');
  head.tailColorAttribute = tailGeometry.getAttribute('color');
  head.tailSizeAttribute = tailGeometry.getAttribute('size');
  head.tailOpacityAttribute = tailGeometry.getAttribute('opacity');
  head.lastParticleTime = 0;
  head.particleRate = 8; // 初始每帧产生8个粒子
  
  // 更新函数 - 在updateComets中调用
  head.updateTail = function(deltaTime) {
    // 生成新粒子
    const currentTime = Date.now();
    const targetFrameInterval = 1000 / 60; // 目标帧间隔
    
    // 根据时间间隔调整粒子生成策略，确保即使在低帧率时也能持续生成粒子
    if (currentTime - this.lastParticleTime > targetFrameInterval) {
      this.lastParticleTime = currentTime;
      
      // 获取彗星头部当前位置
      const headX = this.positions[0];
      const headY = this.positions[1];
      const headZ = this.positions[2];
      
      // 计算上次生成粒子以来经过的实际时间比例
      const timeRatio = Math.min(2.0, (currentTime - this.lastParticleTime) / targetFrameInterval);
      // 根据时间动态调整本次生成的粒子数量
      const particlesToGenerate = Math.max(1, Math.round(this.particleRate * timeRatio));
      
      // 生成新粒子
      for (let i = 0; i < particlesToGenerate; i++) {
        // 在球体表面随机位置生成粒子
        const phi = Math.random() * Math.PI * 2;
        const theta = Math.random() * Math.PI;
        const radius = this.radius;
        
        // 球坐标转笛卡尔坐标
        const x = headX + radius * Math.sin(theta) * Math.cos(phi);
        const y = headY + radius * Math.sin(theta) * Math.sin(phi);
        const z = headZ + radius * Math.cos(theta);
        
        // 计算到中心的距离比例（0为中心，1为边缘）
        const distanceFromCenter = Math.sqrt(
          Math.pow(Math.sin(theta) * Math.cos(phi), 2) + 
          Math.pow(Math.sin(theta) * Math.sin(phi), 2) + 
          Math.pow(Math.cos(theta), 2)
        );
        
        // 边缘粒子生命周期短，中心粒子生命周期长
        // 将生命周期延长为原来的两倍
        const particleLife = 2000 * (1 - distanceFromCenter * 0.7);
        
        // 粒子初始速度（从头部速度稍微减小）
        const speedFactor = 0.9 - Math.random() * 0.2;
        
        // 创建粒子
        const particle = {
          index: this.tailParticles.length,
          x: x,
          y: y,
          z: z,
          speedX: this.speedX * speedFactor,
          speedY: this.speedY * speedFactor,
          speedZ: this.speedZ * speedFactor,
          life: particleLife,
          maxLife: particleLife,
          distanceFromCenter: distanceFromCenter,
          size: 5 + Math.random() * 3, // 粒子大小
          opacity: 0.9 - distanceFromCenter * 0.3 // 边缘粒子更透明
        };
        
        // 添加到粒子数组
        if (this.tailParticles.length < this.maxTailParticles) {
          this.tailParticles.push(particle);
        } else {
          // 替换最老的粒子
          const oldestIndex = this.tailParticles.findIndex(p => p.life <= 0);
          if (oldestIndex >= 0) {
            this.tailParticles[oldestIndex] = particle;
          }
        }
      }
    }
    
    // 更新所有尾部粒子
    for (let i = 0; i < this.tailParticles.length; i++) {
      const particle = this.tailParticles[i];
      
      // 更新生命周期
      particle.life -= 1000 * deltaTime;
      
      if (particle.life <= 0) {
        // 粒子死亡，设为不可见
        this.tailOpacities[particle.index] = 0;
        this.tailSizes[particle.index] = 0;
        continue;
      }
      
      // 计算生命周期比例
      const lifeRatio = particle.life / particle.maxLife;
      
      // 应用物理
      // 1. 空气阻力 - 边缘粒子阻力大
      const airDrag = Math.pow(0.97 - particle.distanceFromCenter * 0.02, 60 * deltaTime);
      
      // 2. 向中心靠拢的力 - 边缘粒子向中心靠拢更快
      const centeringForce = particle.distanceFromCenter * 0.015;
      
      // 获取彗星头部当前位置
      const headX = this.positions[0];
      const headY = this.positions[1];
      const headZ = this.positions[2];
      
      // 计算粒子到头部的方向向量
      const dx = headX - particle.x;
      const dy = headY - particle.y;
      const dz = headZ - particle.z;
      const dist = Math.sqrt(dx*dx + dy*dy + dz*dz);
      
      // 归一化
      const dirX = dist > 0 ? dx / dist : 0;
      const dirY = dist > 0 ? dy / dist : 0;
      const dirZ = dist > 0 ? dz / dist : 0;
      
      // 应用向中心的力
      particle.speedX += dirX * centeringForce;
      particle.speedY += dirY * centeringForce;
      particle.speedZ += dirZ * centeringForce;
      
      // 应用空气阻力
      particle.speedX *= airDrag;
      particle.speedY *= airDrag;
      particle.speedZ *= airDrag;
      
      // 应用重力
      particle.speedY -= GRAVITY * 0.2 * 60 * deltaTime; // 减小重力影响，从0.3降低到0.2
      
      // 更新位置
      particle.x += particle.speedX * 60 * deltaTime;
      particle.y += particle.speedY * 60 * deltaTime;
      particle.z += particle.speedZ * 60 * deltaTime;
      
      // 更新几何体中的位置
      const idx = particle.index * 3;
      this.tailPositions[idx] = particle.x;
      this.tailPositions[idx + 1] = particle.y;
      this.tailPositions[idx + 2] = particle.z;
      
      // 更新颜色和透明度 - 随生命周期变化
      const opacity = Math.pow(lifeRatio, 0.8) * particle.opacity; // 使透明度变化更平滑
      this.tailOpacities[particle.index] = opacity;
      
      // 更新大小 - 随生命周期变化，但保持更长时间的可见性
      this.tailSizes[particle.index] = particle.size * (0.3 + lifeRatio * 0.7);
      
      // 颜色随生命周期变化 - 逐渐变红
      const colorObj = new THREE.Color(this.color);
      this.tailColors[idx] = Math.min(1.0, colorObj.r * (1 + (1-lifeRatio) * 0.3));
      this.tailColors[idx + 1] = Math.min(1.0, colorObj.g * (0.4 + lifeRatio * 0.6)); // 保持一定的绿色
      this.tailColors[idx + 2] = Math.min(1.0, colorObj.b * (0.3 + lifeRatio * 0.7)); // 保持一定的蓝色
    }
    
    // 标记属性需要更新
    this.tailPositionAttribute.needsUpdate = true;
    this.tailColorAttribute.needsUpdate = true;
    this.tailSizeAttribute.needsUpdate = true;
    this.tailOpacityAttribute.needsUpdate = true;
  };
  
  return head;
}

// 创建爆炸粒子 - 使用对象池
function createParticles(count, x, y, z, color, minSpeed, maxSpeed, life, onDeath = null) {
  const particles = [];
  const colorValue = typeof color === 'object' ? color.value : color;

  // 使用对象池获取几何体
  const geometry = getGeometryFromPool();
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const colorObj = new THREE.Color(colorValue);

  for (let i = 0; i < count; i++) {
    const idx = i * 3;
    positions[idx] = x;
    positions[idx + 1] = y;
    positions[idx + 2] = z;

    // 增强颜色亮度
    colors[idx] = Math.min(1.0, colorObj.r * 1.4);
    colors[idx + 1] = Math.min(1.0, colorObj.g * 1.4);
    colors[idx + 2] = Math.min(1.0, colorObj.b * 1.4);
  }

  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

  // 使用彩色粒子贴图
  const texture = createColoredParticleTexture(colorValue);
  const material = new THREE.PointsMaterial({
    size: 50.0,
    map: texture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    vertexColors: true,
    opacity: 1.0
  });

  material.userData.originalSize = 5.0;

  const pointCloud = new THREE.Points(geometry, material);
  scene.add(pointCloud);

  // 为每个粒子添加属性
  for (let i = 0; i < count; i++) {
    // 3D空间分布 - 球形爆炸
    const phi = Math.random() * PI_2; // 水平角度
    const theta = Math.random() * Math.PI; // 垂直角度

    const speed = minSpeed + Math.random() * (maxSpeed - minSpeed);

    const particle = {
      index: i,
      x: x,
      y: y,
      z: z,
      speedX: Math.sin(theta) * Math.cos(phi) * speed,
      speedY: Math.sin(theta) * Math.sin(phi) * speed,
      speedZ: Math.cos(theta) * speed,
      life: life * (0.8 + Math.random() * 0.4), // 随机化生命周期
      maxLife: life,
      startTime: Date.now(), // 记录开始时间用于贝塞尔曲线
      color: colorValue,
      geometry: geometry,
      positionAttribute: geometry.getAttribute('position'),
      pointCloud: pointCloud,
      onDeath: onDeath,
      mass: 0.1 // 爆炸后的粒子质量较小
    };

    particles.push(particle);
  }

  return particles;
}

// 计算贝塞尔曲线值 (三次贝塞尔曲线)
function cubicBezier(t, p0, p1, p2, p3) {
  const mt = 1 - t;
  return mt * mt * mt * p0 + 3 * mt * mt * t * p1 + 3 * mt * t * t * p2 + t * t * t * p3;
}

// 计算粒子透明度的贝塞尔曲线
function calculateOpacity(currentLife, maxLife) {
  const t = 1 - (currentLife / maxLife);
  if (t < 0.5) {
    return cubicBezier(t * 2, 1.0, 0.8, 0.8, 0.8);
  } else {
    return cubicBezier((t - 0.5) * 2, 0.8, 0.8, 0.3, 0.0);
  }
}

// 更新粒子位置和状态 - 使用批量更新减少needsUpdate调用
function updateParticles(deltaTime) {
  // 收集需要更新的几何体
  const needsUpdateGeometries = new Set();
  
  // 定义速度增强系数
  const speedFactor = 1.35; // 增加35%的速度
  
  // 更新星星粒子
  for (let i = stars.length - 1; i >= 0; i--) {
    const star = stars[i];

    // 更新生命周期 - 使用deltaTime使动画与帧率无关
    star.life -= 1500 * deltaTime; // 基于时间的生命值减少

    if (star.life <= 0) {
      // 如果有死亡回调，调用它
      if (star.onDeath) {
        star.onDeath(star);
      }

      // 移除星星并回收资源
      scene.remove(star.pointCloud);
      if (star.geometry) recycleGeometry(star.geometry);
      if (star.pointCloud && star.pointCloud.material) {
        recycleMaterial(star.pointCloud.material);
      }
      
      stars.splice(i, 1);
      continue;
    }

    // 如果有自定义更新函数，使用它
    if (star.update) {
      star.update(deltaTime);
    } else {
      // 应用物理（重力和空气阻力）- 基于deltaTime调整
      const airDrag = Math.pow(0.98, 60 * deltaTime); // 基于时间的空气阻力
      star.speedX *= airDrag;
      star.speedY -= GRAVITY * (star.mass || 0.1) * 60 * deltaTime; // 基于时间的重力效应
      star.speedY *= airDrag;
      star.speedZ *= airDrag; // Z轴也应用空气阻力

      // 更新位置 - 基于deltaTime调整位移，并应用速度增强系数
      star.x += star.speedX * 60 * deltaTime * speedFactor;
      star.y += star.speedY * 60 * deltaTime * speedFactor;
      star.z += star.speedZ * 60 * deltaTime * speedFactor; // 更新Z轴位置
    }

    // 更新几何体中的位置
    const idx = star.index * 3;
    const positions = star.positionAttribute.array;
    positions[idx] = star.x;
    positions[idx + 1] = star.y;
    positions[idx + 2] = star.z; // 设置Z轴坐标

    // 将几何体添加到需要更新的集合中
    if (star.positionAttribute) {
      needsUpdateGeometries.add(star.positionAttribute);
    }

    // 使用贝塞尔曲线计算透明度
    const opacity = calculateOpacity(star.life, star.maxLife);
    if (star.pointCloud && star.pointCloud.material) {
      star.pointCloud.material.opacity = opacity;

      // 根据生命周期也调整粒子大小，但确保粒子尺寸在生命周期结束前仍然足够明显
      const lifeFactor = star.life / star.maxLife;
      // 使用自定义缩放曲线，使粒子在前70%的生命周期几乎保持原始大小
      const sizeScale = lifeFactor > 0.3 ? 0.8 + lifeFactor * 0.2 : 0.5 + lifeFactor * 0.3;
      star.pointCloud.material.size = (star.pointCloud.material.userData.originalSize || 50.0) * sizeScale;
    }
  }

  // 更新火花粒子（如果有的话）
  for (let i = sparks.length - 1; i >= 0; i--) {
    const spark = sparks[i];

    // 更新生命周期 - 使用deltaTime使动画与帧率无关
    spark.life -= 1500 * deltaTime;

    if (spark.life <= 0) {
      scene.remove(spark.pointCloud);
      if (spark.geometry) recycleGeometry(spark.geometry);
      if (spark.pointCloud && spark.pointCloud.material) {
        recycleMaterial(spark.pointCloud.material);
      }
      sparks.splice(i, 1);
      continue;
    }

    // 如果有自定义更新函数，使用它
    if (spark.update) {
      spark.update(deltaTime);
    } else {
      // 应用物理 - 基于deltaTime调整
      const airDrag = Math.pow(0.9, 60 * deltaTime); // 基于时间的空气阻力
      spark.speedX *= airDrag;
      spark.speedY -= GRAVITY * 0.05 * 60 * deltaTime; // 基于时间的重力效应，火花质量更小
      spark.speedY *= airDrag;
      spark.speedZ *= airDrag; // Z轴也应用空气阻力

      // 更新位置 - 基于deltaTime调整位移，应用速度增强系数
      spark.x += spark.speedX * 60 * deltaTime * speedFactor;
      spark.y += spark.speedY * 60 * deltaTime * speedFactor;
      spark.z += spark.speedZ * 60 * deltaTime * speedFactor; // 更新Z轴位置
    }

    // 更新几何体中的位置
    const positions = spark.positionAttribute.array;
    positions[0] = spark.x;
    positions[1] = spark.y;
    positions[2] = spark.z; // 设置Z轴坐标

    if (spark.positionAttribute) {
      needsUpdateGeometries.add(spark.positionAttribute);
    }

    // 使用贝塞尔曲线计算透明度
    const opacity = calculateOpacity(spark.life, spark.maxLife);
    if (spark.pointCloud && spark.pointCloud.material) {
      spark.pointCloud.material.opacity = opacity;
    }
  }
  
  // 批量更新几何体，减少needsUpdate调用
  needsUpdateGeometries.forEach(attr => {
    attr.needsUpdate = true;
  });
}

// 创建实例化爆炸效果
function createInstancedBurst(count, x, y, z, color, speed, life) {
  // 创建实例化粒子系统
  const instancedSystem = createInstancedParticles(count, color);

  // 设置每个粒子的初始位置和速度
  const particles = [];
  const positions = instancedSystem.positionAttribute.array;

  for (let i = 0; i < count; i++) {
    // 3D空间分布 - 球形爆炸
    const phi = Math.random() * PI_2; // 水平角度
    const theta = Math.random() * Math.PI; // 垂直角度

    const particleSpeed = speed * (0.8 + Math.random() * 0.4);

    const particle = {
      index: i,
      x: x,
      y: y,
      z: z,
      speedX: Math.sin(theta) * Math.cos(phi) * particleSpeed,
      speedY: Math.sin(theta) * Math.sin(phi) * particleSpeed,
      speedZ: Math.cos(theta) * particleSpeed,
      life: life * (0.8 + Math.random() * 0.4), // 随机化生命周期
      maxLife: life,
      startTime: Date.now(), // 记录开始时间
      color: color
    };

    // 设置初始位置
    const idx = i * 3;
    positions[idx] = particle.x;
    positions[idx + 1] = particle.y;
    positions[idx + 2] = particle.z;

    // 设置初始透明度
    instancedSystem.opacities[i] = 1.0;

    particles.push(particle);
  }

  // 标记属性需要更新
  instancedSystem.positionAttribute.needsUpdate = true;
  instancedSystem.opacityAttribute.needsUpdate = true;

  // 每帧更新函数
  const updateFunction = function () {
    const positions = instancedSystem.positionAttribute.array;
    const opacities = instancedSystem.opacityAttribute.array;

    let aliveCount = 0;

    for (let i = 0; i < particles.length; i++) {
      const particle = particles[i];

      // 更新生命周期
      particle.life -= 16;

      if (particle.life <= 0) {
        opacities[i] = 0;
        continue;
      }

      aliveCount++;

      // 应用物理（重力和空气阻力）
      const airDrag = 0.98;
      particle.speedX *= airDrag;
      particle.speedY -= GRAVITY;
      particle.speedY *= airDrag;
      particle.speedZ *= airDrag;

      // 更新位置
      particle.x += particle.speedX;
      particle.y += particle.speedY;
      particle.z += particle.speedZ;

      // 更新实例位置
      const idx = i * 3;
      positions[idx] = particle.x;
      positions[idx + 1] = particle.y;
      positions[idx + 2] = particle.z;

      // 使用贝塞尔曲线更新透明度
      opacities[i] = calculateOpacity(particle.life, particle.maxLife);
    }

    // 标记属性需要更新
    instancedSystem.positionAttribute.needsUpdate = true;
    instancedSystem.opacityAttribute.needsUpdate = true;

    // 如果所有粒子都死亡，移除系统
    if (aliveCount === 0) {
      scene.remove(instancedSystem.mesh);
      return false; // 停止更新
    }

    return true; // 继续更新
  };

  // 添加到更新列表
  instancedSystems.push({
    update: updateFunction,
    system: instancedSystem
  });

  return instancedSystem;
}

// 创建爆炸效果
function createBurst(count, x, y, z, color, speed, life, onDeath = null) {
  // 创建爆炸粒子，提高速度系数
  const particles = createParticles(count, x, y, z, color, speed * 1.2, speed * 1.6, life, onDeath);

  // 将粒子添加到跟踪数组
  stars.push(...particles);

  // 创建爆炸闪光效果
  addBurstFlash(x, y, z, color);

  return particles;
}

// 添加爆炸闪光 - 优化闪光效果的粒子数量
function addBurstFlash(x, y, z, colorHex = 0xffaa55) {
  const baseColor = new THREE.Color(colorHex);

  // 使用彩色粒子纹理生成渐变贴图
  const texture = createColoredParticleTexture(colorHex, 1.2);

  // 创建更大的光晕闪光 - 减小50%大小
  const flashGeometry = new THREE.PlaneGeometry(10, 10); // 从20降至10，减少50%
  const flashMaterial = new THREE.MeshBasicMaterial({
    map: texture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    opacity: 1.0,
    side: THREE.DoubleSide,
    color: baseColor.clone().multiplyScalar(1.4) // 更亮的颜色
  });

  const flash = new THREE.Mesh(flashGeometry, flashMaterial);
  flash.position.set(x, y, z);
  flash.lookAt(camera.position);
  scene.add(flash);

  // 获取当前时间
  const startTime = Date.now();
  const duration = 250; // 减少闪光持续时间，使效果更快速、更强烈

  function animateFlash() {
    const elapsed = Date.now() - startTime;
    const progress = Math.min(elapsed / duration, 1.0);

    if (progress >= 1.0) {
      scene.remove(flash);
      flashGeometry.dispose();
      flashMaterial.dispose();
      return;
    }

    // 使用新的缓动曲线，使闪光更明亮更快
    const opacity = cubicBezier(progress, 1.0, 0.8, 0.4, 0.0);
    const scale = cubicBezier(progress, 1.0, 1.4, 2.1, 3.0); // 从(1.0, 2.8, 4.2, 6.0)降低50%

    flash.material.opacity = opacity;
    flash.scale.set(scale, scale, 1);

    // 添加颜色变化，使闪光在消失时略微变暖
    const r = lerp(baseColor.r, Math.min(1.0, baseColor.r * 1.2), progress);
    const g = lerp(baseColor.g, Math.min(0.9, baseColor.g * 0.9), progress);
    const b = lerp(baseColor.b, Math.min(0.8, baseColor.b * 0.8), progress);
    flash.material.color.setRGB(r, g, b);

    requestAnimationFrame(animateFlash);
  }

  animateFlash();

  // 添加额外的爆炸闪光效果 - 多层次闪光，减小50%大小
  addSecondaryFlash(x, y, z, colorHex, startTime);

  // 添加外围粒子光晕 - 减少粒子数量和半径
  const particleCount = 40; // 从80减少到40，但保持视觉效果
  const particleGeometry = new THREE.BufferGeometry();
  const particlePositions = new Float32Array(particleCount * 3);

  for (let i = 0; i < particleCount; i++) {
    const i3 = i * 3;
    const radius = 2.0 + Math.random() * 5.0; // 从(4.0 + Math.random() * 10.0)减少50%
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.random() * Math.PI;

    particlePositions[i3] = x + radius * Math.sin(phi) * Math.cos(theta);
    particlePositions[i3 + 1] = y + radius * Math.sin(phi) * Math.sin(theta);
    particlePositions[i3 + 2] = z + radius * Math.cos(phi);
  }

  particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));

  // 创建一个增强的基础颜色，使光晕更亮
  const enhancedColor = baseColor.clone().multiplyScalar(1.6);
  
  const particleMaterial = new THREE.PointsMaterial({
    size: 4.5, // 从9.0减少50%
    map: texture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    color: enhancedColor
  });

  const particles = new THREE.Points(particleGeometry, particleMaterial);
  scene.add(particles);

  const particleStartTime = Date.now();
  const particleDuration = 200; // 减少粒子动画持续时间，但使效果更强烈

  function animateParticles() {
    const elapsed = Date.now() - particleStartTime;
    const progress = Math.min(elapsed / particleDuration, 1.0);

    if (progress >= 1.0) {
      scene.remove(particles);
      particleGeometry.dispose();
      particleMaterial.dispose();
      return;
    }

    // 使用不同的透明度曲线，快速开始然后缓慢消失
    const opacity = cubicBezier(progress, 1.0, 0.7, 0.4, 0.0);
    particleMaterial.opacity = opacity;

    // 颜色逐渐变暗但保持暖色调
    const r = lerp(enhancedColor.r, 0.2, progress);
    const g = lerp(enhancedColor.g, 0.1, progress);
    const b = lerp(enhancedColor.b, 0.05, progress);
    particleMaterial.color.setRGB(r, g, b);

    requestAnimationFrame(animateParticles);
  }

  animateParticles();
}

// 添加次级闪光效果，使爆炸看起来更真实
function addSecondaryFlash(x, y, z, colorHex, primaryStartTime) {
  const baseColor = new THREE.Color(colorHex);
  // 使闪光颜色略有不同
  const flashColor = baseColor.clone().multiplyScalar(0.9);
  if (flashColor.r > 0.5) flashColor.g *= 0.9;
  else if (flashColor.b > 0.5) flashColor.r *= 1.2;
  
  const texture = createColoredParticleTexture(colorHex, 1.0);
  
  // 创建多个随机位置的小闪光
  const flashCount = Math.floor(Math.random() * 3) + 2; // 2-4个小闪光
  
  for (let i = 0; i < flashCount; i++) {
    // 随机位置偏移
    const offset = 3 + Math.random() * 2; // 恢复原始值，不减少
    const angle = Math.random() * Math.PI * 2;
    const xOffset = x + Math.cos(angle) * offset;
    const yOffset = y + Math.sin(angle) * offset;
    const zOffset = z + (Math.random() - 0.5) * offset * 0.5;
    
    // 随机延迟
    const delay = Math.random() * 80;
    
    setTimeout(() => {
      const flashSize = 6 + Math.random() * 4; // 恢复原始值，不减少
      const flashGeometry = new THREE.PlaneGeometry(flashSize, flashSize);
      const flashMaterial = new THREE.MeshBasicMaterial({
        map: texture,
        transparent: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        opacity: 0.8,
        side: THREE.DoubleSide,
        color: flashColor
      });
      
      const flash = new THREE.Mesh(flashGeometry, flashMaterial);
      flash.position.set(xOffset, yOffset, zOffset);
      flash.lookAt(camera.position);
      scene.add(flash);
      
      const duration = 100 + Math.random() * 100;
      const startTime = Date.now();
      
      function animateSecondaryFlash() {
        const elapsed = Date.now() - startTime;
        const progress = Math.min(elapsed / duration, 1.0);
        
        if (progress >= 1.0) {
          scene.remove(flash);
          return;
        }
        
        const opacity = 0.8 * (1.0 - progress * progress);
        const scale = 1.0 + progress * 2; // 恢复原始值，不减少
        
        flash.material.opacity = opacity;
        flash.scale.set(scale, scale, 1);
        
        requestAnimationFrame(animateSecondaryFlash);
      }
      
      animateSecondaryFlash();
    }, delay);
  }
}

// 线性插值函数
function lerp(a, b, t) {
  return a + (b - a) * t;
}

// 创建交叉效果
function crossetteEffect(particle) {
  const count = 4;
  const startAngle = Math.random() * PI_HALF;

  for (let i = 0; i < count; i++) {
    const angle = startAngle + (i * PI_2 / count);
    const speed = 1.0 + Math.random() * 0.5; // 增加交叉效果速度
    const life = 450; // 减少生命周期使效果更快速

    const newParticle = {
      x: particle.x,
      y: particle.y,
      speedX: Math.sin(angle) * speed,
      speedY: Math.cos(angle) * speed,
      life: life,
      maxLife: life,
      color: particle.color,
      geometry: new THREE.BufferGeometry(),
      positionAttribute: null,
      pointCloud: null
    };

    // 创建几何体和点云
    const positions = new Float32Array(3);
    positions[0] = particle.x;
    positions[1] = particle.y;
    positions[2] = 0;

    newParticle.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    newParticle.positionAttribute = newParticle.geometry.getAttribute('position');

    const material = starMaterial.clone();
    material.color.set(particle.color);

    newParticle.pointCloud = new THREE.Points(newParticle.geometry, material);
    scene.add(newParticle.pointCloud);

    stars.push(newParticle);
  }
}

// 花形效果
function floralEffect(particle) {
  const count = 12;

  // 创建环状粒子爆炸
  for (let i = 0; i < count; i++) {
    const angle = (i / count) * PI_2;
    const speed = 2.0; // 增加花形效果速度
    const life = 800 + Math.random() * 200; // 缩短生命周期使效果更快速

    const newParticle = {
      x: particle.x,
      y: particle.y,
      speedX: Math.sin(angle) * speed + particle.speedX * 0.2,
      speedY: Math.cos(angle) * speed + particle.speedY * 0.2,
      life: life,
      maxLife: life,
      color: particle.color,
      geometry: new THREE.BufferGeometry(),
      positionAttribute: null,
      pointCloud: null
    };

    // 创建几何体和点云
    const positions = new Float32Array(3);
    positions[0] = particle.x;
    positions[1] = particle.y;
    positions[2] = 0;

    newParticle.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    newParticle.positionAttribute = newParticle.geometry.getAttribute('position');

    const material = starMaterial.clone();
    material.color.set(particle.color);

    newParticle.pointCloud = new THREE.Points(newParticle.geometry, material);
    scene.add(newParticle.pointCloud);

    stars.push(newParticle);
  }

  // 添加爆炸闪光
  addBurstFlash(particle.x, particle.y, particle.z, particle.color);
}

// 环形效果
function ringEffect(particle) {
  const count = 20;
  const ringRadius = 2 + Math.random() * 3;

  for (let i = 0; i < count; i++) {
    const angle = (i / count) * PI_2;

    const newParticle = {
      x: particle.x + Math.cos(angle) * ringRadius,
      y: particle.y + Math.sin(angle) * ringRadius,
      z: particle.z,
      speedX: Math.cos(angle) * 1.2, // 从0.8增加到1.2，增加环形效果速度
      speedY: Math.sin(angle) * 1.2, // 从0.8增加到1.2，增加环形效果速度
      speedZ: (Math.random() - 0.5) * 0.6, // 从0.4增加到0.6，增加Z轴速度
      life: 600 + Math.random() * 200, // 保持生命周期
      maxLife: 700, // 保持最大生命周期
      color: particle.color,
      geometry: new THREE.BufferGeometry(),
      positionAttribute: null,
      pointCloud: null
    };

    // 创建几何体和点云
    const positions = new Float32Array(3);
    positions[0] = newParticle.x;
    positions[1] = newParticle.y;
    positions[2] = newParticle.z;

    newParticle.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    newParticle.positionAttribute = newParticle.geometry.getAttribute('position');

    const material = starMaterial.clone();
    material.color.set(newParticle.color);

    newParticle.pointCloud = new THREE.Points(newParticle.geometry, material);
    scene.add(newParticle.pointCloud);

    stars.push(newParticle);
  }
}

// 螺旋效果
function spiralEffect(particle) {
  const count = 30;
  const baseSpeed = 1.3; // 从0.9增加到1.3，增加基础速度
  const spiralTightness = 0.6; // 从0.4增加到0.6，增加螺旋紧凑度

  for (let i = 0; i < count; i++) {
    const angle = (i / count) * PI_2 * 3; // 三圈螺旋
    const distance = (i / count) * 8; // 增加距离

    const newParticle = {
      x: particle.x,
      y: particle.y,
      z: particle.z,
      spiralAngle: angle,
      spiralDistance: distance,
      speedX: Math.cos(angle) * baseSpeed,
      speedY: Math.sin(angle) * baseSpeed,
      speedZ: (Math.random() - 0.5) * 0.4, // 增加Z轴速度
      spiralSpeed: spiralTightness,
      life: 900 + Math.random() * 300, // 保持生命周期
      maxLife: 1000, // 保持最大生命周期
      color: particle.color,
      geometry: new THREE.BufferGeometry(),
      positionAttribute: null,
      pointCloud: null,
      update: function () {
        this.spiralAngle += this.spiralSpeed;
        this.x += this.speedX + Math.cos(this.spiralAngle) * this.spiralDistance * 0.18; // 增加螺旋效果速度
        this.y += this.speedY + Math.sin(this.spiralAngle) * this.spiralDistance * 0.18; // 增加螺旋效果速度
        this.z += this.speedZ;
      }
    };

    // 创建几何体和点云
    const positions = new Float32Array(3);
    positions[0] = newParticle.x;
    positions[1] = newParticle.y;
    positions[2] = newParticle.z;

    newParticle.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    newParticle.positionAttribute = newParticle.geometry.getAttribute('position');

    const material = starMaterial.clone();
    material.color.set(newParticle.color);
    material.size = 1.0;

    newParticle.pointCloud = new THREE.Points(newParticle.geometry, material);
    scene.add(newParticle.pointCloud);

    stars.push(newParticle);
  }
}

// 心形效果
function heartEffect(particle) {
  const count = 30;
  const scale = 4; // 从3增加到4，增大心形比例

  for (let i = 0; i < count; i++) {
    const t = (i / count) * PI_2;

    // 心形函数参数方程
    const x = 16 * Math.pow(Math.sin(t), 3);
    const y = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);

    // 缩放并偏移
    const heartX = x * scale * 0.07; // 增加心形大小
    const heartY = -y * scale * 0.07; // 增加心形大小并翻转Y轴使心形正向

    const newParticle = {
      x: particle.x + heartX,
      y: particle.y + heartY,
      z: particle.z + (Math.random() - 0.5) * 0.5,
      initialX: heartX,
      initialY: heartY,
      t: t,
      speedX: heartX * 0.05, // 增加心形扩散速度
      speedY: heartY * 0.05, // 增加心形扩散速度
      speedZ: (Math.random() - 0.5) * 0.3, // 增加Z轴变化
      life: 1000 + Math.random() * 500, // 保持生命周期
      maxLife: 1500, // 保持最大生命周期
      color: 0xff0044, // 心形通常是红色
      geometry: new THREE.BufferGeometry(),
      positionAttribute: null,
      pointCloud: null
    };

    // 创建几何体和点云
    const positions = new Float32Array(3);
    positions[0] = newParticle.x;
    positions[1] = newParticle.y;
    positions[2] = newParticle.z;

    newParticle.geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    newParticle.positionAttribute = newParticle.geometry.getAttribute('position');

    const material = starMaterial.clone();
    material.color.set(newParticle.color);

    newParticle.pointCloud = new THREE.Points(newParticle.geometry, material);
    scene.add(newParticle.pointCloud);

    stars.push(newParticle);
  }
}

// 创建烟花弹
function createShell(options) {
  const defaults = {
    position: { x: 0, y: -20, z: 0 },
    velocity: 7.5 + Math.random() * 1.5, // 从5+增加到7.5+，提高发射速度
    color: randomColor(),
    spreadSize: 180 + Math.random() * 100, // 从120+增加到180+，扩大爆炸范围
    starCount: 600 + Math.floor(Math.random() * 300), // 保持粒子数量
    starLife: 1200 + Math.random() * 600, // 保持粒子生命周期
    effects: []
  };

  const config = { ...defaults, ...options };

  // 创建彗星 - 修正角度为0（向上）
  const angle = 0; // 朝上发射，而不是Math.PI
  const comet = createComet(
    config.position.x,
    config.position.y,
    config.position.z || 0, // 添加z坐标
    config.color.value,
    angle,
    config.velocity,
    1500 // 减半生命周期，使烟花更快爆炸
  );

  // 扩展彗星对象
  comet.config = config;
  comet.lifetime = 0;

  // 当彗星到达顶点时爆炸
  comet.explode = function () {
    const x = this.positions[0];
    const y = this.positions[1];
    const z = this.positions[2];

    // 移除彗星
    scene.remove(this);
    scene.remove(this.tail);
    
    // 不再需要清除历史记录，因为我们不再使用history属性
    // this.history = [];

    // 处理多彩效果
    if (config.effects.includes('multicolor')) {
      // 为每个粒子批次选择不同颜色
      const particleColors = [];
      const colorCount = Math.floor(Math.random() * 3) + 3; // 3-5种颜色
      
      for (let i = 0; i < colorCount; i++) {
        const colorName = COLOR_NAMES[Math.floor(Math.random() * COLOR_NAMES.length)];
        particleColors.push(colors[colorName]);
      }
      
      // 创建多种颜色的爆炸
      for (let i = 0; i < colorCount; i++) {
        const particleCount = Math.floor(config.starCount / colorCount);
        // 创建第一批粒子
        createBurst(
          particleCount,
          x, y, z,
          particleColors[i],
          config.spreadSize / 10,
          config.starLife,
          config.effects.includes('crossette') ? crossetteEffect : null
        );
        
        // 使用实例化渲染创建额外的爆炸效果
        if (config.starCount > 180) {
          createInstancedBurst(
            Math.floor(450 / colorCount),
            x, y, z,
            particleColors[i],
            config.spreadSize / 12,
            config.starLife * 0.8
          );
        }
      }
    } else {
      // 使用普通渲染创建爆炸核心
      const particles = createBurst(
        Math.min(config.starCount, 180), // 增加普通渲染粒子数量
        x, y, z,
        config.color.value,
        config.spreadSize / 10, // 再次大幅增加爆炸速度，提高扩散半径
        config.starLife,
        config.effects.includes('crossette') ? crossetteEffect : null
      );

      // 使用实例化渲染创建额外的爆炸效果（大量粒子）
      if (config.starCount > 180) {
        createInstancedBurst(
          450, // 增加实例化渲染粒子数量
          x, y, z,
          config.color.value,
          config.spreadSize / 12, // 大幅增加实例化粒子的速度
          config.starLife * 0.8
        );
      }

      // 特效处理
      if (config.effects.includes('floral')) {
        // 500ms 后触发花形效果
        setTimeout(() => {
          particles.forEach(particle => {
            if (particle.life > 0) {
              floralEffect(particle);
            }
          });
        }, 500);
      }
    }

    // 环形效果
    if (config.effects.includes('ring')) {
      ringEffect({ x, y, z, color: config.color.value });
    }

    // 螺旋效果
    if (config.effects.includes('spiral')) {
      spiralEffect({ x, y, z, color: config.color.value });
    }

    // 心形效果
    if (config.effects.includes('heart')) {
      heartEffect({ x, y, z, color: config.color.value });
    }
  };

  shells.push(comet);
  return comet;
}

// 发射烟花
function launchShell() {
  // 随机烟花类型
  const shellTypes = [
    { name: 'basic', effects: [], probability: 0.15 },
    { name: 'crossette', effects: ['crossette'], probability: 0.15 },
    { name: 'floral', effects: ['floral'], probability: 0.15 },
    { name: 'double', effects: [], probability: 0.10 }, // 双发烟花
    { name: 'triple', effects: [], probability: 0.05 }, // 三发烟花
    { name: 'finale', effects: ['crossette', 'floral'], probability: 0.10 }, // 终极烟花
    { name: 'ring', effects: ['ring'], probability: 0.10 }, // 环形烟花
    { name: 'spiral', effects: ['spiral'], probability: 0.10 }, // 螺旋形烟花
    { name: 'heart', effects: ['heart'], probability: 0.05 }, // 心形烟花
    { name: 'multicolor', effects: ['multicolor'], probability: 0.05 } // 多彩烟花
  ];

  // 根据草地范围随机生成发射点坐标
  // 草地大小是 2000x2000，中心在(0,-20,0)
  const groundRadius = 1000; // 扩大草地半径
  const position = {
    x: (Math.random() * 2 - 1) * groundRadius * 0.8, // 在草地范围内随机 x 坐标（使用80%的范围以确保在草地上）
    y: -20, // 从地面发射
    z: (Math.random() * 2 - 1) * groundRadius * 0.8  // 在草地范围内随机 z 坐标
  };

  // 根据概率选择烟花类型
  let cumulativeProbability = 0;
  const random = Math.random();
  let type = shellTypes[0]; // 默认选择第一种类型
  
  for (const shellType of shellTypes) {
    cumulativeProbability += shellType.probability;
    if (random < cumulativeProbability) {
      type = shellType;
      break;
    }
  }
  
  // 选择颜色
  let color;
  
  // 多彩烟花使用特定的颜色组合
  if (type.name === 'multicolor') {
    const colorName = COLOR_NAMES[Math.floor(Math.random() * COLOR_NAMES.length)];
    color = { name: colorName, value: colors[colorName] };
    // 多彩效果添加到effects中
    type.effects = ['multicolor'];
  } else {
    color = randomColor();
  }

  // 创建烟花弹
  const shell = createShell({
    position,
    color,
    effects: type.effects,
    spreadSize: 20 + Math.random() * 30
  });

  // 如果是双发烟花，发射另一个相似的烟花
  if (type.name === 'double') {
    setTimeout(() => {
      createShell({
        position: {
          x: position.x + (Math.random() * 10 - 5),
          y: position.y,
          z: position.z + (Math.random() * 10 - 5)
        },
        color,
        effects: type.effects,
        spreadSize: 10 + Math.random() * 15
      });
    }, 100 + Math.random() * 200);
  }
  
  // 如果是三发烟花，发射两个额外的相似烟花
  if (type.name === 'triple') {
    // 发射第二个烟花
    setTimeout(() => {
      createShell({
        position: {
          x: position.x + (Math.random() * 15 - 7.5),
          y: position.y,
          z: position.z + (Math.random() * 15 - 7.5)
        },
        color,
        effects: type.effects,
        spreadSize: 10 + Math.random() * 20
      });
    }, 80 + Math.random() * 150);
    
    // 发射第三个烟花
    setTimeout(() => {
      // 选择互补色或类似色
      let thirdColor;
      if (Math.random() > 0.5) {
        // 使用互补色
        const currentIndex = COLOR_NAMES.indexOf(color.name);
        const complementIndex = (currentIndex + 6) % COLOR_NAMES.length;
        thirdColor = { 
          name: COLOR_NAMES[complementIndex], 
          value: colors[COLOR_NAMES[complementIndex]] 
        };
      } else {
        // 使用原始颜色
        thirdColor = color;
      }
      
      createShell({
        position: {
          x: position.x + (Math.random() * 15 - 7.5),
          y: position.y,
          z: position.z + (Math.random() * 15 - 7.5)
        },
        color: thirdColor,
        effects: type.effects,
        spreadSize: 15 + Math.random() * 25
      });
    }, 150 + Math.random() * 200);
  }
}


// 更新彗星位置
function updateComets(deltaTime) {
  // 彗星速度增强系数
  const cometSpeedFactor = 1.4; // 增加40%的彗星移动速度
  
  for (let i = shells.length - 1; i >= 0; i--) {
    const comet = shells[i];

    // 更新彗星生命周期 - 使用deltaTime使动画与帧率无关
    comet.lifetime += 1000 * deltaTime; // 基于时间的生命值增加

    // 应用物理（重力和空气阻力）- 基于deltaTime调整
    const airDrag = Math.pow(comet.heavy ? 0.998 : 0.95, 60 * deltaTime); // 基于时间的空气阻力
    comet.speedX *= airDrag;
    // 根据质量调整重力效应
    comet.speedY -= GRAVITY * comet.mass * 60 * deltaTime; // 基于时间的重力效应
    comet.speedY *= airDrag;
    comet.speedZ *= airDrag; // Z轴也受空气阻力影响

    // 更新彗星头部位置
    const positions = comet.positions;
    const oldX = positions[0];
    const oldY = positions[1];
    const oldZ = positions[2];
    const newX = oldX + comet.speedX * 60 * deltaTime * cometSpeedFactor;
    const newY = oldY + comet.speedY * 60 * deltaTime * cometSpeedFactor;
    const newZ = oldZ + comet.speedZ * 60 * deltaTime * cometSpeedFactor;

    positions[0] = newX;
    positions[1] = newY;
    positions[2] = newZ;

    comet.positionAttribute.needsUpdate = true;
    
    // 无论彗星状态如何，都保持尾部粒子生成，直到爆炸前
    // 增加粒子生成率，确保尾部可见性
    if (comet.speedY <= 0) {
      // 开始下降时，增加粒子生成率
      comet.particleRate = 12; // 从8增加到12
    }
    
    // 更新尾部粒子
    comet.updateTail(deltaTime);

    // 当速度转为负时（开始下落）或生命周期结束时爆炸
    // 由于我们现在使用角度0（向上发射），所以判断条件需要修改
    if ((comet.speedY <= 0 && comet.lifetime > 500) || comet.lifetime >= comet.life) {
      comet.explode();
      // 移除彗星及其尾部
      scene.remove(comet);
      scene.remove(comet.tail);
      shells.splice(i, 1);
    }
  }
}

// 使用实例化技术创建大量粒子
function createInstancedParticles(count, color, size = 0.6) { // 减小默认粒子尺寸
  // 创建一个单一的粒子形状
  const particleGeometry = new THREE.PlaneGeometry(size, size);

  // 创建实例化Buffer几何体
  const instancedGeometry = new THREE.InstancedBufferGeometry();
  instancedGeometry.index = particleGeometry.index;
  instancedGeometry.attributes = particleGeometry.attributes;

  // 为每个实例创建位置、颜色、大小属性
  const positions = new Float32Array(count * 3);
  const scales = new Float32Array(count);
  const colorArray = new Float32Array(count * 3);
  const opacities = new Float32Array(count);

  // 将十六进制颜色转换为RGB
  const r = ((color >> 16) & 255) / 255;
  const g = ((color >> 8) & 255) / 255;
  const b = (color & 255) / 255;

  // 初始化颜色
  for (let i = 0; i < count; i++) {
    const i3 = i * 3;
    colorArray[i3] = r;
    colorArray[i3 + 1] = g;
    colorArray[i3 + 2] = b;

    // 初始不可见
    opacities[i] = 0;

    // 随机化大小，但整体偏小
    scales[i] = (0.4 + Math.random() * 0.3) * size;
  }

  // 添加实例属性
  instancedGeometry.setAttribute('instancePosition', new THREE.InstancedBufferAttribute(positions, 3));
  instancedGeometry.setAttribute('instanceColor', new THREE.InstancedBufferAttribute(colorArray, 3));
  instancedGeometry.setAttribute('instanceScale', new THREE.InstancedBufferAttribute(scales, 1));
  instancedGeometry.setAttribute('instanceOpacity', new THREE.InstancedBufferAttribute(opacities, 1));

  // 创建彩色贴图
  const particleTexture = createColoredParticleTexture(color);

  // 创建着色器材质
  const material = new THREE.ShaderMaterial({
    uniforms: {
      map: { value: particleTexture },
      isWarm: { value: r > b } // 传递是否为暖色调的标志
    },
    vertexShader: `
      attribute vec3 instancePosition;
      attribute vec3 instanceColor;
      attribute float instanceScale;
      attribute float instanceOpacity;
      
      varying vec3 vColor;
      varying float vOpacity;
      varying vec2 vUv;
      
      void main() {
        vUv = uv;
        vColor = instanceColor;
        vOpacity = instanceOpacity;
        
        // 应用实例位置和缩放
        vec3 transformed = position * instanceScale;
        transformed += instancePosition;
        
        vec4 mvPosition = modelViewMatrix * vec4(transformed, 1.0);
        gl_Position = projectionMatrix * mvPosition;
      }
    `,
    fragmentShader: `
      uniform sampler2D map;
      uniform bool isWarm;
      
      varying vec3 vColor;
      varying float vOpacity;
      varying vec2 vUv;
      
      void main() {
        // 获取基本纹理颜色
        vec4 texColor = texture2D(map, vUv);
        
        // 调整颜色，使其更符合焰火特性
        vec3 finalColor = vColor;
        
        // 为暖色调增加一点红色和黄色，为冷色调增加一点蓝色
        if (isWarm) {
          finalColor.r = min(finalColor.r * 1.1, 1.0);
          finalColor.g = min(finalColor.g * 1.05, 1.0);
        } else {
          finalColor.b = min(finalColor.b * 1.1, 1.0);
        }
        
        // 亮度增强
        float brightness = (finalColor.r + finalColor.g + finalColor.b) / 3.0;
        float boost = 1.5 + brightness * 0.8;
        finalColor *= boost;
        
        gl_FragColor = vec4(finalColor, vOpacity) * texColor;
      }
    `,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false
  });

  // 创建实例化网格
  const instancedMesh = new THREE.Mesh(instancedGeometry, material);
  scene.add(instancedMesh);

  return {
    mesh: instancedMesh,
    positions: positions,
    opacities: opacities,
    scales: scales,
    colors: colorArray,
    count: count,
    positionAttribute: instancedGeometry.getAttribute('instancePosition'),
    opacityAttribute: instancedGeometry.getAttribute('instanceOpacity')
  };
}

// 更新实例化粒子系统 - 添加视锥体剔除
function updateInstancedSystems() {
  // 创建视锥体用于视锥剔除
  const frustum = new THREE.Frustum();
  const cameraViewProjectionMatrix = new THREE.Matrix4();
  
  // 计算当前视锥体
  camera.updateMatrixWorld();
  cameraViewProjectionMatrix.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
  frustum.setFromProjectionMatrix(cameraViewProjectionMatrix);
  
  for (let i = instancedSystems.length - 1; i >= 0; i--) {
    const system = instancedSystems[i];
    
    // 如果粒子系统不在视锥体内，跳过更新
    if (system.mesh && !frustum.intersectsObject(system.mesh)) {
      continue;
    }
    
    const continueUpdating = system.update();
    if (!continueUpdating) {
      // 回收资源
      if (system.mesh) {
        scene.remove(system.mesh);
        if (system.mesh.geometry) system.mesh.geometry.dispose();
        if (system.mesh.material) system.mesh.material.dispose();
      }
      instancedSystems.splice(i, 1);
    }
  }
}

// 更新烟花状态
function updateFireworks(deltaTime) {
  updateComets(deltaTime);
  updateParticles(deltaTime);
  updateInstancedSystems(deltaTime); // 添加实例化系统更新
}

// 创建全局时钟对象，用于时间控制
const clock = new THREE.Clock();
// 上一次发射烟花的时间
let lastFireworkTime = 0;
// 烟花发射间隔
let fireworkInterval = 0.5; // 初始值，将在发射后更新
// 上一帧的时间，用于计算增量时间
let lastFrameTime = 0;
// 帧率控制
let frameCount = 0;

// 动画循环 - 优化渲染循环
function animate() {
  requestAnimationFrame(animate);
  
  // 计算上一帧到当前帧的时间增量
  const currentTime = clock.getElapsedTime();
  const deltaTime = currentTime - lastFrameTime;
  
  // 确保deltaTime有一个合理的上限，防止因页面非活动时产生过大的deltaTime
  const cappedDeltaTime = Math.min(deltaTime, 0.1);
  
  // 性能监控
  if (window.statsInstance) {
    window.statsInstance.begin();
  }
  
  // 增量帧计数器
  frameCount++;
  
  // 在重型场景中（大量粒子）可选择跳过某些帧的非关键计算
  // 当粒子数量过多时，可以跳过一些帧的物理更新但保持视觉渲染
  const isHeavyScene = stars.length > 500 || instancedSystems.length > 5;
  const skipComputation = isHeavyScene && (frameCount % 2 !== 0);
  
  // 相机更新
  updateCameraPosition(cappedDeltaTime);
  if (controls) controls.update();
  
  // 只有在不跳过计算的帧中更新物理
  if (!skipComputation) {
    updateFireworks(cappedDeltaTime);
    
    // 烟花发射控制
    if (currentTime - lastFireworkTime > fireworkInterval && shells.length < 10) {
      launchShell();
      lastFireworkTime = currentTime;
      fireworkInterval = getRandomFireworkInterval();
    }
  }
  
  // 总是渲染场景
  renderer.render(scene, camera);
  
  // 更新时间状态
  lastFrameTime = currentTime;
  
  // 结束性能统计
  if (window.statsInstance) {
    window.statsInstance.end();
  }
}

// 处理窗口大小变化
function onWindowResize() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}

// 随机化烟花发射间隔
function getRandomFireworkInterval() {
  // 生成0.3到1.2秒的随机间隔
  return 0.3 + Math.random() * 0.9;
}

// 对象池实现 - 为粒子几何体和材质
const geometryPool = [];
const materialPool = [];

// 从对象池获取几何体
function getGeometryFromPool() {
  if (geometryPool.length > 0) {
    return geometryPool.pop();
  }
  return new THREE.BufferGeometry();
}

// 回收几何体到对象池
function recycleGeometry(geometry) {
  if (geometry && geometryPool.length < 100) {
    geometry.dispose(); // 清理
    geometryPool.push(geometry);
  }
}

// 从对象池获取材质
function getMaterialFromPool(color) {
  if (materialPool.length > 0) {
    const material = materialPool.pop();
    material.color.set(color);
    return material;
  }
  
  // 创建新材质
  const texture = createColoredParticleTexture(color);
  const material = new THREE.PointsMaterial({
    size: 5.0,
    map: texture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
    vertexColors: true
  });
  material.userData.originalSize = 5.0;
  return material;
}

// 回收材质到对象池
function recycleMaterial(material) {
  if (material && materialPool.length < 30) {
    material.dispose();
    materialPool.push(material);
  }
}

// 生命周期钩子
onMounted(() => {
  // 创建一个新的统计面板
  const stats = new Stats();
  document.body.appendChild(stats.dom); // 将统计面板添加到页面上
  stats.dom.style.position = 'absolute'; // 设置定位为绝对定位
  stats.dom.style.top = '0px'; // 靠顶部定位
  stats.dom.style.left = '0px'; // 靠左侧定位
  stats.dom.style.zIndex = '100'; // 确保在最上层
  
  // stats模式: 0:FPS, 1:MS, 2:MB
  stats.showPanel(0); // 默认显示FPS面板
  
  initScene();
  
  // 初始化烟花发射间隔
  fireworkInterval = getRandomFireworkInterval();
  
  // 动画循环中使用这个stats实例
  window.statsInstance = stats;
  
  animate();
  window.addEventListener('resize', onWindowResize);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize);
  
  // 清理资源
  if (renderer) renderer.dispose();
  
  // 清理所有几何体和材质
  scene.traverse(object => {
    if (object.geometry) object.geometry.dispose();
    if (object.material) {
      if (Array.isArray(object.material)) {
        object.material.forEach(material => material.dispose());
      } else {
        object.material.dispose();
      }
    }
  });
  
  // 移除性能统计面板
  if (window.statsInstance && window.statsInstance.dom && window.statsInstance.dom.parentNode) {
    window.statsInstance.dom.parentNode.removeChild(window.statsInstance.dom);
    window.statsInstance = null;
  }
});
</script>

<style scoped>
.firework-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}
</style>