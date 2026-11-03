<template>
  <div class="earth-3d-container">
    <!-- Three.js 渲染器会把 WebGL canvas 插入这个容器。ref 让脚本可以拿到真实 DOM 尺寸。 -->
    <div ref="canvasContainer" class="canvas-container"></div>

    <!--
      这些卡片不是 Three.js 物体，而是普通 HTML。
      updateHudPositions() 每帧把 3D 坐标换算成 screenX/screenY，Vue 再通过 :style 更新位置。
    -->
    <div v-for="tag in hudTags" :key="tag.id"
         class="hud-card"
         :style="{
           left: tag.screenX + 'px',
           top: tag.screenY + 'px',
           opacity: tag.visible ? 1 : 0,
           display: tag.onScreen ? 'block' : 'none'
         }">
      <div class="hud-line"></div>
      <div class="hud-content">
        <div class="hud-title">{{ tag.title }}</div>
        <div class="hud-value">{{ tag.value }}</div>
        <div class="hud-sub">{{ tag.sub }}</div>
      </div>
    </div>

    <!-- 远程地图还未完成采样时显示加载层；loading 变为 false 后整个节点会被 Vue 移除。 -->
    <div v-if="loading" class="loading-overlay">
      <div class="spinner"></div>
      <div class="loading-text">正在初始化3D数字地球... {{ loadingProgress }}%</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, reactive } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// 保存模板需要访问的 DOM 容器。组件挂载前它是 null，onMounted 后才有真实元素。
const canvasContainer = ref<HTMLDivElement | null>(null);
// loading 控制遮罩是否显示，loadingProgress 控制遮罩中的百分比文字。
const loading = ref(true);
const loadingProgress = ref(0);

// 一个 HUD 标签同时保存“3D 世界数据”和“HTML 屏幕数据”。
interface HudTag {
  id: string; // Vue v-for 使用的唯一 key。
  title: string; // 标签标题，例如“IT 设备节点”。
  value: string; // 需要突出显示的统计数值。
  sub: string; // 标签底部的补充说明。
  pos: THREE.Vector3; // 标签在 globeGroup 局部坐标中的 3D 位置。
  screenX: number; // 投影后的 HTML 横坐标，单位是 px。
  screenY: number; // 投影后的 HTML 纵坐标，单位是 px。
  visible: boolean; // 3D 点是否朝向摄像机，决定透明度。
  onScreen: boolean; // 投影结果是否在摄像机可视深度内，决定是否显示。
}

const hudTags = reactive<HudTag[]>([
  {
    id: 'tag-1',
    title: 'IT设备节点',
    value: '26 台 在线',
    sub: '1F 机房核心群组',
    pos: new THREE.Vector3(1.5, 1.2, 1.0),
    screenX: 0,
    screenY: 0,
    visible: false,
    onScreen: false,
  },
  {
    id: 'tag-2',
    title: '智能物联网关',
    value: '26 个 活跃',
    sub: '星潮全域网关路由',
    pos: new THREE.Vector3(-1.8, -0.5, 1.2),
    screenX: 0,
    screenY: 0,
    visible: false,
    onScreen: false,
  },
  {
    id: 'tag-3',
    title: '安全运维告警',
    value: '178 条 未处理',
    sub: '今日新增 23 条',
    pos: new THREE.Vector3(1.2, -1.2, -1.5),
    screenX: 0,
    screenY: 0,
    visible: false,
    onScreen: false,
  }
]);

// Three.js 的四个基础对象：场景装载所有物体，相机决定观察角度，渲染器输出像素，控制器处理鼠标交互。
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let animationFrameId: number;

// 按功能拆分的组。旋转组本身即可带动内部所有子物体一起运动。
let globeGroup: THREE.Group;
let baseGroup: THREE.Group;
let orbitsGroup: THREE.Group;

// 每颗“数据卫星”记录轨迹参数和当前进度，animate() 会持续更新它们的位置。
interface Satellite {
  mesh: THREE.Group; // 卫星主体加拖尾球体组成的组。
  curve: { a: number; b: number; rx: number; ry: number; rz: number }; // 椭圆长短轴和空间旋转角。
  speed: number; // 每帧前进的角度。
  progress: number; // 当前在椭圆上的弧度位置。
}
const satellites: Satellite[] = [];

// 光柱和火花粒子的共享数据。BufferGeometry 直接引用 sparksPositions，因此改数组即可更新顶点。
let pillarMesh: THREE.Mesh;
let sparksGeometry: THREE.BufferGeometry;
let sparksPositions: Float32Array;
let sparksVelocities: number[];
const sparkCount = 60;

// 组件挂载后初始化 Three.js 场景、创建模型，并开始逐帧渲染。
onMounted(() => {
  if (!canvasContainer.value) return;

  initThree();
  createScene();
  animate();

  window.addEventListener('resize', onWindowResize);
});

// 组件卸载时停止动画、解绑窗口事件，并释放 Three.js 控制器和渲染器。
onUnmounted(() => {
  cancelAnimationFrame(animationFrameId);
  window.removeEventListener('resize', onWindowResize);
  
  // Dispose resources
  if (renderer) {
    renderer.dispose();
  }
  if (controls) {
    controls.dispose();
  }
});

/**
 * 初始化 Three.js 的基础运行环境。
 * 创建场景、透视相机、WebGL 渲染器和鼠标轨道控制器。
 */
