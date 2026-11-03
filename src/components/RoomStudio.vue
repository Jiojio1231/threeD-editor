<template>
  <div class="studio-shell">
    <header class="topbar">
      <a class="brand" href="#" aria-label="空间设计工作台">
        <span class="brand-mark"><span></span><span></span><span></span><span></span></span>
        <span class="brand-name">NEXUS<span> / SPACE STUDIO</span></span>
      </a>
      <div class="project-name">
        <span class="live-dot"></span>
        <span>数据中心 · 1F 机房</span>
        <span class="saved-state">本地编辑</span>
      </div>
      <div class="top-actions">
        <button class="top-button subtle" type="button" @click="resetCamera">
          <span class="button-icon">⌖</span> 重置视角
        </button>
        <button class="top-button primary" type="button" @click="addRack">
          <span class="plus">＋</span> 添加机柜
        </button>
        <button class="top-button partition-top-button" type="button" @click="addPartition">
          <span class="plus">＋</span> 添加隔板
        </button>
      </div>
    </header>

    <div class="workspace">
      <aside class="sidebar left-sidebar">
        <div class="sidebar-heading">
          <div>
            <div class="eyebrow">PROJECT SETUP</div>
            <h1>机房编辑器</h1>
          </div>
          <span class="heading-icon">▧</span>
        </div>

        <section class="side-section">
          <div class="section-title">
            <span>空间尺寸</span>
            <span class="section-note">单位：米</span>
          </div>
          <div class="dimension-grid">
            <label class="dimension-field">
              <span>长度 <small>L</small></span>
              <div><input v-model.number="roomDraft.length" type="number" min="4" max="40" step="0.5"><b>m</b></div>
            </label>
            <label class="dimension-field">
              <span>宽度 <small>W</small></span>
              <div><input v-model.number="roomDraft.width" type="number" min="4" max="40" step="0.5"><b>m</b></div>
            </label>
            <label class="dimension-field">
              <span>高度 <small>H</small></span>
              <div><input v-model.number="roomDraft.height" type="number" min="2.5" max="12" step="0.5"><b>m</b></div>
            </label>
          </div>
          <button class="apply-button" type="button" @click="applyRoomDimensions">应用空间尺寸 <span>↗</span></button>
          <div class="dimension-hint"><span class="tiny-grid">▦</span> 修改尺寸后，机柜贴地，隔板保持在房间范围内</div>
        </section>

        <section class="side-section inventory-section">
          <div class="section-title">
            <span>场景对象</span>
            <span class="count-pill">{{ (racks.length + partitions.length).toString().padStart(2, '0') }}</span>
          </div>
          <div class="object-list">
            <button
              v-for="rack in racks"
              :key="rack.id"
              class="object-row"
              :class="{ selected: selectedId === rack.id }"
              type="button"
              @click="selectItem(rack.id)"
            >
              <span class="object-thumb"><span class="rack-glyph"></span></span>
              <span class="object-copy"><strong>{{ rack.name }}</strong><small>{{ rack.kind === 'imported' ? '导入模型' : '标准机柜' }} · {{ rack.height.toFixed(1) }} m</small></span>
              <span class="object-chevron">›</span>
            </button>
            <button
              v-for="partition in partitions"
              :key="partition.id"
              class="object-row"
              :class="{ selected: selectedId === partition.id }"
              type="button"
              @click="selectItem(partition.id)"
            >
              <span class="object-thumb partition-thumb"><span class="partition-glyph"></span></span>
              <span class="object-copy"><strong>{{ partition.name }}</strong><small>隔板 · {{ partition.length.toFixed(1) }} m{{ partition.hasDoor ? ' · 含门' : '' }}</small></span>
              <span class="object-chevron">›</span>
            </button>
            <div v-if="!racks.length && !partitions.length" class="empty-objects">还没有机柜或隔板，添加一个开始布局。</div>
          </div>
          <button class="import-button" type="button" @click="triggerImport">
            <span class="upload-icon">↑</span><span>导入 3D 模型</span><small>GLB</small>
          </button>
          <button class="import-button partition-add-button" type="button" @click="addPartition">
            <span class="partition-add-icon">╱</span><span>添加空间隔板</span><small>墙体</small>
          </button>
          <div class="asset-source">
            <span>{{ assetStatus }}</span>
            <div class="asset-links">
              <a href="https://grab3d.com/models/electronics/server-rack/" target="_blank" rel="noreferrer">机柜模型 · CC0 ↗</a>
              <a href="https://grab3d.com/models/buildings/roller-shutter-door/" target="_blank" rel="noreferrer">工业门资源 · CC0 ↗</a>
            </div>
          </div>
          <input ref="fileInput" class="visually-hidden" type="file" accept=".glb,model/gltf-binary" @change="importModel">
          <p v-if="importError" class="error-note">{{ importError }}</p>
          <p v-if="assetError" class="error-note">{{ assetError }}</p>
        </section>

        <section class="side-section room-summary">
          <div class="section-title"><span>空间概览</span><span class="status-label"><i></i> 已就绪</span></div>
          <div class="summary-metrics">
            <div><strong>{{ room.width }}<small>m</small></strong><span>宽度</span></div>
            <div><strong>{{ room.length }}<small>m</small></strong><span>长度</span></div>
            <div><strong>{{ room.width * room.length }}<small>m²</small></strong><span>占地面积</span></div>
          </div>
        </section>

        <div class="sidebar-footer">
          <span class="footer-avatar">N</span>
          <span><strong>设计工作区</strong><small>所有变更保存在当前会话</small></span>
          <span class="footer-more">···</span>
        </div>
      </aside>

      <main class="viewport-column">
        <div class="viewport-toolbar">
          <div class="breadcrumbs"><span>项目</span><i>/</i><strong>机房空间</strong><span class="view-chip">3D 视图</span></div>
          <div class="view-tools">
            <span class="view-tool-label"><i></i> 正交辅助网格</span>
            <button type="button" class="icon-button" title="重置视角" aria-label="重置视角" @click="resetCamera">⟳</button>
          </div>
        </div>
        <div ref="viewport" class="viewport">
          <div class="viewport-wash"></div>
          <div class="viewport-caption">
            <span class="caption-kicker">SPATIAL VIEW / 01</span>
            <strong>机房空间预览</strong>
            <span>拖动机柜或隔板调整位置 · 滚轮缩放 · 右键旋转</span>
          </div>
          <div class="axis-indicator"><span class="axis-y">Y</span><span class="axis-x">X</span><span class="axis-z">Z</span><i></i></div>
          <div class="view-scale"><span>0</span><i></i><span>5 m</span></div>
          <div class="viewport-status"><span class="live-dot"></span> 实时渲染 <i></i> {{ racks.length + partitions.length }} 个对象</div>
          <div v-if="selectedRack || selectedPartition" class="selection-hud">
            <span class="selection-dot"></span>
            <span><small>当前选择</small><strong>{{ selectedRack?.name ?? selectedPartition?.name }}</strong></span>
            <span class="hud-coordinates">X {{ (selectedRack?.x ?? selectedPartition?.x ?? 0).toFixed(1) }} <i>/</i> Z {{ (selectedRack?.z ?? selectedPartition?.z ?? 0).toFixed(1) }}</span>
          </div>
        </div>
        <div class="viewport-footer">
          <div class="legend"><span><i class="legend-room"></i>机房边界</span><span><i class="legend-rack"></i>机柜</span><span><i class="legend-door"></i>出入口</span></div>
          <span class="camera-label">PERSPECTIVE <i>·</i> 01</span>
        </div>
      </main>

      <aside class="sidebar right-sidebar">
        <div class="right-heading">
          <div><div class="eyebrow">OBJECT DETAILS</div><h2>{{ selectedPartition ? '隔板属性' : selectedRack ? '机柜属性' : '场景工具' }}</h2></div>
          <span class="more-dots">···</span>
        </div>

        <template v-if="selectedPartition">
          <div class="selected-card">
            <div class="selected-preview partition-preview"><span class="partition-glyph large"></span><span class="preview-glow"></span></div>
            <div class="selected-name"><span class="selected-type">ROOM PARTITION</span><strong>{{ selectedPartition.name }}</strong><small>对象 ID · {{ selectedPartition.id.slice(-6).toUpperCase() }}</small></div>
          </div>
          <section class="property-section">
            <div class="section-title"><span>位置坐标</span><span class="edit-hint">拖动隔板可快速调整</span></div>
            <div class="coordinate-grid">
              <label><span>X</span><input :value="selectedPartition.x.toFixed(1)" type="number" step="0.1" @change="updatePartitionPosition('x', $event)"></label>
              <label><span>Y</span><input value="0.0" type="number" disabled></label>
              <label><span>Z</span><input :value="selectedPartition.z.toFixed(1)" type="number" step="0.1" @change="updatePartitionPosition('z', $event)"></label>
            </div>
          </section>
          <section class="property-section">
            <div class="section-title"><span>隔板尺寸</span><span class="unit-tag">m</span></div>
            <label class="partition-length-field"><span>长度</span><input :value="selectedPartition.length.toFixed(1)" type="number" min="2" :max="partitionMaxLength(selectedPartition)" step="0.1" @change="updatePartitionLength($event)"><b>m</b></label>
            <div class="size-row"><span>厚度</span><strong>{{ selectedPartition.thickness.toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>高度</span><strong>{{ room.height.toFixed(2) }} <small>m</small></strong></div>
            <button class="rotate-button" type="button" @click="rotateSelectedPartition"><span>⟳</span> 旋转 90° <small>当前 {{ selectedPartition.rotation === 0 ? '横向' : '纵向' }}</small></button>
          </section>
          <section class="partition-door-panel">
            <div class="section-title"><span>隔板门</span><span class="door-status" :class="{ installed: selectedPartition.hasDoor }">{{ selectedPartition.hasDoor ? '已安装' : '未安装' }}</span></div>
            <p>{{ selectedPartition.hasDoor ? '门洞位于隔板中央，可切换门扇开合状态。' : '在隔板中央开设门洞并安装门扇。' }}</p>
            <button v-if="!selectedPartition.hasDoor" class="annotation-add" type="button" @click="addPartitionDoor"><span>＋</span> 在隔板上添加门</button>
            <div v-else class="door-actions">
              <button class="annotation-add" type="button" @click="togglePartitionDoor"><span>{{ selectedPartition.doorOpen ? '↶' : '↗' }}</span> {{ selectedPartition.doorOpen ? '关闭门扇' : '打开门扇' }}</button>
              <button class="remove-door-button" type="button" @click="removePartitionDoor">移除门</button>
            </div>
          </section>
          <button class="danger-button" type="button" @click="removeSelectedPartition"><span>⌫</span> 删除此隔板</button>
        </template>
        <template v-else-if="selectedRack">
          <div class="selected-card">
            <div class="selected-preview"><span class="rack-glyph large"></span><span class="preview-glow"></span></div>
            <div class="selected-name"><span class="selected-type">{{ selectedRack.kind === 'imported' ? 'IMPORT MODEL' : 'SERVER RACK' }}</span><strong>{{ selectedRack.name }}</strong><small>对象 ID · {{ selectedRack.id.slice(-6).toUpperCase() }}</small></div>
          </div>
          <section class="property-section">
            <div class="section-title"><span>位置坐标</span><span class="edit-hint">拖动模型可快速调整</span></div>
            <div class="coordinate-grid">
              <label><span>X</span><input :value="selectedRack.x.toFixed(1)" type="number" step="0.1" @change="updatePosition('x', $event)"></label>
              <label><span>Y</span><input value="0.0" type="number" disabled></label>
              <label><span>Z</span><input :value="selectedRack.z.toFixed(1)" type="number" step="0.1" @change="updatePosition('z', $event)"></label>
            </div>
          </section>
          <section class="property-section dimension-properties">
            <div class="section-title"><span>模型尺寸</span><span class="unit-tag">m</span></div>
            <div class="size-row"><span>宽度</span><strong>{{ selectedRack.width.toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>深度</span><strong>{{ selectedRack.depth.toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>高度</span><strong>{{ selectedRack.height.toFixed(2) }} <small>m</small></strong></div>
          </section>
          <button class="danger-button" type="button" @click="removeSelected"><span>⌫</span> 删除此机柜</button>
        </template>
        <template v-else>
          <div class="tool-intro">
            <div class="tool-illustration"><span class="tool-cube"></span><i></i><b></b></div>
            <strong>构建你的空间</strong>
            <p>添加机柜或导入模型，再点击场景中的对象查看属性。</p>
          </div>
          <div class="quick-actions">
            <button type="button" @click="addRack"><span class="quick-icon">▤</span><span><strong>添加标准机柜</strong><small>快速放置到房间地板</small></span><b>＋</b></button>
            <button type="button" @click="triggerImport"><span class="quick-icon upload">↑</span><span><strong>导入自定义模型</strong><small>支持 GLB 格式文件</small></span><b>↗</b></button>
          </div>
        </template>

        <section class="annotation-section">
          <div class="annotation-heading">
            <div class="section-title"><span>空间标注</span><span class="count-pill">{{ annotations.length.toString().padStart(2, '0') }}</span></div>
            <span class="annotation-helper">2D 卡片</span>
          </div>
          <div class="annotation-form">
            <label class="form-label">标注标题</label>
            <input v-model="annotationTitle" class="text-field" type="text" maxlength="28" placeholder="例如：核心交换机">
            <label class="form-label">描述信息</label>
            <textarea v-model="annotationDescription" class="text-field description-field" maxlength="120" placeholder="填写设备用途、维护信息或备注…"></textarea>
            <button class="annotation-add" type="button" :disabled="!selectedRack || !annotationTitle.trim()" @click="addAnnotation"><span>＋</span> 添加 2D 标注</button>
            <p v-if="!selectedRack" class="annotation-tip">先选择一个机柜，标注将固定在该对象上。</p>
          </div>
          <div v-if="annotations.length" class="annotation-list">
            <div v-for="annotation in annotations" :key="annotation.id" class="annotation-row" :class="{ active: selectedId === annotation.itemId }" @click="selectItem(annotation.itemId)">
              <span class="annotation-pin">⌖</span>
              <span class="annotation-copy"><strong>{{ annotation.title }}</strong><small>{{ annotation.description || '暂无描述' }}</small></span>
              <button type="button" aria-label="删除标注" @click.stop="removeAnnotation(annotation.id)">×</button>
            </div>
          </div>
          <div v-else class="annotation-empty"><span>↗</span><p>添加标注后，信息卡片会显示在场景对象上方。</p></div>
        </section>
        <div class="right-footer"><span>✦</span> Spatial tools <i>v1.0</i></div>
      </aside>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, markRaw, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { CSS2DObject, CSS2DRenderer } from 'three/addons/renderers/CSS2DRenderer.js';
import { GLTFLoader, type GLTF } from 'three/addons/loaders/GLTFLoader.js';

type RackItem = {
  id: string;
  name: string;
  kind: 'rack' | 'imported';
  group: THREE.Group;
  width: number;
  depth: number;
  height: number;
  x: number;
  z: number;
};

type PartitionItem = {
  id: string;
  name: string;
  group: THREE.Group;
  length: number;
  thickness: number;
  rotation: 0 | 1;
  x: number;
  z: number;
  fullSpan: boolean;
  hasDoor: boolean;
  doorOpen: boolean;
};

type Annotation = {
  id: string;
  itemId: string;
  title: string;
  description: string;
  label: CSS2DObject;
};

const viewport = ref<HTMLDivElement | null>(null);
const fileInput = ref<HTMLInputElement | null>(null);
const room = reactive({ width: 12, length: 9, height: 4 });
const roomDraft = reactive({ width: 12, length: 9, height: 4 });
const racks = ref<RackItem[]>([]);
const partitions = ref<PartitionItem[]>([]);
const annotations = ref<Annotation[]>([]);
const selectedId = ref<string | null>(null);
const annotationTitle = ref('');
const annotationDescription = ref('');
const importError = ref('');
const assetStatus = ref('CC0 模型加载中');
const assetError = ref('');

const selectedRack = computed(() => racks.value.find((rack) => rack.id === selectedId.value) ?? null);
const selectedPartition = computed(() => partitions.value.find((partition) => partition.id === selectedId.value) ?? null);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let labelRenderer: CSS2DRenderer;
let controls: OrbitControls;
let roomGroup: THREE.Group;
let roomOutline: THREE.Group;
let floorPlane: THREE.Mesh;
let freeRackPrototype: THREE.Group | null = null;
let resizeObserver: ResizeObserver;
let animationFrame = 0;
let activeDrag: { kind: 'rack' | 'partition'; id: string } | null = null;
let floorPlaneForDrag: THREE.Plane;
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const gltfLoader = new GLTFLoader();

function makeMaterial(color: number, roughness = 0.7, metalness = 0) {
  return new THREE.MeshStandardMaterial({ color, roughness, metalness });
}

function makeBox(
  parent: THREE.Object3D,
  width: number,
  height: number,
  depth: number,
  material: THREE.Material,
  x: number,
  y: number,
  z: number,
) {
  const mesh = new THREE.Mesh(new THREE.BoxGeometry(width, height, depth), material);
  mesh.position.set(x, y, z);
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  parent.add(mesh);
  return mesh;
}

function buildRoom() {
  if (!scene) return;
  if (roomGroup) {
    scene.remove(roomGroup);
    disposeGroup(roomGroup);
  }
  if (roomOutline) {
    scene.remove(roomOutline);
    disposeGroup(roomOutline);
  }

  for (const partition of partitions.value) clampPartitionToRoom(partition);

  roomGroup = new THREE.Group();
  roomOutline = new THREE.Group();
  const halfW = room.width / 2;
  const halfL = room.length / 2;
  const wallColor = makeMaterial(0x26384b, 0.52, 0.3);
  const trimColor = makeMaterial(0x49647a, 0.4, 0.55);
  const floorColor = makeMaterial(0x26313b, 0.84, 0.08);

  floorPlane = new THREE.Mesh(new THREE.PlaneGeometry(room.width, room.length), floorColor);
  floorPlane.rotation.x = -Math.PI / 2;
  floorPlane.position.y = 0.01;
  floorPlane.receiveShadow = true;
  roomGroup.add(floorPlane);

  const gridMaterial = new THREE.LineBasicMaterial({ color: 0x71879a, transparent: true, opacity: 0.18 });
  const gridPoints: THREE.Vector3[] = [];
  for (let x = -halfW; x <= halfW + 0.001; x += 0.5) {
    gridPoints.push(new THREE.Vector3(x, 0.025, -halfL), new THREE.Vector3(x, 0.025, halfL));
  }
  for (let z = -halfL; z <= halfL + 0.001; z += 0.5) {
    gridPoints.push(new THREE.Vector3(-halfW, 0.025, z), new THREE.Vector3(halfW, 0.025, z));
  }
  const grid = new THREE.LineSegments(new THREE.BufferGeometry().setFromPoints(gridPoints), gridMaterial);
  roomGroup.add(grid);

  const wallHeight = room.height;
  const wallThickness = 0.16;
  const sideWallMaterial = new THREE.MeshStandardMaterial({
    color: 0x34506a,
    roughness: 0.55,
    metalness: 0.3,
    transparent: true,
    opacity: 0.25,
    side: THREE.DoubleSide,
  });
  makeBox(roomGroup, wallThickness, wallHeight, room.length, sideWallMaterial, -halfW, wallHeight / 2, 0);
  makeBox(roomGroup, wallThickness, wallHeight, room.length, sideWallMaterial, halfW, wallHeight / 2, 0);
  makeBox(roomGroup, room.width, wallHeight, wallThickness, sideWallMaterial, 0, wallHeight / 2, -halfL);

  const doorWidth = Math.min(2.2, room.width * 0.34);
  const sideSpan = Math.max(0, (room.width - doorWidth) / 2);
  if (sideSpan > 0.15) {
    makeBox(roomGroup, sideSpan, wallHeight, wallThickness, sideWallMaterial, -(doorWidth + sideSpan) / 2, wallHeight / 2, halfL);
    makeBox(roomGroup, sideSpan, wallHeight, wallThickness, sideWallMaterial, (doorWidth + sideSpan) / 2, wallHeight / 2, halfL);
  }

  const doorHeight = Math.min(2.45, wallHeight - 0.25);
  const doorMaterial = new THREE.MeshStandardMaterial({ color: 0x42566a, roughness: 0.42, metalness: 0.45 });
  makeBox(roomGroup, 0.055, doorHeight, 0.12, trimColor, -doorWidth / 2, doorHeight / 2, halfL);
  makeBox(roomGroup, 0.055, doorHeight, 0.12, trimColor, doorWidth / 2, doorHeight / 2, halfL);
  makeBox(roomGroup, doorWidth + 0.11, 0.08, 0.12, trimColor, 0, doorHeight, halfL);
  const leafWidth = doorWidth / 2 - 0.035;
  for (const sign of [-1, 1]) {
    const leaf = makeBox(roomGroup, leafWidth, doorHeight - 0.08, 0.055, doorMaterial, sign * (doorWidth / 4), (doorHeight - 0.08) / 2, halfL - 0.04);
    const inset = makeMaterial(0x344858, 0.5, 0.45);
    makeBox(leaf, leafWidth * 0.62, doorHeight * 0.62, 0.015, inset, 0, 0, 0.037);
    makeBox(roomGroup, 0.025, 0.14, 0.025, trimColor, sign * (doorWidth / 4) + sign * (leafWidth * 0.34), 1.03, halfL - 0.005);
  }

  const edgeMaterial = new THREE.LineBasicMaterial({ color: 0x6bb2c3, transparent: true, opacity: 0.75 });
  const perimeter = [
    new THREE.Vector3(-halfW, 0.035, -halfL), new THREE.Vector3(halfW, 0.035, -halfL),
    new THREE.Vector3(halfW, 0.035, halfL), new THREE.Vector3(-halfW, 0.035, halfL),
    new THREE.Vector3(-halfW, 0.035, -halfL),
  ];
  roomOutline.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(perimeter), edgeMaterial));
  const topEdge = [
    new THREE.Vector3(-halfW, wallHeight, -halfL), new THREE.Vector3(halfW, wallHeight, -halfL),
    new THREE.Vector3(halfW, wallHeight, halfL), new THREE.Vector3(-halfW, wallHeight, halfL),
    new THREE.Vector3(-halfW, wallHeight, -halfL),
  ];
  roomOutline.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(topEdge), edgeMaterial));
  for (const x of [-halfW, halfW]) {
    for (const z of [-halfL, halfL]) {
      roomOutline.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints([
        new THREE.Vector3(x, 0, z),
        new THREE.Vector3(x, wallHeight, z),
      ]), edgeMaterial));
    }
  }

  const ceilingBars = makeMaterial(0x557388, 0.5, 0.6);
  for (const x of [-halfW, halfW]) {
    makeBox(roomGroup, 0.08, 0.08, room.length, ceilingBars, x, wallHeight, 0);
  }
  for (const z of [-halfL, halfL]) {
    makeBox(roomGroup, room.width, 0.08, 0.08, ceilingBars, 0, wallHeight, z);
  }
  scene.add(roomGroup, roomOutline);
}

