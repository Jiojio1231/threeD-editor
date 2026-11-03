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
        <span class="saved-state">{{ saveStatus }}</span>
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

        <section class="side-section scene-file-section">
          <div class="section-title"><span>场景文件</span><span class="section-note">本地 JSON</span></div>
          <div class="scene-file-actions">
            <button type="button" @click="saveSceneToBrowser">保存</button>
            <button type="button" @click="triggerSceneImport">导入</button>
            <button type="button" @click="exportSceneFile">导出</button>
          </div>
          <p>保存到当前浏览器，或导出 JSON 文件备份并在其他设备导入。</p>
          <input ref="sceneFileInput" class="visually-hidden" type="file" accept=".json,application/json" @change="importSceneFile">
          <p v-if="sceneFileError" class="error-note">{{ sceneFileError }}</p>
        </section>

        <section class="side-section inventory-section">
          <div class="section-title">
            <span>场景对象</span>
            <span class="count-pill">{{ (racks.length + partitions.length).toString().padStart(2, '0') }}</span>
          </div>
          <div class="object-list">
            <div
              v-for="rack in racks"
              :key="rack.id"
              class="object-row"
              :class="{ selected: selectedId === rack.id, locked: rack.locked }"
            >
              <button class="object-select" type="button" @click="selectItem(rack.id)">
                <span class="object-thumb"><span class="rack-glyph"></span></span>
                <span class="object-copy"><strong>{{ rack.name }}</strong><small>{{ rack.kind === 'imported' ? '导入模型' : '标准机柜' }} · {{ rack.height.toFixed(1) }} m</small></span>
                <span class="object-chevron">›</span>
              </button>
              <button class="lock-toggle" :class="{ active: rack.locked }" type="button" :aria-label="rack.locked ? `解锁${rack.name}` : `锁定${rack.name}`" :title="rack.locked ? '解锁物体' : '锁定物体'" @click.stop="toggleLock(rack.id)">
                <svg viewBox="0 0 16 16" aria-hidden="true"><rect x="3" y="7" width="10" height="7" rx="1.2"></rect><path v-if="rack.locked" d="M5 7V5a3 3 0 0 1 6 0v2"></path><path v-else d="M5 7V5a3 3 0 0 1 5.2-2"></path></svg>
              </button>
            </div>
            <div
              v-for="partition in partitions"
              :key="partition.id"
              class="object-row"
              :class="{ selected: selectedId === partition.id, locked: partition.locked }"
            >
              <button class="object-select" type="button" @click="selectItem(partition.id)">
                <span class="object-thumb partition-thumb"><span class="partition-glyph"></span></span>
                <span class="object-copy"><strong>{{ partition.name }}</strong><small>{{ partition.roomWallSide ? '机房外墙' : '隔板' }} · {{ partition.length.toFixed(1) }} m{{ partition.doors.length ? ` · ${partition.doors.length} 扇门` : '' }}</small></span>
                <span class="object-chevron">›</span>
              </button>
              <button v-if="!partition.roomWallSide" class="lock-toggle" :class="{ active: partition.locked }" type="button" :aria-label="partition.locked ? `解锁${partition.name}` : `锁定${partition.name}`" :title="partition.locked ? '解锁物体' : '锁定物体'" @click.stop="toggleLock(partition.id)">
                <svg viewBox="0 0 16 16" aria-hidden="true"><rect x="3" y="7" width="10" height="7" rx="1.2"></rect><path v-if="partition.locked" d="M5 7V5a3 3 0 0 1 6 0v2"></path><path v-else d="M5 7V5a3 3 0 0 1 5.2-2"></path></svg>
              </button>
            </div>
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
            <span>先选择对象，再拖动已选对象移动 · 滚轮缩放 · 右键旋转</span>
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
          <div><div class="eyebrow">OBJECT DETAILS</div><h2>{{ selectedPartition ? selectedPartition.roomWallSide ? '机房外墙属性' : '隔板属性' : selectedRack ? '机柜属性' : '场景工具' }}</h2></div>
          <span class="more-dots">···</span>
        </div>

        <template v-if="selectedPartition">
          <div class="selected-card">
            <div class="selected-preview partition-preview"><span class="partition-glyph large"></span><span class="preview-glow"></span></div>
            <div class="selected-name"><span class="selected-type">{{ selectedPartition.roomWallSide ? 'ROOM WALL' : 'ROOM PARTITION' }}</span><strong>{{ selectedPartition.name }}</strong><small>对象 ID · {{ selectedPartition.id.slice(-6).toUpperCase() }}</small></div>
            <button v-if="!selectedPartition.roomWallSide" class="selected-lock-button" :class="{ active: selectedPartition.locked }" type="button" :aria-label="selectedPartition.locked ? '解锁隔板' : '锁定隔板'" @click="toggleLock(selectedPartition.id)">
              <svg viewBox="0 0 16 16" aria-hidden="true"><rect x="3" y="7" width="10" height="7" rx="1.2"></rect><path v-if="selectedPartition.locked" d="M5 7V5a3 3 0 0 1 6 0v2"></path><path v-else d="M5 7V5a3 3 0 0 1 5.2-2"></path></svg>
              <span>{{ selectedPartition.locked ? '解锁' : '锁定' }}</span>
            </button>
          </div>
          <section class="property-section">
            <div class="section-title"><span>{{ selectedPartition.roomWallSide ? '边界位置' : '位置坐标' }}</span><span class="edit-hint">{{ selectedPartition.roomWallSide ? '固定贴合房间边界' : '拖动隔板可快速调整' }}</span></div>
            <div v-if="selectedPartition.roomWallSide" class="size-row"><span>墙面</span><strong>{{ roomWallLabel(selectedPartition.roomWallSide) }}</strong></div>
            <div v-else class="coordinate-grid">
              <label><span>X</span><input :value="selectedPartition.x.toFixed(1)" type="number" step="0.1" :disabled="selectedPartition.locked" @change="updatePartitionPosition('x', $event)"></label>
              <label><span>Y</span><input value="0.0" type="number" disabled></label>
              <label><span>Z</span><input :value="selectedPartition.z.toFixed(1)" type="number" step="0.1" :disabled="selectedPartition.locked" @change="updatePartitionPosition('z', $event)"></label>
            </div>
          </section>
          <section class="property-section">
            <div class="section-title"><span>隔板尺寸</span><span class="unit-tag">m</span></div>
            <label class="partition-length-field"><span>长度</span><input :value="selectedPartition.length.toFixed(1)" type="number" :min="partitionMinLength(selectedPartition)" :max="partitionMaxLength(selectedPartition)" step="0.1" :disabled="selectedPartition.locked || Boolean(selectedPartition.roomWallSide)" @change="updatePartitionLength($event)"><b>m</b></label>
            <div class="size-row"><span>厚度</span><strong>{{ selectedPartition.thickness.toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>高度</span><strong>{{ room.height.toFixed(2) }} <small>m</small></strong></div>
            <button class="rotate-button" type="button" :disabled="selectedPartition.locked || Boolean(selectedPartition.roomWallSide)" @click="rotateSelectedPartition"><span>⟳</span> 旋转 90° <small>{{ selectedPartition.roomWallSide ? '边界墙固定' : selectedPartition.locked ? '已锁定' : `当前 ${selectedPartition.rotation === 0 ? '横向' : '纵向'}` }}</small></button>
          </section>
          <section class="property-section wall-appearance-section">
            <div class="section-title"><span>墙面外观</span><span class="edit-hint">材质与颜色</span></div>
            <label class="wall-material-field">
              <span>材质</span>
              <select :value="selectedPartition.wallMaterialType" aria-label="墙体材质" @change="updatePartitionMaterial($event)">
                <option value="paint">喷涂金属板</option>
                <option value="concrete">混凝土</option>
                <option value="steel">不锈钢板</option>
                <option value="brick">砖墙</option>
              </select>
            </label>
            <label class="wall-color-field">
              <span>颜色</span>
              <input :value="selectedPartition.wallColor" type="color" aria-label="墙体颜色" @input="updatePartitionColor($event)">
              <strong>{{ selectedPartition.wallColor.toUpperCase() }}</strong>
            </label>
          </section>
          <section class="partition-door-panel">
            <div class="section-title"><span>隔板门</span><span class="door-status" :class="{ installed: selectedPartition.doors.length }">{{ selectedPartition.doors.length ? `${selectedPartition.doors.length} 扇门` : '未安装' }}</span></div>
            <p>可在隔板上添加多扇门，分别调整位置和开合状态。</p>
            <button class="annotation-add" type="button" :disabled="selectedPartition.locked || selectedPartition.doors.length >= partitionMaxDoorCount(selectedPartition)" @click="addPartitionDoor"><span>＋</span> 添加一扇门</button>
            <div v-for="(door, index) in selectedPartition.doors" :key="door.id" class="partition-door-item">
              <div class="door-item-heading"><strong>门 {{ index + 1 }}</strong><span>{{ door.doorOpen ? '已打开' : '已关闭' }}</span></div>
              <div class="door-actions">
                <button class="annotation-add" type="button" @click="togglePartitionDoor(door.id)"><span>{{ door.doorOpen ? '↶' : '↗' }}</span> {{ door.doorOpen ? '关闭门扇' : '打开门扇' }}</button>
                <button class="remove-door-button" type="button" :disabled="selectedPartition.locked" @click="removePartitionDoor(door.id)">移除</button>
              </div>
              <label class="door-position-control">
                <span class="door-position-label"><span>门洞位置 <small>距左端</small></span><strong>{{ (selectedPartition.length / 2 + door.doorOffset).toFixed(2) }} m</strong></span>
                <input
                  type="range"
                  :min="partitionDoorPositionMin(selectedPartition, index)"
                  :max="partitionDoorPositionMax(selectedPartition, index)"
                  step="0.05"
                  :value="selectedPartition.length / 2 + door.doorOffset"
                  :disabled="selectedPartition.locked"
                  :aria-label="`门 ${index + 1} 距隔板左端距离`"
                  @input="updatePartitionDoorPosition(door.id, $event)"
                >
                <span class="door-position-ends"><span>左</span><span>右</span></span>
              </label>
            </div>
          </section>
          <button v-if="!selectedPartition.roomWallSide" class="danger-button" type="button" @click="removeSelectedPartition"><span>⌫</span> 删除此隔板</button>
        </template>
        <template v-else-if="selectedRack">
          <div class="selected-card">
            <div class="selected-preview"><span class="rack-glyph large"></span><span class="preview-glow"></span></div>
            <div class="selected-name"><span class="selected-type">{{ selectedRack.kind === 'imported' ? 'IMPORT MODEL' : 'SERVER RACK' }}</span><strong>{{ selectedRack.name }}</strong><small>对象 ID · {{ selectedRack.id.slice(-6).toUpperCase() }}</small></div>
            <button class="selected-lock-button" :class="{ active: selectedRack.locked }" type="button" :aria-label="selectedRack.locked ? '解锁物体' : '锁定物体'" @click="toggleLock(selectedRack.id)">
              <svg viewBox="0 0 16 16" aria-hidden="true"><rect x="3" y="7" width="10" height="7" rx="1.2"></rect><path v-if="selectedRack.locked" d="M5 7V5a3 3 0 0 1 6 0v2"></path><path v-else d="M5 7V5a3 3 0 0 1 5.2-2"></path></svg>
              <span>{{ selectedRack.locked ? '解锁' : '锁定' }}</span>
            </button>
          </div>
          <section class="property-section">
            <div class="section-title"><span>位置坐标</span><span class="edit-hint">拖动模型可快速调整</span></div>
            <div class="coordinate-grid">
              <label><span>X</span><input :value="selectedRack.x.toFixed(1)" type="number" step="0.1" :disabled="selectedRack.locked" @change="updatePosition('x', $event)"></label>
              <label><span>Y</span><input value="0.0" type="number" disabled></label>
              <label><span>Z</span><input :value="selectedRack.z.toFixed(1)" type="number" step="0.1" :disabled="selectedRack.locked" @change="updatePosition('z', $event)"></label>
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

        <details class="lighting-panel">
          <summary><span>场景光照</span><small>{{ lighting.keyEnabled ? '主光开启' : '主光关闭' }}</small></summary>
          <div class="lighting-controls">
            <label class="lighting-range">
              <span>环境光强度 <strong>{{ lighting.ambientIntensity.toFixed(1) }}</strong></span>
              <input type="range" min="0" max="4" step="0.1" :value="lighting.ambientIntensity" aria-label="环境光强度" @input="updateLightingNumber('ambientIntensity', $event)">
            </label>
            <label class="lighting-color">
              <span>环境光颜色</span>
              <input type="color" :value="lighting.ambientColor" aria-label="环境光颜色" @input="updateLightingColor('ambientColor', $event)">
            </label>

            <div class="lighting-subheading">主光源</div>
            <label class="lighting-toggle"><input type="checkbox" :checked="lighting.keyEnabled" @change="updateLightingToggle('keyEnabled', $event)"><span>启用主光</span></label>
            <label class="lighting-range">
              <span>强度 <strong>{{ lighting.keyIntensity.toFixed(1) }}</strong></span>
              <input type="range" min="0" max="5" step="0.1" :value="lighting.keyIntensity" :disabled="!lighting.keyEnabled" aria-label="主光源强度" @input="updateLightingNumber('keyIntensity', $event)">
            </label>
            <label class="lighting-color">
              <span>颜色</span>
              <input type="color" :value="lighting.keyColor" :disabled="!lighting.keyEnabled" aria-label="主光源颜色" @input="updateLightingColor('keyColor', $event)">
            </label>
            <label class="lighting-range">
              <span>方位角 <strong>{{ lighting.keyAzimuth }}°</strong></span>
              <input type="range" min="-180" max="180" step="1" :value="lighting.keyAzimuth" :disabled="!lighting.keyEnabled" aria-label="主光源方位角" @input="updateLightingNumber('keyAzimuth', $event)">
            </label>
            <label class="lighting-range">
              <span>高度角 <strong>{{ lighting.keyElevation }}°</strong></span>
              <input type="range" min="5" max="85" step="1" :value="lighting.keyElevation" :disabled="!lighting.keyEnabled" aria-label="主光源高度角" @input="updateLightingNumber('keyElevation', $event)">
            </label>

            <div class="lighting-subheading">补光</div>
            <label class="lighting-toggle"><input type="checkbox" :checked="lighting.fillEnabled" @change="updateLightingToggle('fillEnabled', $event)"><span>启用补光</span></label>
            <label class="lighting-range">
              <span>强度 <strong>{{ lighting.fillIntensity.toFixed(1) }}</strong></span>
              <input type="range" min="0" max="3" step="0.1" :value="lighting.fillIntensity" :disabled="!lighting.fillEnabled" aria-label="补光强度" @input="updateLightingNumber('fillIntensity', $event)">
            </label>
            <label class="lighting-color">
              <span>颜色</span>
              <input type="color" :value="lighting.fillColor" :disabled="!lighting.fillEnabled" aria-label="补光颜色" @input="updateLightingColor('fillColor', $event)">
            </label>

            <div class="lighting-subheading">天花板灯具</div>
            <label class="lighting-toggle"><input type="checkbox" :checked="ceilingLighting.enabled" @change="updateCeilingToggle($event)"><span>开启顶灯</span></label>
            <label class="lighting-toggle"><input type="checkbox" :checked="ceilingLighting.showFixtures" @change="updateCeilingFixtureVisibility($event)"><span>显示灯具模型</span></label>
            <label class="lighting-range">
              <span>灯具间距 <strong>{{ ceilingLighting.spacing.toFixed(1) }} m · {{ ceilingFixtureCount }} 盏</strong></span>
              <input type="range" min="2" max="8" step="0.5" :value="ceilingLighting.spacing" aria-label="天花板灯具间距" @input="updateCeilingNumber('spacing', $event)">
            </label>
            <label class="lighting-range">
              <span>顶灯亮度 <strong>{{ ceilingLighting.intensity.toFixed(1) }}</strong></span>
              <input type="range" min="0" max="20" step="0.5" :value="ceilingLighting.intensity" :disabled="!ceilingLighting.enabled" aria-label="顶灯亮度" @input="updateCeilingNumber('intensity', $event)">
            </label>
            <label class="lighting-range">
              <span>色温 <strong>{{ ceilingLighting.colorTemperature }} K</strong></span>
              <input type="range" min="3000" max="6500" step="100" :value="ceilingLighting.colorTemperature" :disabled="!ceilingLighting.enabled" aria-label="顶灯色温" @input="updateCeilingNumber('colorTemperature', $event)">
            </label>
            <p class="lighting-help">灯板按房间尺寸自动均布；面光不单独投射阴影。</p>

            <label class="lighting-toggle"><input type="checkbox" :checked="lighting.shadowsEnabled" @change="updateLightingToggle('shadowsEnabled', $event)"><span>启用阴影</span></label>
            <label class="lighting-range">
              <span>曝光度 <strong>{{ lighting.exposure.toFixed(2) }}</strong></span>
              <input type="range" min="0.6" max="1.6" step="0.05" :value="lighting.exposure" aria-label="场景曝光度" @input="updateLightingNumber('exposure', $event)">
            </label>
          </div>
        </details>

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
import { computed, markRaw, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { CSS2DObject, CSS2DRenderer } from 'three/addons/renderers/CSS2DRenderer.js';
import { GLTFLoader, type GLTF } from 'three/addons/loaders/GLTFLoader.js';
import { GLTFExporter } from 'three/addons/exporters/GLTFExporter.js';
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';

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
  locked: boolean;
};

type RoomWallSide = 'north' | 'south' | 'east' | 'west';
type WallMaterialType = 'paint' | 'concrete' | 'steel' | 'brick';

type LightingSettings = {
  ambientIntensity: number;
  ambientColor: string;
  keyEnabled: boolean;
  keyIntensity: number;
  keyColor: string;
  keyAzimuth: number;
  keyElevation: number;
  fillEnabled: boolean;
  fillIntensity: number;
  fillColor: string;
  shadowsEnabled: boolean;
  exposure: number;
};

type CeilingLightingSettings = {
  enabled: boolean;
  showFixtures?: boolean;
  spacing: number;
  intensity: number;
  colorTemperature: number;
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
  locked: boolean;
  roomWallSide: RoomWallSide | null;
  wallMaterialType: WallMaterialType;
  wallColor: string;
  doors: PartitionDoor[];
};

type PartitionDoor = {
  id: string;
  doorOpen: boolean;
  doorOffset: number;
};

type Annotation = {
  id: string;
  itemId: string;
  title: string;
  description: string;
  label: CSS2DObject;
};

type SceneRack = {
  id: string;
  name: string;
  kind: RackItem['kind'];
  width: number;
  depth: number;
  height: number;
  x: number;
  z: number;
  locked: boolean;
  modelAsset?: Record<string, unknown>;
};

type ScenePartition = {
  id: string;
  name: string;
  length: number;
  thickness: number;
  rotation: 0 | 1;
  x: number;
  z: number;
  fullSpan: boolean;
  locked: boolean;
  roomWallSide: RoomWallSide | null;
  wallMaterialType: WallMaterialType;
  wallColor: string;
  doors: PartitionDoor[];
};

type SceneDocument = {
  format: 'nexus-space-studio';
  version: 1;
  room: { width: number; length: number; height: number };
  racks: SceneRack[];
  partitions: ScenePartition[];
  annotations: Array<Omit<Annotation, 'label'>>;
  lighting?: LightingSettings;
  ceilingLighting?: CeilingLightingSettings;
};

const viewport = ref<HTMLDivElement | null>(null);
const fileInput = ref<HTMLInputElement | null>(null);
const sceneFileInput = ref<HTMLInputElement | null>(null);
const room = reactive({ width: 12, length: 9, height: 4 });
const roomDraft = reactive({ width: 12, length: 9, height: 4 });
const defaultLightingSettings: LightingSettings = {
  ambientIntensity: 2.1,
  ambientColor: '#c9e5ff',
  keyEnabled: true,
  keyIntensity: 3.2,
  keyColor: '#e4f2ff',
  keyAzimuth: 131,
  keyElevation: 55,
  fillEnabled: true,
  fillIntensity: 1.25,
  fillColor: '#61b9cf',
  shadowsEnabled: true,
  exposure: 1.15,
};
const lighting = reactive({ ...defaultLightingSettings });
const defaultCeilingLightingSettings: Required<CeilingLightingSettings> = {
  enabled: true,
  showFixtures: true,
  spacing: 3.5,
  intensity: 6,
  colorTemperature: 4000,
};
const ceilingLighting = reactive({ ...defaultCeilingLightingSettings });
const ceilingFixtureCount = computed(() => (
  Math.max(1, Math.floor(room.width / ceilingLighting.spacing))
  * Math.max(1, Math.floor(room.length / ceilingLighting.spacing))
));
const racks = ref<RackItem[]>([]);
const partitions = ref<PartitionItem[]>([]);
const annotations = ref<Annotation[]>([]);
const selectedId = ref<string | null>(null);
const annotationTitle = ref('');
const annotationDescription = ref('');
const importError = ref('');
const assetStatus = ref('CC0 模型加载中');
const assetError = ref('');
const saveStatus = ref('尚未保存');
const sceneFileError = ref('');

const selectedRack = computed(() => racks.value.find((rack) => rack.id === selectedId.value) ?? null);
const selectedPartition = computed(() => partitions.value.find((partition) => partition.id === selectedId.value) ?? null);
watch(selectedId, updateSelectionOutline);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let labelRenderer: CSS2DRenderer;
let controls: OrbitControls;
let roomGroup: THREE.Group;
let roomOutline: THREE.Group;
let floorPlane: THREE.Mesh;
let hemisphereLight: THREE.HemisphereLight | null = null;
let keyLight: THREE.DirectionalLight | null = null;
let fillLight: THREE.DirectionalLight | null = null;
let ceilingLightPanels: THREE.InstancedMesh | null = null;
let ceilingAreaLights: THREE.RectAreaLight[] = [];
let selectionOutline: THREE.BoxHelper | null = null;
let freeRackPrototype: THREE.Group | null = null;
let resizeObserver: ResizeObserver;
let animationFrame = 0;
let activeDrag: { kind: 'rack' | 'partition'; id: string } | null = null;
let floorPlaneForDrag: THREE.Plane;
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const gltfLoader = new GLTFLoader();
const gltfExporter = new GLTFExporter();
const localSceneStorageKey = 'nexus-space-studio:scene:v1';
const wallMaterialPresets: Record<WallMaterialType, { roughness: number; metalness: number }> = {
  paint: { roughness: 0.72, metalness: 0.12 },
  concrete: { roughness: 0.94, metalness: 0.02 },
  steel: { roughness: 0.38, metalness: 0.64 },
  brick: { roughness: 0.9, metalness: 0 },
};

function makeMaterial(color: number, roughness = 0.7, metalness = 0) {
  return new THREE.MeshStandardMaterial({ color, roughness, metalness });
}

function toggleLock(id: string) {
  const rack = racks.value.find((item) => item.id === id);
  const partition = partitions.value.find((item) => item.id === id);
  const item = rack ?? partition;
  if (!item) return;
  selectedId.value = id;
  item.locked = !item.locked;
  markSceneDirty();
  if (item.locked && activeDrag?.id === id) {
    activeDrag = null;
    controls.enabled = true;
  }
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
  ceilingLightPanels = null;
  ceilingAreaLights = [];
  createCeilingFixtures(roomGroup);
  scene.add(roomGroup, roomOutline);
}

function evenlySpacedPositions(size: number, spacing: number) {
  const count = Math.max(1, Math.floor(size / spacing));
  return Array.from({ length: count }, (_, index) => -size / 2 + ((index + 1) * size) / (count + 1));
}

function colorTemperatureHex(kelvin: number) {
  const temperature = kelvin / 100;
  const red = temperature <= 66 ? 255 : 329.698727446 * (temperature - 60) ** -0.1332047592;
  const green = temperature <= 66
    ? 99.4708025861 * Math.log(temperature) - 161.1195681661
    : 288.1221695283 * (temperature - 60) ** -0.0755148492;
  const blue = temperature >= 66
    ? 255
    : temperature <= 19
      ? 0
      : 138.5177312231 * Math.log(temperature - 10) - 305.0447927307;
  const hex = [red, green, blue]
    .map((channel) => Math.round(THREE.MathUtils.clamp(channel, 0, 255)).toString(16).padStart(2, '0'))
    .join('');
  return `#${hex}`;
}

function createCeilingFixtures(parent: THREE.Group) {
  const xPositions = evenlySpacedPositions(room.width, ceilingLighting.spacing);
  const zPositions = evenlySpacedPositions(room.length, ceilingLighting.spacing);
  const lampColor = colorTemperatureHex(ceilingLighting.colorTemperature);
  const panelMaterial = new THREE.MeshStandardMaterial({
    color: 0xb9c4c9,
    emissive: lampColor,
    emissiveIntensity: 1.2,
    roughness: 0.32,
    metalness: 0.08,
  });
  ceilingLightPanels = new THREE.InstancedMesh(
    new THREE.BoxGeometry(0.82, 0.055, 1.25),
    panelMaterial,
    xPositions.length * zPositions.length,
  );
  ceilingLightPanels.name = 'ceiling-light-panels';
  ceilingLightPanels.castShadow = false;
  ceilingLightPanels.receiveShadow = false;
  const transform = new THREE.Object3D();
  let panelIndex = 0;
  for (const z of zPositions) {
    for (const x of xPositions) {
      transform.position.set(x, room.height - 0.12, z);
      transform.updateMatrix();
      ceilingLightPanels.setMatrixAt(panelIndex, transform.matrix);
      panelIndex += 1;
    }
  }
  ceilingLightPanels.instanceMatrix.needsUpdate = true;
  parent.add(ceilingLightPanels);

  const areaLightCount = Math.min(zPositions.length, 4);
  for (let index = 0; index < areaLightCount; index += 1) {
    const rowIndex = areaLightCount === 1
      ? Math.floor((zPositions.length - 1) / 2)
      : Math.round((index * (zPositions.length - 1)) / (areaLightCount - 1));
    const areaLight = new THREE.RectAreaLight(
      lampColor,
      ceilingLighting.intensity,
      room.width * 0.8,
      Math.min(1.25, ceilingLighting.spacing * 0.55),
    );
    areaLight.position.set(0, room.height - 0.17, zPositions[rowIndex]);
    areaLight.rotation.x = -Math.PI / 2;
    parent.add(areaLight);
    ceilingAreaLights.push(areaLight);
  }
  applyCeilingLighting();
}

function rebuildCeilingFixtures() {
  if (!roomGroup) return;
  if (ceilingLightPanels) {
    roomGroup.remove(ceilingLightPanels);
    ceilingLightPanels.geometry.dispose();
    const materials = Array.isArray(ceilingLightPanels.material)
      ? ceilingLightPanels.material
      : [ceilingLightPanels.material];
    for (const material of materials) material.dispose();
  }
  for (const areaLight of ceilingAreaLights) roomGroup.remove(areaLight);
  ceilingLightPanels = null;
  ceilingAreaLights = [];
  createCeilingFixtures(roomGroup);
}

function roomWallLabel(side: RoomWallSide) {
  const labels: Record<RoomWallSide, string> = {
    north: '北墙',
    south: '南墙',
    east: '东墙',
    west: '西墙',
  };
  return labels[side];
}

function createRoomWalls() {
  const definitions: RoomWallSide[] = ['north', 'south', 'east', 'west'];
  for (const side of definitions) {
    const horizontal = side === 'north' || side === 'south';
    const id = createId();
    const partition: PartitionItem = reactive({
      id,
      name: `机房外墙 - ${roomWallLabel(side)}`,
      group: markRaw(new THREE.Group()),
      length: horizontal ? room.width : room.length,
      thickness: 0.14,
      rotation: horizontal ? 0 : 1,
      x: 0,
      z: 0,
      fullSpan: true,
      locked: false,
      roomWallSide: side,
      wallMaterialType: 'paint',
      wallColor: '#78919c',
      doors: [],
    });
    partition.group.name = partition.name;
    partition.group.userData.itemId = id;
    scene.add(partition.group);
    partitions.value.push(partition);
    clampPartitionToRoom(partition);
    rebuildPartition(partition);
  }
}

function buildRack(name: string, id: string) {
  const group = new THREE.Group();
  group.name = name;
  if (freeRackPrototype) {
    const model = freeRackPrototype.clone(true);
    group.userData.sharedAssets = true;
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
    locked: false,
  });
  selectedId.value = id;
  markSceneDirty();
}

function addPartition() {
  if (!scene) return;
  const id = createId();
  let index = partitions.value.filter((partition) => !partition.roomWallSide).length + 1;
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
    locked: false,
    roomWallSide: null,
    wallMaterialType: 'paint',
    wallColor: '#78919c',
    doors: [],
  });
  partition.group.name = partition.name;
  partition.group.userData.itemId = id;
  scene.add(partition.group);
  partitions.value.push(partition);
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
  selectedId.value = id;
  markSceneDirty();
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
  clampPartitionDoorPositions(partition);
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
  const wallPreset = wallMaterialPresets[partition.wallMaterialType];
  const wallMaterial = new THREE.MeshStandardMaterial({
    color: partition.wallColor,
    roughness: wallPreset.roughness,
    metalness: wallPreset.metalness,
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
  const openingWidth = partitionDoorWidth(partition);
  const doorHeight = Math.min(2.2, wallHeight - 0.12);
  const doorHeaderHeight = 0.09;
  const capHeight = 0.045;
  const wallBodyHeight = wallHeight - capHeight;
  const opensIntoRoom = partition.roomWallSide === null
    || partition.roomWallSide === 'north'
    || partition.roomWallSide === 'west';

  if (!partition.doors.length) {
    makeBox(partition.group, length, wallBodyHeight, partition.thickness, wallMaterial, 0, wallBodyHeight / 2, 0);
  } else {
    let wallCursor = -halfLength;
    for (const door of partition.doors) {
      const doorCenter = door.doorOffset;
      const doorLeft = doorCenter - openingWidth / 2;
      const doorRight = doorCenter + openingWidth / 2;
      const wallSegmentLength = doorLeft - wallCursor;
      makeBox(
        partition.group,
        wallSegmentLength,
        wallBodyHeight,
        partition.thickness,
        wallMaterial,
        (wallCursor + doorLeft) / 2,
        wallBodyHeight / 2,
        0,
      );
      const upperWallHeight = wallBodyHeight - doorHeight - doorHeaderHeight;
      makeBox(
        partition.group,
        openingWidth,
        upperWallHeight,
        partition.thickness,
        wallMaterial,
        doorCenter,
        doorHeight + doorHeaderHeight + upperWallHeight / 2,
        0,
      );
      makeBox(partition.group, openingWidth + partition.thickness, doorHeaderHeight, partition.thickness + 0.025, trimMaterial, doorCenter, doorHeight + doorHeaderHeight / 2, 0);
      for (const side of [-1, 1]) {
        makeBox(partition.group, 0.06, doorHeight, partition.thickness + 0.035, trimMaterial, doorCenter + side * openingWidth / 2, doorHeight / 2, 0);
      }

      const hinge = new THREE.Group();
      hinge.position.set(doorLeft + 0.035, 0, (opensIntoRoom ? 1 : -1) * (halfThickness + 0.025));
      hinge.name = `Partition door hinge ${door.id}`;
      const leafWidth = openingWidth - 0.07;
      makeBox(hinge, leafWidth, doorHeight - 0.035, 0.055, doorMaterial, leafWidth / 2, (doorHeight - 0.035) / 2, 0);
      makeBox(hinge, 0.025, 0.12, 0.025, trimMaterial, leafWidth - 0.1, 1.02, 0.04);
      makeBox(hinge, 0.018, doorHeight - 0.18, 0.018, markerMaterial, 0.08, (doorHeight - 0.18) / 2, 0.035);
      hinge.rotation.y = door.doorOpen ? (opensIntoRoom ? -1 : 1) * Math.PI / 2 : 0;
      partition.group.add(hinge);
      wallCursor = doorRight;
    }

    const finalWallLength = halfLength - wallCursor;
    makeBox(partition.group, finalWallLength, wallBodyHeight, partition.thickness, wallMaterial, (wallCursor + halfLength) / 2, wallBodyHeight / 2, 0);
  }

  const baseMaterial = makeMaterial(0x617c83, 0.56, 0.46);
  makeBox(partition.group, length, 0.07, partition.thickness + 0.08, baseMaterial, 0, 0.035, 0);
  const capMaterial = makeMaterial(0x9bb7b7, 0.42, 0.36);
  makeBox(partition.group, length, capHeight, partition.thickness + 0.025, capMaterial, 0, wallHeight - capHeight / 2, 0);

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

function partitionMinLength(partition: PartitionItem) {
  return Math.max(2, 0.15 + partition.doors.length * 0.75);
}

function partitionDoorWidth(partition: PartitionItem, doorCount = Math.max(1, partition.doors.length)) {
  const edgeMargin = 0.15;
  const gap = 0.15;
  return Math.max(0.01, Math.min(0.95, (partition.length - edgeMargin * 2 - gap * (doorCount - 1)) / doorCount));
}

function partitionMaxDoorCount(partition: PartitionItem) {
  const minimumDoorWidth = 0.6;
  const edgeMargin = 0.15;
  const gap = 0.15;
  return Math.max(1, Math.floor((partition.length - edgeMargin * 2 + gap) / (minimumDoorWidth + gap)));
}

function partitionDoorPositionMin(partition: PartitionItem, index: number) {
  const width = partitionDoorWidth(partition);
  const previousDoor = partition.doors[index - 1];
  return previousDoor
    ? partition.length / 2 + previousDoor.doorOffset + width + 0.15
    : 0.15 + width / 2;
}

function partitionDoorPositionMax(partition: PartitionItem, index: number) {
  const width = partitionDoorWidth(partition);
  const nextDoor = partition.doors[index + 1];
  return nextDoor
    ? partition.length / 2 + nextDoor.doorOffset - width - 0.15
    : partition.length - 0.15 - width / 2;
}

function updatePartitionDoorPosition(doorId: string, event: Event) {
  const partition = selectedPartition.value;
  const door = partition?.doors.find((item) => item.id === doorId);
  const input = event.target as HTMLInputElement;
  const index = partition?.doors.findIndex((item) => item.id === doorId) ?? -1;
  if (!partition || !door || index < 0 || partition.locked || !Number.isFinite(input.valueAsNumber)) return;
  const centerFromLeft = clampNumber(input.valueAsNumber, partitionDoorPositionMin(partition, index), partitionDoorPositionMax(partition, index), partition.length / 2);
  door.doorOffset = centerFromLeft - partition.length / 2;
  rebuildPartition(partition);
  markSceneDirty();
}

function clampPartitionDoorPositions(partition: PartitionItem) {
  if (!partition.doors.length) return;
  partition.doors.sort((a, b) => a.doorOffset - b.doorOffset);
  const spacing = partitionDoorWidth(partition) + 0.15;
  const minimum = 0.15 + partitionDoorWidth(partition) / 2;
  const maximum = partition.length - minimum;
  let previousCenter = Number.NEGATIVE_INFINITY;
  partition.doors.forEach((door, index) => {
    const lowerBound = Math.max(minimum + index * spacing, previousCenter + spacing);
    const upperBound = maximum - (partition.doors.length - index - 1) * spacing;
    const center = clampNumber(partition.length / 2 + door.doorOffset, lowerBound, upperBound, partition.length / 2);
    door.doorOffset = center - partition.length / 2;
    previousCenter = center;
  });
}

function clampPartitionToRoom(partition: PartitionItem) {
  if (partition.roomWallSide) {
    const horizontal = partition.roomWallSide === 'north' || partition.roomWallSide === 'south';
    partition.rotation = horizontal ? 0 : 1;
    partition.length = partitionMaxLength(partition);
    partition.x = partition.roomWallSide === 'west' ? -room.width / 2 : partition.roomWallSide === 'east' ? room.width / 2 : 0;
    partition.z = partition.roomWallSide === 'north' ? -room.length / 2 : partition.roomWallSide === 'south' ? room.length / 2 : 0;
    partition.group.position.set(partition.x, 0, partition.z);
    return;
  }
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
  if (!partition || partition.roomWallSide || partition.locked || !Number.isFinite(input.valueAsNumber)) return;
  partition[axis] = input.valueAsNumber;
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
  markSceneDirty();
}

function updatePartitionLength(event: Event) {
  const partition = selectedPartition.value;
  const input = event.target as HTMLInputElement;
  if (!partition || partition.roomWallSide || partition.locked || !Number.isFinite(input.valueAsNumber)) return;
  partition.length = clampNumber(input.valueAsNumber, partitionMinLength(partition), partitionMaxLength(partition), partition.length);
  partition.fullSpan = false;
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
  markSceneDirty();
}

function updatePartitionMaterial(event: Event) {
  const partition = selectedPartition.value;
  const input = event.target as HTMLSelectElement;
  if (!partition || !isWallMaterialType(input.value)) return;
  partition.wallMaterialType = input.value;
  rebuildPartition(partition);
  markSceneDirty();
}

function isWallMaterialType(value: string): value is WallMaterialType {
  return value === 'paint' || value === 'concrete' || value === 'steel' || value === 'brick';
}

function updatePartitionColor(event: Event) {
  const partition = selectedPartition.value;
  const input = event.target as HTMLInputElement;
  if (!partition || !/^#[\da-f]{6}$/i.test(input.value)) return;
  partition.wallColor = input.value;
  rebuildPartition(partition);
  markSceneDirty();
}

function rotateSelectedPartition() {
  const partition = selectedPartition.value;
  if (!partition || partition.roomWallSide || partition.locked) return;
  partition.rotation = partition.rotation === 0 ? 1 : 0;
  partition.length = partition.fullSpan
    ? partitionMaxLength(partition)
    : Math.min(partition.length, partitionMaxLength(partition));
  clampPartitionToRoom(partition);
  rebuildPartition(partition);
  markSceneDirty();
}

function addPartitionDoor() {
  const partition = selectedPartition.value;
  if (!partition || partition.locked || partition.doors.length >= partitionMaxDoorCount(partition)) return;
  const nextCount = partition.doors.length + 1;
  const width = partitionDoorWidth(partition, nextCount);
  const gap = 0.15;
  const minCenter = 0.15 + width / 2;
  const maxCenter = partition.length - minCenter;
  const spacing = width + gap;
  const candidates = Array.from(
    { length: Math.floor((maxCenter - minCenter) / 0.05) + 1 },
    (_, index) => minCenter + index * 0.05,
  ).sort((a, b) => Math.abs(a - partition.length / 2) - Math.abs(b - partition.length / 2));
  const centerFromLeft = candidates.find((candidate) => {
    return partition.doors.every((door) => Math.abs(partition.length / 2 + door.doorOffset - candidate) >= spacing - 0.001);
  });
  if (centerFromLeft === undefined) return;
  partition.doors.push({
    id: createId(),
    doorOpen: false,
    doorOffset: centerFromLeft - partition.length / 2,
  });
  partition.doors.sort((a, b) => a.doorOffset - b.doorOffset);
  rebuildPartition(partition);
  markSceneDirty();
}

function togglePartitionDoor(doorId: string) {
  const partition = selectedPartition.value;
  const door = partition?.doors.find((item) => item.id === doorId);
  if (!partition || !door) return;
  door.doorOpen = !door.doorOpen;
  rebuildPartition(partition);
  markSceneDirty();
}

function removePartitionDoor(doorId: string) {
  const partition = selectedPartition.value;
  if (!partition || partition.locked || !partition.doors.some((door) => door.id === doorId)) return;
  partition.doors = partition.doors.filter((door) => door.id !== doorId);
  rebuildPartition(partition);
  markSceneDirty();
}

function removeSelectedPartition() {
  const partition = selectedPartition.value;
  if (!partition || partition.roomWallSide || !scene) return;
  scene.remove(partition.group);
  disposeGroup(partition.group);
  partitions.value = partitions.value.filter((item) => item.id !== partition.id);
  selectedId.value = partitions.value.at(-1)?.id ?? racks.value.at(-1)?.id ?? null;
  markSceneDirty();
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
  markSceneDirty();
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

function updateSelectionOutline() {
  if (!scene) return;
  if (selectionOutline) {
    scene.remove(selectionOutline);
    selectionOutline.geometry.dispose();
    selectionOutline.material.dispose();
    selectionOutline = null;
  }
  const selectedItem = racks.value.find((rack) => rack.id === selectedId.value)
    ?? partitions.value.find((partition) => partition.id === selectedId.value);
  if (!selectedItem) return;
  selectionOutline = new THREE.BoxHelper(selectedItem.group, 0x72e2d5);
  selectionOutline.material.transparent = true;
  selectionOutline.material.opacity = 0.95;
  selectionOutline.renderOrder = 2;
  scene.add(selectionOutline);
}

function removeSelected() {
  if (!selectedRack.value || !scene) return;
  const id = selectedRack.value.id;
  scene.remove(selectedRack.value.group);
  disposeGroup(selectedRack.value.group);
  racks.value = racks.value.filter((rack) => rack.id !== id);
  for (const annotation of annotations.value.filter((item) => item.itemId === id)) removeAnnotation(annotation.id);
  selectedId.value = racks.value.at(-1)?.id ?? null;
  markSceneDirty();
}

function updatePosition(axis: 'x' | 'z', event: Event) {
  const rack = selectedRack.value;
  const input = event.target as HTMLInputElement;
  if (!rack || rack.locked || !Number.isFinite(input.valueAsNumber)) return;
  rack.group.position[axis] = input.valueAsNumber;
  clampRackToRoom(rack);
  markSceneDirty();
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
        locked: false,
      });
      selectedId.value = id;
      markSceneDirty();
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
  attachAnnotation(rack, createId(), title, annotationDescription.value.trim());
  annotationTitle.value = '';
  annotationDescription.value = '';
  markSceneDirty();
}

function attachAnnotation(rack: RackItem, id: string, title: string, description: string) {
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
  if (description) {
    const descriptionElement = document.createElement('small');
    descriptionElement.textContent = description;
    content.appendChild(descriptionElement);
  }
  element.append(pointerMark, content);
  const label = markRaw(new CSS2DObject(element));
  label.position.set(0, rack.height + 0.36, 0);
  rack.group.add(label);
  annotations.value.push({
    id,
    itemId: rack.id,
    title,
    description,
    label,
  });
}

function removeAnnotation(id: string) {
  const annotation = annotations.value.find((item) => item.id === id);
  if (!annotation) return;
  annotation.label.parent?.remove(annotation.label);
  annotations.value = annotations.value.filter((item) => item.id !== id);
  markSceneDirty();
}

function createId() {
  return globalThis.crypto?.randomUUID?.() ?? `item-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

function markSceneDirty() {
  saveStatus.value = '未保存';
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value);
}

function isRoomWallSide(value: unknown): value is RoomWallSide {
  return value === 'north' || value === 'south' || value === 'east' || value === 'west';
}

function isSceneDocument(value: unknown): value is SceneDocument {
  if (!isRecord(value) || value.format !== 'nexus-space-studio' || value.version !== 1) return false;
  if (!isRecord(value.room) || !Array.isArray(value.racks) || !Array.isArray(value.partitions) || !Array.isArray(value.annotations)) return false;
  if (value.lighting !== undefined && !isLightingSettings(value.lighting)) return false;
  if (value.ceilingLighting !== undefined && !isCeilingLightingSettings(value.ceilingLighting)) return false;
  const { width, length, height } = value.room;
  if (!isFiniteInRange(width, 4, 40) || !isFiniteInRange(length, 4, 40) || !isFiniteInRange(height, 2.5, 12)) return false;
  if (!value.racks.every(isSceneRack) || !value.partitions.every(isScenePartition) || !value.annotations.every(isSceneAnnotation)) return false;
  const roomWallSides = value.partitions.flatMap((partition) => isRecord(partition) && isRoomWallSide(partition.roomWallSide) ? [partition.roomWallSide] : []);
  if (roomWallSides.length !== 4 || new Set(roomWallSides).size !== 4) return false;
  const rackIds = value.racks.map((rack) => rack.id);
  const objectIds = [...rackIds, ...value.partitions.map((partition) => partition.id)];
  if (new Set(objectIds).size !== objectIds.length) return false;
  const annotationIds = value.annotations.map((annotation) => annotation.id);
  if (new Set(annotationIds).size !== annotationIds.length) return false;
  return value.annotations.every((annotation) => rackIds.includes(annotation.itemId));
}

function isLightingSettings(value: unknown): value is LightingSettings {
  return isRecord(value)
    && isFiniteInRange(value.ambientIntensity, 0, 4)
    && typeof value.ambientColor === 'string' && /^#[\da-f]{6}$/i.test(value.ambientColor)
    && typeof value.keyEnabled === 'boolean'
    && isFiniteInRange(value.keyIntensity, 0, 5)
    && typeof value.keyColor === 'string' && /^#[\da-f]{6}$/i.test(value.keyColor)
    && isFiniteInRange(value.keyAzimuth, -180, 180)
    && isFiniteInRange(value.keyElevation, 5, 85)
    && typeof value.fillEnabled === 'boolean'
    && isFiniteInRange(value.fillIntensity, 0, 3)
    && typeof value.fillColor === 'string' && /^#[\da-f]{6}$/i.test(value.fillColor)
    && typeof value.shadowsEnabled === 'boolean'
    && isFiniteInRange(value.exposure, 0.6, 1.6);
}

function isCeilingLightingSettings(value: unknown): value is CeilingLightingSettings {
  return isRecord(value)
    && typeof value.enabled === 'boolean'
    && (value.showFixtures === undefined || typeof value.showFixtures === 'boolean')
    && isFiniteInRange(value.spacing, 2, 8)
    && isFiniteInRange(value.intensity, 0, 20)
    && isFiniteInRange(value.colorTemperature, 3000, 6500);
}

function isSceneRack(value: unknown): value is SceneRack {
  if (!isRecord(value)) return false;
  if (typeof value.id !== 'string' || typeof value.name !== 'string' || value.name.length > 128) return false;
  if (value.kind !== 'rack' && value.kind !== 'imported') return false;
  if (!isFiniteInRange(value.width, 0.01, 40) || !isFiniteInRange(value.depth, 0.01, 40) || !isFiniteInRange(value.height, 0.01, 20)) return false;
  if (!isFiniteInRange(value.x, -100, 100) || !isFiniteInRange(value.z, -100, 100) || typeof value.locked !== 'boolean') return false;
  return value.kind !== 'imported' || isRecord(value.modelAsset);
}

function isScenePartition(value: unknown): value is ScenePartition {
  if (!isRecord(value) || typeof value.id !== 'string' || typeof value.name !== 'string' || value.name.length > 128) return false;
  if (!isFiniteInRange(value.length, 2, 40) || !isFiniteInRange(value.thickness, 0.01, 1)) return false;
  if ((value.rotation !== 0 && value.rotation !== 1) || !isFiniteInRange(value.x, -100, 100) || !isFiniteInRange(value.z, -100, 100)) return false;
  if (typeof value.fullSpan !== 'boolean' || typeof value.locked !== 'boolean') return false;
  if (value.roomWallSide !== null && !isRoomWallSide(value.roomWallSide)) return false;
  if (!isWallMaterialType(String(value.wallMaterialType)) || typeof value.wallColor !== 'string' || !/^#[\da-f]{6}$/i.test(value.wallColor)) return false;
  return Array.isArray(value.doors) && value.doors.every((door) => {
    return isRecord(door)
      && typeof door.id === 'string'
      && typeof door.doorOpen === 'boolean'
      && isFiniteInRange(door.doorOffset, -40, 40);
  });
}

function isSceneAnnotation(value: unknown): value is Omit<Annotation, 'label'> {
  return isRecord(value)
    && typeof value.id === 'string'
    && typeof value.itemId === 'string'
    && typeof value.title === 'string'
    && value.title.length <= 256
    && typeof value.description === 'string'
    && value.description.length <= 4000;
}

function isFiniteInRange(value: unknown, min: number, max: number): value is number {
  return typeof value === 'number' && Number.isFinite(value) && value >= min && value <= max;
}

type LightingNumericKey = 'ambientIntensity' | 'keyIntensity' | 'keyAzimuth' | 'keyElevation' | 'fillIntensity' | 'exposure';
type LightingColorKey = 'ambientColor' | 'keyColor' | 'fillColor';
type LightingToggleKey = 'keyEnabled' | 'fillEnabled' | 'shadowsEnabled';
type CeilingLightingNumericKey = 'spacing' | 'intensity' | 'colorTemperature';

const lightingRanges: Record<LightingNumericKey, { min: number; max: number }> = {
  ambientIntensity: { min: 0, max: 4 },
  keyIntensity: { min: 0, max: 5 },
  keyAzimuth: { min: -180, max: 180 },
  keyElevation: { min: 5, max: 85 },
  fillIntensity: { min: 0, max: 3 },
  exposure: { min: 0.6, max: 1.6 },
};

function updateLightingNumber(key: LightingNumericKey, event: Event) {
  const input = event.currentTarget as HTMLInputElement;
  const { min, max } = lightingRanges[key];
  if (!Number.isFinite(input.valueAsNumber)) return;
  lighting[key] = THREE.MathUtils.clamp(input.valueAsNumber, min, max);
  applyLightingSettings();
  markSceneDirty();
}

function updateLightingColor(key: LightingColorKey, event: Event) {
  lighting[key] = (event.currentTarget as HTMLInputElement).value;
  applyLightingSettings();
  markSceneDirty();
}

function updateLightingToggle(key: LightingToggleKey, event: Event) {
  lighting[key] = (event.currentTarget as HTMLInputElement).checked;
  applyLightingSettings();
  markSceneDirty();
}

function updateCeilingNumber(key: CeilingLightingNumericKey, event: Event) {
  const input = event.currentTarget as HTMLInputElement;
  const ranges: Record<CeilingLightingNumericKey, { min: number; max: number }> = {
    spacing: { min: 2, max: 8 },
    intensity: { min: 0, max: 20 },
    colorTemperature: { min: 3000, max: 6500 },
  };
  if (!Number.isFinite(input.valueAsNumber)) return;
  ceilingLighting[key] = THREE.MathUtils.clamp(input.valueAsNumber, ranges[key].min, ranges[key].max);
  if (key === 'spacing') rebuildCeilingFixtures();
  else applyCeilingLighting();
  markSceneDirty();
}

function updateCeilingToggle(event: Event) {
  ceilingLighting.enabled = (event.currentTarget as HTMLInputElement).checked;
  applyCeilingLighting();
  markSceneDirty();
}

function updateCeilingFixtureVisibility(event: Event) {
  ceilingLighting.showFixtures = (event.currentTarget as HTMLInputElement).checked;
  applyCeilingLighting();
  markSceneDirty();
}

function applyCeilingLighting() {
  if (!ceilingLightPanels) return;
  const enabled = ceilingLighting.enabled;
  const color = colorTemperatureHex(ceilingLighting.colorTemperature);
  ceilingLightPanels.visible = enabled && ceilingLighting.showFixtures;
  const panelMaterials = Array.isArray(ceilingLightPanels.material)
    ? ceilingLightPanels.material
    : [ceilingLightPanels.material];
  for (const material of panelMaterials) {
    if (material instanceof THREE.MeshStandardMaterial) {
      material.emissive.set(color);
      material.emissiveIntensity = 1.2;
    }
  }
  for (const areaLight of ceilingAreaLights) {
    areaLight.visible = enabled;
    areaLight.color.set(color);
    areaLight.intensity = ceilingLighting.intensity;
  }
}

function applyLightingSettings() {
  if (!hemisphereLight || !keyLight || !fillLight || !renderer) return;
  hemisphereLight.color.set(lighting.ambientColor);
  hemisphereLight.intensity = lighting.ambientIntensity;
  keyLight.visible = lighting.keyEnabled;
  keyLight.color.set(lighting.keyColor);
  keyLight.intensity = lighting.keyIntensity;
  const azimuth = THREE.MathUtils.degToRad(lighting.keyAzimuth);
  const elevation = THREE.MathUtils.degToRad(lighting.keyElevation);
  const horizontalDistance = 20 * Math.cos(elevation);
  keyLight.position.set(
    horizontalDistance * Math.cos(azimuth),
    20 * Math.sin(elevation),
    horizontalDistance * Math.sin(azimuth),
  );
  keyLight.castShadow = lighting.shadowsEnabled;
  fillLight.visible = lighting.fillEnabled;
  fillLight.color.set(lighting.fillColor);
  fillLight.intensity = lighting.fillIntensity;
  renderer.shadowMap.enabled = lighting.shadowsEnabled;
  renderer.shadowMap.needsUpdate = true;
  renderer.toneMappingExposure = lighting.exposure;
}

async function serializeScene(): Promise<SceneDocument> {
  const serializedRacks: SceneRack[] = [];
  for (const rack of racks.value) {
    const serializedRack: SceneRack = {
      id: rack.id,
      name: rack.name,
      kind: rack.kind,
      width: rack.width,
      depth: rack.depth,
      height: rack.height,
      x: rack.group.position.x,
      z: rack.group.position.z,
      locked: rack.locked,
    };
    if (rack.kind === 'imported') {
      const modelRoot = new THREE.Group();
      for (const child of rack.group.children) {
        if (!(child instanceof CSS2DObject)) modelRoot.add(child.clone(true));
      }
      const exportedModel = await gltfExporter.parseAsync(modelRoot, { embedImages: true });
      if (exportedModel instanceof ArrayBuffer) throw new Error('导出导入模型时返回了无效的数据格式。');
      serializedRack.modelAsset = exportedModel;
    }
    serializedRacks.push(serializedRack);
  }
  return {
    format: 'nexus-space-studio',
    version: 1,
    room: { width: room.width, length: room.length, height: room.height },
    racks: serializedRacks,
    partitions: partitions.value.map((partition) => ({
      id: partition.id,
      name: partition.name,
      length: partition.length,
      thickness: partition.thickness,
      rotation: partition.rotation,
      x: partition.x,
      z: partition.z,
      fullSpan: partition.fullSpan,
      locked: partition.locked,
      roomWallSide: partition.roomWallSide,
      wallMaterialType: partition.wallMaterialType,
      wallColor: partition.wallColor,
      doors: partition.doors.map((door) => ({ ...door })),
    })),
    annotations: annotations.value.map(({ id, itemId, title, description }) => ({ id, itemId, title, description })),
    lighting: { ...lighting },
    ceilingLighting: { ...ceilingLighting },
  };
}

async function saveSceneToBrowser() {
  sceneFileError.value = '';
  try {
    const serializedScene = await serializeScene();
    localStorage.setItem(localSceneStorageKey, JSON.stringify(serializedScene));
    saveStatus.value = '已保存';
  } catch (error) {
    console.error('Unable to save the scene in browser storage.', error);
    sceneFileError.value = error instanceof DOMException && error.name === 'QuotaExceededError'
      ? '浏览器本地空间不足，请使用“导出”保存场景文件。'
      : '保存场景失败，请尝试导出文件。';
  }
}

async function exportSceneFile() {
  sceneFileError.value = '';
  try {
    const serializedScene = await serializeScene();
    const blob = new Blob([JSON.stringify(serializedScene, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    link.href = url;
    link.download = `nexus-space-scene-${timestamp}.json`;
    document.body.appendChild(link);
    link.click();
    link.remove();
    window.setTimeout(() => URL.revokeObjectURL(url), 1000);
    saveStatus.value = '已导出';
  } catch (error) {
    console.error('Unable to export the scene file.', error);
    sceneFileError.value = '导出场景失败，请检查导入模型是否可正常读取。';
  }
}

function triggerSceneImport() {
  sceneFileError.value = '';
  sceneFileInput.value?.click();
}

function importSceneFile(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  if (!window.confirm('导入场景会替换当前场景，尚未保存的编辑将丢失。确定继续吗？')) {
    input.value = '';
    return;
  }
  sceneFileError.value = '';
  file.text()
    .then((text) => {
      const parsed: unknown = JSON.parse(text);
      if (!isSceneDocument(parsed)) throw new Error('场景文件格式无效或版本不受支持。');
      return loadSceneDocument(parsed);
    })
    .then(() => {
      saveStatus.value = '已导入';
    })
    .catch((error: unknown) => {
      console.error('Unable to import the scene file.', error);
      sceneFileError.value = error instanceof Error ? error.message : '导入场景失败，请检查文件后重试。';
    })
    .finally(() => {
      input.value = '';
    });
}

async function loadSceneDocument(documentData: SceneDocument) {
  const loadedRacks: RackItem[] = [];
  for (const rackData of documentData.racks) {
    let group: THREE.Group;
    if (rackData.kind === 'imported') {
      const gltf = await new Promise<GLTF>((resolve, reject) => {
        gltfLoader.parse(JSON.stringify(rackData.modelAsset), '', resolve, reject);
      });
      group = new THREE.Group();
      group.add(gltf.scene);
    } else {
      group = buildRack(rackData.name, rackData.id).group;
    }
    group.name = rackData.name;
    group.position.set(rackData.x, 0.035, rackData.z);
    group.traverse((child) => {
      if (child instanceof THREE.Mesh) {
        child.userData.itemId = rackData.id;
        child.castShadow = true;
        child.receiveShadow = true;
      }
    });
    loadedRacks.push({
      id: rackData.id,
      name: rackData.name,
      kind: rackData.kind,
      group: markRaw(group),
      width: rackData.width,
      depth: rackData.depth,
      height: rackData.height,
      x: rackData.x,
      z: rackData.z,
      locked: rackData.locked,
    });
  }

  const loadedPartitions = documentData.partitions.map((partitionData) => reactive({
    ...partitionData,
    group: markRaw(new THREE.Group()),
    doors: partitionData.doors.map((door) => ({ ...door })),
  }));
  for (const item of [...racks.value, ...partitions.value]) {
    scene.remove(item.group);
    disposeGroup(item.group);
  }
  for (const annotation of annotations.value) annotation.label.parent?.remove(annotation.label);
  annotations.value = [];
  racks.value = [];
  partitions.value = [];
  room.width = documentData.room.width;
  room.length = documentData.room.length;
  room.height = documentData.room.height;
  roomDraft.width = room.width;
  roomDraft.length = room.length;
  roomDraft.height = room.height;
  Object.assign(lighting, documentData.lighting ?? defaultLightingSettings);
  Object.assign(ceilingLighting, { ...defaultCeilingLightingSettings, ...documentData.ceilingLighting });
  applyLightingSettings();
  buildRoom();

  partitions.value = loadedPartitions;
  for (const partition of partitions.value) {
    scene.add(partition.group);
    clampPartitionToRoom(partition);
    rebuildPartition(partition);
  }
  racks.value = loadedRacks;
  for (const rack of racks.value) {
    scene.add(rack.group);
    clampRackToRoom(rack);
  }
  for (const annotation of documentData.annotations) {
    const rack = racks.value.find((item) => item.id === annotation.itemId);
    if (rack) attachAnnotation(rack, annotation.id, annotation.title, annotation.description);
  }
  selectedId.value = null;
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
    activeDrag = null;
    controls.enabled = true;
    return;
  }
  const id = hit.object.userData.itemId as string | undefined;
  const rack = racks.value.find((item) => item.id === id);
  const partition = partitions.value.find((item) => item.id === id);
  if (!rack && !partition) return;
  if (selectedId.value !== id) {
    selectedId.value = id ?? null;
    activeDrag = null;
    controls.enabled = true;
    return;
  }
  if (rack?.locked || partition?.locked || partition?.roomWallSide) {
    activeDrag = null;
    controls.enabled = true;
    return;
  }
  activeDrag = rack ? { kind: 'rack', id: rack.id } : partition ? { kind: 'partition', id: partition.id } : null;
  floorPlaneForDrag = new THREE.Plane(new THREE.Vector3(0, 1, 0), -0.035);
  controls.enabled = false;
  renderer.domElement.setPointerCapture(event.pointerId);
  moveRack(event);
}

function moveRack(event: PointerEvent) {
  if (!activeDrag || selectedId.value !== activeDrag.id) return;
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const point = raycaster.ray.intersectPlane(floorPlaneForDrag, new THREE.Vector3());
  if (!point) return;
  const step = 0.1;
  const rack = activeDrag.kind === 'rack' ? racks.value.find((item) => item.id === activeDrag?.id) : null;
  if (rack) {
    if (rack.locked) return;
    rack.group.position.x = Math.round(point.x / step) * step;
    rack.group.position.z = Math.round(point.z / step) * step;
    clampRackToRoom(rack);
    markSceneDirty();
    return;
  }
  const partition = activeDrag.kind === 'partition' ? partitions.value.find((item) => item.id === activeDrag?.id) : null;
  if (partition) {
    if (partition.locked) return;
    partition.x = Math.round(point.x / step) * step;
    partition.z = Math.round(point.z / step) * step;
    clampPartitionToRoom(partition);
    markSceneDirty();
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
  selectionOutline?.update();
  renderer.render(scene, camera);
  labelRenderer.render(scene, camera);
}

onMounted(async () => {
  await nextTick();
  if (!viewport.value) return;
  RectAreaLightUniformsLib.init();
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

  hemisphereLight = new THREE.HemisphereLight(0xc9e5ff, 0x38414a, 2.1);
  scene.add(hemisphereLight);
  keyLight = new THREE.DirectionalLight(0xe4f2ff, 3.2);
  keyLight.castShadow = true;
  keyLight.shadow.mapSize.set(2048, 2048);
  keyLight.shadow.camera.left = -18;
  keyLight.shadow.camera.right = 18;
  keyLight.shadow.camera.top = 18;
  keyLight.shadow.camera.bottom = -18;
  scene.add(keyLight);
  fillLight = new THREE.DirectionalLight(0x61b9cf, 1.25);
  fillLight.position.set(10, 8, -8);
  scene.add(fillLight);
  applyLightingSettings();
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(200, 200),
    new THREE.MeshStandardMaterial({ color: 0x111a22, roughness: 0.94 }),
  );
  ground.rotation.x = -Math.PI / 2;
  ground.position.y = -0.035;
  ground.receiveShadow = true;
  scene.add(ground);
  buildRoom();
  createRoomWalls();

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
  let restoredLocalScene = false;
  try {
    const savedScene = localStorage.getItem(localSceneStorageKey);
    if (savedScene) {
      const parsed: unknown = JSON.parse(savedScene);
      if (!isSceneDocument(parsed)) throw new Error('浏览器中保存的场景格式无效。');
      await loadSceneDocument(parsed);
      saveStatus.value = '已载入';
      restoredLocalScene = true;
    }
  } catch (error) {
    console.error('Unable to restore the saved scene from browser storage.', error);
    sceneFileError.value = error instanceof Error ? error.message : '载入本地场景失败。';
  }
  if (!restoredLocalScene) {
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
  }
  selectedId.value = null;
});

onBeforeUnmount(() => {
  if (animationFrame) window.cancelAnimationFrame(animationFrame);
  resizeObserver?.disconnect();
  window.removeEventListener('pointerup', onPointerUp);
  if (selectionOutline) {
    scene.remove(selectionOutline);
    selectionOutline.geometry.dispose();
    selectionOutline.material.dispose();
    selectionOutline = null;
  }
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
.scene-file-section { padding-top: 14px; padding-bottom: 14px; }
.scene-file-section .section-title { margin-bottom: 8px; }
.scene-file-actions { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; }
.scene-file-actions button { min-width: 0; height: 30px; border: 1px solid rgba(172,199,211,.13); border-radius: 3px; background: rgba(255,255,255,.025); color: #a9c8c7; font-size: 9px; cursor: pointer; }
.scene-file-actions button:hover { border-color: rgba(131,217,212,.4); background: rgba(91,174,170,.08); }
.scene-file-section > p { margin: 7px 0 0; color: #70818a; font-size: 8px; line-height: 1.5; }
.inventory-section { padding-top: 17px; }
.count-pill { margin-left: auto; color: #8ccbc6; font-size: 9px; padding: 3px 6px; background: rgba(98, 188, 184, .11); border-radius: 3px; font-variant-numeric: tabular-nums; }
.object-list { max-height: 207px; overflow-y: auto; margin: 0 -7px; padding: 0 3px; }
.object-row { width: 100%; min-height: 51px; display: flex; align-items: center; gap: 3px; padding: 2px 4px 2px 0; background: transparent; border: 1px solid transparent; border-radius: 4px; }
.object-row:hover { background: rgba(255,255,255,.025); }
.object-row.selected { background: rgba(99, 183, 180, .085); border-color: rgba(99, 183, 180, .2); }
.object-row.locked .object-copy strong { color: #b9d1cf; }
.object-select { flex: 1; min-width: 0; min-height: 45px; display: flex; align-items: center; gap: 9px; text-align: left; padding: 5px 5px 5px 6px; color: inherit; background: transparent; border: 0; border-radius: 3px; cursor: pointer; }
.object-select:hover { background: rgba(255,255,255,.025); }
.lock-toggle { width: 25px; height: 28px; flex: 0 0 25px; display: grid; place-items: center; padding: 0; color: #65777e; background: transparent; border: 1px solid transparent; border-radius: 3px; cursor: pointer; opacity: .55; transition: .15s ease; }
.object-row:hover .lock-toggle, .object-row.selected .lock-toggle, .lock-toggle.active { opacity: 1; }
.lock-toggle:hover { color: #a4d8d1; background: rgba(111,191,185,.1); border-color: rgba(111,191,185,.16); }
.lock-toggle.active { color: #8ed3c6; }
.lock-toggle svg, .selected-lock-button svg { width: 14px; height: 14px; fill: none; stroke: currentColor; stroke-width: 1.35; stroke-linecap: round; stroke-linejoin: round; }
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
.selected-lock-button { width: 43px; flex: 0 0 43px; align-self: stretch; display: flex; flex-direction: column; justify-content: center; align-items: center; gap: 4px; padding: 0; color: #778a8e; background: transparent; border: 0; border-left: 1px solid rgba(179,205,220,.08); font-size: 7px; cursor: pointer; }
.selected-lock-button:hover, .selected-lock-button.active { color: #87cfc6; background: rgba(111,191,185,.055); }
.selected-lock-button.active span { color: #91d6c6; }
.property-section { padding: 0 18px 16px; margin-bottom: 14px; border-bottom: 1px solid var(--line); }
.partition-length-field { height: 34px; display: flex; align-items: center; gap: 8px; padding: 0 8px; margin: -2px 0 12px; border: 1px solid rgba(172,199,211,.11); background: #11191f; border-radius: 3px; color: #829198; font-size: 9px; }
.partition-length-field input { width: 100%; min-width: 0; border: 0; outline: 0; background: transparent; color: #d6e0e2; font-size: 10px; }
.partition-length-field b { color: #75858b; font-size: 8px; font-weight: 400; }
.wall-material-field, .wall-color-field { min-height: 34px; display: flex; align-items: center; gap: 8px; margin-top: 7px; padding: 0 8px; border: 1px solid rgba(172,199,211,.11); background: #11191f; border-radius: 3px; color: #829198; font-size: 9px; }
.wall-material-field select { flex: 1; min-width: 0; height: 30px; border: 0; outline: 0; background: #11191f; color: #d6e0e2; font-size: 9px; }
.wall-color-field input[type="color"] { width: 42px; height: 25px; padding: 2px; border: 1px solid rgba(172,199,211,.18); border-radius: 3px; background: #182229; cursor: pointer; }
.wall-color-field strong { margin-left: auto; color: #aab8bc; font-size: 9px; font-weight: 500; font-variant-numeric: tabular-nums; }
.rotate-button { width: 100%; height: 31px; display: flex; align-items: center; gap: 6px; margin-top: 11px; padding: 0 8px; border: 1px solid rgba(172,199,211,.11); background: rgba(255,255,255,.02); color: #a9b8bd; border-radius: 3px; font-size: 9px; text-align: left; cursor: pointer; }
.rotate-button:hover { border-color: rgba(112,193,188,.32); color: #d1e5e3; }
.rotate-button:disabled { opacity: .58; cursor: not-allowed; }
.rotate-button:disabled:hover { border-color: rgba(172,199,211,.11); color: #a9b8bd; }
.rotate-button > span { color: #85c9c3; font-size: 14px; }
.rotate-button small { margin-left: auto; color: #71838a; font-size: 8px; }
.partition-door-panel { padding: 0 18px 15px; margin-bottom: 14px; border-bottom: 1px solid var(--line); }
.partition-door-panel .section-title { margin-bottom: 8px; }
.partition-door-panel p { margin: 0 0 10px; color: #7e8e94; font-size: 8px; line-height: 1.6; }
.door-status { margin-left: auto; color: #849397; font-size: 8px; font-weight: 400; }
.door-status.installed { color: #83c4a8; }
.partition-door-item { margin-top: 12px; padding-top: 10px; border-top: 1px solid rgba(172,199,211,.08); }
.door-item-heading { display: flex; justify-content: space-between; align-items: center; margin-bottom: 7px; color: #a7b5b9; font-size: 8px; }
.door-item-heading span { color: #72838a; }
.door-actions { display: flex; align-items: center; gap: 7px; }
.door-actions .annotation-add { flex: 1; }
.remove-door-button { height: 31px; padding: 0 8px; background: transparent; border: 1px solid rgba(219,116,105,.17); border-radius: 3px; color: #c5877f; font-size: 8px; cursor: pointer; }
.remove-door-button:hover { border-color: rgba(219,116,105,.4); }
.remove-door-button:disabled { opacity: .45; cursor: not-allowed; }
.door-position-control { display: block; margin-top: 12px; }
.door-position-label { display: flex; align-items: center; justify-content: space-between; color: #a7b5b9; font-size: 8px; }
.door-position-label small { color: #72838a; font-size: 8px; margin-left: 4px; }
.door-position-label strong { color: #8ed1c8; font-size: 9px; font-weight: 500; font-variant-numeric: tabular-nums; }
.door-position-control input[type="range"] { display: block; width: 100%; height: 15px; margin: 7px 0 1px; accent-color: #75c9c1; cursor: pointer; }
.door-position-control input[type="range"]:disabled { opacity: .48; cursor: not-allowed; }
.door-position-ends { display: flex; justify-content: space-between; color: #6f8087; font-size: 7px; }
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
.lighting-panel { margin: 0 18px; border-bottom: 1px solid var(--line); }
.lighting-panel > summary { min-height: 43px; display: flex; align-items: center; justify-content: space-between; color: #d3dfdf; font-size: 10px; cursor: pointer; list-style: none; }
.lighting-panel > summary::-webkit-details-marker { display: none; }
.lighting-panel > summary::after { content: "+"; color: #78c5bf; font-size: 15px; }
.lighting-panel[open] > summary::after { content: "−"; }
.lighting-panel > summary small { margin-left: auto; margin-right: 9px; color: #778b90; font-size: 8px; }
.lighting-controls { display: flex; flex-direction: column; gap: 9px; padding: 1px 0 14px; }
.lighting-subheading { padding-top: 5px; border-top: 1px solid rgba(179,205,220,.07); color: #7d999a; font-size: 8px; letter-spacing: .5px; text-transform: uppercase; }
.lighting-range { display: flex; flex-direction: column; gap: 5px; color: #94a3a8; font-size: 8px; }
.lighting-range > span { display: flex; justify-content: space-between; }
.lighting-range strong { color: #a9bfbe; font-weight: 500; }
.lighting-range input { width: 100%; height: 12px; margin: 0; accent-color: #75c4c0; }
.lighting-range input:disabled { opacity: .4; }
.lighting-color { display: flex; align-items: center; justify-content: space-between; color: #94a3a8; font-size: 8px; }
.lighting-color input { width: 31px; height: 19px; padding: 1px; border: 1px solid rgba(172,199,211,.2); border-radius: 3px; background: #11191f; cursor: pointer; }
.lighting-color input:disabled { opacity: .4; cursor: not-allowed; }
.lighting-toggle { display: flex; align-items: center; gap: 7px; color: #94a3a8; font-size: 8px; cursor: pointer; }
.lighting-toggle input { margin: 0; accent-color: #75c4c0; }
.lighting-help { margin: -2px 0 0; color: #71838a; font-size: 8px; line-height: 1.5; }
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
