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
      <button class="fresky-flow__tool-btn fresky-flow__tool-btn--reset" @click.stop="resetView">
        1:1
      </button>
      <span class="fresky-flow__zoom-text">{{ Math.round(zoom * 100) }}%</span>
    </div>

    <div class="fresky-flow__stage" :style="stageStyle">
      <div
        class="fresky-flow__board"
        :style="{ width: `${boardWidth}px`, height: `${boardHeight}px` }"
      >
        <svg
          class="fresky-flow__edges"
          :viewBox="`0 0 ${boardWidth} ${boardHeight}`"
          preserveAspectRatio="none"
        >
          <defs>
            <marker
              v-for="edge in edgeViews"
              :id="markerId(edge.id)"
              :key="`mk_${edge.id}`"
              markerWidth="8"
              markerHeight="8"
              refX="7"
              refY="4"
              orient="auto"
              markerUnits="userSpaceOnUse"
            >
              <path d="M 0 0 L 8 4 L 0 8 z" :fill="edge.stroke" :fill-opacity="edge.opacity" />
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
              stroke-linejoin="round"
              :marker-end="`url(#${markerId(edge.id)})`"
            />
          </g>
        </svg>

        <div
          v-for="node in nodeViews"
          :key="node.uid"
          class="fresky-node"
          :class="[
            `fresky-node--${node.kind}`,
            {
              'fresky-node--selected': props.selectedNodeId === node.id,
              'fresky-node--surplus': node.isSurplus,
              'fresky-node--duplicate': node.duplicate,
            },
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
            <div class="fresky-node__machine-sub">{{ node.subtitle }}</div>
            <div class="fresky-node__machine-title">{{ node.title }}</div>
            <div v-if="node.machineCountText" class="fresky-node__machine-count">
              ×{{ node.machineCountText }}
            </div>
          </template>

          <template v-else>
            <div
              class="fresky-node__icon-wrap"
              @click.stop="onNodeIconClick(node)"
              @mouseenter="onNodeHoverEnter(node)"
              @mouseleave="emit('item-leave')"
            >
              <template v-if="node.duplicate">
                <div class="fresky-node__duplicate-dot">...</div>
              </template>
              <template v-else>
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
                  <div class="fresky-node__sprite-image" :style="spriteImageStyle(node.icon, 48)" />
                </div>
                <div v-else class="fresky-node__icon-fallback">{{ node.shortName }}</div>
              </template>
            </div>
            <div class="fresky-node__title">{{ node.duplicate ? '...' : node.title }}</div>
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

type NodeKind = 'item' | 'machine' | 'fluid';

