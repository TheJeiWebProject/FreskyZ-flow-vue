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
          <g v-for="edge in edgeViews" :key="edge.id">
            <path
              :d="edge.path"
              fill="none"
              :stroke="edge.stroke"
              :stroke-width="edge.lineWidth"
              :stroke-opacity="edge.opacity"
              :stroke-dasharray="edge.strokeDasharray"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </g>
        </svg>

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
                <img
                  v-if="node.icon.kind === 'image'"
                  class="fresky-node__machine-icon"
                  :src="node.icon.src"
                  :alt="node.title"
                />
                <div
                  v-else-if="node.icon.kind === 'sprite'"
                  class="fresky-node__sprite-wrap fresky-node__sprite-wrap--small"
                  :style="spriteWrapStyle(node.icon)"
                >
                  <div class="fresky-node__sprite-image" :style="spriteImageStyle(node.icon, 32)" />
                </div>
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
              <img
                v-if="node.icon.kind === 'image'"
                class="fresky-node__icon"
                :src="node.icon.src"
                :alt="node.title"
              />
              <div
                v-else-if="node.icon.kind === 'sprite'"
                class="fresky-node__sprite-wrap"
                :style="spriteWrapStyle(node.icon)"
              >
                <div class="fresky-node__sprite-image" :style="spriteImageStyle(node.icon, 50)" />
              </div>
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
  key?: ItemKeyLike;
  icon?: string;
  iconSprite?: {
    url?: string;
    position?: string;
    size?: number;
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
  style?: unknown;
};

type IconVisual =
  | { kind: 'none' }
  | { kind: 'image'; src: string }
  | { kind: 'sprite'; url: string; position: string; size: number; color: string | null };

type NodeView = {
  uid: string;
  id: string;
  kind: 'item' | 'machine' | 'fluid';
  title: string;
  subtitle: string;
  x: number;
  y: number;
  width: number;
  height: number;
  icon: IconVisual;
  shortName: string;
  itemKey: ItemKeyLike | null;
  accent: string;
  isSurplus: boolean;
  isRoot: boolean;
  duplicate: boolean;
};

type EdgeView = {
  id: string;
  sourceUid: string;
  targetUid: string;
  path: string;
  lineWidth: number;
  stroke: string;
  opacity: number;
  strokeDasharray: string;
};

