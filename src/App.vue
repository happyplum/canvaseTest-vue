<template>
  <div class="app">
    <aside class="palette">
      <h3>组件库</h3>
      <div
        class="item"
        draggable="true"
        @dragstart="(e) => onDragStart('circle', e)"
      >
        画圆
      </div>
      <div
        class="item"
        draggable="true"
        @dragstart="(e) => onDragStart('rect', e)"
      >
        画方
      </div>
      <p class="hint">将上方元素拖拽到右侧画布</p>
    </aside>
    <main class="board" ref="board" :class="{ 'is-panning': panDragging }">
      <canvas
        ref="canvas"
        @dragover.prevent
        @drop="onDrop"
        @wheel.prevent="onWheel"
      ></canvas>
    </main>
    <aside class="props">
      <h3>属性</h3>
      <div class="row">
        <button class="reset-btn" @click="resetView">重置点位</button>
        <span class="hint">{{ scale.toFixed(2) }}x</span>
      </div>
      <div v-if="selectedShape">
        <div class="row">类型：{{ getShapeType(selectedShape.type) }}</div>
        <label class="row">
          标签：
          <input v-model="selectedShape.label" @input="draw" />
        </label>
        <label class="row">
          X：
          <input type="number" v-model.number="selectedShape.x" @input="draw" />
        </label>
        <label class="row">
          Y：
          <input type="number" v-model.number="selectedShape.y" @input="draw" />
        </label>
        <p class="hint">提示：按住图形可在画布中拖动</p>
      </div>
      <div v-else class="empty">点击画布中图形以编辑标签</div>
    </aside>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onBeforeUnmount } from "vue";

type ShapeType = "circle" | "rect";
type Shape = {
  id: number;
  type: ShapeType;
  x: number;
  y: number;
  radius?: number;
  size?: number;
  label: string;
};

const canvas = ref<HTMLCanvasElement | null>(null);
const board = ref<HTMLElement | null>(null);
const ctx = ref<CanvasRenderingContext2D | null>(null);
const shapes = reactive<Shape[]>([]);
const selectedId = ref<number | null>(null);
const dragging = ref(false);
const dragOffset = ref({ dx: 0, dy: 0 });
const scale = ref(1);
const pan = ref({ x: 0, y: 0 });
const panDragging = ref(false);
const panDragStart = ref({ x: 0, y: 0, panX: 0, panY: 0 });

let nextId = 1;

const selectedShape = computed<Shape | null>(
  () => shapes.find((s) => s.id === selectedId.value) ?? null,
);

function draw() {
  const c = ctx.value;
  const cvs = canvas.value;
  if (!c || !cvs) return;
  // Reset transform and clear canvas
  c.setTransform(1, 0, 0, 1, 0, 0);
  c.clearRect(0, 0, cvs.width, cvs.height);
  // Apply scale and pan (translate)
  c.setTransform(scale.value, 0, 0, scale.value, pan.value.x, pan.value.y);
  c.textAlign = "center";
  c.textBaseline = "middle";
  c.font = "14px system-ui, -apple-system, Segoe UI, Roboto, sans-serif";
  for (const s of shapes) {
    switch (s.type) {
      case "circle":
        c.beginPath();
        const r = s.radius ?? 40;
        c.arc(s.x, s.y, r, 0, Math.PI * 2);
        c.fillStyle = selectedId.value === s.id ? "#4f46e5" : "#60a5fa";
        c.fill();
        c.strokeStyle = "#1f2937";
        c.lineWidth = selectedId.value === s.id ? 2 : 1;
        c.stroke();
        c.fillStyle = "#ffffff";
        c.fillText(s.label, s.x, s.y);
        break;
      case "rect":
        const sz = s.size ?? 80;
        const left = s.x - sz / 2;
        const top = s.y - sz / 2;
        c.beginPath();
        c.rect(left, top, sz, sz);
        c.fillStyle = selectedId.value === s.id ? "#16a34a" : "#34d399";
        c.fill();
        c.strokeStyle = "#1f2937";
        c.lineWidth = selectedId.value === s.id ? 2 : 1;
        c.stroke();
        c.fillStyle = "#ffffff";
        c.fillText(s.label, s.x, s.y);
        break;
      default:
    }
  }
}
function getShapeType(type: ShapeType) {
  switch (type) {
    case "circle":
      return "圆";
    case "rect":
      return "方";
    default:
      return "未知";
  }
}
function addShape(type: ShapeType, x: number, y: number) {
  const id = nextId++;
  const label = getShapeType(type);
  const shape: Shape = { id, type, x, y, label };
  switch (type) {
    case "circle":
      shape.radius = 40;
      break;
    case "rect":
      shape.size = 80;
      break;
    default:
      break;
  }
  shapes.push(shape);
  selectedId.value = id;
  draw();
}

function hitTest(x: number, y: number): Shape | null {
  for (let i = shapes.length - 1; i >= 0; i--) {
    const s = shapes[i];
    if (!s) return null;
    switch (s.type) {
      case "circle":
        const r = s.radius ?? 40;
        const dx = x - s.x;
        const dy = y - s.y;
        if (dx * dx + dy * dy <= r * r) return s;
        break;
      case "rect":
        const sz = s.size ?? 80;
        const left = s.x - sz / 2;
        const top = s.y - sz / 2;
        if (x >= left && x <= left + sz && y >= top && y <= top + sz) return s;
        break;
      default:
        break;
    }
  }
  return null;
}

function getCanvasPos(e: MouseEvent | DragEvent) {
  const rect = canvas.value!.getBoundingClientRect();
  const x = ("clientX" in e ? e.clientX : 0) - rect.left;
  const y = ("clientY" in e ? e.clientY : 0) - rect.top;
  return { x, y };
}