type NodeView = {
  uid: string;
  id: string;
  kind: NodeKind;
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
  machineCountText?: string;
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
  kind: NodeKind;
  depth: number;
  position: number;
  duplicate: boolean;
  children: TreeNode[];
  thread: TreeNode | null;
  threadOffset: number | undefined;
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
  if (type === 'lineMachineNode') return { width: 96, height: 48 };
  if (type === 'lineFluidNode') return { width: 72, height: 72 };
  return { width: 72, height: 72 };
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

function pickItemDef(
  itemKey: ItemKeyLike | null,
  machineItemId: string | null,
): ItemDefLike | undefined {
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

function machineCountText(value: unknown): string | undefined {
  if (typeof value === 'string') {
    const t = value.trim();
    if (t) return t;
  }
  const n = finiteOr(value, 0);
  if (!Number.isFinite(n) || n <= 0) return undefined;
  const rounded = Math.round(n * 1000) / 1000;
  return `${rounded}`.replace(/\.?0+$/, '');
}

function layoutTree(root: TreeNode) {
  const MinDistance = 1;
  const setup = (thisNode: TreeNode) => {
    thisNode.children.filter((c) => c.children.length).forEach(setup);
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

    let activeIndexes = childIndexes.map((i) => i);
    const leftCursors: Array<TreeNode | null> = thisNode.children.map((n) => n);
    const leftCursorOffsets = thisNode.children.map(() => 0);
    const rightCursors: Array<TreeNode | null> = thisNode.children.map((n) => n);
    const rightCursorOffsets = thisNode.children.map(() => 0);

    let leftmostDescendantPosition = 0;
    const rightmostNodes = childIndexes.map((idx) =>
      idx === childCount - 1
        ? { node: thisNode.children[idx]!, offset: 0 }
        : { node: null as TreeNode | null, offset: undefined as number | undefined },
    );

    while (activeIndexes.length) {
      for (const childIndex of activeIndexes) {
        if (leftCursors[childIndex]!.thread) {
          leftCursorOffsets[childIndex]! += leftCursors[childIndex]!.threadOffset ?? 0;
          leftCursors[childIndex] = leftCursors[childIndex]!.thread;
        } else if (leftCursors[childIndex]!.children.length) {
          leftCursors[childIndex] = leftCursors[childIndex]!.children[0]!;
          leftCursorOffsets[childIndex]! += leftCursors[childIndex].position;
        } else if (childIndex !== activeIndexes[0] || activeIndexes.length === 1) {
          leftCursors[childIndex] = null;
        }

        if (rightCursors[childIndex]!.thread) {
          rightCursorOffsets[childIndex]! += rightCursors[childIndex]!.threadOffset ?? 0;
          rightCursors[childIndex] = rightCursors[childIndex]!.thread;
        } else if (rightCursors[childIndex]!.children.length) {
          rightCursors[childIndex] =
            rightCursors[childIndex]!.children[rightCursors[childIndex]!.children.length - 1]!;
          rightCursorOffsets[childIndex]! += rightCursors[childIndex].position;
        } else if (
          childIndex !== activeIndexes[activeIndexes.length - 1] ||
          activeIndexes.length === 1
        ) {
          rightCursors[childIndex] = null;
        }
      }

      const newActiveIndexes = activeIndexes.filter((childIndex, i) =>
        i === 0 ? !!rightCursors[childIndex] : !!leftCursors[childIndex],
      );

      if (newActiveIndexes.length > 1) {
        for (let i = 0; i < newActiveIndexes.length - 1; i += 1) {
          const leftIndex = newActiveIndexes[i]!;
          const rightIndex = newActiveIndexes[i + 1]!;
          let subtreeDistance = 0;
          for (let childIndex = leftIndex + 1; childIndex <= rightIndex; childIndex += 1) {
            subtreeDistance += thisNode.children[childIndex]!.position;
          }
          if (
            subtreeDistance + leftCursorOffsets[rightIndex]! - rightCursorOffsets[leftIndex]! <
            MinDistance
          ) {
            const increaseDistance =
              MinDistance -
              leftCursorOffsets[rightIndex]! +
              rightCursorOffsets[leftIndex]! -
              subtreeDistance;
            if (leftIndex + 1 === rightIndex) {
              thisNode.children[rightIndex]!.position += increaseDistance;
            } else {
              for (let childIndex = leftIndex + 1; childIndex <= rightIndex; childIndex += 1) {
                thisNode.children[childIndex]!.position +=
                  increaseDistance / (rightIndex - leftIndex);
              }
            }
          }
        }
      }

      let leftmostChildIndex = activeIndexes[0]!;
      if (activeIndexes.length > 1 && !rightCursors[leftmostChildIndex]) {
        let subtreeDistance = 0;
        let nextLeftmostChildIndexIndex = 1;
        while (
          nextLeftmostChildIndexIndex < activeIndexes.length &&
          !leftCursors[activeIndexes[nextLeftmostChildIndexIndex]!]
        ) {
          subtreeDistance +=
            thisNode.children[activeIndexes[nextLeftmostChildIndexIndex]!]!.position;
          nextLeftmostChildIndexIndex += 1;
        }
        if (nextLeftmostChildIndexIndex < activeIndexes.length) {
          const nextLeftmostChildIndex = activeIndexes[nextLeftmostChildIndexIndex]!;
          subtreeDistance += thisNode.children[nextLeftmostChildIndex]!.position;
          leftCursors[leftmostChildIndex]!.thread = leftCursors[nextLeftmostChildIndex] ?? null;
          leftCursors[leftmostChildIndex]!.threadOffset =
            leftCursorOffsets[nextLeftmostChildIndex]! -
            leftCursorOffsets[leftmostChildIndex]! +
            subtreeDistance;
          leftmostChildIndex = nextLeftmostChildIndex;
        }
      }

      let rightmostChildIndex = activeIndexes[activeIndexes.length - 1]!;
      if (activeIndexes.length > 1 && !leftCursors[rightmostChildIndex]) {
        let subtreeDistance = thisNode.children[rightmostChildIndex]!.position;
        let nextRightmostChildIndexIndex = activeIndexes.length - 2;
        while (
          nextRightmostChildIndexIndex >= 0 &&
          !rightCursors[activeIndexes[nextRightmostChildIndexIndex]!]
        ) {
          subtreeDistance +=
            thisNode.children[activeIndexes[nextRightmostChildIndexIndex]!]!.position;
          nextRightmostChildIndexIndex -= 1;
        }
        if (nextRightmostChildIndexIndex >= 0) {
          const nextRightmostChildIndex = activeIndexes[nextRightmostChildIndexIndex]!;
          rightCursors[rightmostChildIndex]!.thread = rightCursors[nextRightmostChildIndex] ?? null;
          rightCursors[rightmostChildIndex]!.threadOffset =
            rightCursorOffsets[nextRightmostChildIndex]! -
            rightCursorOffsets[rightmostChildIndex]! -
            subtreeDistance;
          rightmostChildIndex = nextRightmostChildIndex;
        }
      }

      activeIndexes = newActiveIndexes;
      if (activeIndexes.length) {
        leftmostDescendantPosition = Math.min(
          leftmostDescendantPosition,
          leftCursorOffsets[leftmostChildIndex]!,
        );
        if (
          !rightmostNodes[rightmostChildIndex]!.node ||
          (rightmostNodes[rightmostChildIndex]!.offset ?? Number.NEGATIVE_INFINITY) <
            rightCursorOffsets[rightmostChildIndex]!
        ) {
          rightmostNodes[rightmostChildIndex]!.node = rightCursors[rightmostChildIndex] ?? null;
          rightmostNodes[rightmostChildIndex]!.offset = rightCursorOffsets[rightmostChildIndex]!;
        }
      }
    }

    let subtreeDistance = 0;
    let rightmostDescendantPosition = 0;
    for (const childIndex of childIndexes) {
      if (childIndex !== 0) {
        subtreeDistance += thisNode.children[childIndex]!.position;
      }
      if (rightmostNodes[childIndex]!.node) {
        rightmostDescendantPosition = Math.max(
          rightmostDescendantPosition,
          subtreeDistance + (rightmostNodes[childIndex]!.offset ?? 0),
        );
      }
    }

    let currentPosition = -(leftmostDescendantPosition + rightmostDescendantPosition) / 2;
    for (const child of thisNode.children) {
      currentPosition = child.position += currentPosition;
    }
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
    node.threadOffset = undefined;
    node.children.forEach((child) => setPosition(child, position + child.position));
  };
  setPosition(root, -minCursorPos);
}

const baseNodeById = computed(() => {
  const map = new Map<string, Omit<NodeView, 'uid' | 'x' | 'y' | 'shortName' | 'duplicate'>>();
  props.nodes.forEach((node) => {
    const data = node.data ?? {};
    const kind =
      node.type === 'lineMachineNode'
        ? 'machine'
        : node.type === 'lineFluidNode'
          ? 'fluid'
          : 'item';
    const { width, height } = nodeSize(node.type);
    const itemKeyCandidate = data.itemKey as ItemKeyLike | undefined;
    const itemKey = itemKeyCandidate?.id ? itemKeyCandidate : null;
    const machineItemId = typeof data.machineItemId === 'string' ? data.machineItemId : null;
    const machineCount = machineCountText(data.machineCount);
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
      ...(machineCount ? { machineCountText: machineCount } : {}),
    });
  });
  return map;
});