function initThree() {
  const container = canvasContainer.value!;
  const width = container.clientWidth;
  const height = container.clientHeight;

  // 场景是所有灯光、模型和粒子的根容器；雾会让远处轨迹自然融入深色背景。
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x020515, 0.015);

  // 透视相机模拟真实镜头。45 是视野角，near/far 是可见深度范围。
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100);
  camera.position.set(0, 2, 10);

  // WebGLRenderer 负责把 scene 和 camera 渲染到 canvas；alpha 允许看到组件的背景色。
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  container.appendChild(renderer.domElement);

  // OrbitControls 让用户可以拖拽旋转、滚轮缩放；阻尼使停止操作后的运动更平滑。
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 5;
  controls.maxDistance = 20;
  controls.maxPolarAngle = Math.PI / 2 + 0.15; // 限制仰角，避免镜头钻到底座下方。
}

/**
 * 绘制一个简化的世界地图。
 * 当远程地图图片加载失败时，用这张 Canvas 图作为地球陆地采样依据。
 */
function drawFallbackWorldMap(ctx: CanvasRenderingContext2D, w: number, h: number) {
  // 黑色代表海洋，白色代表陆地。后续 buildProceduralGlobe() 会按像素亮度读取这张图。
  ctx.fillStyle = '#000000';
  ctx.fillRect(0, 0, w, h);
  ctx.fillStyle = '#ffffff';

  // 这里不是要绘制精确地图，而是提供网络失败时仍可工作的稳定轮廓。
  // 欧亚大陆和非洲先绘制成一个大块，再用下面的非洲细节修正形状。
  ctx.beginPath();
  ctx.moveTo(w * 0.45, h * 0.2);
  ctx.lineTo(w * 0.85, h * 0.15);
  ctx.lineTo(w * 0.95, h * 0.35);
  ctx.lineTo(w * 0.9, h * 0.55);
  ctx.lineTo(w * 0.8, h * 0.5);
  ctx.lineTo(w * 0.75, h * 0.85); // India / Africa split
  ctx.lineTo(w * 0.55, h * 0.8);
  ctx.lineTo(w * 0.5, h * 0.4);
  ctx.closePath();
  ctx.fill();

  // 非洲单独补一块更窄的轮廓，让默认地图生成的粒子形状更容易辨认。
  ctx.beginPath();
  ctx.moveTo(w * 0.48, h * 0.38);
  ctx.lineTo(w * 0.62, h * 0.38);
  ctx.lineTo(w * 0.65, h * 0.55);
  ctx.lineTo(w * 0.58, h * 0.85);
  ctx.lineTo(w * 0.5, h * 0.75);
  ctx.closePath();
  ctx.fill();

  // 北美洲轮廓。
  ctx.beginPath();
  ctx.moveTo(w * 0.05, h * 0.15);
  ctx.lineTo(w * 0.35, h * 0.12);
  ctx.lineTo(w * 0.42, h * 0.35);
  ctx.lineTo(w * 0.3, h * 0.45);
  ctx.lineTo(w * 0.22, h * 0.45);
  ctx.closePath();
  ctx.fill();

  // 南美洲轮廓。
  ctx.beginPath();
  ctx.moveTo(w * 0.25, h * 0.48);
  ctx.lineTo(w * 0.35, h * 0.48);
  ctx.lineTo(w * 0.38, h * 0.65);
  ctx.lineTo(w * 0.32, h * 0.88);
  ctx.lineTo(w * 0.28, h * 0.75);
  ctx.closePath();
  ctx.fill();

  // 澳大利亚用椭圆点表示，避免绘制复杂路径。
  ctx.beginPath();
  ctx.arc(w * 0.85, h * 0.72, w * 0.05, 0, Math.PI * 2);
  ctx.fill();

  // 格陵兰岛轮廓。
  ctx.beginPath();
  ctx.ellipse(w * 0.38, h * 0.1, w * 0.04, h * 0.06, Math.PI / 4, 0, Math.PI * 2);
  ctx.fill();

  // 用几个小矩形补充日本、东南亚等岛屿区域，避免地图过于空洞。
  ctx.fillStyle = '#ffffff';
  ctx.fillRect(w * 0.78, h * 0.45, w * 0.02, h * 0.03); // Japan / Islands
  ctx.fillRect(w * 0.82, h * 0.52, w * 0.03, h * 0.04); // SE Asia
  ctx.fillRect(w * 0.92, h * 0.25, w * 0.02, h * 0.02); // NE Russia
}

/**
 * 组装完整的 3D 场景。
 * 这里负责添加灯光、地球、底座和轨迹线三个主要模块。
 */
function createScene() {
  // Add lighting
  const ambientLight = new THREE.AmbientLight(0x0a1128, 2);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0x00ffff, 2.5);
  dirLight.position.set(5, 5, 5);
  scene.add(dirLight);

  const blueLight = new THREE.PointLight(0x3a0ca3, 10, 30);
  blueLight.position.set(-4, 3, -4);
  scene.add(blueLight);

  const cyanLight = new THREE.PointLight(0x00f0ff, 15, 20);
  cyanLight.position.set(0, -2.8, 0); // At the pedestal center
  scene.add(cyanLight);

  // Initialize Groups
  globeGroup = new THREE.Group();
  baseGroup = new THREE.Group();
  orbitsGroup = new THREE.Group();
  
  scene.add(globeGroup);
  scene.add(baseGroup);
  scene.add(orbitsGroup);

  // Position base group below
  baseGroup.position.y = -2.8;

  // 1. Create Pedestal/Base
  createPedestal();

  // 2. Create Earth (Dotted Globe)
  createDottedGlobe();

  // 3. Create Orbits & Trajectories
  createOrbits();
}

// ------------------------------
// 1. Pedestal Creation
// ------------------------------
/**
 * 创建地球下方的科幻底座。
 * 包含 Canvas 科技纹理、实体环形结构、中心反应堆、光柱和能量粒子。
 */