function screenToWorld(pos: { x: number; y: number }) {
  return {
    x: (pos.x - pan.value.x) / scale.value,
    y: (pos.y - pan.value.y) / scale.value,
  };
}

function onDrop(e: DragEvent) {
  e.preventDefault();
  const type =
    e.dataTransfer?.getData("shape") || e.dataTransfer?.getData("text/plain");
  if (!type) return;
  const { x, y } = screenToWorld(getCanvasPos(e));
  addShape(type as ShapeType, x, y);
}

function onMouseDown(e: MouseEvent) {
  const { x, y } = screenToWorld(getCanvasPos(e));
  const s = hitTest(x, y);
  if (s) {
    selectedId.value = s.id;
    dragging.value = true;
    dragOffset.value.dx = x - s.x;
    dragOffset.value.dy = y - s.y;
    draw();
  } else {
    selectedId.value = null;
    panDragging.value = true;
    panDragStart.value = {
      x: (e as MouseEvent).clientX,
      y: (e as MouseEvent).clientY,
      panX: pan.value.x,
      panY: pan.value.y,
    };
  }
}

function onMouseMove(e: MouseEvent) {
  if (panDragging.value) {
    const dx = e.clientX - panDragStart.value.x;
    const dy = e.clientY - panDragStart.value.y;
    pan.value.x = panDragStart.value.panX + dx;
    pan.value.y = panDragStart.value.panY + dy;
    draw();
    return;
  }
  if (!dragging.value || selectedId.value == null) return;
  const s = shapes.find((sh) => sh.id === selectedId.value);
  if (!s) return;
  const { x, y } = screenToWorld(getCanvasPos(e));
  s.x = x - dragOffset.value.dx;
  s.y = y - dragOffset.value.dy;
  draw();
}

function onMouseUp() {
  dragging.value = false;
  panDragging.value = false;
}

function onDragStart(type: ShapeType, e: DragEvent) {
  e.dataTransfer?.setData("shape", type);
  e.dataTransfer?.setData("text/plain", type);
}

function resizeCanvas() {
  const el = board.value!;
  const cvs = canvas.value!;
  const w = el.clientWidth;
  const h = el.clientHeight;
  cvs.width = w;
  cvs.height = h;
  draw();
}

function onWheel(e: WheelEvent) {
  // 鼠标为缩放中心：计算当前鼠标的世界坐标，更新缩放后调整平移使其保持在原位置
  const mouse = getCanvasPos(e);
  const world = screenToWorld(mouse);
  const factor = e.deltaY < 0 ? 1.1 : 0.9;
  const next = scale.value * factor;
  scale.value = Number(next.toFixed(3));
  pan.value.x = mouse.x - world.x * scale.value;
  pan.value.y = mouse.y - world.y * scale.value;
  draw();
}

function onKeyDown(e: KeyboardEvent) {
  if (e.key !== "Delete" && e.key !== "Backspace") return;
  const target = e.target as HTMLElement | null;
  if (
    target &&
    (target.tagName === "INPUT" ||
      target.tagName === "TEXTAREA" ||
      target.isContentEditable)
  ) {
    return;
  }
  if (selectedId.value == null) return;
  const idx = shapes.findIndex((s) => s.id === selectedId.value);
  if (idx >= 0) {
    shapes.splice(idx, 1);
    selectedId.value = null;
    dragging.value = false;
    draw();
  }
}

function resetView() {
  scale.value = 1;
  pan.value.x = 0;
  pan.value.y = 0;
  draw();
}

onMounted(() => {
  ctx.value = canvas.value!.getContext("2d");
  resizeCanvas();
  canvas.value!.addEventListener("mousedown", onMouseDown);
  window.addEventListener("mousemove", onMouseMove);
  window.addEventListener("mouseup", onMouseUp);
  window.addEventListener("resize", resizeCanvas);
  window.addEventListener("keydown", onKeyDown);
  draw();
});

onBeforeUnmount(() => {
  canvas.value?.removeEventListener("mousedown", onMouseDown);
  window.removeEventListener("mousemove", onMouseMove);
  window.removeEventListener("mouseup", onMouseUp);
  window.removeEventListener("resize", resizeCanvas);
  window.removeEventListener("keydown", onKeyDown);
});
</script>

<style scoped>
.app {
  display: grid;
  grid-template-columns: 220px 1fr 280px;
  grid-template-rows: 100%;
  height: 100vh;
  font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
  color: #111827;
}
.palette {
  border-right: 1px solid #e5e7eb;
  padding: 12px;
  background: #f9fafb;
}
.palette h3,
.props h3 {
  font-size: 16px;
  margin: 0 0 10px 0;
  color: #374151;
}
.item {
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 10px 12px;
  margin-bottom: 10px;
  cursor: grab;
  user-select: none;
}
.item:active {
  cursor: grabbing;
}
.hint {
  font-size: 12px;
  color: #6b7280;
  margin-top: 8px;
}
.board {
  position: relative;
  background: #ffffff;
}
.board canvas {
  width: 100%;
  height: 100%;
  display: block;
  cursor: grab;
  background: linear-gradient(90deg, #f3f4f6 1px, transparent 1px) 0 0 / 24px
      24px,
    linear-gradient(#f3f4f6 1px, transparent 1px) 0 0 / 24px 24px;
}
.is-panning canvas {
  cursor: grabbing;
}
.props {
  border-left: 1px solid #e5e7eb;
  padding: 12px;
  background: #f9fafb;
}
.row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}
.row input {
  flex: 1;
  padding: 6px 8px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  outline: none;
}
.reset-btn {
  padding: 6px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
  color: #000;
}
.reset-btn:hover {
  border-color: #9ca3af;
}
.empty {
  color: #6b7280;
  font-size: 13px;
}
</style>