function buildRack(name: string, id: string) {
  const group = new THREE.Group();
  group.name = name;
  if (freeRackPrototype) {
    const model = freeRackPrototype.clone(true);
    model.updateMatrixWorld(true);
    const bounds = new THREE.Box3().setFromObject(model);
    const size = bounds.getSize(new THREE.Vector3());
    const height = 2.05;
    const scale = height / size.y;
    model.scale.multiplyScalar(scale);
    model.position.set(
      -((bounds.min.x + bounds.max.x) / 2) * scale,
      -bounds.min.y * scale,
      -((bounds.min.z + bounds.max.z) / 2) * scale,
    );
    group.add(model);
    group.userData.sharedAssets = true;
    group.traverse((child) => {
      if (child instanceof THREE.Mesh) {
        child.userData.itemId = id;
        child.castShadow = true;
        child.receiveShadow = true;
      }
    });
    return { group, width: size.x * scale, depth: size.z * scale, height };
  }

  const width = 0.72;
  const depth = 0.92;
  const height = 2.05;
  const frame = makeMaterial(0x283641, 0.42, 0.72);
  const panel = makeMaterial(0x18232c, 0.48, 0.52);
  const dark = makeMaterial(0x101a22, 0.64, 0.34);
  const cyan = new THREE.MeshStandardMaterial({ color: 0x50c5d5, emissive: 0x127080, emissiveIntensity: 0.85, roughness: 0.32 });

  makeBox(group, width, height - 0.08, depth, panel, 0, (height - 0.08) / 2 + 0.04, 0);
  for (const x of [-width / 2 + 0.045, width / 2 - 0.045]) {
    for (const z of [-depth / 2 + 0.04, depth / 2 - 0.04]) {
      makeBox(group, 0.055, height, 0.055, frame, x, height / 2, z);
    }
  }
  for (const y of [0.06, height - 0.055]) {
    makeBox(group, width, 0.07, depth, frame, 0, y, 0);
  }

  makeBox(group, width - 0.13, height - 0.22, 0.045, dark, 0, height / 2, depth / 2 + 0.018);
  const units = 42;
  for (let unit = 0; unit < units; unit += 1) {
    const y = 0.15 + unit * 0.041;
    makeBox(group, width - 0.2, 0.012, 0.018, unit % 7 === 0 ? frame : dark, 0, y, depth / 2 + 0.047);
  }
  for (const x of [-width / 2 + 0.11, width / 2 - 0.11]) {
    makeBox(group, 0.017, height - 0.28, 0.012, frame, x, height / 2, depth / 2 + 0.055);
  }
  makeBox(group, 0.018, 0.07, 0.018, cyan, -width / 2 + 0.13, height - 0.16, depth / 2 + 0.065);
  makeBox(group, 0.018, 0.07, 0.018, makeMaterial(0x65d89c, 0.4, 0.2), width / 2 - 0.13, height - 0.16, depth / 2 + 0.065);
  makeBox(group, width - 0.12, 0.035, depth - 0.14, dark, 0, 0.04, 0);

  group.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.userData.itemId = id;
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  return { group, width, depth, height };
}