function createPedestal() {
  // We will create multiple concentric rings lying flat on XZ plane
  
  // Custom canvas for sci-fi tech texture (highly detailed)
  const size = 512;
  const canvas = document.createElement('canvas');
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext('2d')!;

  // Clear transparent
  ctx.clearRect(0, 0, size, size);

  const cx = size / 2;
  const cy = size / 2;

  // 在底座 Canvas 纹理上绘制圆弧，支持实线和虚线。
  const drawArc = (r: number, start: number, end: number, color: string, width: number, dashed: number[] = []) => {
    ctx.strokeStyle = color;
    ctx.lineWidth = width;
    ctx.beginPath();
    ctx.arc(cx, cy, r, start, end);
    ctx.setLineDash(dashed);
    ctx.stroke();
  };

  // Outer Ring 1: Very thin circle with tick marks
  drawArc(240, 0, Math.PI * 2, 'rgba(0, 240, 255, 0.2)', 1.5);
  
  // Outer Ring 2: Ticks and Degrees (compass style)
  ctx.strokeStyle = 'rgba(0, 240, 255, 0.4)';
  ctx.lineWidth = 1;
  ctx.setLineDash([]);
  for (let i = 0; i < 72; i++) {
    const angle = (i * 5 * Math.PI) / 180;
    const len = i % 2 === 0 ? 10 : 5;
    const startX = cx + Math.cos(angle) * 235;
    const startY = cy + Math.sin(angle) * 235;
    const endX = cx + Math.cos(angle) * (235 - len);
    const endY = cy + Math.sin(angle) * (235 - len);
    ctx.beginPath();
    ctx.moveTo(startX, startY);
    ctx.lineTo(endX, endY);
    ctx.stroke();
  }

  // Middle Ring 3: Segmented bold arcs
  for (let i = 0; i < 4; i++) {
    const start = (i * 90 + 10) * Math.PI / 180;
    const end = (i * 90 + 80) * Math.PI / 180;
    drawArc(210, start, end, 'rgba(0, 160, 255, 0.6)', 3, []);
  }

  // Inner Ring 4: Dashed quick rotation ring
  drawArc(180, 0, Math.PI * 2, 'rgba(0, 240, 255, 0.5)', 2, [5, 10]);

  // Tech Ring 5: Blocks and markers
  for (let i = 0; i < 8; i++) {
    const start = (i * 45 + 5) * Math.PI / 180;
    const end = (i * 45 + 25) * Math.PI / 180;
    drawArc(150, start, end, 'rgba(0, 255, 255, 0.8)', 6, []);
  }

  // Core ring
  drawArc(100, 0, Math.PI * 2, 'rgba(0, 120, 255, 0.3)', 1, []);
  drawArc(90, 0, Math.PI * 2, 'rgba(0, 240, 255, 0.7)', 2, [20, 10]);
  drawArc(60, 0, Math.PI * 2, 'rgba(0, 240, 255, 0.3)', 1, []);

  // Central filled circle representing core reactor
  const radGrad = ctx.createRadialGradient(cx, cy, 0, cx, cy, 50);
  radGrad.addColorStop(0, 'rgba(0, 255, 255, 0.8)');
  radGrad.addColorStop(0.3, 'rgba(0, 150, 255, 0.4)');
  radGrad.addColorStop(1, 'rgba(0, 50, 255, 0)');
  ctx.fillStyle = radGrad;
  ctx.beginPath();
  ctx.arc(cx, cy, 50, 0, Math.PI * 2);
  ctx.fill();

  const baseTexture = new THREE.CanvasTexture(canvas);
  
  // Create 3D planes for rotating base elements
  const planeGeo = new THREE.PlaneGeometry(8, 8);
  const planeMat = new THREE.MeshBasicMaterial({
    map: baseTexture,
    transparent: true,
    side: THREE.DoubleSide,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });

  const basePlateMesh = new THREE.Mesh(planeGeo, planeMat);
  basePlateMesh.rotation.x = -Math.PI / 2;
  baseGroup.add(basePlateMesh);

  // 下面的 TorusGeometry 是真正有厚度的 3D 环，和上面的平面纹理叠加后，底座才不会像一张贴图。
  const ringMaterials = [
    new THREE.MeshBasicMaterial({
      color: 0x006dff,
      transparent: true,
      opacity: 0.72,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    }),
    new THREE.MeshBasicMaterial({
      color: 0x00eaff,
      transparent: true,
      opacity: 0.9,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    }),
    new THREE.MeshBasicMaterial({
      color: 0x8c4dff,
      transparent: true,
      opacity: 0.75,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    }),
  ];

  const baseRings = [
    { radius: 3.5, tube: 0.018, y: 0.06, material: ringMaterials[0], speed: 0.0012 },
    { radius: 2.85, tube: 0.045, y: 0.12, material: ringMaterials[1], speed: -0.0022 },
    { radius: 2.35, tube: 0.025, y: 0.2, material: ringMaterials[2], speed: 0.003 },
    { radius: 1.75, tube: 0.055, y: 0.27, material: ringMaterials[1], speed: -0.004 },
  ];
  // 将每组底座环参数转换为可旋转的 TorusGeometry 实体。
  baseRings.forEach((ring, index) => {
    const mesh = new THREE.Mesh(
      new THREE.TorusGeometry(ring.radius, ring.tube, 8, 128),
      ring.material,
    );
    mesh.rotation.x = Math.PI / 2;
    mesh.position.y = ring.y;
    mesh.name = `baseRing${index}`;
    mesh.userData.speed = ring.speed;
    baseGroup.add(mesh);
  });

  // 中央反应堆分成上下两层圆柱：下层是外壳，上层是发光平台，用高度差制造立体台阶。
  const lowerHousing = new THREE.Mesh(
    new THREE.CylinderGeometry(1.28, 1.7, 0.18, 64),
    new THREE.MeshBasicMaterial({
      color: 0x061a45,
      transparent: true,
      opacity: 0.95,
      side: THREE.DoubleSide,
    }),
  );
  lowerHousing.position.y = 0.02;
  baseGroup.add(lowerHousing);

  const reactorPlate = new THREE.Mesh(
    new THREE.CylinderGeometry(0.86, 1.16, 0.1, 64),
    new THREE.MeshBasicMaterial({
      color: 0x0a3d82,
      transparent: true,
      opacity: 0.9,
      blending: THREE.AdditiveBlending,
    }),
  );
  reactorPlate.position.y = 0.18;
  baseGroup.add(reactorPlate);

  // 12 个径向鳍片均匀分布在中心周围，形成科幻机械底座的分段轮廓。
  for (let i = 0; i < 12; i++) {
    const angle = (i / 12) * Math.PI * 2;
    const fin = new THREE.Mesh(
      new THREE.BoxGeometry(0.62, 0.035, 0.12),
      new THREE.MeshBasicMaterial({
        color: i % 3 === 0 ? 0xff6c3b : 0x00dfff,
        transparent: true,
        opacity: 0.8,
        blending: THREE.AdditiveBlending,
      }),
    );
    fin.position.set(Math.cos(angle) * 2.45, 0.16, Math.sin(angle) * 2.45);
    fin.rotation.y = -angle;
    baseGroup.add(fin);
  }

  // We can also create a separate outer tick mesh rotating in the opposite direction
  const outerCanvas = document.createElement('canvas');
  outerCanvas.width = 512;
  outerCanvas.height = 512;
  const oCtx = outerCanvas.getContext('2d')!;
  oCtx.clearRect(0, 0, 512, 512);
  // Outer rings
  oCtx.strokeStyle = 'rgba(0, 240, 255, 0.6)';
  oCtx.lineWidth = 2;
  // Draw some triangles/arrows pointing inwards
  for (let i = 0; i < 12; i++) {
    const angle = (i * 30 * Math.PI) / 180;
    const x = cx + Math.cos(angle) * 215;
    const y = cy + Math.sin(angle) * 215;
    
    oCtx.save();
    oCtx.translate(x, y);
    oCtx.rotate(angle + Math.PI / 2);
    oCtx.fillStyle = 'rgba(0, 240, 255, 0.8)';
    oCtx.beginPath();
    oCtx.moveTo(0, -6);
    oCtx.lineTo(-4, 2);
    oCtx.lineTo(4, 2);
    oCtx.closePath();
    oCtx.fill();
    oCtx.restore();
  }
  
  const outerTexture = new THREE.CanvasTexture(outerCanvas);
  const outerPlateMesh = new THREE.Mesh(
    new THREE.PlaneGeometry(7.5, 7.5),
    new THREE.MeshBasicMaterial({
      map: outerTexture,
      transparent: true,
      side: THREE.DoubleSide,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
  );
  outerPlateMesh.rotation.x = -Math.PI / 2;
  outerPlateMesh.name = "rotatingOuterRing";
  baseGroup.add(outerPlateMesh);

  // 1.2 Glow Grid Ground
  const gridTextureCanvas = document.createElement('canvas');
  gridTextureCanvas.width = 256;
  gridTextureCanvas.height = 256;
  const gCtx = gridTextureCanvas.getContext('2d')!;
  const gGrad = gCtx.createRadialGradient(128, 128, 0, 128, 128, 128);
  gGrad.addColorStop(0, 'rgba(0, 100, 255, 0.16)');
  gGrad.addColorStop(0.5, 'rgba(0, 50, 200, 0.06)');
  gGrad.addColorStop(1, 'rgba(0, 0, 0, 0)');
  gCtx.fillStyle = gGrad;
  gCtx.fillRect(0, 0, 256, 256);
  
  const groundGlowGeo = new THREE.PlaneGeometry(12, 12);
  const groundGlowMat = new THREE.MeshBasicMaterial({
    map: new THREE.CanvasTexture(gridTextureCanvas),
    transparent: true,
    side: THREE.DoubleSide,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });
  const groundGlow = new THREE.Mesh(groundGlowGeo, groundGlowMat);
  groundGlow.rotation.x = -Math.PI / 2;
  groundGlow.position.y = -0.05; // Slightly below other rings
  baseGroup.add(groundGlow);

  // 1.3 Central Vertical Light Pillar
  // Linear gradient texture for cylinder fading to top
  const pCanvas = document.createElement('canvas');
  pCanvas.width = 16;
  pCanvas.height = 256;
  const pCtx = pCanvas.getContext('2d')!;
  const pGrad = pCtx.createLinearGradient(0, 256, 0, 0); // bottom to top
  pGrad.addColorStop(0.0, 'rgba(0, 255, 255, 0.85)');
  pGrad.addColorStop(0.1, 'rgba(0, 200, 255, 0.55)');
  pGrad.addColorStop(0.4, 'rgba(0, 100, 255, 0.25)');
  pGrad.addColorStop(0.7, 'rgba(0, 50, 255, 0.08)');
  pGrad.addColorStop(1.0, 'rgba(0, 0, 100, 0)');
  pCtx.fillStyle = pGrad;
  pCtx.fillRect(0, 0, 16, 256);
  
  const pillarTexture = new THREE.CanvasTexture(pCanvas);
  
  // Cylinder pointing up from -2.8 to 0. Height is 2.8, positioned at y = 1.4 (relative to -2.8 is y = -1.4)
  const pillarGeo = new THREE.CylinderGeometry(0.5, 1.4, 3.2, 32, 1, true);
  const pillarMat = new THREE.MeshBasicMaterial({
    map: pillarTexture,
    transparent: true,
    side: THREE.DoubleSide,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });
  
  pillarMesh = new THREE.Mesh(pillarGeo, pillarMat);
  pillarMesh.position.y = 1.6; // Center of 3.2 height cylinder starting at y=0 relative to base
  baseGroup.add(pillarMesh);

  // 1.4 Floating Sparks (Particles inside light pillar)
  createSparks();
}

/**
 * 创建光柱内部的能量火花粒子。
 * 粒子初始分布在圆柱体内，后续由 animate() 负责向上移动和重置。
 */
function createSparks() {
  sparksGeometry = new THREE.BufferGeometry();
  sparksPositions = new Float32Array(sparkCount * 3);
  sparksVelocities = [];

  for (let i = 0; i < sparkCount; i++) {
    // sparks are placed in a cylinder-like shape around center
    const radius = Math.random() * 0.9;
    const angle = Math.random() * Math.PI * 2;
    const px = Math.cos(angle) * radius;
    const py = Math.random() * 3.2; // vertical position up to cylinder height
    const pz = Math.sin(angle) * radius;

    sparksPositions[i * 3] = px;
    sparksPositions[i * 3 + 1] = py;
    sparksPositions[i * 3 + 2] = pz;

    // Upward velocity and a tiny wiggle
    sparksVelocities.push(
      0.015 + Math.random() * 0.02, // speed y
      (Math.random() - 0.5) * 0.005, // speed x
      (Math.random() - 0.5) * 0.005  // speed z
    );
  }

  sparksGeometry.setAttribute('position', new THREE.BufferAttribute(sparksPositions, 3));

  // Particle texture
  const sparkTextureCanvas = document.createElement('canvas');
  sparkTextureCanvas.width = 16;
  sparkTextureCanvas.height = 16;
  const sCtx = sparkTextureCanvas.getContext('2d')!;
  const sGrad = sCtx.createRadialGradient(8, 8, 0, 8, 8, 8);
  sGrad.addColorStop(0, 'rgba(255, 255, 255, 1)');
  sGrad.addColorStop(0.4, 'rgba(0, 240, 255, 0.8)');
  sGrad.addColorStop(1, 'rgba(0, 100, 255, 0)');
  sCtx.fillStyle = sGrad;
  sCtx.fillRect(0, 0, 16, 16);

  const sparkTexture = new THREE.CanvasTexture(sparkTextureCanvas);

  const sparksMat = new THREE.PointsMaterial({
    size: 0.12,
    map: sparkTexture,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    opacity: 0.8,
  });

  const sparks = new THREE.Points(sparksGeometry, sparksMat);
  sparks.name = "sparks";
  baseGroup.add(sparks);
}

// ------------------------------
// 2. Dotted Globe Creation
// ------------------------------
/**
 * 创建数字地球的基础外观。
 * 先创建深色球体、边缘大气光晕和经纬网，再异步加载地图生成陆地点阵。
 */
function createDottedGlobe() {
  const globeRadius = 2.0;

  // Let's create an inner glowing sphere mesh to give the globe physical volume
  const sphereGeo = new THREE.SphereGeometry(globeRadius * 0.98, 32, 32);
  const sphereMat = new THREE.MeshBasicMaterial({
    color: 0x051330,
    transparent: true,
    opacity: 0.7,
  });
  const sphere = new THREE.Mesh(sphereGeo, sphereMat);
  globeGroup.add(sphere);
  const innerSphere = new THREE.Mesh(sphereGeo, sphereMat);
  globeGroup.add(innerSphere);

  // Atmosphere Glow Mesh
  // Let's write a simple shader for the glow edge effect
  const glowGeo = new THREE.SphereGeometry(globeRadius * 1.05, 32, 32);
  const glowMat = new THREE.ShaderMaterial({
    vertexShader: `
      varying vec3 vNormal;
      varying vec3 vPositionNormal;
      void main() {
        vNormal = normalize(normalMatrix * normal);
        vPositionNormal = normalize((modelViewMatrix * vec4(position, 1.0)).xyz);
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      varying vec3 vNormal;
      varying vec3 vPositionNormal;
      void main() {
        float intensity = pow(0.65 - dot(vNormal, vec3(0.0, 0.0, 1.0)), 2.5);
        gl_FragColor = vec4(0.0, 0.75, 1.0, 1.0) * intensity * 1.5;
      }
    `,
    transparent: true,
    blending: THREE.AdditiveBlending,
    side: THREE.BackSide,
    depthWrite: false,
  });
  const glowMesh = new THREE.Mesh(glowGeo, glowMat);
  globeGroup.add(glowMesh);

  // Lat / Long wireframe grids
  const gridGeo = new THREE.SphereGeometry(globeRadius * 1.005, 24, 24);
  const gridMat = new THREE.MeshBasicMaterial({
    color: 0x0066aa,
    wireframe: true,
    transparent: true,
    opacity: 0.15,
    depthWrite: false,
  });
  const gridMesh = new THREE.Mesh(gridGeo, gridMat);
  globeGroup.add(gridMesh);

  // Try to load world map texture, fall back to procedural if it fails
  const imgUrl = 'https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/textures/planets/earth_specular_2048.jpg';
  const loader = new THREE.ImageLoader();
  loader.setCrossOrigin('anonymous');

  // Load image
  // 图片成功、加载进度和加载失败分别进入三个回调。
  loader.load(
    imgUrl,
    // 地图加载成功：根据图片像素生成地球粒子。
    (image) => {
      buildGlobeFromMap(image);
    },
    // 地图加载进度：同步更新页面上的加载百分比。
    (xhr) => {
      if (xhr.total > 0) {
        loadingProgress.value = Math.floor((xhr.loaded / xhr.total) * 100);
      }
    },
    // 地图加载失败：切换到本地 Canvas 简化地图。
    () => {
      // Error loading, use fallback
      console.warn("Failed to load map texture. Creating procedural high-tech fallback globe!");
      buildProceduralGlobe();
    }
  );
}

// Uniformly distribute points on sphere using Fibonacci Grid
// and keep only those on landmasses
/**
 * 从地图图片中提取陆地像素，并转换为球面粒子坐标。
 * 通过 Fibonacci 球面采样让点分布均匀，再根据像素亮度区分陆地和海洋。
 */
function buildGlobeFromMap(image: HTMLImageElement) {
  const w = 512;
  const h = 256;
  const canvas = document.createElement('canvas');
  canvas.width = w;
  canvas.height = h;
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(image, 0, 0, w, h);
  const imgData = ctx.getImageData(0, 0, w, h).data;

  // Let's sample points using Fibonacci Sphere
  const numPoints = 12000;
  const globeRadius = 2.0;
  
  const positions: number[] = [];
  const colors: number[] = [];

  for (let i = 0; i < numPoints; i++) {
    // Fibonacci Sphere math
    const phi = Math.acos(1 - (2 * i) / numPoints);
    const theta = Math.sqrt(numPoints * Math.PI) * phi;

    const x = globeRadius * Math.sin(phi) * Math.cos(theta);
    const y = globeRadius * Math.cos(phi);
    const z = globeRadius * Math.sin(phi) * Math.sin(theta);

    // Map to UV coordinates
    const u = 1 - (theta / (Math.PI * 2) % 1 + 1) % 1;
    const v = phi / Math.PI;

    const px = Math.floor(u * w);
    const py = Math.floor(v * h);
    const index = (py * w + px) * 4;

    // Check specular map (in specular maps, water is highly reflective/white, land is dark/black)
    // Actually, mrdoob specular map: land is black (0), oceans are white/light (~200+).
    // Let's check brightness
    const r = imgData[index];
    const isLand = r < 50; // threshold for land

    if (isLand) {
      positions.push(x, y, z);
      // Bright cyan-blue color
      colors.push(0.0, 1.0, 1.0); // R, G, B
    } else {
      // Ocean point - keep 10% of them with dark blue color to show sphere shape
      if (Math.random() < 0.12) {
        positions.push(x, y, z);
        colors.push(0.0, 0.15, 0.4); // Very dim blue
      }
    }
  }

  createGlobePoints(positions, colors);
  loading.value = false;
}

/**
 * 使用本地绘制的简化地图生成备用数字地球。
 * 这是远程地图加载失败时的离线兜底方案。
 */
function buildProceduralGlobe() {
  const w = 256;
  const h = 128;
  const canvas = document.createElement('canvas');
  canvas.width = w;
  canvas.height = h;
  const ctx = canvas.getContext('2d')!;
  
  drawFallbackWorldMap(ctx, w, h);
  const imgData = ctx.getImageData(0, 0, w, h).data;

  const numPoints = 10000;
  const globeRadius = 2.0;
  const positions: number[] = [];
  const colors: number[] = [];

  for (let i = 0; i < numPoints; i++) {
    const phi = Math.acos(1 - (2 * i) / numPoints);
    const theta = Math.sqrt(numPoints * Math.PI) * phi;

    const x = globeRadius * Math.sin(phi) * Math.cos(theta);
    const y = globeRadius * Math.cos(phi);
    const z = globeRadius * Math.sin(phi) * Math.sin(theta);

    // UV projection
    // theta is longitude in range (-inf, inf)
    // we want u in [0, 1]
    let longitude = Math.atan2(z, x); // [-PI, PI]
    if (longitude < 0) longitude += Math.PI * 2;
    const u = longitude / (Math.PI * 2);
    const v = phi / Math.PI;

    const px = Math.floor(u * w);
    const py = Math.floor(v * h);
    const index = (py * w + px) * 4;

    // In our fallback map, land is white (255) and ocean is black (0)
    const r = imgData[index];
    const isLand = r > 100;

    if (isLand) {
      positions.push(x, y, z);
      colors.push(0.0, 0.95, 1.0); // Bright cyan
    } else {
      if (Math.random() < 0.12) {
        positions.push(x, y, z);
        colors.push(0.0, 0.12, 0.35); // Dim blue
      }
    }
  }

  createGlobePoints(positions, colors);
  loading.value = false;
}

/**
 * 将位置数组和颜色数组创建为 Three.js 点云对象。
 * 点云材质使用发光圆形贴图，从而得到粒子地球效果。
 */
function createGlobePoints(positions: number[], colors: number[]) {
  const pointsGeo = new THREE.BufferGeometry();
  pointsGeo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  pointsGeo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));

  // Circle particle texture
  const particleCanvas = document.createElement('canvas');
  particleCanvas.width = 16;
  particleCanvas.height = 16;
  const pCtx = particleCanvas.getContext('2d')!;
  const grad = pCtx.createRadialGradient(8, 8, 0, 8, 8, 8);
  grad.addColorStop(0, 'rgba(255, 255, 255, 1)');
  grad.addColorStop(0.3, 'rgba(0, 240, 255, 0.8)');
  grad.addColorStop(1, 'rgba(0, 100, 255, 0)');
  pCtx.fillStyle = grad;
  pCtx.fillRect(0, 0, 16, 16);
  const particleTexture = new THREE.CanvasTexture(particleCanvas);

  const pointsMat = new THREE.PointsMaterial({
    size: 0.18,
    map: particleTexture,
    transparent: true,
    vertexColors: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });


  const globePoints = new THREE.Points(pointsGeo, pointsMat);
  globePoints.name = "globePoints";
  globeGroup.add(globePoints);
}