function stableNodeOrder(a: string, b: string): number {
  const at = (baseNodeById.value.get(a)?.title ?? a).trim();
  const bt = (baseNodeById.value.get(b)?.title ?? b).trim();
  const cmp = at.localeCompare(bt, 'zh-Hans-CN');
  if (cmp !== 0) return cmp;
  return a.localeCompare(b);
}

const kindById = computed(() => {
  const map = new Map<string, NodeKind>();
  baseNodeById.value.forEach((v, k) => map.set(k, v.kind));
  return map;
});

const producersByResource = computed(() => {
  const map = new Map<string, string[]>();
  props.nodes.forEach((n) => {
    const kind = kindById.value.get(n.id);
    if (kind === 'item' || kind === 'fluid') map.set(n.id, []);
  });
  props.edges.forEach((edge) => {
    const sk = kindById.value.get(edge.source);
    const tk = kindById.value.get(edge.target);
    if (sk === 'machine' && (tk === 'item' || tk === 'fluid')) {
      map.get(edge.target)?.push(edge.source);
    }
  });
  map.forEach((arr) => arr.sort(stableNodeOrder));
  return map;
});

const ingredientsByMachine = computed(() => {
  const map = new Map<string, string[]>();
  props.nodes.forEach((n) => {
    if (kindById.value.get(n.id) === 'machine') map.set(n.id, []);
  });
  props.edges.forEach((edge) => {
    const sk = kindById.value.get(edge.source);
    const tk = kindById.value.get(edge.target);
    if ((sk === 'item' || sk === 'fluid') && tk === 'machine') {
      map.get(edge.target)?.push(edge.source);
    }
  });
  map.forEach((arr) => arr.sort(stableNodeOrder));
  return map;
});

