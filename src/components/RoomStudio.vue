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
        <div class="mode-switch" role="group" aria-label="工作模式">
          <button type="button" :class="{ active: !isPreviewMode }" :aria-pressed="!isPreviewMode" @click="setEditMode">编辑</button>
          <button type="button" :class="{ active: isPreviewMode }" :aria-pressed="isPreviewMode" @click="setPreviewMode">预览</button>
        </div>
        <button class="top-button subtle" type="button" @click="resetCamera()">
          <span class="button-icon">⌖</span> 重置视角
        </button>
        <button class="top-button subtle undo-button" type="button" :disabled="isPreviewMode || historyBusy || undoCount === 0" title="撤销上一步（Ctrl+Z）" @click="undoScene">
          <span class="button-icon">↶</span> 撤销 <small>{{ undoCount }}/30</small>
        </button>
        <button v-if="!isPreviewMode" class="top-button primary" type="button" @click="addRack">
          <span class="plus">＋</span> 添加机柜
        </button>
        <button v-if="!isPreviewMode" class="top-button subtle fire-smoke-top-button" type="button" @click="addFireSmokeObject">
          <span class="button-icon">✦</span> 添加火焰/烟雾
        </button>
        <button v-if="!isPreviewMode" class="top-button partition-top-button" type="button" @click="addPartition">
          <span class="plus">＋</span> 添加隔板
        </button>
      </div>
    </header>

    <div class="workspace" :class="{ 'preview-workspace': isPreviewMode }">
      <aside v-if="!isPreviewMode" class="sidebar left-sidebar">
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
          <label class="wall-material-field floor-material-field">
            <span>地板材质</span>
            <select :value="floorMaterialType" aria-label="地板材质" @change="updateFloorMaterial">
              <option value="concrete">混凝土</option>
              <option value="epoxy">环氧树脂</option>
              <option value="tile">瓷砖</option>
              <option value="rubber">橡胶地板</option>
            </select>
          </label>
          <p v-if="floorMaterialError" class="error-note">{{ floorMaterialError }}</p>
          <div class="dimension-hint"><span class="tiny-grid">▦</span> 可拖动 XYZ 轴调整物体位置，机柜与隔板的水平位置保持在房间范围内</div>
        </section>

        <section class="side-section scene-file-section">
          <div class="section-title"><span>场景文件</span><span class="section-note">本地 JSON</span></div>
          <div class="scene-file-actions">
            <button type="button" @click="saveSceneToBrowser">保存</button>
            <button type="button" @click="triggerSceneImport">导入</button>
            <button type="button" @click="exportSceneFile">导出</button>
          </div>
          <p>保存到当前浏览器，或导出 JSON 文件备份并在其他设备导入。</p>
          <p v-if="historyError" class="error-note">{{ historyError }}</p>
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
                <span class="object-copy"><strong>{{ rack.name }}</strong><small>{{ rackKindLabel(rack.kind) }} · {{ rack.height.toFixed(1) }} m</small></span>
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
          <p v-if="fireSmokeError" class="error-note">{{ fireSmokeError }}</p>
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
          <div class="breadcrumbs"><span>项目</span><i>/</i><strong>机房空间</strong>          <span class="view-chip">{{ isPreviewMode ? '预览模式' : '编辑模式' }}</span></div>
          <div class="view-tools">
            <span class="view-tool-label"><i></i> 正交辅助网格</span>
            <button type="button" class="icon-button" title="重置视角" aria-label="重置视角" @click="resetCamera()">⟳</button>
          </div>
        </div>
        <div ref="viewport" class="viewport">
          <div class="viewport-wash"></div>
          <div class="viewport-caption">
            <span class="caption-kicker">SPATIAL VIEW / 01</span>
            <strong>机房空间预览</strong>
            <span>{{ isPreviewMode ? '预览模式：仅查看场景 · 滚轮缩放 · 右键旋转' : '选中物体后，在右侧切换移动、旋转或缩放工具' }}</span>
          </div>
          <div class="axis-indicator"><span class="axis-y">Y</span><span class="axis-x">X</span><span class="axis-z">Z</span><i></i></div>
          <div class="view-scale"><span>0</span><i></i><span>5 m</span></div>
          <div class="viewport-status"><span class="live-dot"></span> 实时渲染 <i></i> {{ racks.length + partitions.length }} 个对象</div>
          <div v-if="selectedRack || selectedPartition" class="selection-hud">
            <span class="selection-dot"></span>
            <span><small>当前选择</small><strong>{{ selectedRack?.name ?? selectedPartition?.name }}</strong></span>
            <span class="hud-coordinates">X {{ (selectedRack?.x ?? selectedPartition?.x ?? 0).toFixed(1) }} <i>/</i> Y {{ (selectedRack?.y ?? selectedPartition?.y ?? 0).toFixed(1) }} <i>/</i> Z {{ (selectedRack?.z ?? selectedPartition?.z ?? 0).toFixed(1) }}</span>
          </div>
        </div>
        <div class="viewport-footer">
          <div class="legend"><span><i class="legend-room"></i>机房边界</span><span><i class="legend-rack"></i>机柜</span><span><i class="legend-door"></i>出入口</span></div>
          <span class="camera-label">PERSPECTIVE <i>·</i> 01</span>
        </div>
      </main>

      <aside v-if="!isPreviewMode" class="sidebar right-sidebar">
        <div class="right-heading">
          <div><div class="eyebrow">OBJECT DETAILS</div><h2>{{ selectedPartition ? selectedPartition.roomWallSide ? '机房外墙属性' : '隔板属性' : selectedRack ? `${selectedRack.kind === 'fire-smoke' ? '效果' : isDeviceKind(selectedRack.kind) ? '设备' : '机柜'}属性` : '场景工具' }}</h2></div>
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
            <div class="section-title"><span>{{ selectedPartition.roomWallSide ? '边界位置' : '位置坐标' }}</span><span class="edit-hint">{{ selectedPartition.roomWallSide ? '固定贴合房间边界' : '拖动 XYZ 轴可调整' }}</span></div>
            <div v-if="selectedPartition.roomWallSide" class="size-row"><span>墙面</span><strong>{{ roomWallLabel(selectedPartition.roomWallSide) }}</strong></div>
            <div v-else class="coordinate-grid">
              <label><span>X</span><input :value="selectedPartition.x.toFixed(1)" type="number" step="0.1" :disabled="selectedPartition.locked" @change="updatePartitionPosition('x', $event)"></label>
              <label><span>Y</span><input :value="selectedPartition.y.toFixed(1)" type="number" step="0.1" :disabled="selectedPartition.locked" @change="updatePartitionPosition('y', $event)"></label>
              <label><span>Z</span><input :value="selectedPartition.z.toFixed(1)" type="number" step="0.1" :disabled="selectedPartition.locked" @change="updatePartitionPosition('z', $event)"></label>
            </div>
          </section>
          <section v-if="!selectedPartition.roomWallSide" class="property-section transform-properties">
            <div class="section-title"><span>模型变换</span><span class="unit-tag">XYZ</span></div>
            <div class="transform-mode-switch" role="group" aria-label="模型变换模式">
              <button type="button" :class="{ active: transformMode === 'translate' }" @click="setTransformMode('translate')">移动</button>
              <button type="button" :class="{ active: transformMode === 'rotate' }" @click="setTransformMode('rotate')">旋转</button>
              <button type="button" :class="{ active: transformMode === 'scale' }" @click="setTransformMode('scale')">缩放</button>
            </div>
            <div class="transform-value-heading">旋转角度 <small>°</small></div>
            <div class="coordinate-grid transform-value-grid">
              <label v-for="axis in transformAxes" :key="`partition-rotation-${axis}`"><span>{{ axis.toUpperCase() }}</span><input :value="selectedPartition.transform.rotation[axis].toFixed(0)" type="number" min="-360" max="360" step="1" :disabled="selectedPartition.locked" @change="updateTransformValue('rotation', axis, $event)"></label>
            </div>
            <div class="transform-value-heading">缩放比例 <small>0.1–10</small></div>
            <div class="coordinate-grid transform-value-grid">
              <label v-for="axis in transformAxes" :key="`partition-scale-${axis}`"><span>{{ axis.toUpperCase() }}</span><input :value="selectedPartition.transform.scale[axis].toFixed(2)" type="number" min="0.1" max="10" step="0.1" :disabled="selectedPartition.locked" @change="updateTransformValue('scale', axis, $event)"></label>
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
                <button class="annotation-add" type="button" :disabled="selectedPartition.locked" @click="togglePartitionDoor(door.id)"><span>{{ door.doorOpen ? '↶' : '↗' }}</span> {{ door.doorOpen ? '关闭门扇' : '打开门扇' }}</button>
                <button class="remove-door-button" type="button" :disabled="selectedPartition.locked" @click="removePartitionDoor(door.id)">移除</button>
              </div>
              <button
                class="rotate-button door-reverse-button"
                type="button"
                :disabled="selectedPartition.locked"
                :aria-pressed="Boolean(door.reverseInstall)"
                @click="togglePartitionDoorInstallation(door.id)"
              ><span>⇄</span> 反向安装 <small>{{ door.reverseInstall ? '已反向' : '标准安装' }}</small></button>
              <div class="door-swing-control">
                <span>开门方向 <small>从门的安装侧面向门洞判断</small></span>
                <div class="door-swing-options" role="group" :aria-label="`门 ${index + 1} 开门方向`">
                  <button type="button" :disabled="selectedPartition.locked" :aria-pressed="(door.hingeSide ?? 'left') === 'left'" @click="setPartitionDoorHingeSide(door.id, 'left')">左侧开门</button>
                  <button type="button" :disabled="selectedPartition.locked" :aria-pressed="door.hingeSide === 'right'" @click="setPartitionDoorHingeSide(door.id, 'right')">右侧开门</button>
                </div>
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
            <div class="selected-name"><span class="selected-type">{{ selectedRack.kind === 'fire-smoke' ? 'FIRE / SMOKE EFFECT' : isDeviceKind(selectedRack.kind) ? 'DEVICE' : selectedRack.kind === 'imported' ? 'IMPORT MODEL' : 'SERVER RACK' }}</span><strong>{{ selectedRack.name }}</strong><small>对象 ID · {{ selectedRack.id.slice(-6).toUpperCase() }}</small></div>
            <button class="selected-lock-button" :class="{ active: selectedRack.locked }" type="button" :aria-label="selectedRack.locked ? '解锁物体' : '锁定物体'" @click="toggleLock(selectedRack.id)">
              <svg viewBox="0 0 16 16" aria-hidden="true"><rect x="3" y="7" width="10" height="7" rx="1.2"></rect><path v-if="selectedRack.locked" d="M5 7V5a3 3 0 0 1 6 0v2"></path><path v-else d="M5 7V5a3 3 0 0 1 5.2-2"></path></svg>
              <span>{{ selectedRack.locked ? '解锁' : '锁定' }}</span>
            </button>
          </div>
          <section class="property-section">
            <div class="section-title"><span>位置坐标</span><span class="edit-hint">{{ isDeviceKind(selectedRack.kind) ? '拖动 XYZ 轴可调整' : '机柜保持贴合地板' }}</span></div>
            <div class="coordinate-grid">
              <label><span>X</span><input :value="selectedRack.x.toFixed(1)" type="number" step="0.1" :disabled="selectedRack.locked" @change="updatePosition('x', $event)"></label>
              <label><span>Y</span><input :value="selectedRack.y.toFixed(1)" type="number" step="0.1" :disabled="selectedRack.locked || (!isDeviceKind(selectedRack.kind) && selectedRack.kind !== 'fire-smoke')" @change="updatePosition('y', $event)"></label>
              <label><span>Z</span><input :value="selectedRack.z.toFixed(1)" type="number" step="0.1" :disabled="selectedRack.locked" @change="updatePosition('z', $event)"></label>
            </div>
          </section>
          <section class="property-section transform-properties">
            <div class="section-title"><span>模型变换</span><span class="unit-tag">XYZ</span></div>
            <div class="transform-mode-switch" role="group" aria-label="模型变换模式">
              <button type="button" :class="{ active: transformMode === 'translate' }" @click="setTransformMode('translate')">移动</button>
              <button type="button" :class="{ active: transformMode === 'rotate' }" @click="setTransformMode('rotate')">旋转</button>
              <button type="button" :class="{ active: transformMode === 'scale' }" @click="setTransformMode('scale')">缩放</button>
            </div>
            <div class="transform-value-heading">旋转角度 <small>°</small></div>
            <div class="coordinate-grid transform-value-grid">
              <label v-for="axis in transformAxes" :key="`rack-rotation-${axis}`"><span>{{ axis.toUpperCase() }}</span><input :value="selectedRack.transform.rotation[axis].toFixed(0)" type="number" min="-360" max="360" step="1" :disabled="selectedRack.locked" @change="updateTransformValue('rotation', axis, $event)"></label>
            </div>
            <div class="transform-value-heading">缩放比例 <small>0.1–10</small></div>
            <div class="coordinate-grid transform-value-grid">
              <label v-for="axis in transformAxes" :key="`rack-scale-${axis}`"><span>{{ axis.toUpperCase() }}</span><input :value="selectedRack.transform.scale[axis].toFixed(2)" type="number" min="0.1" max="10" step="0.1" :disabled="selectedRack.locked" @change="updateTransformValue('scale', axis, $event)"></label>
            </div>
          </section>
          <section v-if="selectedRack.kind === 'fire-smoke'" class="property-section fire-smoke-panel">
            <div class="section-title"><span>火焰与烟雾模拟</span><span class="unit-tag">粒子</span></div>
            <p class="fire-smoke-help">程序化粒子效果，非真实流体模拟。</p>
            <label class="lighting-toggle"><input type="checkbox" :checked="selectedRack.fireSmoke?.enabled ?? false" @change="updateFireSmokeEnabled(selectedRack.id, $event)"><span>启用效果</span></label>
            <label class="lighting-range fire-smoke-intensity">
              <span>效果强度 <strong>{{ (selectedRack.fireSmoke?.intensity ?? 1).toFixed(1) }}</strong></span>
              <input type="range" min="0.2" max="2" step="0.1" :value="selectedRack.fireSmoke?.intensity ?? 1" :disabled="!selectedRack.fireSmoke?.enabled" aria-label="火焰与烟雾强度" @input="updateFireSmokeIntensity(selectedRack.id, $event)">
            </label>
            <button class="remove-fire-smoke" type="button" @click="removeSelected"><span>⌫</span> 删除效果</button>
            <p v-if="fireSmokeError" class="error-note">{{ fireSmokeError }}</p>
          </section>
          <section class="property-section dimension-properties">
            <div class="section-title"><span>模型尺寸</span><span class="unit-tag">m</span></div>
            <div class="size-row"><span>宽度</span><strong>{{ (selectedRack.width * selectedRack.transform.scale.x).toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>深度</span><strong>{{ (selectedRack.depth * selectedRack.transform.scale.z).toFixed(2) }} <small>m</small></strong></div>
            <div class="size-row"><span>高度</span><strong>{{ (selectedRack.height * selectedRack.transform.scale.y).toFixed(2) }} <small>m</small></strong></div>
          </section>
          <button class="danger-button" type="button" @click="removeSelected"><span>⌫</span> 删除此对象</button>
        </template>
        <template v-else>
          <div class="tool-intro">
            <div class="tool-illustration"><span class="tool-cube"></span><i></i><b></b></div>
            <strong>构建你的空间</strong>
            <p>添加机柜、设备或导入模型，再点击场景中的对象查看属性。</p>
          </div>
          <div class="quick-actions">
            <button type="button" @click="addRack"><span class="quick-icon">▤</span><span><strong>添加标准机柜</strong><small>快速放置到房间地板</small></span><b>＋</b></button>
            <button type="button" @click="addFireSmokeObject"><span class="quick-icon fire-smoke-icon">✦</span><span><strong>添加火焰与烟雾</strong><small>独立场景对象，可自由移动</small></span><b>＋</b></button>
            <button type="button" @click="addDevice('camera')"><span class="quick-icon">◉</span><span><strong>添加摄像头</strong><small>临时使用提供模型 · 自由三维移动</small></span><b>＋</b></button>
            <button type="button" @click="addDevice('smoke-detector')"><span class="quick-icon">◌</span><span><strong>添加烟雾报警器</strong><small>临时使用提供模型 · 自由三维移动</small></span><b>＋</b></button>
            <button type="button" @click="addDevice('temperature-sensor')"><span class="quick-icon">⊙</span><span><strong>添加温度传感器</strong><small>临时使用提供模型 · 自由三维移动</small></span><b>＋</b></button>
            <button type="button" @click="triggerImport"><span class="quick-icon upload">↑</span><span><strong>导入自定义模型</strong><small>支持 GLB 格式文件</small></span><b>↗</b></button>
          </div>
        </template>

        <details class="lighting-panel">
          <summary><span>场景视角</span><small>初始视角</small></summary>
          <div class="lighting-controls">
            <label class="lighting-range">
              <span>缩放距离 <strong>{{ sceneView.distance.toFixed(1) }} m</strong></span>
              <input type="range" min="5" max="42" step="0.1" :value="sceneView.distance" aria-label="场景缩放距离" @input="updateSceneViewNumber('distance', $event)">
            </label>
            <label class="lighting-range">
              <span>水平角度 <strong>{{ sceneView.azimuth }}°</strong></span>
              <input type="range" min="-180" max="180" step="1" :value="sceneView.azimuth" aria-label="相机水平角度" @input="updateSceneViewNumber('azimuth', $event)">
            </label>
            <label class="lighting-range">
              <span>俯仰角 <strong>{{ sceneView.elevation }}°</strong></span>
              <input type="range" min="1" max="88" step="1" :value="sceneView.elevation" aria-label="相机俯仰角" @input="updateSceneViewNumber('elevation', $event)">
            </label>
            <div class="camera-view-actions">
              <button class="annotation-add" type="button" @click="resetCamera()">恢复默认视角</button>
              <button class="annotation-add" type="button" @click="applyCameraSettings">应用设置</button>
            </div>
            <p class="lighting-help">{{ cameraConfigStatus }}。应用后的配置会作为场景加载和重置视角的默认值；普通场景保存不会覆盖它。</p>
            <p v-if="cameraConfigError" class="error-note">{{ cameraConfigError }}</p>
          </div>
        </details>

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
            <div v-for="annotation in annotations" :key="annotation.id" class="annotation-row" :class="{ active: selectedAnnotationId === annotation.id }" @click="selectAnnotation(annotation.id)">
              <span class="annotation-pin">⌖</span>
              <span class="annotation-copy"><strong>{{ annotation.title }}</strong><small>{{ annotation.description || '暂无描述' }}</small></span>
              <button type="button" aria-label="删除标注" @click.stop="removeAnnotation(annotation.id)">×</button>
            </div>
          </div>
          <div v-else class="annotation-empty"><span>↗</span><p>添加标注后，信息卡片会显示在场景对象上方。</p></div>
          <div v-if="selectedAnnotation" class="annotation-settings">
            <div class="annotation-settings-title">标注外观与位置</div>
            <div class="annotation-settings-grid">
              <label><span>宽度 (px)</span><input type="number" min="20" max="400" step="10" :value="selectedAnnotation.width" @change="updateAnnotationSize(selectedAnnotation.id, 'width', $event)"></label>
              <label><span>高度 (px)</span><input type="number" min="20" max="240" step="4" :value="selectedAnnotation.height" @change="updateAnnotationSize(selectedAnnotation.id, 'height', $event)"></label>
            </div>
            <div class="annotation-settings-title offset-title">相对物体偏移 (m)</div>
            <div class="annotation-settings-grid offset-grid">
              <label v-for="axis in transformAxes" :key="`annotation-offset-${axis}`"><span>{{ axis.toUpperCase() }}</span><input type="number" min="-10" max="10" step="0.05" :value="selectedAnnotation.offset[axis]" @change="updateAnnotationOffset(selectedAnnotation.id, axis, $event)"></label>
            </div>
            <p class="annotation-tip">Y 偏移以物体上方的默认标注位置为基准。</p>
          </div>
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
import { TransformControls } from 'three/addons/controls/TransformControls.js';
import { CSS2DObject, CSS2DRenderer } from 'three/addons/renderers/CSS2DRenderer.js';
import { GLTFLoader, type GLTF } from 'three/addons/loaders/GLTFLoader.js';
import { GLTFExporter } from 'three/addons/exporters/GLTFExporter.js';
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';

type DeviceKind = 'camera' | 'smoke-detector' | 'temperature-sensor';
type RackKind = 'rack' | 'imported' | 'fire-smoke' | DeviceKind;
type TransformMode = 'translate' | 'rotate' | 'scale';
type TransformAxis = 'x' | 'y' | 'z';
type ObjectTransform = {
  rotation: Record<TransformAxis, number>;
  scale: Record<TransformAxis, number>;
};

type FireSmokeSettings = {
  enabled: boolean;
  intensity: number;
};

type FireSmokeParticle = {
  sprite: THREE.Sprite;
  material: THREE.SpriteMaterial;
  age: number;
  lifetime: number;
  phase: number;
  originX: number;
  originZ: number;
};

type FireSmokeEffect = {
  group: THREE.Group;
  flameParticles: FireSmokeParticle[];
  smokeParticles: FireSmokeParticle[];
  light: THREE.PointLight;
  settings: FireSmokeSettings;
};

type RackItem = {
  id: string;
  name: string;
  kind: RackKind;
  group: THREE.Group;
  width: number;
  depth: number;
  height: number;
  x: number;
  y: number;
  z: number;
  transform: ObjectTransform;
  locked: boolean;
  fireSmoke: FireSmokeSettings | null;
};

type RoomWallSide = 'north' | 'south' | 'east' | 'west';
type WallMaterialType = 'paint' | 'concrete' | 'steel' | 'brick';
type FloorMaterialType = 'concrete' | 'epoxy' | 'tile' | 'rubber';
type DoorHingeSide = 'left' | 'right';

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

type SceneViewSettings = {
  distance: number;
  azimuth: number;
  elevation: number;
  targetX: number;
  targetY: number;
  targetZ: number;
};

type PartitionItem = {
  id: string;
  name: string;
  group: THREE.Group;
  length: number;
  thickness: number;
  rotation: 0 | 1;
  x: number;
  y: number;
  z: number;
  transform: ObjectTransform;
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
  reverseInstall?: boolean;
  hingeSide?: DoorHingeSide;
};

type Annotation = {
  id: string;
  itemId: string;
  title: string;
  description: string;
  width: number;
  height: number;
  offset: Record<TransformAxis, number>;
  label: CSS2DObject;
};

type SceneAnnotation = {
  id: string;
  itemId: string;
  title: string;
  description: string;
  width?: number;
  height?: number;
  offset?: Record<TransformAxis, number>;
};

type SceneRack = {
  id: string;
  name: string;
  kind: RackKind;
  width: number;
  depth: number;
  height: number;
  x: number;
  y?: number;
  z: number;
  transform?: ObjectTransform;
  locked: boolean;
  modelAsset?: Record<string, unknown>;
  fireSmoke?: FireSmokeSettings;
};

type ScenePartition = {
  id: string;
  name: string;
  length: number;
  thickness: number;
  rotation: 0 | 1;
  x: number;
  y?: number;
  z: number;
  transform?: ObjectTransform;
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
  annotations: SceneAnnotation[];
  lighting?: LightingSettings;
  ceilingLighting?: CeilingLightingSettings;
  floorMaterial?: FloorMaterialType;
};

type HistorySnapshot = {
  document: SceneDocument;
  selectedId: string | null;
};

const viewport = ref<HTMLDivElement | null>(null);
const fileInput = ref<HTMLInputElement | null>(null);
const sceneFileInput = ref<HTMLInputElement | null>(null);
const isPreviewMode = ref(false);
const cameraConfigStatus = ref('尚未应用');
const cameraConfigError = ref('');
const sceneView = reactive<SceneViewSettings>({
  distance: 25.4,
  azimuth: 39.8,
  elevation: 22.7,
  targetX: 0,
  targetY: 1.2,
  targetZ: 0,
});
const room = reactive({ width: 12, length: 9, height: 4 });
const roomDraft = reactive({ width: 12, length: 9, height: 4 });
const defaultFloorMaterialType: FloorMaterialType = 'concrete';
const floorMaterialType = ref<FloorMaterialType>(defaultFloorMaterialType);
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
const selectedAnnotationId = ref<string | null>(null);
const undoCount = ref(0);
const historyBusy = ref(false);
const historyError = ref('');
const transformMode = ref<TransformMode>('translate');
const transformAxes: TransformAxis[] = ['x', 'y', 'z'];
const rackFloorY = 0.035;
const annotationTitle = ref('');
const annotationDescription = ref('');
const floorMaterialError = ref('');
const importError = ref('');
const assetStatus = ref('CC0 模型加载中');
const assetError = ref('');
const fireSmokeError = ref('');
const saveStatus = ref('尚未保存');
const sceneFileError = ref('');

const selectedRack = computed(() => racks.value.find((rack) => rack.id === selectedId.value) ?? null);
const selectedPartition = computed(() => partitions.value.find((partition) => partition.id === selectedId.value) ?? null);
const selectedAnnotation = computed(() => annotations.value.find((annotation) => annotation.id === selectedAnnotationId.value) ?? null);
watch(selectedId, () => {
  updateSelectionOutline();
  updateTransformControlsSelection();
});

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let labelRenderer: CSS2DRenderer;
let controls: OrbitControls;
let transformControls: TransformControls | null = null;
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
let freeDevicePrototype: THREE.Group | null = null;
let resizeObserver: ResizeObserver;
let animationFrame = 0;
let lastAnimationTimestamp = 0;
let activeDrag: { kind: 'rack' | 'device' | 'partition'; id: string } | null = null;
let activePointerId: number | null = null;
let applyingSceneView = false;
let appliedCameraConfig: SceneViewSettings | null = null;
let floorPlaneForDrag: THREE.Plane;
const dragOffset = new THREE.Vector3();
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const gltfLoader = new GLTFLoader();
const gltfExporter = new GLTFExporter();
const localSceneStorageKey = 'nexus-space-studio:scene:v1';
const cameraConfigStorageKey = 'nexus-space-studio:camera-view:v1';
const historyLimit = 30;
const historyCaptureDelay = 300;
const historyUndoStack: HistorySnapshot[] = [];
let historyInitialized = false;
let historyGroupOpen = false;
let historyInteractionActive = false;
let historyCaptureTimer: number | null = null;
let historySnapshotPromise: Promise<HistorySnapshot | null> | null = null;
let historyPendingRecords: Promise<void>[] = [];
const historyModelAssetCache = new Map<string, Record<string, unknown>>();
const fireSmokeEffects = new Map<string, FireSmokeEffect>();
let fireParticleTexture: THREE.DataTexture | null = null;
let smokeParticleTexture: THREE.DataTexture | null = null;
const wallMaterialPresets: Record<WallMaterialType, { roughness: number; metalness: number }> = {
  paint: { roughness: 0.72, metalness: 0.12 },
  concrete: { roughness: 0.94, metalness: 0.02 },
  steel: { roughness: 0.38, metalness: 0.64 },
  brick: { roughness: 0.9, metalness: 0 },
};

function makeMaterial(color: number, roughness = 0.7, metalness = 0) {
  return new THREE.MeshStandardMaterial({ color, roughness, metalness });
}

function createObjectTransform(): ObjectTransform {
  return {
    rotation: { x: 0, y: 0, z: 0 },
    scale: { x: 1, y: 1, z: 1 },
  };
}

function cloneObjectTransform(transform?: ObjectTransform): ObjectTransform {
  const defaults = createObjectTransform();
  return transform
    ? {
        rotation: { ...defaults.rotation, ...transform.rotation },
        scale: { ...defaults.scale, ...transform.scale },
      }
    : defaults;
}

function applyObjectTransform(group: THREE.Group, transform: ObjectTransform) {
  group.rotation.set(
    THREE.MathUtils.degToRad(transform.rotation.x),
    THREE.MathUtils.degToRad(transform.rotation.y),
    THREE.MathUtils.degToRad(transform.rotation.z),
  );
  group.scale.set(transform.scale.x, transform.scale.y, transform.scale.z);
}

function normalizeDegrees(degrees: number) {
  const normalized = THREE.MathUtils.euclideanModulo(degrees + 180, 360) - 180;
  return Number(normalized.toFixed(1));
}

function syncObjectTransform(group: THREE.Group, transform: ObjectTransform) {
  for (const axis of transformAxes) {
    transform.rotation[axis] = normalizeDegrees(THREE.MathUtils.radToDeg(group.rotation[axis]));
    transform.scale[axis] = clampNumber(group.scale[axis], 0.1, 10, 1);
  }
  applyObjectTransform(group, transform);
}

function updateTransformValue(
  property: 'rotation' | 'scale',
  axis: TransformAxis,
  event: Event,
) {
  const rack = selectedRack.value;
  const partition = selectedPartition.value;
  const item = rack ?? partition;
  const input = event.target as HTMLInputElement;
  if (!item || item.locked || partition?.roomWallSide || !Number.isFinite(input.valueAsNumber)) return;
  const value = property === 'rotation'
    ? clampNumber(input.valueAsNumber, -360, 360, item.transform.rotation[axis])
    : clampNumber(input.valueAsNumber, 0.1, 10, item.transform.scale[axis]);
  item.transform[property][axis] = property === 'rotation' ? normalizeDegrees(value) : value;
  applyObjectTransform(item.group, item.transform);
  if (rack) clampRackToRoom(rack);
  if (partition) clampPartitionToRoom(partition);
  markSceneDirty();
}

function setTransformMode(mode: TransformMode) {
  transformMode.value = mode;
  transformControls?.setMode(mode);
  updateTransformControlsSelection();
}

function isDeviceKind(kind: unknown): kind is DeviceKind {
  return kind === 'camera' || kind === 'smoke-detector' || kind === 'temperature-sensor';
}

function rackKindLabel(kind: RackKind) {
  const labels: Record<RackKind, string> = {
    rack: '标准机柜',
    imported: '导入模型',
    'fire-smoke': '火焰与烟雾',
    camera: '摄像头',
    'smoke-detector': '烟雾报警器',
    'temperature-sensor': '温度传感器',
  };
  return labels[kind];
}

function createFloorPatternTexture(pattern: 'tile' | 'rubber') {
  const canvas = document.createElement('canvas');
  canvas.width = 128;
  canvas.height = 128;
  const context = canvas.getContext('2d');
  if (!context) throw new Error('无法创建地板材质纹理。');

  if (pattern === 'tile') {
    context.fillStyle = '#929b9d';
    context.fillRect(0, 0, 128, 128);
    context.fillStyle = '#596467';
    context.fillRect(0, 0, 3, 128);
    context.fillRect(0, 0, 128, 3);
    context.fillRect(125, 0, 3, 128);
    context.fillRect(0, 125, 128, 3);
    context.strokeStyle = 'rgba(240, 245, 245, 0.12)';
    context.strokeRect(4, 4, 120, 120);
  } else {
    context.fillStyle = '#343b3d';
    context.fillRect(0, 0, 128, 128);
    for (let y = 8; y < 128; y += 16) {
      for (let x = 8; x < 128; x += 16) {
        context.beginPath();
        context.arc(x, y, 3, 0, Math.PI * 2);
        context.fillStyle = '#4b5355';
        context.fill();
        context.beginPath();
        context.arc(x - 1, y - 1, 1.4, 0, Math.PI * 2);
        context.fillStyle = '#252b2d';
        context.fill();
      }
    }
  }

  const texture = new THREE.CanvasTexture(canvas);
  texture.colorSpace = THREE.SRGBColorSpace;
  texture.wrapS = THREE.RepeatWrapping;
  texture.wrapT = THREE.RepeatWrapping;
  const repeatSize = pattern === 'tile' ? 0.6 : 0.5;
  texture.repeat.set(room.width / repeatSize, room.length / repeatSize);
  return texture;
}

function createFloorMaterial(materialType: FloorMaterialType): THREE.MeshStandardMaterial {
  if (materialType === 'epoxy') {
    return new THREE.MeshPhysicalMaterial({
      color: 0x9ca8aa,
      roughness: 0.2,
      metalness: 0.04,
      clearcoat: 0.55,
      clearcoatRoughness: 0.18,
    });
  }
  if (materialType === 'tile') {
    return new THREE.MeshStandardMaterial({
      color: 0xffffff,
      map: createFloorPatternTexture('tile'),
      roughness: 0.5,
      metalness: 0.01,
    });
  }
  if (materialType === 'rubber') {
    return new THREE.MeshStandardMaterial({
      color: 0xffffff,
      map: createFloorPatternTexture('rubber'),
      roughness: 0.94,
      metalness: 0,
    });
  }
  return makeMaterial(0x737b7d, 0.96, 0.01);
}

function disposeFloorMaterialMap() {
  const material = floorPlane?.material;
  if (!material) return;
  for (const surfaceMaterial of Array.isArray(material) ? material : [material]) {
    if (surfaceMaterial instanceof THREE.MeshStandardMaterial) surfaceMaterial.map?.dispose();
  }
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
    finishContinuousHistoryChange();
  }
  updateTransformControlsSelection();
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
    disposeFloorMaterialMap();
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

  floorPlane = new THREE.Mesh(new THREE.PlaneGeometry(room.width, room.length), createFloorMaterial(floorMaterialType.value));
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
      y: 0,
      z: 0,
      transform: createObjectTransform(),
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

function buildDevice(kind: DeviceKind, name: string, id: string) {
  const group = new THREE.Group();
  group.name = name;
  const model = new THREE.Group();

  if (freeDevicePrototype) {
    model.add(freeDevicePrototype.clone(true));
    group.userData.sharedAssets = true;
  } else if (kind === 'camera') {
    const body = makeMaterial(0x26343b, 0.48, 0.55);
    const lens = new THREE.MeshStandardMaterial({ color: 0x142a37, metalness: 0.72, roughness: 0.18 });
    makeBox(model, 0.16, 0.025, 0.16, body, 0, 0.045, 0);
    makeBox(model, 0.035, 0.09, 0.035, body, 0, -0.01, 0);
    makeBox(model, 0.12, 0.055, 0.085, body, 0, -0.075, 0.025);
    const lensMesh = new THREE.Mesh(new THREE.CylinderGeometry(0.026, 0.026, 0.024, 24), lens);
    lensMesh.rotation.x = Math.PI / 2;
    lensMesh.position.set(0, -0.075, 0.08);
    model.add(lensMesh);
  } else if (kind === 'smoke-detector') {
    const shell = makeMaterial(0xe4e9e9, 0.82);
    const trim = makeMaterial(0x7d8a8e, 0.7);
    const indicator = new THREE.MeshStandardMaterial({ color: 0xf05a55, emissive: 0x631c1a, emissiveIntensity: 0.45 });
    const base = new THREE.Mesh(new THREE.CylinderGeometry(0.105, 0.105, 0.012, 32), trim);
    base.position.y = -0.018;
    model.add(base);
    const shellMesh = new THREE.Mesh(new THREE.CylinderGeometry(0.098, 0.105, 0.045, 32), shell);
    shellMesh.position.y = 0.008;
    model.add(shellMesh);
    const light = new THREE.Mesh(new THREE.SphereGeometry(0.008, 12, 8), indicator);
    light.position.set(0.048, 0.034, 0.025);
    model.add(light);
    for (let index = 0; index < 8; index += 1) {
      const angle = (index / 8) * Math.PI * 2;
      const vent = new THREE.Mesh(new THREE.CylinderGeometry(0.004, 0.004, 0.002, 8), trim);
      vent.position.set(Math.cos(angle) * 0.072, 0.031, Math.sin(angle) * 0.072);
      model.add(vent);
    }
  } else {
    const casing = makeMaterial(0xd5dfe2, 0.68, 0.08);
    const screen = makeMaterial(0x1b2b32, 0.38, 0.12);
    const indicator = new THREE.MeshStandardMaterial({ color: 0x48d7a2, emissive: 0x145e43, emissiveIntensity: 0.6 });
    makeBox(model, 0.14, 0.12, 0.045, casing, 0, 0, 0);
    makeBox(model, 0.082, 0.045, 0.004, screen, 0, 0.026, 0.0245);
    const statusLight = new THREE.Mesh(new THREE.SphereGeometry(0.006, 12, 8), indicator);
    statusLight.position.set(0.048, -0.038, 0.025);
    model.add(statusLight);
    for (let index = 0; index < 3; index += 1) {
      makeBox(model, 0.055, 0.003, 0.003, screen, 0, -0.022 - index * 0.008, 0.024);
    }
  }

  model.updateMatrixWorld(true);
  const bounds = new THREE.Box3().setFromObject(model);
  const size = bounds.getSize(new THREE.Vector3());
  const maxDimension = Math.max(size.x, size.y, size.z);
  const targetSize = 0.24;
  const scale = targetSize / maxDimension;
  const center = bounds.getCenter(new THREE.Vector3());
  model.scale.multiplyScalar(scale);
  model.position.set(-center.x * scale, -center.y * scale, -center.z * scale);
  group.add(model);
  group.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.userData.itemId = id;
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  return {
    group,
    width: size.x * scale,
    depth: size.z * scale,
    height: size.y * scale,
  };
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
    y: 0.035,
    z: position.z,
    transform: createObjectTransform(),
    locked: false,
    fireSmoke: null,
  });
  selectedId.value = id;
  markSceneDirty();
}