// ------------------------------
// 3. Orbits & Trajectories Creation
// ------------------------------
/**
 * 创建地球周围的椭圆轨迹和运动中的数据卫星。
 * 每条轨迹由一条线和多个带透明度衰减的球体组成，形成拖尾效果。
 */
function createOrbits() {
  // Define 3 elliptic orbits parameters
  const orbitParams = [
    { a: 4.0, b: 3.5, rx: 0.6, ry: 0.4, rz: 0.2, color: 0x00f0ff },
    { a: 4.6, b: 3.0, rx: -0.5, ry: -0.6, rz: 0.3, color: 0x0090ff },
    { a: 4.8, b: 3.8, rx: 0.3, ry: -0.2, rz: 1.0, color: 0x0055ff }
  ];

  // 为每组轨迹参数创建轨道线和对应的数据卫星。
  orbitParams.forEach((params) => {
    // 3.1 Draw orbit path line loop
    const points: THREE.Vector3[] = [];
    const segments = 120;
    
    const euler = new THREE.Euler(params.rx, params.ry, params.rz);

    for (let i = 0; i <= segments; i++) {
      const angle = (i / segments) * Math.PI * 2;
      const localPos = new THREE.Vector3(
        params.a * Math.cos(angle),
        params.b * Math.sin(angle),
        0
      );
      // Rotate path in 3D
      localPos.applyEuler(euler);
      points.push(localPos);
    }

    const orbitGeo = new THREE.BufferGeometry().setFromPoints(points);
    const orbitMat = new THREE.LineBasicMaterial({
      color: params.color,
      transparent: true,
      opacity: 0.25,
      blending: THREE.AdditiveBlending,
    });
    const orbitLine = new THREE.LineLoop(orbitGeo, orbitMat);
    orbitsGroup.add(orbitLine);

    // 3.2 Create trailing satellite (signal packet)
    // We render a group of 6 spheres that follow each other in decreasing size and opacity
    const satGroup = new THREE.Group();
    const trailCount = 6;
    const satMeshList: THREE.Mesh[] = [];

    const satTextureCanvas = document.createElement('canvas');
    satTextureCanvas.width = 16;
    satTextureCanvas.height = 16;
    const sCtx = satTextureCanvas.getContext('2d')!;
    const sGrad = sCtx.createRadialGradient(8, 8, 0, 8, 8, 8);
    sGrad.addColorStop(0, 'rgba(255, 255, 255, 1)');
    sGrad.addColorStop(0.4, 'rgba(0, 240, 255, 0.9)');
    sGrad.addColorStop(1, 'rgba(0, 50, 255, 0)');
    sCtx.fillStyle = sGrad;
    sCtx.fillRect(0, 0, 16, 16);
    for (let i = 0; i < trailCount; i++) {
      // Sphere geometry
      const size = 0.16 * (1 - i / (trailCount + 1));
      const geom = new THREE.SphereGeometry(size, 8, 8);
      const mat = new THREE.MeshBasicMaterial({
        color: params.color,
        transparent: true,
        opacity: 0.9 * (1 - i / trailCount),
        blending: THREE.AdditiveBlending,
      });
      const mesh = new THREE.Mesh(geom, mat);
      satGroup.add(mesh);
      satMeshList.push(mesh);
    }

    orbitsGroup.add(satGroup);

    satellites.push({
      mesh: satGroup,
      curve: params,
      speed: 0.003 + Math.random() * 0.002,
      progress: Math.random() * Math.PI * 2, // random starting progress
    });
  });
}