const outDegreeById = computed(() => {
  const map = new Map<string, number>();
  props.nodes.forEach((n) => map.set(n.id, 0));
  props.edges.forEach((e) => map.set(e.source, (map.get(e.source) ?? 0) + 1));
  return map;
});

const chosenRootId = computed(() => {
  if (props.selectedNodeId) {
    const k = kindById.value.get(props.selectedNodeId);
    if (k === 'item' || k === 'fluid') return props.selectedNodeId;
  }
  const explicit = Array.from(baseNodeById.value.values()).find(
    (n) => (n.kind === 'item' || n.kind === 'fluid') && n.isRoot,
  );
  if (explicit) return explicit.id;
  const sink = Array.from(baseNodeById.value.values()).find(
    (n) => (n.kind === 'item' || n.kind === 'fluid') && (outDegreeById.value.get(n.id) ?? 0) === 0,
  );
  if (sink) return sink.id;
  const firstResource = Array.from(baseNodeById.value.values()).find(
    (n) => n.kind === 'item' || n.kind === 'fluid',
  );
  if (firstResource) return firstResource.id;
  return props.nodes[0]?.id ?? '';
});

function resourceCenterY(node: NodeView): number {
  return node.y + 24;
}

function machineCenterY(node: NodeView): number {
  return node.y + 24;
}

function orthPath(parent: NodeView, child: NodeView): string {
  if ((parent.kind === 'item' || parent.kind === 'fluid') && child.kind === 'machine') {
    const px = parent.x;
    const py = resourceCenterY(parent);
    const cx = child.x + child.width;
    const cy = machineCenterY(child);
    const trunkX = parent.x - 8;
    return `M ${cx} ${cy} L ${trunkX} ${cy} L ${trunkX} ${py} L ${px} ${py}`;
  }
  if (parent.kind === 'machine' && (child.kind === 'item' || child.kind === 'fluid')) {
    const px = parent.x;
    const py = machineCenterY(parent);
    const cx = child.x + child.width;
    const cy = resourceCenterY(child);
    const trunkX = parent.x - 8;
    return `M ${px} ${py} L ${trunkX} ${py} L ${trunkX} ${cy} L ${cx} ${cy}`;
  }
  const sx = parent.x + parent.width;
  const sy = parent.y + parent.height / 2;
  const tx = child.x;
  const ty = child.y + child.height / 2;
  const mx = (sx + tx) / 2;
  return `M ${sx} ${sy} L ${mx} ${sy} L ${mx} ${ty} L ${tx} ${ty}`;
}