function addRack() {
  const id = createId();
  const index = racks.value.length + 1;
  const name = `机柜 ${String(index).padStart(2, '0')}`;
  const built = buildRack(name, id);
  const position = findFreePosition(built.width, built.depth);
  built.group.position.set(position.x, 0.035, position.z);
  scene.add(built.group);
  racks.value.push({
    id,
    name,
    kind: 'rack',
    ...built,
    group: markRaw(built.group),
    x: position.x,
    z: position.z,
  });
  selectedId.value = id;
}

function addPartition() {
  if (!scene) return;
  const id = createId();
  let index = partitions.value.length + 1;
  while (partitions.value.some((partition) => partition.name === `隔板 ${String(index).padStart(2, '0')}`)) index += 1;
  const length = room.width;
  const position = findPartitionPosition(length, 0.14, 0);
  const partition: PartitionItem = reactive({
    id,
    name: `隔板 ${String(index).padStart(2, '0')}`,
    group: markRaw(new THREE.Group()),
    length,
    thickness: 0.14,
    rotation: 0,
    x: position.x,
    z: position.z,
    fullSpan: true,
    hasDoor: false,
    doorOpen: false,
  });
  partition.group.name = partition.name;
  partition.group.userData.itemId = id;
  scene.add(partition.group);
  partitions.value.push(partition);
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
  selectedId.value = id;
}

function findPartitionPosition(length: number, thickness: number, rotation: 0 | 1) {
  const alongX = rotation === 0;
  const offsetLimit = alongX ? room.length / 2 - thickness / 2 - 0.08 : room.width / 2 - thickness / 2 - 0.08;
  const spacing = Math.max(1.5, thickness + 0.3);
  const offsets = [0];
  for (let distance = spacing; distance <= offsetLimit; distance += spacing) {
    offsets.push(distance, -distance);
  }
  const extentX = alongX ? length : thickness;
  const extentZ = alongX ? thickness : length;
  for (const offset of offsets) {
    const x = alongX ? 0 : offset;
    const z = alongX ? offset : 0;
    const overlaps = partitions.value.some((partition) => {
      const otherExtentX = partition.rotation === 0 ? partition.length : partition.thickness;
      const otherExtentZ = partition.rotation === 0 ? partition.thickness : partition.length;
      return Math.abs(partition.x - x) < (otherExtentX + extentX) / 2 + 0.2
        && Math.abs(partition.z - z) < (otherExtentZ + extentZ) / 2 + 0.2;
    });
    if (!overlaps) return { x, z };
  }
  return { x: 0, z: 0 };
}