// ------------------------------
// Animation & Resizing
// ------------------------------
/**
 * Three.js 主动画循环。
 * 每一帧更新地球、底座、轨迹、卫星、光柱、粒子和 HUD 坐标，然后渲染画面。
 */
function animate() {
  animationFrameId = requestAnimationFrame(animate);

  // Rotate Earth
  if (globeGroup) {
    globeGroup.rotation.y += 0.002;
    // Slight tilt wobble
    globeGroup.rotation.x = Math.sin(Date.now() * 0.0002) * 0.05;
  }

  // Rotate Pedestal elements
  if (baseGroup) {
    // Base texture plate 1
    baseGroup.children[0].rotation.z += 0.001; // outer ticks
    
    // Rotating outer ring with triangles (element 1)
    const rotatingRing = baseGroup.getObjectByName("rotatingOuterRing");
    if (rotatingRing) {
      rotatingRing.rotation.z -= 0.004; // rotate counter-clockwise
    }

    // 根据每个环的独立速度，让底座产生错落的旋转层次。
    baseGroup.children.forEach((child) => {
      if (child.name.startsWith('baseRing')) {
        child.rotation.z += Number(child.userData.speed);
      }
    });
  }

  // Rotate orbits slowly
  if (orbitsGroup) {
    orbitsGroup.rotation.y += 0.0005;
  }

  // Update satellites progress along curves
  // 更新每颗卫星在椭圆轨迹上的进度。
  satellites.forEach((sat) => {
    sat.progress += sat.speed;
    if (sat.progress > Math.PI * 2) sat.progress -= Math.PI * 2;

    const euler = new THREE.Euler(sat.curve.rx, sat.curve.ry, sat.curve.rz);

    // Update each trailing segment in satellite group
    const trailOffset = 0.02; // spacing between trail points in radians
    // 同步更新卫星主体及其拖尾球体的位置。
    sat.mesh.children.forEach((mesh, index) => {
      const angle = sat.progress - index * trailOffset;
      const pos = new THREE.Vector3(
        sat.curve.a * Math.cos(angle),
        sat.curve.b * Math.sin(angle),
        0
      );
      pos.applyEuler(euler);
      mesh.position.copy(pos);
    });
  });

  // Animate light pillar pulses
  if (pillarMesh) {
    pillarMesh.rotation.y += 0.01;
    // Pulsing scale
    const pulse = 1.0 + Math.sin(Date.now() * 0.0015) * 0.05;
    pillarMesh.scale.set(pulse, 1.0, pulse);
  }

  // Animate sparks
  if (sparksGeometry) {
    const posAttr = sparksGeometry.getAttribute('position') as THREE.BufferAttribute;
    const positions = posAttr.array as Float32Array;

    for (let i = 0; i < sparkCount; i++) {
      const index = i * 3;
      
      // Update position based on velocity
      positions[index] += sparksVelocities[i * 3 + 1]; // x
      positions[index + 1] += sparksVelocities[i * 3]; // y (upwards)
      positions[index + 2] += sparksVelocities[i * 3 + 2]; // z

      // Reset sparks when they rise above cylinder height (3.2)
      if (positions[index + 1] > 3.2) {
        positions[index + 1] = 0; // back to base
        const radius = Math.random() * 0.8;
        const angle = Math.random() * Math.PI * 2;
        positions[index] = Math.cos(angle) * radius;
        positions[index + 2] = Math.sin(angle) * radius;
      }
    }
    posAttr.needsUpdate = true;
  }

  // Update OrbitControls
  if (controls) {
    controls.update();
  }

  // Update Projected Screen Coordinates for HUD Tags
  updateHudPositions();

  // Render
  if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}