const layoutResult = computed(() => {
  const rootId = chosenRootId.value;
  if (!rootId || !baseNodeById.value.has(rootId)) {
    return { nodes: [] as NodeView[], edges: [] as EdgeView[] };
  }

  let uidSeq = 0;
  let builtCount = 0;
  const MAX_NODES = 10000;
  const MAX_DEPTH = 40;
  const visitedIds = new Set<string>();
  const buildTree = (id: string, depth: number, resourcePath: string[]): TreeNode => {
    const kind = kindById.value.get(id) ?? 'item';
    const cycleAncestor = (kind === 'item' || kind === 'fluid') && resourcePath.includes(id);
    const alreadyExpanded = visitedIds.has(id);
    const duplicate = cycleAncestor || alreadyExpanded;
    const node: TreeNode = {
      uid: `tn_${uidSeq++}`,
      id,
      kind,
      depth,
      position: 0,
      duplicate,
      children: [],
      thread: null,
      threadOffset: undefined,
    };

    builtCount += 1;
    if (duplicate || builtCount >= MAX_NODES || depth >= MAX_DEPTH) return node;

    visitedIds.add(id);

    if (kind === 'machine') {
      node.children = (ingredientsByMachine.value.get(id) ?? []).map((childId) =>
        buildTree(childId, depth + 1, resourcePath),
      );
      return node;
    }

    const nextPath = [...resourcePath, id];
    node.children = (producersByResource.value.get(id) ?? []).map((childId) =>
      buildTree(childId, depth, nextPath),
    );
    return node;
  };

  const root = buildTree(rootId, 0, []);
  layoutTree(root);

  const flatTree: TreeNode[] = [];
  const treeEdges: Array<{ source: TreeNode; target: TreeNode }> = [];
  let maxDepth = 0;
  const walk = (node: TreeNode) => {
    flatTree.push(node);
    if (node.kind !== 'machine') {
      maxDepth = Math.max(maxDepth, node.depth);
    }
    node.children.forEach((child) => {
      treeEdges.push({ source: node, target: child });
      walk(child);
    });
  };
  walk(root);

  const CellWidth = 184;
  const CellHeight = 72;
  const LeftPad = 24;
  const TopPad = 24;

  const nodes: NodeView[] = flatTree.map((t) => {
    const base = baseNodeById.value.get(t.id)!;
    const x =
      t.kind === 'machine'
        ? LeftPad + (maxDepth - t.depth) * CellWidth + 100
        : LeftPad + (maxDepth - t.depth + 1) * CellWidth + 20;
    const y = TopPad + t.position * CellHeight + (t.kind === 'machine' ? 12 : 0);
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
      ...(base.machineCountText ? { machineCountText: base.machineCountText } : {}),
    };
  });

  if (nodes.length > 0) {
    let minX = Number.POSITIVE_INFINITY;
    let minY = Number.POSITIVE_INFINITY;
    nodes.forEach((n) => {
      minX = Math.min(minX, n.x);
      minY = Math.min(minY, n.y);
    });
    const shiftX = minX < 20 ? 20 - minX : 0;
    const shiftY = minY < 20 ? 20 - minY : 0;
    if (shiftX || shiftY) {
      nodes.forEach((n) => {
        n.x += shiftX;
        n.y += shiftY;
      });
    }
  }

  const nodeByUid = new Map(nodes.map((n) => [n.uid, n] as const));
  const styleByPair = new Map<
    string,
    { lineWidth: number; stroke: string; opacity: number; strokeDasharray: string }
  >();
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

  const edges: EdgeView[] = treeEdges.map((e, i) => {
    const source = nodeByUid.get(e.source.uid)!;
    const target = nodeByUid.get(e.target.uid)!;
    const style = styleByPair.get(`${e.target.id}->${e.source.id}`) ?? {
      lineWidth: 2,
      stroke: '#666666',
      opacity: 0.95,
      strokeDasharray: '',
    };
    return {
      id: `te_${i}`,
      sourceUid: e.source.uid,
      targetUid: e.target.uid,
      path: orthPath(source, target),
      lineWidth: style.lineWidth,
      stroke: style.stroke,
      opacity: style.opacity,
      strokeDasharray: style.strokeDasharray,
    };
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
    minX = Math.min(minX, n.x - 40);
    minY = Math.min(minY, n.y - 40);
    maxX = Math.max(maxX, n.x + n.width + 80);
    maxY = Math.max(maxY, n.y + n.height + 80);
  });
  return { minX, minY, maxX, maxY };
});

const boardWidth = computed(() =>
  Math.max(960, Math.ceil(bounds.value.maxX - Math.min(0, bounds.value.minX))),
);
const boardHeight = computed(() =>
  Math.max(520, Math.ceil(bounds.value.maxY - Math.min(0, bounds.value.minY))),
);

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
const markerPrefix = `fz_${Math.random().toString(36).slice(2, 8)}`;
function markerId(edgeId: string): string {
  return `${markerPrefix}_${edgeId.replace(/[^a-zA-Z0-9_-]/g, '_')}`;
}
const zoom = ref(1);
const panX = ref(24);
const panY = ref(24);
const isPanning = ref(false);
const hasInteracted = ref(false);

