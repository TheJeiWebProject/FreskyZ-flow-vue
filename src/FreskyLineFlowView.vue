<template>
  <div
    ref="viewportEl"
    class="fresky-flow"
    :class="{ 'fresky-flow--panning': isPanning }"
    @click="onBackgroundClick"
    @wheel.prevent="onWheel"
    @pointerdown="onPointerDown"
  >
    <div class="fresky-flow__toolbar">
      <button class="fresky-flow__tool-btn" @click.stop="zoomOut">-</button>
      <button class="fresky-flow__tool-btn" @click.stop="zoomIn">+</button>
      <button class="fresky-flow__tool-btn fresky-flow__tool-btn--reset" @click.stop="resetView">1:1</button>
      <span class="fresky-flow__zoom-text">{{ Math.round(zoom * 100) }}%</span>
    </div>

    <div class="fresky-flow__stage" :style="stageStyle">
      <div
        class="fresky-flow__board"
        :style="{ width: `${boardWidth}px`, height: `${boardHeight}px` }"
      >
        <svg class="fresky-flow__edges" :viewBox="`0 0 ${boardWidth} ${boardHeight}`" preserveAspectRatio="none">
          <defs>
            <marker
              v-for="edge in edgeViews"
              :key="edge.markerId"
              :id="edge.markerId"
              markerWidth="9"
              markerHeight="9"
              refX="8"
              refY="4.5"
              orient="auto"
              markerUnits="strokeWidth"
            >
              <path d="M0,0 L9,4.5 L0,9 z" :fill="edge.stroke" />
            </marker>
          </defs>
          <g v-for="edge in edgeViews" :key="edge.id">
            <path
              :d="edge.path"
              fill="none"
              :stroke="edge.stroke"
              :stroke-width="edge.lineWidth"
              :stroke-opacity="edge.opacity"
              :stroke-dasharray="edge.strokeDasharray"
              stroke-linecap="round"
              :marker-end="`url(#${edge.markerId})`"
            />
          </g>
        </svg>

        <div
          v-for="edge in edgeViews"
          :key="`${edge.id}:label`"
          class="fresky-flow__edge-label"
          :style="{ left: `${edge.labelX}px`, top: `${edge.labelY}px` }"
        >
          {{ edge.label }}
        </div>

        <div
          v-for="node in nodeViews"
          :key="node.id"
          class="fresky-node"
          :class="[
            `fresky-node--${node.kind}`,
            { 'fresky-node--selected': props.selectedNodeId === node.id, 'fresky-node--surplus': node.isSurplus },
          ]"
          :style="{
            left: `${node.x}px`,
            top: `${node.y}px`,
            width: `${node.width}px`,
            height: `${node.height}px`,
            '--fresky-accent': node.accent,
          }"
          @click.stop="emit('update:selected-node-id', node.id)"
        >
          <template v-if="node.kind === 'machine'">
            <div class="fresky-node__machine-inner">
              <div class="fresky-node__machine-icon-wrap">
                <img v-if="node.iconUrl" class="fresky-node__machine-icon" :src="node.iconUrl" :alt="node.title" />
                <div v-else class="fresky-node__machine-icon-fallback">M</div>
              </div>
              <div class="fresky-node__machine-meta">
                <div class="fresky-node__machine-title">{{ node.title }}</div>
                <div class="fresky-node__machine-sub">{{ node.subtitle }}</div>
              </div>
            </div>
          </template>

          <template v-else>
            <div
              class="fresky-node__icon-wrap"
              @click.stop="onNodeIconClick(node)"
              @mouseenter="onNodeHoverEnter(node)"
              @mouseleave="emit('item-leave')"
            >
              <img v-if="node.iconUrl" class="fresky-node__icon" :src="node.iconUrl" :alt="node.title" />
              <div v-else class="fresky-node__icon-fallback">{{ node.shortName }}</div>
            </div>
            <div class="fresky-node__title">{{ node.title }}</div>
            <div class="fresky-node__sub">{{ node.subtitle }}</div>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, ref, watch } from 'vue';