function addDevice(kind: DeviceKind) {
  if (!scene) return;
  const id = createId();
  const index = racks.value.filter((rack) => isDeviceKind(rack.kind)).length;
  const name = `${rackKindLabel(kind)} ${String(racks.value.filter((rack) => rack.kind === kind).length + 1).padStart(2, '0')}`;
  const built = buildDevice(kind, name, id);
  const row = Math.floor(index / 3);
  const position = {
    x: (index % 3 - 1) * 0.55,
    y: room.height + 0.2 + Math.floor(row / 3) * 0.35,
    z: (row % 3 - 1) * 0.55,
  };
  built.group.position.set(position.x, position.y, position.z);
  scene.add(built.group);
  racks.value.push({
    id,
    name,
    kind,
    ...built,
    group: markRaw(built.group),
    ...position,
    transform: createObjectTransform(),
    locked: false,
    fireSmoke: null,
  });
  selectedId.value = id;
  markSceneDirty();
}

function getParticleTexture(kind: 'flame' | 'smoke') {
  const cachedTexture = kind === 'flame' ? fireParticleTexture : smokeParticleTexture;
  if (cachedTexture) return cachedTexture;
  const size = 64;
  const data = new Uint8Array(size * size * 4);
  const center = (size - 1) / 2;
  const radiusX = kind === 'flame' ? 22 : 29;
  const radiusY = kind === 'flame' ? 30 : 29;
  for (let y = 0; y < size; y += 1) {
    for (let x = 0; x < size; x += 1) {
      const dx = (x - center) / radiusX;
      const dy = (y - center) / radiusY;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const alpha = Math.round(255 * Math.pow(Math.max(0, 1 - distance), kind === 'flame' ? 1.4 : 1.8));
      const index = (y * size + x) * 4;
      data[index] = 255;
      data[index + 1] = 255;
      data[index + 2] = 255;
      data[index + 3] = alpha;
    }
  }
  const texture = new THREE.DataTexture(data, size, size, THREE.RGBAFormat, THREE.UnsignedByteType);
  texture.colorSpace = THREE.SRGBColorSpace;
  texture.magFilter = THREE.LinearFilter;
  texture.minFilter = THREE.LinearFilter;
  texture.generateMipmaps = false;
  texture.needsUpdate = true;
  if (kind === 'flame') fireParticleTexture = texture;
  else smokeParticleTexture = texture;
  return texture;
}