type TreeNode = {
  uid: string;
  id: string;
  depth: number;
  position: number;
  duplicate: boolean;
  children: TreeNode[];
  thread: TreeNode | null;
  threadOffset: number | null;
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

function stableText(v: unknown): string {
  try {
    return JSON.stringify(v);
  } catch {
    return '';
  }
}

function sameItemKey(a: ItemKeyLike | null, b: ItemKeyLike | null): boolean {
  if (!a || !b) return false;
  return a.id === b.id && a.meta === b.meta && stableText(a.nbt) === stableText(b.nbt);
}

function pickItemDef(itemKey: ItemKeyLike | null, machineItemId: string | null): ItemDefLike | undefined {
  const defs = Object.values(props.itemDefsByKeyHash);
  if (itemKey) {
    const exact = defs.find((d) => sameItemKey(itemKey, d.key ?? null));
    if (exact) return exact;
    const byId = defs.find((d) => d.key?.id === itemKey.id);
    if (byId) return byId;
  }
  if (machineItemId) return defs.find((d) => d.key?.id === machineItemId);
  return undefined;
}

function itemAccentColor(itemKey: ItemKeyLike | null, machineItemId: string | null): string {
  const def = pickItemDef(itemKey, machineItemId);
  return def?.rarity?.color || def?.iconSprite?.color || '#9ca3af';
}

function nodeIcon(itemKey: ItemKeyLike | null, machineItemId: string | null): IconVisual {
  const def = pickItemDef(itemKey, machineItemId);
  if (!def) return { kind: 'none' };
  if (typeof def.icon === 'string' && def.icon.trim()) {
    return { kind: 'image', src: def.icon.trim() };
  }
  const sprite = def.iconSprite;
  if (sprite?.url && sprite.position) {
    return {
      kind: 'sprite',
      url: sprite.url,
      position: sprite.position,
      size: Math.max(1, Math.round(finiteOr(sprite.size, 64))),
      color: typeof sprite.color === 'string' && sprite.color.trim() ? sprite.color.trim() : null,
    };
  }
  return { kind: 'none' };
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

function layoutTree(root: TreeNode) {
  const MinDistance = 1;
  const setup = (thisNode: TreeNode) => {
    thisNode.children.filter((c) => c.children.length > 0).forEach(setup);
    if (thisNode.children.length === 1) {
      thisNode.children[0]!.position = 0;
      return;
    }
    if (thisNode.children.length === 0) return;

    const childCount = thisNode.children.length;
    const childIndexes = Array.from({ length: childCount }, (_, i) => i);
    childIndexes.forEach((idx) => {
      thisNode.children[idx]!.position = idx === 0 ? 0 : MinDistance;
    });

    let activeIndexes = childIndexes.slice();
    const leftCursors: Array<TreeNode | null> = thisNode.children.slice();
    const leftOffsets = thisNode.children.map(() => 0);
    const rightCursors: Array<TreeNode | null> = thisNode.children.slice();
    const rightOffsets = thisNode.children.map(() => 0);

    let leftmost = 0;
    const rightmost = childIndexes.map((idx) =>
      idx === childCount - 1
        ? { node: thisNode.children[idx] ?? null, offset: 0 }
        : { node: null, offset: Number.NEGATIVE_INFINITY },
    );

    while (activeIndexes.length > 0) {
      for (const idx of activeIndexes) {
        const l = leftCursors[idx];
        if (l?.thread) {
          leftOffsets[idx] = (leftOffsets[idx] ?? 0) + (l.threadOffset ?? 0);
          leftCursors[idx] = l.thread;
        } else if (l && l.children.length > 0) {
          const child = l.children[0] ?? null;
          leftCursors[idx] = child;
          leftOffsets[idx] = (leftOffsets[idx] ?? 0) + (child?.position ?? 0);
        } else if (idx !== activeIndexes[0] || activeIndexes.length === 1) {
          leftCursors[idx] = null;
        }

        const r = rightCursors[idx];
        if (r?.thread) {
          rightOffsets[idx] = (rightOffsets[idx] ?? 0) + (r.threadOffset ?? 0);
          rightCursors[idx] = r.thread;
        } else if (r && r.children.length > 0) {
          const child = r.children[r.children.length - 1] ?? null;
          rightCursors[idx] = child;
          rightOffsets[idx] = (rightOffsets[idx] ?? 0) + (child?.position ?? 0);
        } else if (idx !== activeIndexes[activeIndexes.length - 1] || activeIndexes.length === 1) {
          rightCursors[idx] = null;
        }
      }

      const nextActive = activeIndexes.filter((idx, i) => (i === 0 ? !!rightCursors[idx] : !!leftCursors[idx]));
      if (nextActive.length > 1) {
        for (let i = 0; i < nextActive.length - 1; i += 1) {
          const li = nextActive[i]!;
          const ri = nextActive[i + 1]!;
          let dist = 0;
          for (let c = li + 1; c <= ri; c += 1) dist += thisNode.children[c]!.position;
          if (dist + leftOffsets[ri]! - rightOffsets[li]! < MinDistance) {
            const inc = MinDistance - leftOffsets[ri]! + rightOffsets[li]! - dist;
            if (li + 1 === ri) thisNode.children[ri]!.position += inc;
            else {
              for (let c = li + 1; c <= ri; c += 1) thisNode.children[c]!.position += inc / (ri - li);
            }
          }
        }
      }

      activeIndexes = nextActive;
      if (activeIndexes.length > 0) {
        const lIdx = activeIndexes[0]!;
        leftmost = Math.min(leftmost, leftOffsets[lIdx]!);
        const rIdx = activeIndexes[activeIndexes.length - 1]!;
        if (!rightmost[rIdx]!.node || rightmost[rIdx]!.offset < rightOffsets[rIdx]!) {
          rightmost[rIdx]!.node = rightCursors[rIdx] ?? null;
          rightmost[rIdx]!.offset = rightOffsets[rIdx]!;
        }
      }
    }

    let dist = 0;
    let rightMostPos = 0;
    for (const idx of childIndexes) {
      if (idx !== 0) dist += thisNode.children[idx]!.position;
      if (rightmost[idx]!.node) rightMostPos = Math.max(rightMostPos, dist + rightmost[idx]!.offset);
    }

    let cur = -(leftmost + rightMostPos) / 2;
    thisNode.children.forEach((child) => {
      cur = child.position += cur;
    });
  };

  setup(root);

  let cursor = root;
  let cursorPos = 0;
  let minCursorPos = 0;
  while (true) {
    if (cursor.thread) {
      cursorPos += cursor.threadOffset ?? 0;
      cursor = cursor.thread;
      minCursorPos = Math.min(minCursorPos, cursorPos);
    } else if (cursor.children.length > 0) {
      cursor = cursor.children[0]!;
      cursorPos += cursor.position;
      minCursorPos = Math.min(minCursorPos, cursorPos);
    } else break;
  }

  const setPosition = (node: TreeNode, position: number) => {
    node.position = position;
    node.thread = null;
    node.threadOffset = null;
    node.children.forEach((child) => setPosition(child, position + child.position));
  };
  setPosition(root, -minCursorPos);
}

const baseNodeById = computed(() => {
  const map = new Map<string, Omit<NodeView, 'uid' | 'x' | 'y' | 'shortName' | 'duplicate'>>();
  props.nodes.forEach((node) => {
    const data = node.data ?? {};
    const kind = node.type === 'lineMachineNode' ? 'machine' : node.type === 'lineFluidNode' ? 'fluid' : 'item';
    const { width, height } = nodeSize(node.type);
    const itemKeyCandidate = data.itemKey as ItemKeyLike | undefined;
    const itemKey = itemKeyCandidate?.id ? itemKeyCandidate : null;
    const machineItemId = typeof data.machineItemId === 'string' ? data.machineItemId : null;
    map.set(node.id, {
      id: node.id,
      kind,
      title: typeof data.title === 'string' ? data.title : node.id,
      subtitle: typeof data.subtitle === 'string' ? data.subtitle : '',
      width,
      height,
      icon: nodeIcon(itemKey, machineItemId),
      itemKey,
      accent: kind === 'machine' ? '#6b7280' : itemAccentColor(itemKey, machineItemId),
      isSurplus: !!data.isSurplus,
      isRoot: !!data.isRoot,
    });
  });
  return map;
});

const orderYById = computed(() => {
  const map = new Map<string, number>();
  props.nodes.forEach((node) => {
    map.set(node.id, finiteOr(node.position?.y, 0));
  });
  return map;
});

const inByTarget = computed(() => {
  const map = new Map<string, string[]>();
  props.nodes.forEach((n) => map.set(n.id, []));
  props.edges.forEach((edge) => {
    if (!map.has(edge.target) || !baseNodeById.value.has(edge.source)) return;
    (map.get(edge.target) ?? []).push(edge.source);
  });
  map.forEach((arr) =>
    arr.sort((a, b) => (orderYById.value.get(a) ?? 0) - (orderYById.value.get(b) ?? 0)),
  );
  return map;
});

const outDegreeById = computed(() => {
  const map = new Map<string, number>();
  props.nodes.forEach((n) => map.set(n.id, 0));
  props.edges.forEach((e) => map.set(e.source, (map.get(e.source) ?? 0) + 1));
  return map;
});

const chosenRootId = computed(() => {
  if (props.selectedNodeId && baseNodeById.value.get(props.selectedNodeId)?.kind === 'item') {
    return props.selectedNodeId;
  }
  const explicit = Array.from(baseNodeById.value.values()).find((n) => n.kind === 'item' && n.isRoot);
  if (explicit) return explicit.id;
  const sink = Array.from(baseNodeById.value.values()).find(
    (n) => n.kind === 'item' && (outDegreeById.value.get(n.id) ?? 0) === 0,
  );
  if (sink) return sink.id;
  return props.nodes[0]?.id ?? '';
});

const layoutResult = computed(() => {
  const rootId = chosenRootId.value;
  if (!rootId || !baseNodeById.value.has(rootId)) {
    return { nodes: [] as NodeView[], edges: [] as EdgeView[] };
  }

  let uidSeq = 0;
  const buildTree = (id: string, depth: number, path: string[]): TreeNode => {
    const duplicate = path.includes(id);
    const node: TreeNode = {
      uid: `tn_${uidSeq++}`,
      id,
      depth,
      position: 0,
      duplicate,
      children: [],
      thread: null,
      threadOffset: null,
    };
    if (duplicate) return node;
    node.children = (inByTarget.value.get(id) ?? []).map((childId) => buildTree(childId, depth + 1, [...path, id]));
    return node;
  };

  const root = buildTree(rootId, 0, []);
  layoutTree(root);

  const flatTree: TreeNode[] = [];
  const treeEdges: Array<{ source: TreeNode; target: TreeNode }> = [];
  let maxDepth = 0;
  const walk = (node: TreeNode) => {
    flatTree.push(node);
    maxDepth = Math.max(maxDepth, node.depth);
    node.children.forEach((child) => {
      treeEdges.push({ source: node, target: child });
      walk(child);
    });
  };
  walk(root);

  const CellWidth = 184;
  const CellHeight = 78;
  const LeftPad = 20;
  const TopPad = 24;

  const nodes: NodeView[] = flatTree.map((t) => {
    const base = baseNodeById.value.get(t.id)!;
    const x = LeftPad + (maxDepth - t.depth) * CellWidth + (base.kind === 'machine' ? 92 : 8);
    const y = TopPad + t.position * CellHeight + (base.kind === 'machine' ? 8 : 0);
    return {
      uid: t.uid,
      id: t.id,
      kind: base.kind,
      title: base.title,
      subtitle: base.subtitle,
      x,
      y,
      width: base.width,
      height: base.height,
      icon: base.icon,
      shortName: initialChar(base.title, t.id),
      itemKey: base.itemKey,
      accent: base.accent,
      isSurplus: base.isSurplus,
      isRoot: base.isRoot,
      duplicate: t.duplicate,
    };
  });

  const nodeByUid = new Map(nodes.map((n) => [n.uid, n] as const));
  const styleByPair = new Map<string, { lineWidth: number; stroke: string; opacity: number; strokeDasharray: string }>();
  props.edges.forEach((e) => {
    if (styleByPair.has(`${e.source}->${e.target}`)) return;
    const style = styleObject(e.style);
    styleByPair.set(`${e.source}->${e.target}`, {
      lineWidth: Math.max(1, finiteOr(style.strokeWidth, 2)),
      stroke: typeof style.stroke === 'string' && style.stroke.trim() ? style.stroke : '#666666',
      opacity: Math.max(0.1, Math.min(1, finiteOr(style.opacity, 0.95))),
      strokeDasharray: parseDash(style.strokeDasharray),
    });
  });

  const grouped = new Map<string, Array<{ sourceUid: string; targetUid: string; sourceId: string; targetId: string }>>();
  treeEdges.forEach((e) => {
    const arr = grouped.get(e.source.uid);
    const item = { sourceUid: e.source.uid, targetUid: e.target.uid, sourceId: e.source.id, targetId: e.target.id };
    if (arr) arr.push(item);
    else grouped.set(e.source.uid, [item]);
  });

  const edges: EdgeView[] = [];
  grouped.forEach((arr, sourceUid) => {
    const source = nodeByUid.get(sourceUid);
    if (!source) return;
    const arranged = arr
      .map((e) => {
        const target = nodeByUid.get(e.targetUid);
        return target ? { ...e, target } : null;
      })
      .filter((x): x is NonNullable<typeof x> => x !== null)
      .sort((a, b) => (a.target.y + a.target.height / 2) - (b.target.y + b.target.height / 2));

    arranged.forEach((e, i) => {
      const sourceCx = source.x + source.width / 2;
      const targetCx = e.target.x + e.target.width / 2;
      const toRight = targetCx >= sourceCx;
      const sx = toRight ? source.x + source.width : source.x;
      const tx = toRight ? e.target.x : e.target.x + e.target.width;
      const sy = source.y + source.height / 2;
      const ty = e.target.y + e.target.height / 2;
      const sign = toRight ? 1 : -1;
      const trunkX = sx + sign * 24;
      const path =
        arranged.length === 1 && Math.abs(ty - sy) < 1
          ? `M ${sx} ${sy} L ${tx} ${ty}`
          : `M ${sx} ${sy} L ${trunkX} ${sy} L ${trunkX} ${ty} L ${tx} ${ty}`;
      const style = styleByPair.get(`${e.targetId}->${e.sourceId}`) ?? {
        lineWidth: 2,
        stroke: '#666666',
        opacity: 0.95,
        strokeDasharray: '',
      };
      edges.push({
        id: `te_${sourceUid}_${i}`,
        sourceUid: e.sourceUid,
        targetUid: e.targetUid,
        path,
        lineWidth: style.lineWidth,
        stroke: style.stroke,
        opacity: style.opacity,
        strokeDasharray: style.strokeDasharray,
      });
    });
  });

  return { nodes, edges };
});

const nodeViews = computed(() => layoutResult.value.nodes);
const edgeViews = computed(() => layoutResult.value.edges);

const bounds = computed(() => {
  const nodes = nodeViews.value;
  if (!nodes.length) return { minX: 0, minY: 0, maxX: 960, maxY: 540 };
  let minX = Number.POSITIVE_INFINITY;
  let minY = Number.POSITIVE_INFINITY;
  let maxX = Number.NEGATIVE_INFINITY;
  let maxY = Number.NEGATIVE_INFINITY;
  nodes.forEach((n) => {
    minX = Math.min(minX, n.x);
    minY = Math.min(minY, n.y);
    maxX = Math.max(maxX, n.x + n.width);
    maxY = Math.max(maxY, n.y + n.height);
  });
  return { minX, minY, maxX, maxY };
});

const boardWidth = computed(() => Math.max(960, Math.ceil(bounds.value.maxX + 60)));
const boardHeight = computed(() => Math.max(520, Math.ceil(bounds.value.maxY + 60)));

function spriteWrapStyle(icon: Extract<IconVisual, { kind: 'sprite' }>) {
  return {
    backgroundColor: icon.color ?? 'transparent',
  };
}

function spriteImageStyle(icon: Extract<IconVisual, { kind: 'sprite' }>, targetSize: number) {
  const scale = targetSize / icon.size;
  return {
    width: `${icon.size}px`,
    height: `${icon.size}px`,
    backgroundImage: `url(${icon.url})`,
    backgroundRepeat: 'no-repeat',
    backgroundPosition: icon.position,
    transform: `scale(${scale})`,
    transformOrigin: 'top left',
  };
}

const viewportEl = ref<HTMLElement | null>(null);
const zoom = ref(1);
const panX = ref(24);
const panY = ref(24);
const isPanning = ref(false);
const hasInteracted = ref(false);

const stageStyle = computed(() => ({
  transform: `translate(${panX.value}px, ${panY.value}px)`,
  zoom: `${zoom.value}`,
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
  if (target?.closest('.fresky-node') || target?.closest('.fresky-flow__toolbar')) return;
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
  image-rendering: -webkit-optimize-contrast;
  image-rendering: crisp-edges;
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
  image-rendering: -webkit-optimize-contrast;
  image-rendering: crisp-edges;
}

.fresky-node__sprite-wrap {
  width: 50px;
  height: 50px;
  overflow: hidden;
  border-radius: 6px;
}

.fresky-node__sprite-wrap--small {
  width: 32px;
  height: 32px;
  border-radius: 4px;
}

.fresky-node__sprite-image {
  background-repeat: no-repeat;
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