/**
 * 把地球上的 3D 数据节点投影到 HTML 屏幕坐标。
 * 同时判断节点是否朝向摄像机以及是否位于可视区域内。
 */
function updateHudPositions() {
  if (!canvasContainer.value || !camera || !renderer) return;

  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;
  const tempV = new THREE.Vector3();

  // 逐个计算 HUD 标签的屏幕位置和正面可见状态。
  hudTags.forEach((tag) => {
    // Since globeGroup is rotating, the tags need to rotate with it!
    // Tag positions are static relative to globeGroup. We convert them to world space:
    tempV.copy(tag.pos);
    tempV.applyMatrix4(globeGroup.matrixWorld);

    // check if point is visible / not facing away from the camera
    // We calculate the dot product between the point's normal (from center) and the camera direction
    const centerToPt = new THREE.Vector3().copy(tempV).normalize();
    const camDir = new THREE.Vector3().copy(camera.position).normalize();
    const facingFactor = centerToPt.dot(camDir);

    // If facingFactor > 0.0, the point is on the front side of the sphere facing the camera
    tag.visible = facingFactor > 0.05;

    // Project to screen space
    tempV.project(camera);

    // Is it in the screen boundaries?
    tag.onScreen = tempV.z <= 1;

    // Map to pixel coords
    tag.screenX = (tempV.x * 0.5 + 0.5) * width;
    tag.screenY = (-(tempV.y * 0.5) + 0.5) * height;
  });
}

