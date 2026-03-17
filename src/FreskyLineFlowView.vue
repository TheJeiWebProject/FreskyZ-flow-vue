<template>
  <div class="fresky-flow" @click="emit('update:selected-node-id', null)">
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
            markerWidth="10"
            markerHeight="10"
            refX="9"
            refY="5"
            orient="auto"
            markerUnits="strokeWidth"
          >
            <path d="M0,0 L10,5 L0,10 z" :fill="edge.stroke" />
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
          <div class="fresky-node__machine-title">{{ node.title }}</div>
          <div class="fresky-node__machine-sub">{{ node.subtitle }}</div>
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
</template>

<script setup lang="ts">
import { computed } from 'vue';

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
  label?: string;
  style?: Record<string, unknown>;
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
  itemKey?: ItemKeyLike;
  accent: string;
  isSurplus: boolean;
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
  if (type === 'lineMachineNode') return { width: 210, height: 86 };
  if (type === 'lineFluidNode') return { width: 120, height: 108 };
  return { width: 112, height: 126 };
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
    const data = (node.data ?? {}) as Record<string, unknown>;
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
      shortName: (typeof data.title === 'string' && data.title.trim().length > 0 ? data.title.trim()[0] : node.id[0] || '?').toUpperCase(),
      itemKey,
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

const edgeViews = computed(() => {
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

      const style = edge.style ?? {};
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
      };
    })
    .filter((v): v is NonNullable<typeof v> => v !== null);
});

function onNodeIconClick(node: NodeView) {
  if (!node.itemKey) return;
  emit('item-click', node.itemKey);
}

function onNodeHoverEnter(node: NodeView) {
  if (!node.itemKey) return;
  emit('item-hover', node.itemKey);
}
</script>

<style scoped>
.fresky-flow {
  width: 100%;
  height: 100%;
  overflow: auto;
  border-radius: 12px;
  background: linear-gradient(180deg, rgba(229, 231, 235, 0.5) 0%, rgba(226, 232, 240, 0.32) 100%);
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
  font-size: 12px;
  line-height: 1.3;
  white-space: nowrap;
  color: #1f2937;
  padding: 2px 6px;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.78);
  backdrop-filter: blur(1px);
  pointer-events: none;
  box-shadow: 0 2px 8px rgba(15, 23, 42, 0.09);
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
  gap: 6px;
}

.fresky-node--machine {
  border-radius: 10px;
  border: 2px solid rgba(75, 85, 99, 0.45);
  background: rgba(255, 255, 255, 0.72);
  box-shadow: 0 4px 16px rgba(2, 6, 23, 0.08);
  padding: 8px 10px;
}

.fresky-node__icon-wrap {
  width: 68px;
  height: 68px;
  border-radius: 10px;
  border: 2px solid color-mix(in srgb, var(--fresky-accent), #374151 35%);
  background: rgba(255, 255, 255, 0.7);
  box-shadow: 0 4px 14px rgba(2, 6, 23, 0.14);
  display: grid;
  place-items: center;
  cursor: pointer;
}

.fresky-node__icon {
  width: 52px;
  height: 52px;
  object-fit: contain;
}

.fresky-node__icon-fallback {
  width: 52px;
  height: 52px;
  border-radius: 8px;
  display: grid;
  place-items: center;
  font-weight: 700;
  color: #334155;
  background: rgba(148, 163, 184, 0.2);
}

.fresky-node__title {
  max-width: 128px;
  font-size: 18px;
  line-height: 1.1;
  text-align: center;
  color: #0f172a;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__sub {
  max-width: 170px;
  font-size: 12px;
  line-height: 1.2;
  text-align: center;
  color: #334155;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__machine-title {
  font-size: 34px;
  line-height: 1;
  color: #0f172a;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__machine-sub {
  margin-top: 4px;
  font-size: 14px;
  color: #334155;
  text-align: center;
}

.fresky-node--selected.fresky-node--machine {
  border-color: var(--q-primary, #1976d2);
  box-shadow: 0 0 0 2px color-mix(in srgb, var(--q-primary, #1976d2), transparent 72%);
}

.fresky-node--selected .fresky-node__icon-wrap {
  border-color: var(--q-primary, #1976d2);
}

.fresky-node--surplus .fresky-node__icon-wrap {
  border-color: #f59e0b;
}

.body--dark .fresky-flow {
  background: linear-gradient(180deg, rgba(30, 41, 59, 0.75) 0%, rgba(15, 23, 42, 0.75) 100%);
}

.body--dark .fresky-flow__edge-label {
  color: #e2e8f0;
  background: rgba(15, 23, 42, 0.76);
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
  background: rgba(15, 23, 42, 0.72);
}

.body--dark .fresky-node__machine-title {
  color: #f8fafc;
}

.body--dark .fresky-node__icon-wrap {
  background: rgba(15, 23, 42, 0.82);
}

.body--dark .fresky-node__icon-fallback {
  color: #cbd5e1;
  background: rgba(148, 163, 184, 0.18);
}
</style>