function rebuildPartition(partition: PartitionItem) {
  for (const child of [...partition.group.children]) {
    partition.group.remove(child);
    disposeGroup(child);
  }
  partition.group.position.set(partition.x, 0, partition.z);
  partition.group.rotation.y = partition.rotation * Math.PI / 2;
  const length = partition.length;
  const halfLength = length / 2;
  const halfThickness = partition.thickness / 2;
  const wallHeight = room.height;
  const wallMaterial = new THREE.MeshStandardMaterial({
    color: 0x78919c,
    roughness: 0.72,
    metalness: 0.12,
    transparent: true,
    opacity: 0.74,
    side: THREE.DoubleSide,
  });
  const trimMaterial = makeMaterial(0x9caeae, 0.45, 0.36);
  const doorMaterial = makeMaterial(0x425967, 0.44, 0.45);
  const markerMaterial = new THREE.MeshStandardMaterial({
    color: 0x71ccc4,
    emissive: 0x245d5a,
    emissiveIntensity: 0.35,
    roughness: 0.45,
  });
  const openingWidth = Math.min(0.95, length - 0.6);
  const doorHeight = Math.min(2.2, wallHeight - 0.12);

  if (!partition.hasDoor) {
    makeBox(partition.group, length, wallHeight, partition.thickness, wallMaterial, 0, wallHeight / 2, 0);
  } else {
    const segmentLength = (length - openingWidth) / 2;
    makeBox(partition.group, segmentLength, wallHeight, partition.thickness, wallMaterial, -(openingWidth + segmentLength) / 2, wallHeight / 2, 0);
    makeBox(partition.group, segmentLength, wallHeight, partition.thickness, wallMaterial, (openingWidth + segmentLength) / 2, wallHeight / 2, 0);
    makeBox(partition.group, openingWidth + partition.thickness, 0.09, partition.thickness + 0.025, trimMaterial, 0, doorHeight, 0);
    for (const side of [-1, 1]) {
      makeBox(partition.group, 0.06, doorHeight, partition.thickness + 0.035, trimMaterial, side * openingWidth / 2, doorHeight / 2, 0);
    }

    const hinge = new THREE.Group();
    hinge.position.set(-openingWidth / 2 + 0.035, 0, halfThickness + 0.025);
    hinge.name = 'Partition door hinge';
    const leafWidth = openingWidth - 0.07;
    makeBox(hinge, leafWidth, doorHeight - 0.035, 0.055, doorMaterial, leafWidth / 2, (doorHeight - 0.035) / 2, 0);
    makeBox(hinge, 0.025, 0.12, 0.025, trimMaterial, leafWidth - 0.1, 1.02, 0.04);
    makeBox(hinge, 0.018, doorHeight - 0.18, 0.018, markerMaterial, 0.08, (doorHeight - 0.18) / 2, 0.035);
    hinge.rotation.y = partition.doorOpen ? -Math.PI / 2 : 0;
    partition.group.add(hinge);
  }

  const baseMaterial = makeMaterial(0x617c83, 0.56, 0.46);
  makeBox(partition.group, length, 0.07, partition.thickness + 0.08, baseMaterial, 0, 0.035, 0);
  const capMaterial = makeMaterial(0x9bb7b7, 0.42, 0.36);
  makeBox(partition.group, length, 0.045, partition.thickness + 0.025, capMaterial, 0, wallHeight - 0.0225, 0);

  partition.group.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.userData.itemId = partition.id;
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
}

function partitionMaxLength(partition: PartitionItem) {
  return partition.rotation === 0 ? room.width : room.length;
}

function clampPartitionToRoom(partition: PartitionItem) {
  partition.length = Math.min(partition.length, partitionMaxLength(partition));
  const alongX = partition.rotation === 0;
  const limitX = alongX
    ? Math.max(0, room.width / 2 - partition.length / 2)
    : Math.max(0, room.width / 2 - partition.thickness / 2 - 0.08);
  const limitZ = alongX
    ? Math.max(0, room.length / 2 - partition.thickness / 2 - 0.08)
    : Math.max(0, room.length / 2 - partition.length / 2);
  partition.x = THREE.MathUtils.clamp(partition.x, -limitX, limitX);
  partition.z = THREE.MathUtils.clamp(partition.z, -limitZ, limitZ);
  partition.group.position.set(partition.x, 0, partition.z);
}

function updatePartitionPosition(axis: 'x' | 'z', event: Event) {
  const partition = selectedPartition.value;
  const input = event.target as HTMLInputElement;
  if (!partition || !Number.isFinite(input.valueAsNumber)) return;
  partition[axis] = input.valueAsNumber;
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
}

function updatePartitionLength(event: Event) {
  const partition = selectedPartition.value;
  const input = event.target as HTMLInputElement;
  if (!partition || !Number.isFinite(input.valueAsNumber)) return;
  partition.length = clampNumber(input.valueAsNumber, 2, partitionMaxLength(partition), partition.length);
  partition.fullSpan = false;
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
}

function rotateSelectedPartition() {
  const partition = selectedPartition.value;
  if (!partition) return;
  partition.rotation = partition.rotation === 0 ? 1 : 0;
  partition.length = partition.fullSpan
    ? partitionMaxLength(partition)
    : Math.min(partition.length, partitionMaxLength(partition));
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
}

function addPartitionDoor() {
  const partition = selectedPartition.value;
  if (!partition) return;
  partition.hasDoor = true;
  partition.doorOpen = false;
  rebuildPartition(partition);
}

function togglePartitionDoor() {
  const partition = selectedPartition.value;
  if (!partition || !partition.hasDoor) return;
  partition.doorOpen = !partition.doorOpen;
  rebuildPartition(partition);
}

function removePartitionDoor() {
  const partition = selectedPartition.value;
  if (!partition) return;
  partition.hasDoor = false;
  partition.doorOpen = false;
  rebuildPartition(partition);
}

function removeSelectedPartition() {
  const partition = selectedPartition.value;
  if (!partition || !scene) return;
  scene.remove(partition.group);
  disposeGroup(partition.group);
  partitions.value = partitions.value.filter((item) => item.id !== partition.id);
  selectedId.value = partitions.value.at(-1)?.id ?? racks.value.at(-1)?.id ?? null;
}

function findFreePosition(width: number, depth: number) {
  const maxX = Math.max(0, room.width / 2 - width / 2 - 0.25);
  const maxZ = Math.max(0, room.length / 2 - depth / 2 - 0.45);
  const stepX = width + 0.55;
  const stepZ = depth + 0.75;
  for (let row = 0; row < 100; row += 1) {
    const z = -maxZ + row * stepZ;
    if (z > maxZ) break;
    const rowCount = Math.max(1, Math.floor((maxX * 2) / stepX) + 1);
    for (let column = 0; column < rowCount; column += 1) {
      const x = -maxX + column * stepX;
      if (x > maxX + 0.01) break;
      const occupied = racks.value.some((rack) => {
        return Math.abs(rack.group.position.x - x) < (rack.width + width) / 2 + 0.15
          && Math.abs(rack.group.position.z - z) < (rack.depth + depth) / 2 + 0.15;
      });
      if (!occupied) return { x, z };
    }
  }
  return { x: 0, z: 0 };
}

function applyRoomDimensions() {
  roomDraft.width = clampNumber(roomDraft.width, 4, 40, room.width);
  roomDraft.length = clampNumber(roomDraft.length, 4, 40, room.length);
  roomDraft.height = clampNumber(roomDraft.height, 2.5, 12, room.height);
  room.width = roomDraft.width;
  room.length = roomDraft.length;
  room.height = roomDraft.height;
  buildRoom();
  for (const rack of racks.value) clampRackToRoom(rack);
  for (const partition of partitions.value) {
    if (partition.fullSpan) partition.length = partitionMaxLength(partition);
    clampPartitionToRoom(partition);
    rebuildPartition(partition);
  }
}

function clampNumber(value: number, min: number, max: number, fallback: number) {
  return Number.isFinite(value) ? Math.min(max, Math.max(min, value)) : fallback;
}

function clampRackToRoom(rack: RackItem) {
  const limitX = Math.max(0, room.width / 2 - rack.width / 2 - 0.08);
  const limitZ = Math.max(0, room.length / 2 - rack.depth / 2 - 0.08);
  rack.group.position.x = THREE.MathUtils.clamp(rack.group.position.x, -limitX, limitX);
  rack.group.position.z = THREE.MathUtils.clamp(rack.group.position.z, -limitZ, limitZ);
  rack.group.position.y = 0.035;
  rack.x = rack.group.position.x;
  rack.z = rack.group.position.z;
}

function selectItem(id: string) {
  if (racks.value.some((rack) => rack.id === id) || partitions.value.some((partition) => partition.id === id)) selectedId.value = id;
}

function removeSelected() {
  if (!selectedRack.value || !scene) return;
  const id = selectedRack.value.id;
  scene.remove(selectedRack.value.group);
  disposeGroup(selectedRack.value.group);
  racks.value = racks.value.filter((rack) => rack.id !== id);
  for (const annotation of annotations.value.filter((item) => item.itemId === id)) removeAnnotation(annotation.id);
  selectedId.value = racks.value.at(-1)?.id ?? null;
}

function updatePosition(axis: 'x' | 'z', event: Event) {
  const rack = selectedRack.value;
  const input = event.target as HTMLInputElement;
  if (!rack || !Number.isFinite(input.valueAsNumber)) return;
  rack.group.position[axis] = input.valueAsNumber;
  clampRackToRoom(rack);
}

function triggerImport() {
  importError.value = '';
  fileInput.value?.click();
}