/**
 * 响应浏览器尺寸变化。
 * 同步更新相机宽高比和 WebGL 画布尺寸，避免画面变形。
 */
function onWindowResize() {
  if (!canvasContainer.value || !camera || !renderer) return;
  const width = canvasContainer.value.clientWidth;
  const height = canvasContainer.value.clientHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  renderer.setSize(width, height);
}
</script>

<style scoped>
.earth-3d-container {
  position: relative;
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at center, #020b24 0%, #010512 100%);
  overflow: hidden;
}

.canvas-container {
  width: 100%;
  height: 100%;
}

/* 3D HUD Tags styling */
.hud-card {
  position: absolute;
  pointer-events: none;
  transform: translate(-50%, -100%);
  transition: opacity 0.5s ease;
  z-index: 10;
}

.hud-line {
  width: 1px;
  height: 45px;
  background: linear-gradient(to top, rgba(0, 240, 255, 1) 0%, rgba(0, 240, 255, 0) 100%);
  position: absolute;
  left: 50%;
  bottom: 0;
}

.hud-content {
  background: rgba(4, 15, 37, 0.85);
  border: 1px solid #00f0ff;
  box-shadow: 0 0 15px rgba(0, 240, 255, 0.4), inset 0 0 10px rgba(0, 240, 255, 0.2);
  border-radius: 4px;
  padding: 8px 12px;
  min-width: 150px;
  position: absolute;
  left: 50%;
  bottom: 45px;
  transform: translateX(-50%);
  color: #fff;
  font-family: 'Segoe UI', Arial, sans-serif;
  backdrop-filter: blur(4px);
}

.hud-content::before {
  content: '';
  position: absolute;
  width: 6px;
  height: 6px;
  background: #00f0ff;
  border-radius: 50%;
  bottom: -48px;
  left: calc(50% - 3px);
  box-shadow: 0 0 8px #00f0ff;
}

.hud-title {
  font-size: 11px;
  color: #8da4d1;
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-bottom: 2px;
}

.hud-value {
  font-size: 15px;
  font-weight: bold;
  color: #00f0ff;
  text-shadow: 0 0 5px rgba(0, 240, 255, 0.5);
  margin-bottom: 1px;
}

.hud-sub {
  font-size: 10px;
  color: rgba(255, 255, 255, 0.6);
}

/* Spinner & Loading */
.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #020515;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 20;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 3px solid rgba(0, 240, 255, 0.1);
  border-radius: 50%;
  border-top-color: #00f0ff;
  animation: spin 1s ease-in-out infinite;
  margin-bottom: 15px;
}

.loading-text {
  color: #00f0ff;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  font-size: 14px;
  letter-spacing: 1px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
</style>