type ItemKeyLike = {
  id: string;
  meta?: number | string;
  nbt?: unknown;
};

type ItemDefLike = {
  key?: { id?: string };
  icon?: string;
  iconSprite?: {
    url?: string;
    color?: string;
  };
  rarity?: {
    color?: string;
  };
};

type FlowNode = {
  id: string;
  type?: string;
  position?: {
    x?: number;
    y?: number;
  };
  data?: Record<string, unknown>;
};

type FlowEdge = {
  id: string;
  source: string;
  target: string;
  label?: unknown;
  style?: unknown;
};

type NodeView = {
  id: string;
  kind: 'item' | 'machine' | 'fluid';
  title: string;
  subtitle: string;
  x: number;
  y: number;
  width: number;
  height: number;
  iconUrl: string;
  shortName: string;
  itemKey: ItemKeyLike | null;
  accent: string;
  isSurplus: boolean;
};

type EdgeView = {
  id: string;
  markerId: string;
  path: string;
  lineWidth: number;
  stroke: string;
  opacity: number;
  strokeDasharray: string;
  label: string;
  labelX: number;
  labelY: number;
};

const props = defineProps<{
  nodes: FlowNode[];
  edges: FlowEdge[];
  selectedNodeId: string | null;
  itemDefsByKeyHash: Record<string, ItemDefLike>;
}>();

const emit = defineEmits<{
  (e: 'update:selected-node-id', id: string | null): void;
  (e: 'item-click', itemKey: ItemKeyLike): void;
  (e: 'item-hover', itemKey: ItemKeyLike): void;
  (e: 'item-leave'): void;
}>();

function finiteOr(v: unknown, fallback: number): number {
  const n = typeof v === 'number' ? v : Number(v);
  return Number.isFinite(n) ? n : fallback;
}

function nodeSize(type: string | undefined): { width: number; height: number } {
  if (type === 'lineMachineNode') return { width: 188, height: 84 };
  if (type === 'lineFluidNode') return { width: 96, height: 116 };
  return { width: 96, height: 116 };
}

function parseDash(value: unknown): string {
  if (typeof value === 'string') return value.trim();
  if (Array.isArray(value)) {
    return value
      .map((v) => finiteOr(v, 0))
      .filter((v) => v > 0)
      .join(' ');
  }
  return '';
}

function pickItemDefById(itemId: string | undefined): ItemDefLike | undefined {
  if (!itemId) return undefined;
  const defs = Object.values(props.itemDefsByKeyHash);
  return defs.find((d) => d?.key?.id === itemId);
}

function itemAccentColor(itemId: string | undefined): string {
  const def = pickItemDefById(itemId);
  return def?.rarity?.color || def?.iconSprite?.color || '#9ca3af';
}

function itemIcon(itemId: string | undefined): string {
  const def = pickItemDefById(itemId);
  if (!def) return '';
  if (typeof def.icon === 'string' && def.icon.trim()) return def.icon.trim();
  if (typeof def.iconSprite?.url === 'string' && def.iconSprite.url.trim()) return def.iconSprite.url.trim();
  return '';
}

function styleObject(value: unknown): Record<string, unknown> {
  if (value && typeof value === 'object' && !Array.isArray(value)) {
    return value as Record<string, unknown>;
  }
  return {};
}

function initialChar(text: string, fallback: string): string {
  const t = text.trim();
  if (t.length > 0) return t.slice(0, 1).toUpperCase();
  const f = fallback.trim();
  if (f.length > 0) return f.slice(0, 1).toUpperCase();
  return '?';
}

function clamp(value: number, min: number, max: number): number {
  return Math.max(min, Math.min(max, value));
}