function createFireSmokeParticle(
  group: THREE.Group,
  kind: 'flame' | 'smoke',
  index: number,
  count: number,
): FireSmokeParticle {
  const phase = index * 2.399963229728653;
  const material = new THREE.SpriteMaterial({
    map: getParticleTexture(kind),
    color: kind === 'flame' ? 0xffb33e : 0x9ca6aa,
    transparent: true,
    opacity: 1,
    depthWrite: false,
    blending: kind === 'flame' ? THREE.AdditiveBlending : THREE.NormalBlending,
  });
  const sprite = new THREE.Sprite(material);
  sprite.center.set(0.5, kind === 'flame' ? 0.05 : 0.5);
  sprite.renderOrder = kind === 'flame' ? 4 : 3;
  group.add(sprite);
  const lifetime = kind === 'flame' ? 0.65 + (index % 5) * 0.12 : 2 + (index % 6) * 0.35;
  return {
    sprite,
    material,
    age: (index / count) * lifetime,
    lifetime,
    phase,
    originX: Math.cos(phase) * (kind === 'flame' ? 0.07 : 0.12),
    originZ: Math.sin(phase) * (kind === 'flame' ? 0.07 : 0.12),
  };
}

function createFireSmokeEffect(rack: RackItem) {
  if (!rack.fireSmoke || fireSmokeEffects.has(rack.id)) return;
  const visualGroup = markRaw(new THREE.Group());
  visualGroup.name = `${rack.name} 火焰与烟雾`;
  visualGroup.userData.sceneEffect = true;
  const flameParticles = Array.from({ length: 16 }, (_, index) => (
    createFireSmokeParticle(visualGroup, 'flame', index, 16)
  ));
  const smokeParticles = Array.from({ length: 18 }, (_, index) => (
    createFireSmokeParticle(visualGroup, 'smoke', index, 18)
  ));
  const light = new THREE.PointLight(0xff6a24, 0, 5, 2);
  light.position.y = 0.18;
  visualGroup.add(light);
  visualGroup.visible = rack.fireSmoke.enabled;
  const hitbox = new THREE.Mesh(
    new THREE.BoxGeometry(rack.width, rack.height, rack.depth),
    new THREE.MeshBasicMaterial({ colorWrite: false, depthWrite: false, side: THREE.DoubleSide }),
  );
  hitbox.name = `${rack.name} 选择区域`;
  hitbox.position.y = rack.height / 2;
  hitbox.userData.itemId = rack.id;
  rack.group.add(hitbox, visualGroup);
  fireSmokeEffects.set(rack.id, {
    group: visualGroup,
    flameParticles,
    smokeParticles,
    light,
    settings: rack.fireSmoke,
  });
}