const stageStyle = computed(() => ({
  transform: `translate3d(${panX.value}px, ${panY.value}px, 0) scale(${zoom.value})`,
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
  const cx = clientX ?? rect.left + rect.width / 2;
  const cy = clientY ?? rect.top + rect.height / 2;
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

watch(
  [boardWidth, boardHeight],
  async () => {
    if (hasInteracted.value) return;
    await nextTick();
    fitView();
  },
  { immediate: true },
);

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
  border: 1px solid rgba(0, 0, 0, 0.08);
  background: #fff;
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
  background: rgba(255, 255, 255, 0.92);
  border: 1px solid rgba(100, 116, 139, 0.26);
}

.fresky-flow__tool-btn {
  border: 1px solid rgba(100, 116, 139, 0.45);
  background: rgba(255, 255, 255, 0.96);
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
  display: grid;
  grid-template-rows: 48px 24px;
  grid-template-columns: 12px 48px 12px;
}

.fresky-node--machine {
  display: grid;
  grid-template-columns: 12px 72px 12px;
  grid-template-rows: 12px 24px 12px;
  position: relative;
  cursor: default;
}

.fresky-node__machine-inner {
  display: none;
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
  grid-area: 1 / 2 / 1 / 2;
  width: 48px;
  height: 48px;
  border-radius: 4px;
  background: #d7dde4;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
  display: grid;
  place-items: center;
  cursor: pointer;
}

.fresky-node__icon {
  width: 48px;
  height: 48px;
  object-fit: contain;
  image-rendering: auto;
}

.fresky-node__sprite-wrap {
  width: 48px;
  height: 48px;
  overflow: hidden;
  border-radius: 4px;
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
  width: 48px;
  height: 48px;
  border-radius: 4px;
  display: grid;
  place-items: center;
  font-weight: 700;
  color: #1f2937;
}

.fresky-node__duplicate-dot {
  width: 48px;
  height: 48px;
  display: grid;
  place-items: center;
  font-weight: 700;
  font-size: 20px;
  letter-spacing: 1px;
  color: #1f2937;
}

.fresky-node__title {
  grid-area: 2 / 1 / 2 / 4;
  width: 108px;
  margin-left: -18px;
  font-size: 12px;
  line-height: 16px;
  padding: 4px 0;
  text-align: center;
  color: #111827;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__sub {
  display: none;
}

.fresky-node__machine-title {
  grid-area: 2 / 2 / 2 / 2;
  line-height: 18px;
  text-align: center;
  font-size: 12px;
  width: 64px;
  margin: 0 auto;
  padding: 3px 4px 1px 4px;
  border-radius: 4px;
  background: #d7dde4;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
  color: #111827;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.fresky-node__machine-sub {
  grid-area: 1 / 2 / 1 / 2;
  font-size: 11px;
  line-height: 12px;
  text-align: center;
  color: #374151;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.fresky-node__machine-count {
  position: absolute;
  top: -2px;
  right: -2px;
  font-size: 10px;
  line-height: 12px;
  padding: 0 4px;
  border-radius: 6px;
  background: rgba(15, 23, 42, 0.88);
  color: #e2e8f0;
}

.fresky-node--selected.fresky-node--machine {
  border-color: transparent;
  box-shadow: none;
}

.fresky-node--selected.fresky-node--machine .fresky-node__machine-title,
.fresky-node--selected .fresky-node__icon-wrap {
  box-shadow:
    0 0 0 2px rgba(245, 166, 35, 0.85),
    2px 2px 5px rgba(0, 0, 0, 0.2);
}

.fresky-node--surplus .fresky-node__icon-wrap {
  box-shadow:
    0 0 0 2px rgba(245, 158, 11, 0.8),
    2px 2px 5px rgba(0, 0, 0, 0.2);
}

.fresky-node--duplicate .fresky-node__icon-wrap {
  background: transparent;
  box-shadow: none;
}

.body--dark .fresky-flow {
  border-color: rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.04);
}

.body--dark .fresky-flow__toolbar {
  background: rgba(15, 23, 42, 0.78);
  border-color: rgba(148, 163, 184, 0.33);
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
  border-color: transparent;
  background: transparent;
}

.body--dark .fresky-node__icon-wrap {
  background: rgba(73, 95, 123, 0.88);
  border-color: rgba(148, 163, 184, 0.4);
}

.body--dark .fresky-node__icon-fallback {
  color: #cbd5e1;
  background: rgba(148, 163, 184, 0.18);
}

.body--dark .fresky-node__machine-title {
  background: rgba(9, 28, 62, 0.86);
  color: #e2e8f0;
}

.body--dark .fresky-node__machine-count {
  background: rgba(226, 232, 240, 0.16);
  color: #f8fafc;
}
</style>