function cubicPointAt(
  t: number,
  p0: { x: number; y: number },
  p1: { x: number; y: number },
  p2: { x: number; y: number },
  p3: { x: number; y: number },
) {
  const mt = 1 - t;
  const mt2 = mt * mt;
  const t2 = t * t;
  const x = (mt2 * mt * p0.x) + (3 * mt2 * t * p1.x) + (3 * mt * t2 * p2.x) + (t2 * t * p3.x);
  const y = (mt2 * mt * p0.y) + (3 * mt2 * t * p1.y) + (3 * mt * t2 * p2.y) + (t2 * t * p3.y);
  return { x, y };
}

const rawNodeViews = computed(() => {
  return props.nodes.map((node) => {
    const data = node.data ?? {};
    const kind = node.type === 'lineMachineNode' ? 'machine' : node.type === 'lineFluidNode' ? 'fluid' : 'item';
    const { width, height } = nodeSize(node.type);
    const itemKeyCandidate = data.itemKey as ItemKeyLike | undefined;
    const itemKey = itemKeyCandidate?.id ? itemKeyCandidate : undefined;
    const machineItemId = typeof data.machineItemId === 'string' ? data.machineItemId : undefined;
    const itemId = itemKey?.id || machineItemId;
    const subtitleRaw = typeof data.subtitle === 'string' ? data.subtitle : '';
    return {
      id: node.id,
      kind,
      title: typeof data.title === 'string' ? data.title : node.id,
      subtitle: subtitleRaw,
      x: finiteOr(node.position?.x, 0),
      y: finiteOr(node.position?.y, 0),
      width,
      height,
      iconUrl: kind === 'machine' ? '' : itemIcon(itemId),
      shortName: initialChar(typeof data.title === 'string' ? data.title : '', node.id),
      itemKey: itemKey ?? null,
      accent: kind === 'machine' ? '#6b7280' : itemAccentColor(itemId),
      isSurplus: !!data.isSurplus,
    } satisfies NodeView;
  });
});

const bounds = computed(() => {
  const nodes = rawNodeViews.value;
  if (!nodes.length) {
    return { minX: 0, minY: 0, maxX: 960, maxY: 540 };
  }
  let minX = Number.POSITIVE_INFINITY;
  let minY = Number.POSITIVE_INFINITY;
  let maxX = Number.NEGATIVE_INFINITY;
  let maxY = Number.NEGATIVE_INFINITY;
  nodes.forEach((node) => {
    minX = Math.min(minX, node.x);
    minY = Math.min(minY, node.y);
    maxX = Math.max(maxX, node.x + node.width);
    maxY = Math.max(maxY, node.y + node.height);
  });
  return { minX, minY, maxX, maxY };
});

const boardPadding = 48;
const boardOffsetX = computed(() => boardPadding - bounds.value.minX);
const boardOffsetY = computed(() => boardPadding - bounds.value.minY);
const boardWidth = computed(() => Math.max(960, Math.ceil(bounds.value.maxX - bounds.value.minX + boardPadding * 2)));
const boardHeight = computed(() => Math.max(520, Math.ceil(bounds.value.maxY - bounds.value.minY + boardPadding * 2)));

const nodeViews = computed(() =>
  rawNodeViews.value.map((n) => ({
    ...n,
    x: n.x + boardOffsetX.value,
    y: n.y + boardOffsetY.value,
  })),
);

const nodeViewById = computed(() => new Map(nodeViews.value.map((node) => [node.id, node] as const)));