function createFireSmokeRack(
  id: string,
  name: string,
  position: { x: number; y: number; z: number },
  settings: FireSmokeSettings,
): RackItem {
  const width = 1.2;
  const depth = 1.2;
  const height = 3.5;
  const group = markRaw(new THREE.Group());
  group.name = name;
  group.userData.itemId = id;
  group.position.set(position.x, position.y, position.z);
  return reactive({
    id,
    name,
    kind: 'fire-smoke',
    group,
    width,
    depth,
    height,
    x: position.x,
    y: position.y,
    z: position.z,
    transform: createObjectTransform(),
    locked: false,
    fireSmoke: { ...settings },
  });
}

function updateFireSmokeEffect(effect: FireSmokeEffect, delta: number, time: number) {
  if (!effect.settings.enabled) {
    effect.group.visible = false;
    return;
  }
  effect.group.visible = true;
  const intensity = effect.settings.intensity;
  for (const particle of effect.flameParticles) {
    particle.age = (particle.age + delta) % particle.lifetime;
    const progress = particle.age / particle.lifetime;
    const drift = Math.sin(time * 3.2 + particle.phase);
    particle.sprite.position.set(
      particle.originX + drift * 0.045 * intensity,
      progress * 0.85 * intensity,
      particle.originZ + Math.cos(time * 2.7 + particle.phase) * 0.04 * intensity,
    );
    particle.sprite.scale.set(
      (0.13 + progress * 0.09) * intensity,
      (0.3 + progress * 0.16) * intensity,
      1,
    );
    particle.material.opacity = 0.9 * (1 - progress * 0.68);
    particle.material.color.setRGB(1, 0.22 + (1 - progress) * 0.58, 0.04 + (1 - progress) * 0.12);
  }
  for (const particle of effect.smokeParticles) {
    particle.age = (particle.age + delta) % particle.lifetime;
    const progress = particle.age / particle.lifetime;
    const drift = Math.sin(time * 0.7 + particle.phase);
    const size = (0.16 + progress * 0.38) * intensity;
    particle.sprite.position.set(
      particle.originX + drift * progress * 0.3 * intensity,
      0.12 + progress * 1.65 * intensity,
      particle.originZ + Math.cos(time * 0.6 + particle.phase) * progress * 0.24 * intensity,
    );
    particle.sprite.scale.set(size, size * 0.82, 1);
    particle.material.opacity = 0.3 * (1 - progress) * Math.min(intensity, 1.5);
  }
  effect.light.intensity = 0.7 + intensity * (2.1 + Math.sin(time * 12) * 0.55);
  effect.light.distance = 3 + intensity * 2;
}

function addFireSmokeObject() {
  if (!scene) return;
  fireSmokeError.value = '';
  const index = racks.value.filter((item) => item.kind === 'fire-smoke').length + 1;
  const id = createId();
  const name = `火焰与烟雾 ${String(index).padStart(2, '0')}`;
  const position = findFreePosition(1.2, 1.2);
  const rack = createFireSmokeRack(id, name, { ...position, y: rackFloorY }, { enabled: true, intensity: 1 });
  try {
    createFireSmokeEffect(rack);
    scene.add(rack.group);
    racks.value.push(rack);
    clampRackToRoom(rack);
    selectedId.value = id;
    markSceneDirty();
  } catch (error) {
    const effect = fireSmokeEffects.get(id);
    if (effect) {
      effect.group.parent?.remove(effect.group);
      disposeGroup(effect.group);
      fireSmokeEffects.delete(id);
    }
    scene.remove(rack.group);
    racks.value = racks.value.filter((item) => item.id !== id);
    disposeGroup(rack.group);
    console.error('Unable to create the fire and smoke particle effect.', error);
    fireSmokeError.value = error instanceof Error ? `无法创建火焰与烟雾效果：${error.message}` : '无法创建火焰与烟雾效果。';
  }
}

function updateFireSmokeEnabled(id: string, event: Event) {
  const rack = racks.value.find((item) => item.id === id);
  if (!rack?.fireSmoke) return;
  rack.fireSmoke.enabled = (event.target as HTMLInputElement).checked;
  const effect = fireSmokeEffects.get(id);
  if (effect) effect.group.visible = rack.fireSmoke.enabled;
  markSceneDirty();
}

function updateFireSmokeIntensity(id: string, event: Event) {
  const rack = racks.value.find((item) => item.id === id);
  const input = event.target as HTMLInputElement;
  if (!rack?.fireSmoke || !Number.isFinite(input.valueAsNumber)) return;
  rack.fireSmoke.intensity = clampNumber(input.valueAsNumber, 0.2, 2, rack.fireSmoke.intensity);
  input.value = String(rack.fireSmoke.intensity);
  markSceneDirty();
}

function removeFireSmokeEffect(id: string) {
  const rack = racks.value.find((item) => item.id === id);
  const effect = fireSmokeEffects.get(id);
  if (!rack?.fireSmoke) return;
  if (effect) {
    effect.group.parent?.remove(effect.group);
    disposeGroup(effect.group);
    fireSmokeEffects.delete(id);
  }
  rack.fireSmoke = null;
  markSceneDirty();
}