function importModel(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  importError.value = '';
  file.arrayBuffer().then((buffer) => {
    gltfLoader.parse(buffer, '', (gltf) => {
      const id = createId();
      const group = new THREE.Group();
      group.name = file.name.replace(/\.glb$/i, '') || '导入模型';
      const model = gltf.scene;
      model.updateMatrixWorld(true);
      const bounds = new THREE.Box3().setFromObject(model);
      const size = bounds.getSize(new THREE.Vector3());
      if (!Number.isFinite(size.y) || size.y <= 0) {
        importError.value = '模型尺寸无效，请检查 GLB 文件。';
        return;
      }
      const targetHeight = 2.05;
      const scale = targetHeight / size.y;
      model.scale.multiplyScalar(scale);
      model.position.set(
        -((bounds.min.x + bounds.max.x) / 2) * scale,
        -bounds.min.y * scale,
        -((bounds.min.z + bounds.max.z) / 2) * scale,
      );
      group.add(model);
      group.traverse((child) => {
        if (child instanceof THREE.Mesh) {
          child.userData.itemId = id;
          child.castShadow = true;
          child.receiveShadow = true;
        }
      });
      const itemName = file.name.replace(/\.glb$/i, '') || '导入模型';
      group.name = itemName;
      const itemWidth = Math.max(0.25, size.x * scale);
      const itemDepth = Math.max(0.25, size.z * scale);
      const position = findFreePosition(itemWidth, itemDepth);
      group.position.set(position.x, 0.035, position.z);
      scene.add(group);
      racks.value.push({
        id,
        name: itemName,
        kind: 'imported',
        group: markRaw(group),
        width: itemWidth,
        depth: itemDepth,
        height: targetHeight,
        x: position.x,
        z: position.z,
      });
      selectedId.value = id;
    }, () => {
      importError.value = '无法读取此 GLB 模型，请确认文件完整后重试。';
    });
  }).catch(() => {
    importError.value = '读取模型文件失败，请重试。';
  }).finally(() => {
    input.value = '';
  });
}

function addAnnotation() {
  const rack = selectedRack.value;
  const title = annotationTitle.value.trim();
  if (!rack || !title) return;
  const id = createId();
  const element = document.createElement('div');
  element.className = 'scene-annotation';
  const pointerMark = document.createElement('span');
  pointerMark.className = 'scene-annotation-pin';
  pointerMark.textContent = '↗';
  const content = document.createElement('span');
  content.className = 'scene-annotation-content';
  const heading = document.createElement('strong');
  heading.textContent = title;
  content.appendChild(heading);
  if (annotationDescription.value.trim()) {
    const description = document.createElement('small');
    description.textContent = annotationDescription.value.trim();
    content.appendChild(description);
  }
  element.append(pointerMark, content);
  const label = markRaw(new CSS2DObject(element));
  label.position.set(0, rack.height + 0.36, 0);
  rack.group.add(label);
  annotations.value.push({
    id,
    itemId: rack.id,
    title,
    description: annotationDescription.value.trim(),
    label,
  });
  annotationTitle.value = '';
  annotationDescription.value = '';
}

function removeAnnotation(id: string) {
  const annotation = annotations.value.find((item) => item.id === id);
  if (!annotation) return;
  annotation.label.parent?.remove(annotation.label);
  annotations.value = annotations.value.filter((item) => item.id !== id);
}