const edgeViews = computed<EdgeView[]>(() => {
  return props.edges
    .map((edge, index) => {
      const source = nodeViewById.value.get(edge.source);
      const target = nodeViewById.value.get(edge.target);
      if (!source || !target) return null;

      const leftToRight = source.x <= target.x;
      const startX = leftToRight ? source.x + source.width - 8 : source.x + 8;
      const endX = leftToRight ? target.x + 8 : target.x + target.width - 8;
      const startY = source.y + source.height * 0.5;
      const endY = target.y + target.height * 0.5;
      const dx = Math.max(42, Math.abs(endX - startX) * 0.35);
      const cp1 = { x: startX + (leftToRight ? dx : -dx), y: startY };
      const cp2 = { x: endX - (leftToRight ? dx : -dx), y: endY };
      const path = `M ${startX} ${startY} C ${cp1.x} ${cp1.y}, ${cp2.x} ${cp2.y}, ${endX} ${endY}`;
      const mid = cubicPointAt(0.5, { x: startX, y: startY }, cp1, cp2, { x: endX, y: endY });

      const style = styleObject(edge.style);
      const lineWidth = Math.max(1, finiteOr(style.strokeWidth, 2));
      const stroke = typeof style.stroke === 'string' && style.stroke.trim() ? style.stroke : '#636f7f';
      const opacity = Math.max(0.1, Math.min(1, finiteOr(style.opacity, 0.95)));

      return {
        id: edge.id,
        markerId: `fresky-edge-arrow-${index}`,
        path,
        lineWidth,
        stroke,
        opacity,
        strokeDasharray: parseDash(style.strokeDasharray),
        label: typeof edge.label === 'string' ? edge.label : '',
        labelX: mid.x,
        labelY: mid.y,
      } satisfies EdgeView;
    })
    .filter((v): v is NonNullable<typeof v> => v !== null);
});

const viewportEl = ref<HTMLElement | null>(null);
const zoom = ref(1);
const panX = ref(24);
const panY = ref(24);
const isPanning = ref(false);
const hasInteracted = ref(false);

const stageStyle = computed(() => ({
  transform: `translate(${panX.value}px, ${panY.value}px) scale(${zoom.value})`,
}));

let panPointerId: number | null = null;
let panStartClientX = 0;
let panStartClientY = 0;
let panStartX = 0;
let panStartY = 0;

function fitView() {
  const viewport = viewportEl.value;
  if (!viewport) return;
  const vw = Math.max(1, viewport.clientWidth);
  const vh = Math.max(1, viewport.clientHeight);
  const scaleX = (vw - 40) / boardWidth.value;
  const scaleY = (vh - 40) / boardHeight.value;
  zoom.value = clamp(Math.min(scaleX, scaleY, 1), 0.35, 2.8);
  panX.value = (vw - boardWidth.value * zoom.value) * 0.5;
  panY.value = (vh - boardHeight.value * zoom.value) * 0.5;
}

function resetView() {
  hasInteracted.value = false;
  fitView();
}

function zoomBy(factor: number, clientX?: number, clientY?: number) {
  const viewport = viewportEl.value;
  if (!viewport) return;
  const rect = viewport.getBoundingClientRect();
  const cx = clientX ?? (rect.left + rect.width / 2);
  const cy = clientY ?? (rect.top + rect.height / 2);
  const vx = cx - rect.left;
  const vy = cy - rect.top;
  const worldX = (vx - panX.value) / zoom.value;
  const worldY = (vy - panY.value) / zoom.value;
  const nextZoom = clamp(zoom.value * factor, 0.35, 2.8);
  panX.value = vx - worldX * nextZoom;
  panY.value = vy - worldY * nextZoom;
  zoom.value = nextZoom;
}

function zoomIn() {
  hasInteracted.value = true;
  zoomBy(1.12);
}

function zoomOut() {
  hasInteracted.value = true;
  zoomBy(1 / 1.12);
}

function onWheel(evt: WheelEvent) {
  hasInteracted.value = true;
  const factor = evt.deltaY < 0 ? 1.08 : 1 / 1.08;
  zoomBy(factor, evt.clientX, evt.clientY);
}