function clearFireSmokeEffects() {
  for (const effect of fireSmokeEffects.values()) {
    effect.group.parent?.remove(effect.group);
    disposeGroup(effect.group);
  }
  fireSmokeEffects.clear();
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
    y: 0,
    z: position.z,
    transform: createObjectTransform(),
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
  partition.group.position.set(partition.x, partition.y, partition.z);
  applyObjectTransform(partition.group, partition.transform);
  const geometry = new THREE.Group();
  geometry.rotation.y = partition.rotation * Math.PI / 2;
  partition.group.add(geometry);
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
    makeBox(geometry, length, wallBodyHeight, partition.thickness, wallMaterial, 0, wallBodyHeight / 2, 0);
  } else {
    let wallCursor = -halfLength;
    for (const door of partition.doors) {
      const doorCenter = door.doorOffset;
      const doorLeft = doorCenter - openingWidth / 2;
      const doorRight = doorCenter + openingWidth / 2;
      const wallSegmentLength = doorLeft - wallCursor;
      makeBox(
        geometry,
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
        geometry,
        openingWidth,
        upperWallHeight,
        partition.thickness,
        wallMaterial,
        doorCenter,
        doorHeight + doorHeaderHeight + upperWallHeight / 2,
        0,
      );
      makeBox(geometry, openingWidth + partition.thickness, doorHeaderHeight, partition.thickness + 0.025, trimMaterial, doorCenter, doorHeight + doorHeaderHeight / 2, 0);
      for (const side of [-1, 1]) {
        makeBox(geometry, 0.06, doorHeight, partition.thickness + 0.035, trimMaterial, doorCenter + side * openingWidth / 2, doorHeight / 2, 0);
      }

      const hinge = new THREE.Group();
      const mountSide = opensIntoRoom !== door.reverseInstall ? 1 : -1;
      const hingeOnRight = door.hingeSide === 'right';
      const hingeDirection = (hingeOnRight ? 1 : -1) * mountSide;
      const leafDirection = -hingeDirection;
      hinge.position.set(
        hingeDirection > 0 ? doorRight - 0.035 : doorLeft + 0.035,
        0,
        mountSide * (halfThickness + 0.025),
      );
      hinge.name = `Partition door hinge ${door.id}`;
      const leafWidth = openingWidth - 0.07;
      makeBox(hinge, leafWidth, doorHeight - 0.035, 0.055, doorMaterial, leafDirection * leafWidth / 2, (doorHeight - 0.035) / 2, 0);
      makeBox(hinge, 0.025, 0.12, 0.025, trimMaterial, leafDirection * (leafWidth - 0.1), 1.02, 0.04);
      makeBox(hinge, 0.018, doorHeight - 0.18, 0.018, markerMaterial, leafDirection * 0.08, (doorHeight - 0.18) / 2, 0.035);
      hinge.rotation.y = door.doorOpen ? (hingeOnRight ? 1 : -1) * Math.PI / 2 : 0;
      geometry.add(hinge);
      wallCursor = doorRight;
    }

    const finalWallLength = halfLength - wallCursor;
    makeBox(geometry, finalWallLength, wallBodyHeight, partition.thickness, wallMaterial, (wallCursor + halfLength) / 2, wallBodyHeight / 2, 0);
  }

  const baseMaterial = makeMaterial(0x617c83, 0.56, 0.46);
  makeBox(geometry, length, 0.07, partition.thickness + 0.08, baseMaterial, 0, 0.035, 0);
  const capMaterial = makeMaterial(0x9bb7b7, 0.42, 0.36);
  makeBox(geometry, length, capHeight, partition.thickness + 0.025, capMaterial, 0, wallHeight - capHeight / 2, 0);

  geometry.traverse((child) => {
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
    partition.y = 0;
    partition.group.position.set(partition.x, partition.y, partition.z);
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
  partition.group.position.set(partition.x, partition.y, partition.z);
}

function updatePartitionPosition(axis: 'x' | 'y' | 'z', event: Event) {
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

function updateFloorMaterial(event: Event) {
  const input = event.target as HTMLSelectElement;
  if (!isFloorMaterialType(input.value) || !floorPlane) return;
  floorMaterialError.value = '';
  try {
    const nextMaterial = createFloorMaterial(input.value);
    const previousMaterial = floorPlane.material;
    disposeFloorMaterialMap();
    if (Array.isArray(previousMaterial)) previousMaterial.forEach((material) => material.dispose());
    else previousMaterial.dispose();
    floorPlane.material = nextMaterial;
    floorMaterialType.value = input.value;
    markSceneDirty();
  } catch (error) {
    console.error('Unable to apply the selected floor material.', error);
    floorMaterialError.value = '无法切换地板材质，请重试。';
  }
}

function isWallMaterialType(value: string): value is WallMaterialType {
  return value === 'paint' || value === 'concrete' || value === 'steel' || value === 'brick';
}

function isFloorMaterialType(value: unknown): value is FloorMaterialType {
  return value === 'concrete' || value === 'epoxy' || value === 'tile' || value === 'rubber';
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
    reverseInstall: false,
    hingeSide: 'left',
  });
  partition.doors.sort((a, b) => a.doorOffset - b.doorOffset);
  rebuildPartition(partition);
  markSceneDirty();
}

function togglePartitionDoor(doorId: string) {
  const partition = selectedPartition.value;
  const door = partition?.doors.find((item) => item.id === doorId);
  if (!partition || partition.locked || !door) return;
  door.doorOpen = !door.doorOpen;
  rebuildPartition(partition);
  markSceneDirty();
}

function togglePartitionDoorInstallation(doorId: string) {
  const partition = selectedPartition.value;
  const door = partition?.doors.find((item) => item.id === doorId);
  if (!partition || partition.locked || !door) return;
  door.reverseInstall = !door.reverseInstall;
  rebuildPartition(partition);
  markSceneDirty();
}

function setPartitionDoorHingeSide(doorId: string, hingeSide: DoorHingeSide) {
  const partition = selectedPartition.value;
  const door = partition?.doors.find((item) => item.id === doorId);
  if (!partition || partition.locked || !door || (door.hingeSide ?? 'left') === hingeSide) return;
  door.hingeSide = hingeSide;
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
  if (isDeviceKind(rack.kind)) {
    rack.x = rack.group.position.x;
    rack.y = rack.group.position.y;
    rack.z = rack.group.position.z;
    return;
  }
  if (rack.kind !== 'fire-smoke') {
    rack.group.updateWorldMatrix(true, true);
    const bounds = new THREE.Box3().setFromObject(rack.group);
    if (!bounds.isEmpty() && Number.isFinite(bounds.min.y)) {
      rack.group.position.y += rackFloorY - bounds.min.y;
      rack.group.updateWorldMatrix(true, true);
    } else {
      rack.group.position.y = rackFloorY;
    }
  }
  const limitX = Math.max(0, room.width / 2 - rack.width / 2 - 0.08);
  const limitZ = Math.max(0, room.length / 2 - rack.depth / 2 - 0.08);
  rack.group.position.x = THREE.MathUtils.clamp(rack.group.position.x, -limitX, limitX);
  rack.group.position.z = THREE.MathUtils.clamp(rack.group.position.z, -limitZ, limitZ);
  rack.x = rack.group.position.x;
  rack.y = rack.group.position.y;
  rack.z = rack.group.position.z;
}

function selectItem(id: string) {
  if (isPreviewMode.value) return;
  if (racks.value.some((rack) => rack.id === id) || partitions.value.some((partition) => partition.id === id)) selectedId.value = id;
}

function setEditMode() {
  isPreviewMode.value = false;
  if (controls) controls.enabled = true;
}