function createId() {
  return globalThis.crypto?.randomUUID?.() ?? `item-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

async function loadFreeRackModel() {
  const response = await fetch('/models/server-rack.glb');
  if (!response.ok) throw new Error(`Free rack model request failed: ${response.status}`);
  const buffer = await response.arrayBuffer();
  const gltf = await new Promise<GLTF>((resolve, reject) => {
    gltfLoader.parse(buffer, '', resolve, reject);
  });
  freeRackPrototype = gltf.scene;
}

function disposeGroup(group: THREE.Object3D) {
  if (group.userData.sharedAssets) return;
  group.traverse((child) => {
    if (child instanceof THREE.Mesh || child instanceof THREE.Line || child instanceof THREE.LineSegments) {
      child.geometry.dispose();
      const materials = Array.isArray(child.material) ? child.material : [child.material];
      for (const material of materials) material.dispose();
    }
  });
}

function setPointer(event: PointerEvent) {
  if (!renderer || !viewport.value) return;
  const rect = renderer.domElement.getBoundingClientRect();
  pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
}

function onPointerDown(event: PointerEvent) {
  if (event.button !== 0 || !renderer || !camera) return;
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const meshes = [...racks.value, ...partitions.value].flatMap((item) => {
    const children: THREE.Object3D[] = [];
    item.group.traverse((child) => {
      if (child instanceof THREE.Mesh) children.push(child);
    });
    return children;
  });
  const hit = raycaster.intersectObjects(meshes, false)[0];
  if (!hit) {
    selectedId.value = null;
    return;
  }
  const id = hit.object.userData.itemId as string | undefined;
  const rack = racks.value.find((item) => item.id === id);
  const partition = partitions.value.find((item) => item.id === id);
  if (!rack && !partition) return;
  selectedId.value = id ?? null;
  activeDrag = rack ? { kind: 'rack', id: rack.id } : partition ? { kind: 'partition', id: partition.id } : null;
  floorPlaneForDrag = new THREE.Plane(new THREE.Vector3(0, 1, 0), -0.035);
  controls.enabled = false;
  renderer.domElement.setPointerCapture(event.pointerId);
  moveRack(event);
}

function moveRack(event: PointerEvent) {
  if (!activeDrag) return;
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const point = raycaster.ray.intersectPlane(floorPlaneForDrag, new THREE.Vector3());
  if (!point) return;
  const step = 0.1;
  const rack = activeDrag.kind === 'rack' ? racks.value.find((item) => item.id === activeDrag?.id) : null;
  if (rack) {
    rack.group.position.x = Math.round(point.x / step) * step;
    rack.group.position.z = Math.round(point.z / step) * step;
    clampRackToRoom(rack);
    return;
  }
  const partition = activeDrag.kind === 'partition' ? partitions.value.find((item) => item.id === activeDrag?.id) : null;
  if (partition) {
    partition.x = Math.round(point.x / step) * step;
    partition.z = Math.round(point.z / step) * step;
    clampPartitionToRoom(partition);
  }
}

function onPointerMove(event: PointerEvent) {
  if (activeDrag) moveRack(event);
}

function onPointerUp() {
  if (!activeDrag) return;
  if (activeDrag.kind === 'partition') {
    const partition = partitions.value.find((item) => item.id === activeDrag?.id);
    if (partition) rebuildPartition(partition);
  }
  activeDrag = null;
  controls.enabled = true;
}

function resetCamera() {
  if (!camera || !controls) return;
  camera.position.set(room.width * 1.25, room.height * 2.3, room.length * 1.45);
  controls.target.set(0, room.height * 0.3, 0);
  controls.update();
}

function resizeViewport() {
  if (!viewport.value || !renderer || !camera) return;
  const { width, height } = viewport.value.getBoundingClientRect();
  if (!width || !height) return;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  labelRenderer.setSize(width, height);
}

function animate() {
  animationFrame = window.requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
  labelRenderer.render(scene, camera);
}

onMounted(async () => {
  await nextTick();
  if (!viewport.value) return;
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111a22);
  scene.fog = new THREE.Fog(0x111a22, 24, 62);

  const { width, height } = viewport.value.getBoundingClientRect();
  camera = new THREE.PerspectiveCamera(42, width / height, 0.1, 100);
  camera.position.set(15, 11, 18);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFShadowMap;
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.15;
  renderer.domElement.className = 'scene-webgl';
  viewport.value.prepend(renderer.domElement);

  labelRenderer = new CSS2DRenderer();
  labelRenderer.domElement.className = 'scene-label-layer';
  viewport.value.append(labelRenderer.domElement);
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.07;
  controls.minDistance = 5;
  controls.maxDistance = 42;
  controls.maxPolarAngle = Math.PI * 0.49;
  controls.target.set(0, 1.2, 0);

  scene.add(new THREE.HemisphereLight(0xc9e5ff, 0x38414a, 2.1));
  const keyLight = new THREE.DirectionalLight(0xe4f2ff, 3.2);
  keyLight.position.set(-7, 15, 8);
  keyLight.castShadow = true;
  keyLight.shadow.mapSize.set(2048, 2048);
  keyLight.shadow.camera.left = -18;
  keyLight.shadow.camera.right = 18;
  keyLight.shadow.camera.top = 18;
  keyLight.shadow.camera.bottom = -18;
  scene.add(keyLight);
  const fillLight = new THREE.DirectionalLight(0x61b9cf, 1.25);
  fillLight.position.set(10, 8, -8);
  scene.add(fillLight);
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(200, 200),
    new THREE.MeshStandardMaterial({ color: 0x111a22, roughness: 0.94 }),
  );
  ground.rotation.x = -Math.PI / 2;
  ground.position.y = -0.035;
  ground.receiveShadow = true;
  scene.add(ground);
  buildRoom();

  resizeObserver = new ResizeObserver(resizeViewport);
  resizeObserver.observe(viewport.value);
  renderer.domElement.addEventListener('pointerdown', onPointerDown);
  renderer.domElement.addEventListener('pointermove', onPointerMove);
  renderer.domElement.addEventListener('pointerup', onPointerUp);
  renderer.domElement.addEventListener('pointercancel', onPointerUp);
  window.addEventListener('pointerup', onPointerUp);
  resizeViewport();
  animate();
  try {
    await loadFreeRackModel();
    assetStatus.value = '默认机柜为 CC0 免费模型';
  } catch (error) {
    assetStatus.value = '使用内置参数化机柜';
    assetError.value = '在线免费模型暂不可用，已切换为内置机柜。';
    console.error('Unable to load the CC0 rack model.', error);
  }
  addRack();
  addRack();
  addRack();
  addRack();
  racks.value[0].group.position.set(-2.2, 0.035, -1.75);
  racks.value[1].group.position.set(-0.95, 0.035, -1.75);
  racks.value[2].group.position.set(0.95, 0.035, -1.75);
  racks.value[3].group.position.set(2.2, 0.035, -1.75);
  for (const rack of racks.value) {
    rack.x = rack.group.position.x;
    rack.z = rack.group.position.z;
  }
  selectedId.value = null;
});

onBeforeUnmount(() => {
  if (animationFrame) window.cancelAnimationFrame(animationFrame);
  resizeObserver?.disconnect();
  window.removeEventListener('pointerup', onPointerUp);
  if (renderer) {
    renderer.domElement.removeEventListener('pointerdown', onPointerDown);
    renderer.domElement.removeEventListener('pointermove', onPointerMove);
    renderer.domElement.removeEventListener('pointerup', onPointerUp);
    renderer.domElement.removeEventListener('pointercancel', onPointerUp);
    renderer.dispose();
  }
  controls?.dispose();
});
</script>

<style>
:root {
  font-family: Inter, "Segoe UI", "Microsoft YaHei", sans-serif;
  color: #e7edf3;
  background: #10171d;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  --surface: #151e25;
  --surface-deep: #11191f;
  --surface-raised: #1b262e;
  --line: rgba(179, 205, 220, 0.1);
  --muted: #82919d;
  --soft: #a9b5be;
  --accent: #61c2c2;
  --accent-bright: #83d9d4;
}

* { box-sizing: border-box; }
html, body, #app { width: 100%; height: 100%; margin: 0; overflow: hidden; }
body { background: #10171d; }
button, input, textarea { font: inherit; }
button { color: inherit; }

.studio-shell { height: 100%; min-width: 900px; display: flex; flex-direction: column; background: #10171d; }
.topbar { height: 62px; flex: 0 0 62px; display: flex; align-items: center; justify-content: space-between; padding: 0 24px; background: #141d24; border-bottom: 1px solid var(--line); position: relative; z-index: 3; }
.brand { display: flex; align-items: center; gap: 11px; color: #e9f0f3; text-decoration: none; min-width: 255px; }
.brand-mark { width: 22px; height: 22px; display: grid; grid-template-columns: repeat(2, 8px); grid-template-rows: repeat(2, 8px); gap: 3px; transform: rotate(-8deg); }
.brand-mark span { display: block; border: 1px solid #78cfca; background: rgba(97, 194, 194, .15); }
.brand-mark span:nth-child(2), .brand-mark span:nth-child(3) { background: rgba(97, 194, 194, .55); }
.brand-name { font-size: 13px; letter-spacing: 1.15px; font-weight: 700; }
.brand-name span { color: #85949e; font-weight: 500; font-size: 10px; letter-spacing: 1.35px; margin-left: 4px; }
.project-name { display: flex; align-items: center; gap: 9px; color: #d7e0e5; font-size: 12px; position: absolute; left: 50%; transform: translateX(-50%); }
.live-dot { height: 6px; width: 6px; border-radius: 50%; background: #6bc7ae; box-shadow: 0 0 9px rgba(107, 199, 174, .55); display: inline-block; }
.saved-state { font-size: 10px; color: #87949b; padding: 4px 8px; border: 1px solid rgba(255,255,255,.09); border-radius: 3px; margin-left: 4px; }
.top-actions { display: flex; gap: 9px; align-items: center; }
.top-button { height: 34px; border-radius: 4px; padding: 0 12px; font-size: 11px; border: 1px solid transparent; cursor: pointer; transition: .18s ease; }
.top-button.subtle { background: transparent; color: #a7b3bb; border-color: rgba(255,255,255,.12); }
.top-button.subtle:hover { color: #eaf5f3; border-color: rgba(131,217,212,.45); }
.top-button.primary { background: #67bebb; color: #102021; font-weight: 700; padding: 0 14px; }
.top-button.primary:hover { background: #8bd5cf; }
.top-button.partition-top-button { background: transparent; color: #a9c7c7; border-color: rgba(129, 177, 180, .2); }
.top-button.partition-top-button:hover { color: #d3efea; border-color: rgba(131,217,212,.45); background: rgba(91,174,170,.08); }
.button-icon { font-size: 16px; vertical-align: -1px; margin-right: 4px; }
.plus { font-size: 15px; margin-right: 4px; }

.workspace { flex: 1; min-height: 0; display: grid; grid-template-columns: 258px minmax(420px, 1fr) 300px; }
.sidebar { min-height: 0; display: flex; flex-direction: column; background: #151e25; position: relative; z-index: 2; }
.left-sidebar { border-right: 1px solid var(--line); }
.right-sidebar { border-left: 1px solid var(--line); overflow-y: auto; }
.sidebar-heading, .right-heading { padding: 25px 20px 19px; display: flex; align-items: flex-start; justify-content: space-between; }
.eyebrow { color: #75918f; font-size: 8px; letter-spacing: 1.65px; font-weight: 700; margin-bottom: 6px; }
.sidebar-heading h1, .right-heading h2 { font-size: 17px; font-weight: 600; margin: 0; color: #edf2f2; letter-spacing: .1px; }
.heading-icon { color: #75c4c0; font-size: 20px; margin: 6px 1px 0 0; }
.side-section { padding: 0 19px 19px; border-bottom: 1px solid var(--line); }
.section-title { display: flex; align-items: center; gap: 8px; color: #dbe3e7; font-size: 11px; font-weight: 600; margin-bottom: 13px; }
.section-note, .edit-hint { color: #76858e; font-size: 9px; font-weight: 400; margin-left: auto; }
.dimension-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 9px 8px; }
.dimension-field { min-width: 0; }
.dimension-field > span { display: block; color: #91a0a9; font-size: 10px; margin-bottom: 5px; }
.dimension-field > span small { font-size: 8px; color: #586b77; margin-left: 4px; }
.dimension-field > div { height: 32px; border: 1px solid rgba(172, 199, 211, .13); background: #11191f; display: flex; align-items: center; border-radius: 3px; padding: 0 8px; }
.dimension-field input { border: 0; min-width: 0; width: 100%; outline: 0; background: transparent; color: #e3ecef; font-size: 11px; }
.dimension-field b { font-size: 9px; color: #687b84; font-weight: 500; }
input[type="number"]::-webkit-inner-spin-button, input[type="number"]::-webkit-outer-spin-button { opacity: .35; }
.apply-button { width: 100%; height: 32px; margin-top: 11px; border: 1px solid rgba(101, 190, 187, .35); background: rgba(91, 174, 170, .1); color: #8ad2ce; border-radius: 3px; text-align: left; padding: 0 10px; font-size: 10px; cursor: pointer; }
.apply-button span { float: right; font-size: 14px; }
.apply-button:hover { background: rgba(91, 174, 170, .18); }
.dimension-hint { margin-top: 10px; color: #697983; font-size: 9px; line-height: 1.5; }
.tiny-grid { color: #6bb9b5; margin-right: 5px; }
.inventory-section { padding-top: 17px; }
.count-pill { margin-left: auto; color: #8ccbc6; font-size: 9px; padding: 3px 6px; background: rgba(98, 188, 184, .11); border-radius: 3px; font-variant-numeric: tabular-nums; }
.object-list { max-height: 207px; overflow-y: auto; margin: 0 -7px; padding: 0 3px; }
.object-row { width: 100%; min-height: 51px; display: flex; align-items: center; gap: 9px; text-align: left; padding: 6px; background: transparent; border: 1px solid transparent; border-radius: 4px; cursor: pointer; }
.object-row:hover { background: rgba(255,255,255,.025); }
.object-row.selected { background: rgba(99, 183, 180, .085); border-color: rgba(99, 183, 180, .2); }
.object-thumb { width: 34px; height: 35px; display: flex; justify-content: center; align-items: center; border-radius: 3px; background: #202c33; border: 1px solid rgba(255,255,255,.045); }
.rack-glyph { width: 14px; height: 23px; border: 1px solid #8eabb0; border-radius: 2px; position: relative; display: inline-block; background: repeating-linear-gradient(to bottom, transparent 0 3px, rgba(113,169,174,.6) 3px 4px); box-shadow: inset 0 0 0 2px rgba(0,0,0,.15); }
.rack-glyph::before, .rack-glyph::after { content: ""; position: absolute; top: -1px; bottom: -1px; width: 2px; background: #667e84; }
.rack-glyph::before { left: 2px; }.rack-glyph::after { right: 2px; }
.partition-thumb { background: #233138; }
.partition-glyph { width: 23px; height: 15px; display: inline-block; border: 1px solid #8eabb0; background: repeating-linear-gradient(to right, rgba(113,169,174,.3) 0 2px, transparent 2px 5px); }
.partition-glyph.large { width: 34px; height: 20px; border-color: #9bb9b8; background: repeating-linear-gradient(to right, rgba(126,189,187,.4) 0 2px, transparent 2px 6px); transform: rotate(-24deg); box-shadow: 0 0 13px rgba(114,211,198,.1); z-index: 1; }
.partition-add-button { margin-top: 6px; }
.partition-add-icon { font-size: 15px; color: #80c6c1; }
.object-copy { min-width: 0; display: flex; flex-direction: column; gap: 4px; }
.object-copy strong { font-size: 10px; color: #d7e0e4; font-weight: 550; }
.object-copy small { color: #71818a; font-size: 9px; }
.object-chevron { margin-left: auto; color: #697c85; font-size: 18px; }
.empty-objects { padding: 12px 8px; color: #77868e; font-size: 10px; line-height: 1.5; }
.import-button { width: 100%; height: 35px; display: flex; align-items: center; gap: 8px; padding: 0 9px; margin-top: 9px; background: transparent; color: #aab8bf; border: 1px dashed rgba(159, 189, 199, .23); border-radius: 3px; font-size: 10px; cursor: pointer; }
.import-button:hover { border-color: rgba(125, 207, 201, .56); color: #c3e3df; }
.upload-icon { font-size: 15px; color: #73c5c0; }
.import-button small { margin-left: auto; font-size: 8px; color: #71858c; border: 1px solid rgba(255,255,255,.1); padding: 2px 4px; border-radius: 2px; }
.asset-source { display: flex; align-items: flex-start; justify-content: space-between; gap: 6px; margin-top: 7px; color: #72838a; font-size: 8px; }
.asset-links { display: flex; gap: 7px; }
.asset-source a { color: #85c7c1; text-decoration: none; white-space: nowrap; }
.asset-source a:hover { color: #b2e2dc; text-decoration: underline; }
.visually-hidden { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events: none; }
.error-note { color: #f19283; font-size: 9px; line-height: 1.5; margin: 7px 1px 0; }
.room-summary { padding-top: 17px; border-bottom: 0; }
.status-label { margin-left: auto; color: #83b9aa; font-size: 9px; font-weight: 400; }
.status-label i { width: 5px; height: 5px; background: #6bc7a6; border-radius: 50%; display: inline-block; margin-right: 5px; vertical-align: 1px; }
.summary-metrics { display: grid; grid-template-columns: repeat(3, 1fr); gap: 4px; }
.summary-metrics > div { display: flex; flex-direction: column; gap: 5px; }
.summary-metrics strong { color: #dce5e8; font-size: 13px; font-weight: 550; font-variant-numeric: tabular-nums; }
.summary-metrics strong small { font-size: 8px; color: #809099; font-weight: 400; margin-left: 2px; }
.summary-metrics span { color: #70808a; font-size: 9px; }
.sidebar-footer { margin-top: auto; min-height: 57px; display: flex; align-items: center; gap: 9px; border-top: 1px solid var(--line); padding: 0 17px; }
.footer-avatar { width: 25px; height: 25px; display: grid; place-items: center; background: #243b3d; border: 1px solid rgba(120, 198, 187, .22); color: #94d3c8; border-radius: 4px; font-size: 10px; }
.sidebar-footer > span:nth-child(2) { display: flex; flex-direction: column; gap: 3px; }
.sidebar-footer strong { color: #bac6cb; font-size: 9px; font-weight: 550; }
.sidebar-footer small { color: #71808a; font-size: 8px; }
.footer-more { margin-left: auto; color: #7d8b91; font-size: 18px; letter-spacing: 2px; }

.viewport-column { min-width: 0; display: flex; flex-direction: column; position: relative; background: #111920; }
.viewport-toolbar { height: 47px; flex: 0 0 47px; display: flex; justify-content: space-between; align-items: center; padding: 0 19px; border-bottom: 1px solid rgba(179,205,220,.065); }
.breadcrumbs { display: flex; align-items: center; gap: 10px; color: #788993; font-size: 10px; }
.breadcrumbs i { font-style: normal; color: #4a5b64; }
.breadcrumbs strong { color: #c4d0d5; font-size: 10px; font-weight: 500; }
.view-chip { border: 1px solid rgba(126, 201, 198, .18); color: #82c8c3; border-radius: 2px; font-size: 8px; padding: 3px 6px; margin-left: 3px; }
.view-tools { display: flex; align-items: center; gap: 13px; }
.view-tool-label { color: #7d8f97; font-size: 9px; }
.view-tool-label i { display: inline-block; width: 6px; height: 6px; border: 1px solid #77c1bd; margin-right: 5px; transform: rotate(45deg); }
.icon-button { border: 1px solid rgba(255,255,255,.1); background: transparent; color: #8fa0a8; width: 25px; height: 25px; border-radius: 3px; cursor: pointer; font-size: 15px; }
.icon-button:hover { color: #b6e1de; border-color: rgba(129,208,204,.42); }
.viewport { flex: 1; min-height: 0; position: relative; overflow: hidden; background: radial-gradient(ellipse at 48% 56%, #263641, #141e26 72%); }
.scene-webgl, .scene-label-layer { position: absolute; inset: 0; width: 100%; height: 100%; display: block; }
.scene-label-layer { pointer-events: none; overflow: hidden; }
.scene-label-layer .scene-annotation { pointer-events: auto; }
.viewport-wash { pointer-events: none; z-index: 1; position: absolute; inset: 0; background: linear-gradient(180deg, rgba(13,20,26,.36), transparent 22%, transparent 76%, rgba(13,20,26,.16)); }
.viewport-caption { position: absolute; z-index: 1; top: 24px; left: 25px; display: flex; flex-direction: column; gap: 6px; pointer-events: none; }
.caption-kicker { color: #7e9f9e; font-size: 8px; letter-spacing: 1.5px; }
.viewport-caption strong { color: #d5e0e3; font-size: 13px; font-weight: 550; }
.viewport-caption > span:last-child { color: rgba(188, 205, 211, .57); font-size: 9px; }
.axis-indicator { position: absolute; z-index: 1; top: 27px; right: 26px; width: 44px; height: 44px; border-radius: 50%; border: 1px solid rgba(199,224,224,.13); pointer-events: none; color: #8fa7af; font-size: 8px; }
.axis-indicator i { width: 3px; height: 3px; position: absolute; background: #d6e0e0; border-radius: 50%; left: 20px; top: 20px; }
.axis-y, .axis-x, .axis-z { position: absolute; }
.axis-y { left: 19px; top: 4px; color: #77ceca; }.axis-x { right: 6px; top: 22px; color: #e0988d; }.axis-z { left: 9px; top: 29px; color: #91b9e8; }
.view-scale { position: absolute; z-index: 1; right: 24px; bottom: 24px; display: flex; align-items: center; gap: 6px; color: #a2b3b8; font-size: 8px; pointer-events: none; }
.view-scale i { width: 39px; height: 4px; border: 1px solid rgba(173,204,205,.7); border-top: 0; display: inline-block; }
.viewport-status { position: absolute; z-index: 1; left: 24px; bottom: 23px; display: flex; gap: 8px; align-items: center; color: #a4b3b8; font-size: 9px; pointer-events: none; }
.viewport-status .live-dot { width: 5px; height: 5px; }
.viewport-status i { width: 1px; height: 10px; background: rgba(255,255,255,.2); margin: 0 1px; }
.selection-hud { position: absolute; z-index: 2; top: 24px; right: 83px; display: flex; gap: 8px; align-items: center; padding: 7px 9px; border: 1px solid rgba(122,201,196,.24); border-radius: 4px; background: rgba(18,30,36,.82); backdrop-filter: blur(8px); pointer-events: none; }
.selection-dot { width: 6px; height: 6px; background: #7fd0c8; border-radius: 50%; box-shadow: 0 0 7px rgba(127,208,200,.6); }
.selection-hud > span:nth-child(2) { display: flex; flex-direction: column; gap: 3px; }
.selection-hud small { color: #8b9b9f; font-size: 8px; }
.selection-hud strong { color: #e0e9e9; font-size: 9px; font-weight: 550; }
.hud-coordinates { color: #aab8bd; font-size: 8px; margin-left: 6px; }
.hud-coordinates i { color: #5e777d; font-style: normal; padding: 0 3px; }
.viewport-footer { height: 35px; flex: 0 0 35px; display: flex; align-items: center; justify-content: space-between; padding: 0 19px; border-top: 1px solid rgba(179,205,220,.065); }
.legend { display: flex; gap: 15px; }
.legend span { display: flex; align-items: center; gap: 5px; color: #87969c; font-size: 8px; }
.legend i { width: 6px; height: 6px; display: inline-block; border-radius: 1px; }
.legend-room { background: #69b4bd; }.legend-rack { background: #9babaf; }.legend-door { background: #cb967e; }
.camera-label { color: #71818a; font-size: 8px; letter-spacing: .8px; }.camera-label i { color: #74bbb7; font-style: normal; }

.right-heading { padding: 25px 18px 17px; }
.right-heading h2 { font-size: 16px; }
.more-dots { color: #788a91; letter-spacing: 2px; font-size: 16px; }
.selected-card { display: flex; margin: 0 17px 18px; border: 1px solid rgba(179,205,220,.09); background: #19232a; border-radius: 4px; min-height: 83px; overflow: hidden; }
.selected-preview { width: 77px; flex: 0 0 77px; background: radial-gradient(ellipse, #35464d, #202a30 67%); display: grid; place-items: center; position: relative; }
.rack-glyph.large { width: 19px; height: 35px; border-color: #9bb9b8; background: repeating-linear-gradient(to bottom, transparent 0 4px, rgba(126,189,187,.7) 4px 5px); z-index: 1; }
.rack-glyph.large::before, .rack-glyph.large::after { background: #758c8c; }
.preview-glow { width: 44px; height: 10px; background: rgba(114,211,198,.16); filter: blur(8px); position: absolute; bottom: 18px; }
.selected-name { display: flex; flex-direction: column; justify-content: center; min-width: 0; gap: 5px; padding: 9px 10px; }
.selected-type { color: #73bcb8; font-size: 7px; letter-spacing: 1px; }
.selected-name strong { color: #e3eaec; font-size: 11px; font-weight: 550; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.selected-name small { color: #788890; font-size: 8px; }
.property-section { padding: 0 18px 16px; margin-bottom: 14px; border-bottom: 1px solid var(--line); }
.partition-length-field { height: 34px; display: flex; align-items: center; gap: 8px; padding: 0 8px; margin: -2px 0 12px; border: 1px solid rgba(172,199,211,.11); background: #11191f; border-radius: 3px; color: #829198; font-size: 9px; }
.partition-length-field input { width: 100%; min-width: 0; border: 0; outline: 0; background: transparent; color: #d6e0e2; font-size: 10px; }
.partition-length-field b { color: #75858b; font-size: 8px; font-weight: 400; }
.rotate-button { width: 100%; height: 31px; display: flex; align-items: center; gap: 6px; margin-top: 11px; padding: 0 8px; border: 1px solid rgba(172,199,211,.11); background: rgba(255,255,255,.02); color: #a9b8bd; border-radius: 3px; font-size: 9px; text-align: left; cursor: pointer; }
.rotate-button:hover { border-color: rgba(112,193,188,.32); color: #d1e5e3; }
.rotate-button > span { color: #85c9c3; font-size: 14px; }
.rotate-button small { margin-left: auto; color: #71838a; font-size: 8px; }
.partition-door-panel { padding: 0 18px 15px; margin-bottom: 14px; border-bottom: 1px solid var(--line); }
.partition-door-panel .section-title { margin-bottom: 8px; }
.partition-door-panel p { margin: 0 0 10px; color: #7e8e94; font-size: 8px; line-height: 1.6; }
.door-status { margin-left: auto; color: #849397; font-size: 8px; font-weight: 400; }
.door-status.installed { color: #83c4a8; }
.door-actions { display: flex; align-items: center; gap: 7px; }
.door-actions .annotation-add { flex: 1; }
.remove-door-button { height: 31px; padding: 0 8px; background: transparent; border: 1px solid rgba(219,116,105,.17); border-radius: 3px; color: #c5877f; font-size: 8px; cursor: pointer; }
.remove-door-button:hover { border-color: rgba(219,116,105,.4); }
.coordinate-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 7px; }
.coordinate-grid label { height: 35px; display: flex; align-items: center; gap: 5px; border: 1px solid rgba(172,199,211,.11); background: #11191f; padding: 0 6px; border-radius: 3px; min-width: 0; }
.coordinate-grid label > span { color: #70bbb7; font-size: 9px; }
.coordinate-grid input { width: 100%; min-width: 0; border: 0; outline: 0; background: transparent; color: #d6e0e2; font-size: 10px; }
.coordinate-grid input:disabled { color: #75848a; }
.unit-tag { color: #77868e; font-size: 9px; margin-left: auto; font-weight: 400; }
.dimension-properties { padding-bottom: 10px; }
.size-row { display: flex; align-items: center; justify-content: space-between; margin: 0 0 9px; font-size: 9px; color: #829198; }
.size-row strong { color: #d0dbde; font-size: 10px; font-weight: 500; }
.size-row small { color: #74858b; font-size: 8px; font-weight: 400; }
.danger-button { margin: 0 18px 18px; height: 32px; text-align: left; padding: 0 9px; border-radius: 3px; border: 1px solid rgba(219,116,105,.17); background: rgba(163,72,65,.045); color: #c5877f; font-size: 9px; cursor: pointer; }
.danger-button:hover { border-color: rgba(219,116,105,.4); background: rgba(163,72,65,.1); }
.danger-button span { font-size: 13px; margin-right: 5px; vertical-align: -1px; }
.tool-intro { margin: 0 18px 15px; padding: 18px 14px 14px; background: linear-gradient(135deg, rgba(79,142,143,.12), rgba(31,45,51,.1)); border: 1px solid rgba(120,191,187,.11); border-radius: 4px; }
.tool-illustration { height: 65px; position: relative; margin-bottom: 12px; display: grid; place-items: center; }
.tool-cube { width: 35px; height: 35px; border: 1px solid rgba(123,203,198,.73); transform: rotate(30deg) skew(-4deg,-4deg); box-shadow: 8px 7px 0 -1px rgba(83,144,148,.5), 0 0 25px rgba(84,186,183,.13); background: rgba(57,114,118,.13); }
.tool-illustration i, .tool-illustration b { position: absolute; display: block; border: 1px solid rgba(130,192,189,.25); width: 52px; height: 31px; transform: skewY(-24deg); }
.tool-illustration i { left: 42px; top: 10px; border-right: 0; border-bottom: 0; }.tool-illustration b { right: 42px; bottom: 10px; border-left: 0; border-top: 0; }
.tool-intro > strong { color: #d6e2e3; font-size: 11px; font-weight: 550; }
.tool-intro p { margin: 7px 0 0; color: #87969a; font-size: 9px; line-height: 1.7; }
.quick-actions { padding: 0 18px 17px; display: flex; flex-direction: column; gap: 7px; border-bottom: 1px solid var(--line); }
.quick-actions button { min-height: 49px; display: flex; align-items: center; gap: 9px; text-align: left; border: 1px solid rgba(179,205,220,.09); border-radius: 3px; background: rgba(255,255,255,.015); padding: 6px 9px; cursor: pointer; }
.quick-actions button:hover { border-color: rgba(112,193,188,.32); background: rgba(84,152,150,.06); }
.quick-icon { width: 28px; height: 28px; display: grid; place-items: center; border-radius: 3px; color: #88c9c4; background: rgba(91,169,167,.12); font-size: 16px; }
.quick-icon.upload { font-size: 18px; }
.quick-actions button > span:nth-child(2) { display: flex; flex-direction: column; gap: 4px; }
.quick-actions strong { color: #d5e0e1; font-size: 9px; font-weight: 550; }
.quick-actions small { color: #77868b; font-size: 8px; }
.quick-actions button > b { margin-left: auto; color: #758a8e; font-size: 14px; font-weight: 400; }
.annotation-section { padding: 17px 18px 16px; }
.annotation-heading { display: flex; align-items: center; justify-content: space-between; }
.annotation-heading .section-title { margin-bottom: 12px; }
.annotation-helper { margin-bottom: 12px; color: #6f858b; font-size: 8px; }
.annotation-form { padding: 10px; border: 1px solid rgba(179,205,220,.09); background: rgba(9,15,19,.25); border-radius: 3px; }
.form-label { display: block; color: #94a3a8; font-size: 9px; margin-bottom: 5px; }
.text-field { width: 100%; color: #d7e2e3; background: #11191f; border: 1px solid rgba(172,199,211,.12); border-radius: 3px; outline: 0; padding: 8px; font-size: 9px; margin-bottom: 10px; }
.text-field:focus { border-color: rgba(113,197,192,.5); }
.text-field::placeholder { color: #63727a; }
.description-field { resize: vertical; min-height: 55px; max-height: 105px; line-height: 1.5; }
.annotation-add { width: 100%; height: 31px; color: #a2d9d4; border: 1px solid rgba(107,190,185,.24); border-radius: 3px; background: rgba(86,160,157,.1); font-size: 9px; cursor: pointer; }
.annotation-add:hover:not(:disabled) { background: rgba(86,160,157,.2); }
.annotation-add:disabled { opacity: .45; cursor: not-allowed; }
.annotation-add span { font-size: 13px; margin-right: 4px; }
.annotation-tip { margin: 8px 1px 0; color: #708088; font-size: 8px; line-height: 1.5; }
.annotation-list { display: flex; flex-direction: column; gap: 5px; margin-top: 10px; max-height: 180px; overflow-y: auto; }
.annotation-row { display: flex; align-items: center; gap: 7px; min-height: 39px; padding: 5px 7px; background: rgba(255,255,255,.025); border: 1px solid transparent; border-radius: 3px; cursor: pointer; }
.annotation-row:hover, .annotation-row.active { border-color: rgba(112,193,188,.21); }
.annotation-pin { color: #7fcac4; font-size: 14px; }
.annotation-copy { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 3px; }
.annotation-copy strong { color: #d6e0e2; font-size: 9px; font-weight: 500; }
.annotation-copy small { color: #74838a; font-size: 8px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.annotation-row button { color: #809097; border: 0; background: transparent; font-size: 15px; cursor: pointer; }
.annotation-row button:hover { color: #e19488; }
.annotation-empty { display: flex; gap: 8px; align-items: flex-start; margin-top: 11px; padding: 9px; background: rgba(255,255,255,.02); border-radius: 3px; }
.annotation-empty > span { color: #6eaaa8; font-size: 13px; }
.annotation-empty p { margin: 0; color: #77858a; font-size: 8px; line-height: 1.6; }
.right-footer { margin-top: auto; padding: 12px 18px; border-top: 1px solid var(--line); color: #71828a; font-size: 8px; letter-spacing: .4px; }
.right-footer > span { color: #78c5bf; margin-right: 5px; }.right-footer i { float: right; font-style: normal; color: #596a71; }

.scene-annotation { display: flex; align-items: flex-start; gap: 6px; max-width: 190px; transform: translate(-50%, -100%); background: rgba(17, 29, 34, .94); color: #dce8e7; padding: 6px 9px 6px 7px; border: 1px solid rgba(121,205,198,.55); border-radius: 4px; box-shadow: 0 5px 17px rgba(0,0,0,.28); white-space: normal; }
.scene-annotation::after { content: ""; position: absolute; bottom: -5px; left: 50%; width: 8px; height: 8px; background: #17272b; border-right: 1px solid rgba(121,205,198,.55); border-bottom: 1px solid rgba(121,205,198,.55); transform: translateX(-50%) rotate(45deg); }
.scene-annotation-pin { color: #80d5ca; font-size: 12px; line-height: 1.1; }
.scene-annotation-content { display: flex; flex-direction: column; gap: 3px; }
.scene-annotation-content strong { font-size: 9px; font-weight: 600; }
.scene-annotation-content small { max-width: 160px; color: #a8b8b8; font-size: 8px; line-height: 1.4; }

@media (max-width: 1150px) {
  .workspace { grid-template-columns: 225px minmax(390px, 1fr) 265px; }
  .sidebar-heading, .right-heading { padding-left: 16px; padding-right: 16px; }
  .side-section { padding-left: 15px; padding-right: 15px; }
  .project-name { left: 48%; }
}
@media (max-height: 760px) {
  .sidebar-heading, .right-heading { padding-top: 17px; padding-bottom: 13px; }
  .side-section { padding-bottom: 13px; }
  .inventory-section, .room-summary { padding-top: 12px; }
  .object-list { max-height: 152px; }
  .tool-intro { padding-top: 10px; padding-bottom: 10px; }
  .tool-illustration { height: 48px; margin-bottom: 8px; }
}
@media (max-width: 900px) {
  .studio-shell { min-width: 0; overflow-x: auto; }
  .workspace { min-width: 900px; }
  .brand { min-width: auto; }
  .project-name { display: none; }
}
</style>