function onPointerDown(evt: PointerEvent) {
  if (evt.button !== 0) return;
  const target = evt.target as HTMLElement | null;
  if (target?.closest('.fresky-node')) return;
  hasInteracted.value = true;
  isPanning.value = true;
  panPointerId = evt.pointerId;
  panStartClientX = evt.clientX;
  panStartClientY = evt.clientY;
  panStartX = panX.value;
  panStartY = panY.value;
  window.addEventListener('pointermove', onPointerMove);
  window.addEventListener('pointerup', onPointerUp);
  window.addEventListener('pointercancel', onPointerUp);
}

function onPointerMove(evt: PointerEvent) {
  if (!isPanning.value || panPointerId === null || evt.pointerId !== panPointerId) return;
  panX.value = panStartX + (evt.clientX - panStartClientX);
  panY.value = panStartY + (evt.clientY - panStartClientY);
}

function onPointerUp(evt: PointerEvent) {
  if (panPointerId !== null && evt.pointerId !== panPointerId) return;
  isPanning.value = false;
  panPointerId = null;
  window.removeEventListener('pointermove', onPointerMove);
  window.removeEventListener('pointerup', onPointerUp);
  window.removeEventListener('pointercancel', onPointerUp);
}

function onBackgroundClick(evt: MouseEvent) {
  const target = evt.target as HTMLElement | null;
  if (target?.closest('.fresky-node') || target?.closest('.fresky-flow__toolbar')) return;
  emit('update:selected-node-id', null);
}

function onNodeIconClick(node: NodeView) {
  if (!node.itemKey) return;
  emit('item-click', node.itemKey);
}

function onNodeHoverEnter(node: NodeView) {
  if (!node.itemKey) return;
  emit('item-hover', node.itemKey);
}

watch([boardWidth, boardHeight], async () => {
  if (hasInteracted.value) return;
  await nextTick();
  fitView();
}, { immediate: true });

onBeforeUnmount(() => {
  window.removeEventListener('pointermove', onPointerMove);
  window.removeEventListener('pointerup', onPointerUp);
  window.removeEventListener('pointercancel', onPointerUp);
});
</script>

<style scoped>
.fresky-flow {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  border-radius: 12px;
  border: 1px solid rgba(148, 163, 184, 0.45);
  background: #d6d6d6;
  cursor: grab;
}

.fresky-flow--panning {
  cursor: grabbing;
}

.fresky-flow__toolbar {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 9;
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.86);
  border: 1px solid rgba(100, 116, 139, 0.3);
}

.fresky-flow__tool-btn {
  border: 1px solid rgba(100, 116, 139, 0.45);
  background: #fff;
  color: #111827;
  width: 24px;
  height: 24px;
  border-radius: 6px;
  cursor: pointer;
}

.fresky-flow__tool-btn--reset {
  width: 36px;
  font-size: 11px;
}

.fresky-flow__zoom-text {
  min-width: 38px;
  text-align: right;
  font-size: 12px;
  color: #334155;
}

.fresky-flow__stage {
  position: absolute;
  inset: 0;
  transform-origin: 0 0;
  will-change: transform;
}

.fresky-flow__board {
  position: relative;
}

.fresky-flow__edges {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.fresky-flow__edge-label {
  position: absolute;
  transform: translate(-50%, -50%);
  font-size: 11px;
  line-height: 1.3;
  white-space: nowrap;
  color: #111827;
  padding: 1px 4px;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.55);
  pointer-events: none;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.08);
}

.fresky-node {
  position: absolute;
  user-select: none;
}

.fresky-node--item,
.fresky-node--fluid {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 5px;
}

.fresky-node--machine {
  border-radius: 8px;
  border: 1px solid rgba(100, 116, 139, 0.35);
  background: rgba(235, 235, 235, 0.92);
  box-shadow: 0 2px 8px rgba(2, 6, 23, 0.1);
  padding: 7px 8px;
}

.fresky-node__machine-inner {
  display: flex;
  align-items: flex-start;
  gap: 8px;
}

.fresky-node__machine-icon-wrap {
  width: 42px;
  height: 42px;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid rgba(100, 116, 139, 0.25);
  display: grid;
  place-items: center;
  flex: 0 0 auto;
}