function setPreviewMode() {
  isPreviewMode.value = true;
  if (activePointerId !== null && renderer?.domElement.hasPointerCapture(activePointerId)) {
    renderer.domElement.releasePointerCapture(activePointerId);
  }
  activeDrag = null;
  activePointerId = null;
  finishContinuousHistoryChange();
  if (controls) controls.enabled = true;
  selectedId.value = null;
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

function updateTransformControlsSelection() {
  if (!transformControls) return;
  const rack = racks.value.find((item) => item.id === selectedId.value);
  if (rack && !rack.locked) {
    transformControls.showX = true;
    transformControls.showY = isDeviceKind(rack.kind) || rack.kind === 'fire-smoke' || transformMode.value !== 'translate';
    transformControls.showZ = true;
    transformControls.attach(rack.group);
    return;
  }
  const partition = partitions.value.find((item) => item.id === selectedId.value);
  if (partition && !partition.locked && !partition.roomWallSide) {
    transformControls.showX = true;
    transformControls.showY = true;
    transformControls.showZ = true;
    transformControls.attach(partition.group);
    return;
  }
  transformControls.detach();
}

function handleTransformObjectChange() {
  if (!transformControls?.object) return;
  const rack = racks.value.find((item) => item.id === selectedId.value);
  if (rack && transformControls.object === rack.group) {
    if (rack.locked) return;
    syncObjectTransform(rack.group, rack.transform);
    clampRackToRoom(rack);
    markSceneDirty();
    return;
  }
  const partition = partitions.value.find((item) => item.id === selectedId.value);
  if (partition && transformControls.object === partition.group && !partition.locked && !partition.roomWallSide) {
    syncObjectTransform(partition.group, partition.transform);
    partition.x = partition.group.position.x;
    partition.y = partition.group.position.y;
    partition.z = partition.group.position.z;
    clampPartitionToRoom(partition);
    markSceneDirty();
  }
}

function handleTransformDraggingChange(event: { value: boolean }) {
  controls.enabled = !event.value;
  if (event.value) beginContinuousHistoryChange();
  else finishContinuousHistoryChange();
}

function removeSelected() {
  if (!selectedRack.value || !scene) return;
  const id = selectedRack.value.id;
  removeFireSmokeEffect(id);
  scene.remove(selectedRack.value.group);
  disposeGroup(selectedRack.value.group);
  racks.value = racks.value.filter((rack) => rack.id !== id);
  for (const annotation of annotations.value.filter((item) => item.itemId === id)) removeAnnotation(annotation.id);
  selectedId.value = racks.value.at(-1)?.id ?? null;
  markSceneDirty();
}

function updatePosition(axis: 'x' | 'y' | 'z', event: Event) {
  const rack = selectedRack.value;
  const input = event.target as HTMLInputElement;
  if (!rack || rack.locked || !Number.isFinite(input.valueAsNumber)) return;
  rack.group.position[axis] = input.valueAsNumber;
  if (isDeviceKind(rack.kind)) {
    rack.x = rack.group.position.x;
    rack.y = rack.group.position.y;
    rack.z = rack.group.position.z;
  } else {
    clampRackToRoom(rack);
  }
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
        y: 0.035,
        z: position.z,
        transform: createObjectTransform(),
        locked: false,
        fireSmoke: null,
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
  selectedAnnotationId.value = annotations.value.at(-1)?.id ?? null;
  annotationTitle.value = '';
  annotationDescription.value = '';
  markSceneDirty();
}

function attachAnnotation(
  rack: RackItem,
  id: string,
  title: string,
  description: string,
  settings: Pick<SceneAnnotation, 'width' | 'height' | 'offset'> = {},
) {
  const width = clampNumber(settings.width ?? 190, 20, 400, 190);
  const height = clampNumber(settings.height ?? 56, 20, 240, 56);
  const offset = {
    x: clampNumber(settings.offset?.x ?? 0, -10, 10, 0),
    y: clampNumber(settings.offset?.y ?? 0, -10, 10, 0),
    z: clampNumber(settings.offset?.z ?? 0, -10, 10, 0),
  };
  const element = document.createElement('div');
  element.className = 'scene-annotation';
  element.style.width = `${width}px`;
  element.style.height = `${height}px`;
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
  element.append( content);
  const label = markRaw(new CSS2DObject(element));
  label.position.set(offset.x, rack.height + 0.36 + offset.y, offset.z);
  rack.group.add(label);
  annotations.value.push({
    id,
    itemId: rack.id,
    title,
    description,
    width,
    height,
    offset,
    label,
  });
}

function selectAnnotation(id: string) {
  const annotation = annotations.value.find((item) => item.id === id);
  if (!annotation) return;
  selectedAnnotationId.value = id;
  selectItem(annotation.itemId);
}

function updateAnnotationSize(id: string, dimension: 'width' | 'height', event: Event) {
  const annotation = annotations.value.find((item) => item.id === id);
  const input = event.target as HTMLInputElement;
  if (!annotation || !Number.isFinite(input.valueAsNumber)) return;
  const min = 20;
  const max = dimension === 'width' ? 400 : 240;
  annotation[dimension] = clampNumber(input.valueAsNumber, min, max, annotation[dimension]);
  input.value = String(annotation[dimension]);
  annotation.label.element.style[dimension] = `${annotation[dimension]}px`;
  markSceneDirty();
}

function updateAnnotationOffset(id: string, axis: TransformAxis, event: Event) {
  const annotation = annotations.value.find((item) => item.id === id);
  const rack = racks.value.find((item) => item.id === annotation?.itemId);
  const input = event.target as HTMLInputElement;
  if (!annotation || !rack || !Number.isFinite(input.valueAsNumber)) return;
  annotation.offset[axis] = clampNumber(input.valueAsNumber, -10, 10, annotation.offset[axis]);
  input.value = String(annotation.offset[axis]);
  annotation.label.position.set(
    annotation.offset.x,
    rack.height + 0.36 + annotation.offset.y,
    annotation.offset.z,
  );
  markSceneDirty();
}

function removeAnnotation(id: string) {
  const annotation = annotations.value.find((item) => item.id === id);
  if (!annotation) return;
  annotation.label.parent?.remove(annotation.label);
  if (selectedAnnotationId.value === id) selectedAnnotationId.value = null;
  annotations.value = annotations.value.filter((item) => item.id !== id);
  markSceneDirty();
}

function createId() {
  return globalThis.crypto?.randomUUID?.() ?? `item-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

function markSceneDirty() {
  saveStatus.value = '未保存';
  if (!historyInitialized) return;
  if (!historyGroupOpen) {
    historyGroupOpen = true;
    const previousSnapshot = historySnapshotPromise;
    if (previousSnapshot) {
      const record = previousSnapshot.then((snapshot) => {
        if (!snapshot) return;
        historyUndoStack.push(snapshot);
        if (historyUndoStack.length > historyLimit) historyUndoStack.shift();
        undoCount.value = Math.min(historyUndoStack.length + historyPendingRecords.length, historyLimit);
      });
      historyPendingRecords.push(record);
      void record.then(() => {
        historyPendingRecords = historyPendingRecords.filter((pending) => pending !== record);
        undoCount.value = Math.min(historyUndoStack.length + historyPendingRecords.length, historyLimit);
      });
    }
  }
  if (historyCaptureTimer !== null) window.clearTimeout(historyCaptureTimer);
  historyCaptureTimer = window.setTimeout(() => {
    historyCaptureTimer = null;
    if (historyInteractionActive) return;
    historyGroupOpen = false;
    void captureHistorySnapshot();
  }, historyCaptureDelay);
}

function beginContinuousHistoryChange() {
  if (historyGroupOpen && !historyInteractionActive) {
    if (historyCaptureTimer !== null) {
      window.clearTimeout(historyCaptureTimer);
      historyCaptureTimer = null;
    }
    historyGroupOpen = false;
    void captureHistorySnapshot();
  }
  historyInteractionActive = true;
  if (historyCaptureTimer !== null) {
    window.clearTimeout(historyCaptureTimer);
    historyCaptureTimer = null;
  }
}

function finishContinuousHistoryChange() {
  if (!historyInteractionActive) return;
  historyInteractionActive = false;
  if (historyCaptureTimer !== null) {
    window.clearTimeout(historyCaptureTimer);
    historyCaptureTimer = null;
  }
  if (historyGroupOpen) {
    historyGroupOpen = false;
    void captureHistorySnapshot();
  }
}

function captureHistorySnapshot() {
  const selectedAtCapture = selectedId.value;
  const capture = serializeScene(historyModelAssetCache)
    .then((document) => {
      historyError.value = '';
      return { document, selectedId: selectedAtCapture };
    })
    .catch((error: unknown) => {
      console.error('Unable to capture an undo history snapshot.', error);
      historyError.value = error instanceof Error ? `无法记录撤销历史：${error.message}` : '无法记录撤销历史。';
      return null;
    });
  historySnapshotPromise = capture;
  return capture;
}

function initializeUndoHistory() {
  historyUndoStack.length = 0;
  historyPendingRecords = [];
  historyGroupOpen = false;
  historyInteractionActive = false;
  undoCount.value = 0;
  historyError.value = '';
  historyInitialized = true;
  void captureHistorySnapshot();
}

function restoreHistoryModelAssetCache(snapshot: HistorySnapshot) {
  historyModelAssetCache.clear();
  for (const rack of snapshot.document.racks) {
    if (rack.kind === 'imported' && rack.modelAsset) {
      historyModelAssetCache.set(rack.id, rack.modelAsset);
    }
  }
}

async function undoScene() {
  if (historyBusy.value || isPreviewMode.value) return;
  historyBusy.value = true;
  finishContinuousHistoryChange();
  activeDrag = null;
  if (activePointerId !== null && renderer.domElement.hasPointerCapture(activePointerId)) {
    renderer.domElement.releasePointerCapture(activePointerId);
  }
  activePointerId = null;
  transformControls?.detach();
  controls.enabled = true;
  if (historyCaptureTimer !== null) {
    window.clearTimeout(historyCaptureTimer);
    historyCaptureTimer = null;
  }
  historyGroupOpen = false;
  try {
    await Promise.all(historyPendingRecords);
    const previousSnapshot = historyUndoStack.at(-1);
    if (!previousSnapshot) return;
    historyUndoStack.pop();
    undoCount.value = historyUndoStack.length;
    const cameraPosition = camera.position.clone();
    const cameraTarget = controls.target.clone();
    restoreHistoryModelAssetCache(previousSnapshot);
    await loadSceneDocument(previousSnapshot.document);
    const selectedExists = previousSnapshot.selectedId
      && (racks.value.some((rack) => rack.id === previousSnapshot.selectedId)
        || partitions.value.some((partition) => partition.id === previousSnapshot.selectedId));
    selectedId.value = selectedExists ? previousSnapshot.selectedId : null;
    camera.position.copy(cameraPosition);
    controls.target.copy(cameraTarget);
    controls.update();
    syncSceneViewFromCamera();
    historySnapshotPromise = Promise.resolve(previousSnapshot);
    historyError.value = '';
    saveStatus.value = '未保存';
  } catch (error) {
    console.error('Unable to undo the last scene change.', error);
    historyError.value = error instanceof Error ? `撤销失败：${error.message}` : '撤销失败。';
  } finally {
    historyBusy.value = false;
    undoCount.value = Math.min(historyUndoStack.length + historyPendingRecords.length, historyLimit);
  }
}

function handleHistoryKeydown(event: KeyboardEvent) {
  if (!(event.ctrlKey || event.metaKey) || event.shiftKey || event.key.toLowerCase() !== 'z') return;
  if (isPreviewMode.value || historyBusy.value || (!historyUndoStack.length && !historyPendingRecords.length)) return;
  const target = event.target;
  if (target instanceof HTMLTextAreaElement || target instanceof HTMLElement && target.isContentEditable) return;
  if (target instanceof HTMLInputElement && ['text', 'search', 'email', 'password', 'url', 'tel'].includes(target.type)) return;
  event.preventDefault();
  void undoScene();
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
  if (value.floorMaterial !== undefined && !isFloorMaterialType(value.floorMaterial)) return false;
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

function isSceneViewSettings(value: unknown): value is SceneViewSettings {
  return isRecord(value)
    && isFiniteInRange(value.distance, 5, 42)
    && isFiniteInRange(value.azimuth, -180, 180)
    && isFiniteInRange(value.elevation, 1, 88)
    && isFiniteInRange(value.targetX, -100, 100)
    && isFiniteInRange(value.targetY, -100, 100)
    && isFiniteInRange(value.targetZ, -100, 100);
}

function isSceneRack(value: unknown): value is SceneRack {
  if (!isRecord(value)) return false;
  if (typeof value.id !== 'string' || typeof value.name !== 'string' || value.name.length > 128) return false;
  if (value.kind !== 'rack' && value.kind !== 'imported' && value.kind !== 'fire-smoke' && !isDeviceKind(value.kind)) return false;
  if (!isFiniteInRange(value.width, 0.01, 40) || !isFiniteInRange(value.depth, 0.01, 40) || !isFiniteInRange(value.height, 0.01, 20)) return false;
  if (!isFiniteInRange(value.x, -100, 100) || !isFiniteInRange(value.z, -100, 100) || typeof value.locked !== 'boolean') return false;
  if (value.y !== undefined && !isFiniteInRange(value.y, -100, 100)) return false;
  if (value.transform !== undefined && !isObjectTransform(value.transform)) return false;
  if (value.fireSmoke !== undefined && !isFireSmokeSettings(value.fireSmoke)) return false;
  if (value.kind === 'fire-smoke' && value.fireSmoke === undefined) return false;
  return value.kind !== 'imported' || isRecord(value.modelAsset);
}

function isFireSmokeSettings(value: unknown): value is FireSmokeSettings {
  return isRecord(value)
    && typeof value.enabled === 'boolean'
    && isFiniteInRange(value.intensity, 0.2, 2);
}

function isScenePartition(value: unknown): value is ScenePartition {
  if (!isRecord(value) || typeof value.id !== 'string' || typeof value.name !== 'string' || value.name.length > 128) return false;
  if (!isFiniteInRange(value.length, 2, 40) || !isFiniteInRange(value.thickness, 0.01, 1)) return false;
  if ((value.rotation !== 0 && value.rotation !== 1) || !isFiniteInRange(value.x, -100, 100) || !isFiniteInRange(value.z, -100, 100)) return false;
  if (value.y !== undefined && !isFiniteInRange(value.y, -100, 100)) return false;
  if (value.transform !== undefined && !isObjectTransform(value.transform)) return false;
  if (typeof value.fullSpan !== 'boolean' || typeof value.locked !== 'boolean') return false;
  if (value.roomWallSide !== null && !isRoomWallSide(value.roomWallSide)) return false;
  if (!isWallMaterialType(String(value.wallMaterialType)) || typeof value.wallColor !== 'string' || !/^#[\da-f]{6}$/i.test(value.wallColor)) return false;
  return Array.isArray(value.doors) && value.doors.every((door) => {
    return isRecord(door)
      && typeof door.id === 'string'
      && typeof door.doorOpen === 'boolean'
      && isFiniteInRange(door.doorOffset, -40, 40)
      && (door.reverseInstall === undefined || typeof door.reverseInstall === 'boolean')
      && (door.hingeSide === undefined || isDoorHingeSide(door.hingeSide));
  });
}

function isObjectTransform(value: unknown): value is ObjectTransform {
  if (!isRecord(value) || !isRecord(value.rotation) || !isRecord(value.scale)) return false;
  return transformAxes.every((axis) => (
    isFiniteInRange(value.rotation[axis], -360, 360)
    && isFiniteInRange(value.scale[axis], 0.1, 10)
  ));
}

function isDoorHingeSide(value: unknown): value is DoorHingeSide {
  return value === 'left' || value === 'right';
}

function isSceneAnnotation(value: unknown): value is SceneAnnotation {
  return isRecord(value)
    && typeof value.id === 'string'
    && typeof value.itemId === 'string'
    && typeof value.title === 'string'
    && value.title.length <= 256
    && typeof value.description === 'string'
    && value.description.length <= 4000
    && (value.width === undefined || isFiniteInRange(value.width, 20, 400))
    && (value.height === undefined || isFiniteInRange(value.height, 20, 240))
    && (value.offset === undefined || isAnnotationOffset(value.offset));
}

function isAnnotationOffset(value: unknown): value is Record<TransformAxis, number> {
  return isRecord(value) && transformAxes.every((axis) => isFiniteInRange(value[axis], -10, 10));
}

function isFiniteInRange(value: unknown, min: number, max: number): value is number {
  return typeof value === 'number' && Number.isFinite(value) && value >= min && value <= max;
}

type LightingNumericKey = 'ambientIntensity' | 'keyIntensity' | 'keyAzimuth' | 'keyElevation' | 'fillIntensity' | 'exposure';
type LightingColorKey = 'ambientColor' | 'keyColor' | 'fillColor';
type LightingToggleKey = 'keyEnabled' | 'fillEnabled' | 'shadowsEnabled';
type CeilingLightingNumericKey = 'spacing' | 'intensity' | 'colorTemperature';
type SceneViewNumericKey = 'distance' | 'azimuth' | 'elevation';

const sceneViewRanges: Record<SceneViewNumericKey, { min: number; max: number }> = {
  distance: { min: 5, max: 42 },
  azimuth: { min: -180, max: 180 },
  elevation: { min: 1, max: 88 },
};

function updateSceneViewNumber(key: SceneViewNumericKey, event: Event) {
  const input = event.currentTarget as HTMLInputElement;
  if (!Number.isFinite(input.valueAsNumber)) return;
  const value = key === 'distance' ? Math.round(input.valueAsNumber * 10) / 10 : Math.round(input.valueAsNumber);
  sceneView[key] = THREE.MathUtils.clamp(value, sceneViewRanges[key].min, sceneViewRanges[key].max);
  applySceneView();
}

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

async function serializeScene(modelAssetCache?: Map<string, Record<string, unknown>>): Promise<SceneDocument> {
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
      y: rack.group.position.y,
      z: rack.group.position.z,
      transform: cloneObjectTransform(rack.transform),
      locked: rack.locked,
      ...(rack.fireSmoke ? { fireSmoke: { ...rack.fireSmoke } } : {}),
    };
    if (rack.kind === 'imported') {
      const cachedAsset = modelAssetCache?.get(rack.id);
      if (cachedAsset) {
        serializedRack.modelAsset = cachedAsset;
      } else {
        const modelRoot = new THREE.Group();
        for (const child of rack.group.children) {
          if (!(child instanceof CSS2DObject) && !child.userData.sceneEffect) modelRoot.add(child.clone(true));
        }
        const exportedModel = await gltfExporter.parseAsync(modelRoot, { embedImages: true });
        if (exportedModel instanceof ArrayBuffer) throw new Error('导出导入模型时返回了无效的数据格式。');
        serializedRack.modelAsset = exportedModel;
        if (modelAssetCache) {
          modelAssetCache.set(rack.id, exportedModel);
          if (modelAssetCache.size > historyLimit) {
            const oldestModelId = modelAssetCache.keys().next().value;
            if (oldestModelId) modelAssetCache.delete(oldestModelId);
          }
        }
      }
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
      y: partition.y,
      z: partition.z,
      transform: cloneObjectTransform(partition.transform),
      fullSpan: partition.fullSpan,
      locked: partition.locked,
      roomWallSide: partition.roomWallSide,
      wallMaterialType: partition.wallMaterialType,
      wallColor: partition.wallColor,
      doors: partition.doors.map((door) => ({ ...door })),
    })),
    annotations: annotations.value.map(({ id, itemId, title, description, width, height, offset }) => ({
      id,
      itemId,
      title,
      description,
      width,
      height,
      offset: { ...offset },
    })),
    lighting: { ...lighting },
    ceilingLighting: { ...ceilingLighting },
    floorMaterial: floorMaterialType.value,
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
      historyModelAssetCache.clear();
      return loadSceneDocument(parsed);
    })
    .then(() => {
      markSceneDirty();
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
    } else if (rackData.kind === 'fire-smoke') {
      group = new THREE.Group();
    } else if (isDeviceKind(rackData.kind)) {
      group = buildDevice(rackData.kind, rackData.name, rackData.id).group;
    } else {
      group = buildRack(rackData.name, rackData.id).group;
    }
    group.name = rackData.name;
    group.userData.itemId = rackData.id;
    const y = rackData.y ?? (isDeviceKind(rackData.kind) ? documentData.room.height + 0.2 : 0.035);
    group.position.set(rackData.x, y, rackData.z);
    const transform = cloneObjectTransform(rackData.transform);
    applyObjectTransform(group, transform);
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
      y,
      z: rackData.z,
      transform,
      locked: rackData.locked,
      fireSmoke: rackData.kind === 'fire-smoke'
        ? { ...(rackData.fireSmoke ?? { enabled: true, intensity: 1 }) }
        : null,
    });
  }

  const loadedPartitions = documentData.partitions.map((partitionData) => reactive({
    ...partitionData,
    y: partitionData.y ?? 0,
    transform: partitionData.roomWallSide ? createObjectTransform() : cloneObjectTransform(partitionData.transform),
    group: markRaw(new THREE.Group()),
    doors: partitionData.doors.map((door) => ({
      ...door,
      reverseInstall: door.reverseInstall ?? false,
      hingeSide: door.hingeSide ?? 'left',
    })),
  }));
  clearFireSmokeEffects();
  for (const item of [...racks.value, ...partitions.value]) {
    scene.remove(item.group);
    disposeGroup(item.group);
  }
  for (const annotation of annotations.value) annotation.label.parent?.remove(annotation.label);
  annotations.value = [];
  selectedAnnotationId.value = null;
  racks.value = [];
  partitions.value = [];
  room.width = documentData.room.width;
  room.length = documentData.room.length;
  room.height = documentData.room.height;
  roomDraft.width = room.width;
  roomDraft.length = room.length;
  roomDraft.height = room.height;
  floorMaterialType.value = documentData.floorMaterial ?? defaultFloorMaterialType;
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
    if (rack.kind === 'fire-smoke' && rack.fireSmoke) createFireSmokeEffect(rack);
  }
  for (const legacyRack of documentData.racks.filter((item) => item.kind !== 'fire-smoke' && item.fireSmoke)) {
    const sourceRack = racks.value.find((item) => item.id === legacyRack.id);
    if (!sourceRack || !legacyRack.fireSmoke) continue;
    sourceRack.group.updateWorldMatrix(true, false);
    const position = sourceRack.group.localToWorld(new THREE.Vector3(0, sourceRack.height * 0.28, 0));
    const index = racks.value.filter((item) => item.kind === 'fire-smoke').length + 1;
    const effect = createFireSmokeRack(
      createId(),
      `火焰与烟雾 ${String(index).padStart(2, '0')}`,
      position,
      legacyRack.fireSmoke,
    );
    scene.add(effect.group);
    racks.value.push(effect);
    clampRackToRoom(effect);
    createFireSmokeEffect(effect);
  }
  for (const annotation of documentData.annotations) {
    const rack = racks.value.find((item) => item.id === annotation.itemId);
    if (rack) attachAnnotation(rack, annotation.id, annotation.title, annotation.description, annotation);
  }
  restoreInitialCameraView();
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

async function loadFreeDeviceModel() {
  const response = await fetch('/models/cassette-player.glb');
  if (!response.ok) throw new Error(`Device model request failed: ${response.status}`);
  const buffer = await response.arrayBuffer();
  const gltf = await new Promise<GLTF>((resolve, reject) => {
    gltfLoader.parse(buffer, '', resolve, reject);
  });
  const model = gltf.scene;
  model.updateMatrixWorld(true);
  const bounds = new THREE.Box3().setFromObject(model);
  const size = bounds.getSize(new THREE.Vector3());
  const maxDimension = Math.max(size.x, size.y, size.z);
  if (!Number.isFinite(maxDimension) || maxDimension <= 0) throw new Error('Device model has invalid dimensions.');
  const scale = 0.24 / maxDimension;
  const center = bounds.getCenter(new THREE.Vector3());
  model.scale.multiplyScalar(scale);
  model.position.set(-center.x * scale, -center.y * scale, -center.z * scale);
  freeDevicePrototype = model;
}

function disposeGroup(group: THREE.Object3D) {
  if (group.userData.sharedAssets) return;
  group.traverse((child) => {
    if (child instanceof THREE.Sprite) {
      child.material.dispose();
    } else if (child instanceof THREE.Mesh || child instanceof THREE.Line || child instanceof THREE.LineSegments) {
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
  if (isPreviewMode.value || event.button !== 0 || !renderer || !camera) return;
  if (transformControls && (transformControls.dragging || transformControls.axis !== null)) return;
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
  if (rack && isDeviceKind(rack.kind)) {
    activeDrag = { kind: 'device', id: rack.id };
    floorPlaneForDrag = new THREE.Plane().setFromNormalAndCoplanarPoint(
      camera.getWorldDirection(new THREE.Vector3()),
      hit.point,
    );
    dragOffset.copy(rack.group.position).sub(hit.point);
  } else {
    activeDrag = rack ? { kind: 'rack', id: rack.id } : partition ? { kind: 'partition', id: partition.id } : null;
    floorPlaneForDrag = new THREE.Plane(new THREE.Vector3(0, 1, 0), -0.035);
  }
  if (activeDrag) beginContinuousHistoryChange();
  controls.enabled = false;
  renderer.domElement.setPointerCapture(event.pointerId);
  activePointerId = event.pointerId;
  moveRack(event);
}

function moveRack(event: PointerEvent) {
  if (isPreviewMode.value || !activeDrag || selectedId.value !== activeDrag.id) return;
  setPointer(event);
  raycaster.setFromCamera(pointer, camera);
  const point = raycaster.ray.intersectPlane(floorPlaneForDrag, new THREE.Vector3());
  if (!point) return;
  const step = 0.1;
  if (activeDrag.kind === 'device') {
    const device = racks.value.find((item) => item.id === activeDrag?.id);
    if (!device || device.locked) return;
    device.group.position.copy(point.add(dragOffset));
    device.x = device.group.position.x;
    device.y = device.group.position.y;
    device.z = device.group.position.z;
    markSceneDirty();
    return;
  }
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
  if (!activeDrag) {
    activePointerId = null;
    return;
  }
  if (activeDrag.kind === 'partition') {
    const partition = partitions.value.find((item) => item.id === activeDrag?.id);
    if (partition) rebuildPartition(partition);
  }
  activeDrag = null;
  activePointerId = null;
  controls.enabled = true;
  finishContinuousHistoryChange();
}

function updateCameraConfigStatus() {
  if (!appliedCameraConfig) {
    cameraConfigStatus.value = '尚未应用';
    return;
  }
  const matchesAppliedConfig = (Object.keys(appliedCameraConfig) as (keyof SceneViewSettings)[])
    .every((key) => sceneView[key] === appliedCameraConfig?.[key]);
  cameraConfigStatus.value = matchesAppliedConfig ? '已应用' : '有未应用的调整';
}

function syncSceneViewFromCamera() {
  if (!camera || !controls) return;
  const offset = camera.position.clone().sub(controls.target);
  const distance = offset.length();
  const azimuth = THREE.MathUtils.radToDeg(Math.atan2(offset.x, offset.z));
  const elevation = THREE.MathUtils.radToDeg(Math.atan2(offset.y, Math.hypot(offset.x, offset.z)));
  const nextView: SceneViewSettings = {
    distance: Number(distance.toFixed(1)),
    azimuth: Math.round(azimuth),
    elevation: Math.round(elevation),
    targetX: Number(THREE.MathUtils.clamp(controls.target.x, -100, 100).toFixed(1)),
    targetY: Number(THREE.MathUtils.clamp(controls.target.y, -100, 100).toFixed(1)),
    targetZ: Number(THREE.MathUtils.clamp(controls.target.z, -100, 100).toFixed(1)),
  };
  const changed = Object.keys(nextView).some((key) => {
    const viewKey = key as keyof SceneViewSettings;
    return sceneView[viewKey] !== nextView[viewKey];
  });
  if (changed) Object.assign(sceneView, nextView);
  if (!applyingSceneView) updateCameraConfigStatus();
}

function applySceneView() {
  if (!camera || !controls) return;
  applyingSceneView = true;
  const azimuth = THREE.MathUtils.degToRad(sceneView.azimuth);
  const elevation = THREE.MathUtils.degToRad(sceneView.elevation);
  const horizontalDistance = sceneView.distance * Math.cos(elevation);
  controls.target.set(sceneView.targetX, sceneView.targetY, sceneView.targetZ);
  camera.position.set(
    sceneView.targetX + horizontalDistance * Math.sin(azimuth),
    sceneView.targetY + sceneView.distance * Math.sin(elevation),
    sceneView.targetZ + horizontalDistance * Math.cos(azimuth),
  );
  controls.update();
  syncSceneViewFromCamera();
  applyingSceneView = false;
  updateCameraConfigStatus();
}

function resetCamera() {
  if (!camera || !controls) return;
  if (appliedCameraConfig) {
    Object.assign(sceneView, appliedCameraConfig);
    applySceneView();
    return;
  }
  applyingSceneView = true;
  camera.position.set(room.width * 1.25, room.height * 2.3, room.length * 1.45);
  controls.target.set(0, room.height * 0.3, 0);
  controls.update();
  syncSceneViewFromCamera();
  applyingSceneView = false;
  updateCameraConfigStatus();
}

function applyCameraSettings() {
  cameraConfigError.value = '';
  syncSceneViewFromCamera();
  try {
    const config = { ...sceneView };
    localStorage.setItem(cameraConfigStorageKey, JSON.stringify(config));
    appliedCameraConfig = config;
    cameraConfigStatus.value = '已应用';
  } catch (error) {
    console.error('Unable to save the initial camera view configuration.', error);
    cameraConfigStatus.value = '应用失败';
    cameraConfigError.value = '保存初始视角配置失败，请检查浏览器存储权限。';
  }
}

function restoreInitialCameraView() {
  cameraConfigError.value = '';
  try {
    const savedConfig = localStorage.getItem(cameraConfigStorageKey);
    if (!savedConfig) {
      appliedCameraConfig = null;
      resetCamera();
      return;
    }
    const parsed: unknown = JSON.parse(savedConfig);
    if (!isSceneViewSettings(parsed)) throw new Error('已保存的初始视角配置格式无效。');
    appliedCameraConfig = { ...parsed };
    Object.assign(sceneView, parsed);
    applySceneView();
    cameraConfigStatus.value = '已应用';
  } catch (error) {
    console.error('Unable to restore the initial camera view configuration.', error);
    appliedCameraConfig = null;
    cameraConfigError.value = error instanceof Error ? error.message : '读取初始视角配置失败。';
    resetCamera();
    cameraConfigStatus.value = '配置无效，已恢复默认视角';
  }
}

function handleCameraChange() {
  syncSceneViewFromCamera();
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

function animate(timestamp = performance.now()) {
  animationFrame = window.requestAnimationFrame(animate);
  const delta = lastAnimationTimestamp === 0 ? 0 : Math.min((timestamp - lastAnimationTimestamp) / 1000, 0.05);
  lastAnimationTimestamp = timestamp;
  const time = timestamp / 1000;
  for (const effect of fireSmokeEffects.values()) updateFireSmokeEffect(effect, delta, time);
  controls.update();
  selectionOutline?.update();
  renderer.render(scene, camera);
  labelRenderer.render(scene, camera);
}

onMounted(async () => {
  await nextTick();
  if (!viewport.value) return;
  window.addEventListener('keydown', handleHistoryKeydown);
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
  syncSceneViewFromCamera();
  controls.addEventListener('change', handleCameraChange);
  transformControls = new TransformControls(camera, renderer.domElement);
  transformControls.setMode(transformMode.value);
  transformControls.addEventListener('dragging-changed', handleTransformDraggingChange);
  transformControls.addEventListener('objectChange', handleTransformObjectChange);
  scene.add(transformControls.getHelper());
  restoreInitialCameraView();

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
  try {
    await loadFreeDeviceModel();
    assetStatus.value = '设备使用本地提供的临时模型';
  } catch (error) {
    assetError.value = '无法加载设备临时模型，将使用内置几何模型。';
    console.error('Unable to load the provided device model.', error);
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
    restoreInitialCameraView();
  }
  selectedId.value = null;
  initializeUndoHistory();
});

onBeforeUnmount(() => {
  if (animationFrame) window.cancelAnimationFrame(animationFrame);
  clearFireSmokeEffects();
  fireParticleTexture?.dispose();
  smokeParticleTexture?.dispose();
  fireParticleTexture = null;
  smokeParticleTexture = null;
  if (historyCaptureTimer !== null) window.clearTimeout(historyCaptureTimer);
  window.removeEventListener('keydown', handleHistoryKeydown);
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
  controls?.removeEventListener('change', handleCameraChange);
  controls?.dispose();
  if (transformControls) {
    transformControls.removeEventListener('dragging-changed', handleTransformDraggingChange);
    transformControls.removeEventListener('objectChange', handleTransformObjectChange);
    transformControls.detach();
    scene.remove(transformControls.getHelper());
    transformControls.dispose();
    transformControls = null;
  }
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
.mode-switch { height: 34px; display: flex; align-items: center; gap: 2px; padding: 2px; border: 1px solid rgba(255,255,255,.12); border-radius: 4px; background: rgba(0,0,0,.12); }
.mode-switch button { height: 28px; padding: 0 10px; border: 0; border-radius: 3px; background: transparent; color: #91a1a8; font-size: 10px; cursor: pointer; }
.mode-switch button:hover { color: #dce9e8; }
.mode-switch button.active { background: rgba(97,194,194,.17); color: #9de0da; box-shadow: inset 0 0 0 1px rgba(131,217,212,.18); }
.top-button { height: 34px; border-radius: 4px; padding: 0 12px; font-size: 11px; border: 1px solid transparent; cursor: pointer; transition: .18s ease; }
.top-button:disabled { opacity: .46; cursor: not-allowed; }
.undo-button small { margin-left: 4px; color: #788b91; font-size: 8px; }
.top-button.subtle { background: transparent; color: #a7b3bb; border-color: rgba(255,255,255,.12); }
.top-button.subtle:hover { color: #eaf5f3; border-color: rgba(131,217,212,.45); }
.top-button.primary { background: #67bebb; color: #102021; font-weight: 700; padding: 0 14px; }
.top-button.primary:hover { background: #8bd5cf; }
.top-button.partition-top-button { background: transparent; color: #a9c7c7; border-color: rgba(129, 177, 180, .2); }
.top-button.partition-top-button:hover { color: #d3efea; border-color: rgba(131,217,212,.45); background: rgba(91,174,170,.08); }
.button-icon { font-size: 16px; vertical-align: -1px; margin-right: 4px; }
.plus { font-size: 15px; margin-right: 4px; }

.workspace { flex: 1; min-height: 0; display: grid; grid-template-columns: 258px minmax(420px, 1fr) 300px; }
.workspace.preview-workspace { grid-template-columns: minmax(0, 1fr); }
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
.door-reverse-button { margin-top: 8px; }
.door-swing-control { margin-top: 10px; }
.door-swing-control > span { display: flex; align-items: center; justify-content: space-between; color: #a7b5b9; font-size: 8px; }
.door-swing-control > span small { color: #71838a; font-size: 7px; }
.door-swing-options { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 7px; margin-top: 6px; }
.door-swing-options button { min-width: 0; height: 29px; border: 1px solid rgba(172,199,211,.11); border-radius: 3px; background: rgba(255,255,255,.02); color: #899a9f; font-size: 8px; cursor: pointer; }
.door-swing-options button[aria-pressed="true"] { border-color: rgba(107,190,185,.38); background: rgba(86,160,157,.14); color: #a2d9d4; }
.door-swing-options button:disabled { opacity: .48; cursor: not-allowed; }
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
.transform-properties { padding-bottom: 12px; }
.transform-mode-switch { display: grid; grid-template-columns: repeat(3, 1fr); gap: 4px; margin: -2px 0 12px; padding: 3px; border: 1px solid rgba(172,199,211,.1); border-radius: 3px; background: #11191f; }
.transform-mode-switch button { height: 27px; padding: 0; border: 0; border-radius: 2px; background: transparent; color: #87979d; font-size: 9px; cursor: pointer; }
.transform-mode-switch button:hover { color: #dce9e8; }
.transform-mode-switch button.active { background: rgba(97,194,194,.16); color: #9de0da; box-shadow: inset 0 0 0 1px rgba(131,217,212,.16); }
.transform-value-heading { display: flex; align-items: center; gap: 5px; margin: 8px 0 6px; color: #91a0a9; font-size: 9px; }
.transform-value-heading small { margin-left: auto; color: #687b84; font-size: 8px; }
.transform-value-grid { gap: 5px; }
.transform-value-grid label { height: 31px; }
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
.camera-view-actions { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 7px; }
.camera-view-actions .annotation-add { padding: 0 3px; font-size: 8px; white-space: nowrap; }
.fire-smoke-panel > .section-title { margin-bottom: 10px; }
.fire-smoke-help { margin: -4px 0 9px; color: #71838a; font-size: 8px; line-height: 1.5; }
.fire-smoke-panel .fire-smoke-intensity { margin-top: 10px; }
.remove-fire-smoke { margin-top: 10px; padding: 0; color: #b88d82; border: 0; background: transparent; font-size: 8px; cursor: pointer; }
.remove-fire-smoke:hover { color: #e19488; }
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
.annotation-settings { margin-top: 10px; padding: 10px; border: 1px solid rgba(112,193,188,.18); background: rgba(9,15,19,.3); border-radius: 3px; }
.annotation-settings-title { margin-bottom: 8px; color: #a9c5c6; font-size: 9px; }
.annotation-settings-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 7px; }
.annotation-settings-grid label { display: flex; flex-direction: column; gap: 4px; min-width: 0; color: #778990; font-size: 8px; }
.annotation-settings-grid input { width: 100%; min-width: 0; height: 27px; padding: 0 6px; color: #d7e2e3; background: #11191f; border: 1px solid rgba(172,199,211,.12); border-radius: 3px; outline: 0; font-size: 9px; }
.annotation-settings-grid input:focus { border-color: rgba(113,197,192,.5); }
.annotation-settings-title.offset-title { margin-top: 12px; }
.annotation-settings-grid.offset-grid { grid-template-columns: repeat(3, minmax(0, 1fr)); }
.annotation-settings .annotation-tip { margin-bottom: 0; }
.annotation-empty { display: flex; gap: 8px; align-items: flex-start; margin-top: 11px; padding: 9px; background: rgba(255,255,255,.02); border-radius: 3px; }
.annotation-empty > span { color: #6eaaa8; font-size: 13px; }
.annotation-empty p { margin: 0; color: #77858a; font-size: 8px; line-height: 1.6; }
.right-footer { margin-top: auto; padding: 12px 18px; border-top: 1px solid var(--line); color: #71828a; font-size: 8px; letter-spacing: .4px; }
.right-footer > span { color: #78c5bf; margin-right: 5px; }.right-footer i { float: right; font-style: normal; color: #596a71; }

.scene-annotation { position: relative; display: flex; align-items: flex-start; gap: 6px; box-sizing: border-box; min-width: 20px; max-width: none; overflow: visible; transform: translate(-50%, -100%); background: rgba(17, 29, 34, .94); color: #dce8e7; padding: 6px 9px 6px 7px; border: 1px solid rgba(121,205,198,.55); border-radius: 4px; box-shadow: 0 5px 17px rgba(0,0,0,.28); white-space: normal; }
.scene-annotation::after { content: ""; position: absolute; bottom: -5px; left: 50%; width: 8px; height: 8px; background: #17272b; border-right: 1px solid rgba(121,205,198,.55); border-bottom: 1px solid rgba(121,205,198,.55); transform: translateX(-50%) rotate(45deg); }
.scene-annotation-pin { color: #80d5ca; font-size: 6px; line-height: 1.1; }
.scene-annotation-content { display: flex; flex: 1; flex-direction: column; gap: 3px; min-width: 0; }
.scene-annotation-content strong { font-size: 8px; font-weight: 600; }
.scene-annotation-content small { max-width: 100%; color: #a8b8b8; font-size: 8px; line-height: 1.4; }

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