.fresky-node__machine-icon {
  width: 32px;
  height: 32px;
  object-fit: contain;
}

.fresky-node__machine-icon-fallback {
  width: 32px;
  height: 32px;
  border-radius: 4px;
  display: grid;
  place-items: center;
  background: rgba(148, 163, 184, 0.25);
  color: #334155;
  font-weight: 700;
}

.fresky-node__machine-meta {
  min-width: 0;
  flex: 1 1 auto;
}

.fresky-node__icon-wrap {
  width: 66px;
  height: 66px;
  border-radius: 8px;
  border: 1px solid rgba(100, 116, 139, 0.2);
  background: #dfdfdf;
  box-shadow: 0 2px 7px rgba(2, 6, 23, 0.14);
  display: grid;
  place-items: center;
  cursor: pointer;
}

.fresky-node__icon {
  width: 50px;
  height: 50px;
  object-fit: contain;
}

.fresky-node__icon-fallback {
  width: 50px;
  height: 50px;
  border-radius: 6px;
  display: grid;
  place-items: center;
  font-weight: 700;
  color: #1f2937;
  background: rgba(148, 163, 184, 0.2);
}

.fresky-node__title {
  max-width: 112px;
  font-size: 13px;
  line-height: 1.15;
  text-align: center;
  color: #111827;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__sub {
  max-width: 112px;
  font-size: 12px;
  line-height: 1.2;
  text-align: center;
  color: #374151;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__machine-title {
  font-size: 13px;
  line-height: 1.15;
  color: #111827;
  text-align: left;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__machine-sub {
  margin-top: 3px;
  font-size: 11px;
  color: #374151;
  text-align: left;
  line-height: 1.2;
  max-height: 2.4em;
  overflow: hidden;
}

.fresky-node--selected.fresky-node--machine {
  border-color: #3b82f6;
  box-shadow: 0 0 0 1px rgba(59, 130, 246, 0.5);
}

.fresky-node--selected .fresky-node__icon-wrap {
  border-color: #3b82f6;
  box-shadow: 0 0 0 1px rgba(59, 130, 246, 0.45), 0 2px 7px rgba(2, 6, 23, 0.14);
}

.fresky-node--surplus .fresky-node__icon-wrap {
  border-color: #f59e0b;
}

.body--dark .fresky-flow {
  border-color: rgba(148, 163, 184, 0.4);
  background: #30343a;
}

.body--dark .fresky-flow__toolbar {
  background: rgba(15, 23, 42, 0.74);
  border-color: rgba(148, 163, 184, 0.35);
}

.body--dark .fresky-flow__tool-btn {
  background: rgba(15, 23, 42, 0.9);
  color: #e2e8f0;
  border-color: rgba(148, 163, 184, 0.55);
}

.body--dark .fresky-flow__zoom-text {
  color: #e2e8f0;
}

.body--dark .fresky-flow__edge-label {
  color: #e2e8f0;
  background: rgba(15, 23, 42, 0.6);
}

.body--dark .fresky-node__title {
  color: #e2e8f0;
}

.body--dark .fresky-node__sub,
.body--dark .fresky-node__machine-sub {
  color: #cbd5e1;
}

.body--dark .fresky-node--machine {
  border-color: rgba(148, 163, 184, 0.65);
  background: rgba(15, 23, 42, 0.62);
}

.body--dark .fresky-node__icon-wrap {
  background: rgba(15, 23, 42, 0.85);
  border-color: rgba(148, 163, 184, 0.4);
}

.body--dark .fresky-node__machine-icon-wrap {
  background: rgba(15, 23, 42, 0.86);
  border-color: rgba(148, 163, 184, 0.4);
}

.body--dark .fresky-node__icon-fallback {
  color: #cbd5e1;
  background: rgba(148, 163, 184, 0.18);
}
</style>
