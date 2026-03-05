<script setup lang="ts">
import { computed, ref, nextTick, onMounted, onUnmounted, watch } from 'vue';
import { useEditorStore } from '../../stores/editor';
import { calculateSnapping, flattenElements } from '../../utils/snapping';
import type { SnapGuide } from '../../utils/snapping';
import { Code } from 'lucide-vue-next';

const props = defineProps<{
  element: any;
  parent?: any;
}>();

const store = useEditorStore();
const isEditing = ref(false);
const textareaRef = ref<HTMLTextAreaElement | null>(null);

// Frame label rename
const isRenamingFrame = ref(false);
const frameNameInput = ref<HTMLInputElement | null>(null);
const tempFrameName = ref('');

const wrapperStyle = computed(() => {
  const { x, y, width, height, rotation, style, type } = props.element;
  
  const s: any = {
    boxSizing: 'border-box',
    border: style.stroke ? `${style.strokeWidth || 1}px ${style.borderStyle || 'solid'} ${style.stroke}` : 'none',
    backgroundColor: style.fill || 'transparent',
    opacity: style.opacity ?? 1,
    borderRadius: style.borderRadius || '0px',
    borderTopLeftRadius: style.borderTopLeftRadius,
    borderTopRightRadius: style.borderTopRightRadius,
    borderBottomRightRadius: style.borderBottomRightRadius,
    borderBottomLeftRadius: style.borderBottomLeftRadius,
  };

  if (props.parent && props.parent.type === 'flex') {
    s.position = 'relative';
    s.width = `${width}px`;
    s.height = `${height}px`;
    // Flex items usually don't support rotation via transform the same way without affecting flow, 
    // but we'll keep it for now.
    s.transform = `rotate(${rotation}deg)`;
    s.flexShrink = 0; // Prevent shrinking by default
  } else {
    s.position = 'absolute';
    s.left = `${x}px`;
    s.top = `${y}px`;
    s.width = `${width}px`;
    s.height = `${height}px`;
    s.transform = `rotate(${rotation}deg)`;
  }

  if (type === 'frame') {
    s.backgroundColor = style.fill || '#ffffff'; 
    // Removed overflow: hidden from wrapper
  }

  if (type === 'text') {
    s.backgroundColor = 'transparent'; // Text background usually transparent
    // If user wants background, they can set it, but usually text tool implies text color.
    // But we used style.fill as text color for text.
    // So background should be transparent.
  }

  if (!props.element.visible) {
    s.display = 'none';
  }

  return s;
});

const contentStyle = computed(() => {
  const { type } = props.element;
  const s: any = {
    position: 'absolute',
    top: 0,
    left: 0,
    width: '100%',
    height: '100%',
  };
  
  if (type === 'frame' || type === 'carousel' || type === 'flex') {
    s.overflow = 'hidden';
    
    // Apply border radius to content as well to clip children
    const { style } = props.element;
    if (style.borderRadius) s.borderRadius = style.borderRadius;
    if (style.borderTopLeftRadius) s.borderTopLeftRadius = style.borderTopLeftRadius;
    if (style.borderTopRightRadius) s.borderTopRightRadius = style.borderTopRightRadius;
    if (style.borderBottomRightRadius) s.borderBottomRightRadius = style.borderBottomRightRadius;
    if (style.borderBottomLeftRadius) s.borderBottomLeftRadius = style.borderBottomLeftRadius;
  }

  if (type === 'flex') {
    s.display = 'flex';
    s.flexDirection = props.element.style.flexDirection || 'row';
    s.justifyContent = props.element.style.justifyContent || 'flex-start';
    s.alignItems = props.element.style.alignItems || 'flex-start';
    s.flexWrap = props.element.style.flexWrap || 'nowrap';
    s.gap = props.element.style.gap || '0px';
    s.padding = props.element.style.padding || '0px';
  }
  
  return s;
});

const textStyle = computed(() => {
  const { style } = props.element;
  return {
    fontSize: `${style.fontSize || 16}px`,
    fontFamily: style.fontFamily || 'Arial',
    lineHeight: style.lineHeight || 1.2,
    letterSpacing: `${style.letterSpacing || 0}px`,
    textAlign: style.textAlign || 'left',
    color: style.fill || '#000000', // Use fill as text color
    whiteSpace: 'pre-wrap',
    width: '100%',
    height: '100%',
    outline: 'none',
    border: 'none',
    background: 'transparent',
    resize: 'none' as const,
    overflow: 'hidden',
    padding: '0',
    margin: '0',
    display: 'block',
  };
});

function onMouseDown(e: MouseEvent) {
  if (props.element.locked) return;
  
  // Stop propagation to prevent parent elements from being selected
  e.stopPropagation();
  
  // Mark that mouse was pressed on an element
  store.mouseDownOnElement = true;
  
  // If clicking on an already selected element (and not holding Shift), don't change selection
  // This allows dragging multiple selected elements
  if (store.selectedElementIds.includes(props.element.id) && !e.shiftKey) {
    // Element is already selected, don't change selection
    return;
  }
  
  store.selectElement(props.element.id, e.shiftKey);
}

function onDblClick(e: MouseEvent) {
  if (props.element.type === 'text' && !props.element.locked) {
    e.stopPropagation();
    isEditing.value = true;
    nextTick(() => {
      textareaRef.value?.focus();
    });
  }
}

function onBlur() {
  if (isEditing.value) {
    isEditing.value = false;
  }
}

function updateContent(e: Event) {
  const val = (e.target as HTMLTextAreaElement).value;
  store.updateElement(props.element.id, { content: val });
}

// Frame label rename functions
function startFrameRename(e: MouseEvent) {
  e.stopPropagation();
  isRenamingFrame.value = true;
  tempFrameName.value = props.element.name;
  
  nextTick(() => {
    if (frameNameInput.value) {
      frameNameInput.value.focus();
      frameNameInput.value.select();
    }
  });
}

function finishFrameRename() {
  if (tempFrameName.value.trim()) {
    store.updateElement(props.element.id, { name: tempFrameName.value.trim() });
  }
  isRenamingFrame.value = false;
}

function cancelFrameRename() {
  isRenamingFrame.value = false;
  tempFrameName.value = '';
}

function handleFrameRenameKeydown(e: KeyboardEvent) {
  if (e.key === 'Enter') {
    finishFrameRename();
  } else if (e.key === 'Escape') {
    cancelFrameRename();
  }
}

// Copy code function
function copyFrameCode(e: MouseEvent) {
  e.stopPropagation();
  
  if (props.element.type !== 'frame') return;
  
  const code = store.exportToUVUE(props.element);
  
  // Copy to clipboard
  navigator.clipboard.writeText(code).then(() => {
    // Show success feedback (you can add a toast notification here)
    console.log('代码已复制到剪贴板');
  }).catch(err => {
    console.error('复制失败:', err);
  });
}

function onWrapperMouseMove(e: MouseEvent) {
  // Only handle hover logic if not dragging canvas/element
  e.stopPropagation(); // Stop parent from getting hovered
  store.hoveredElementId = props.element.id;
}

function onMouseLeave() {
  // Don't clear hover immediately to allow resize handles to work
}

function onResizeMouseDown(e: MouseEvent, edge: string) {
  if (props.element.locked) return;
  
  e.stopPropagation();
  e.preventDefault();
  
  // Select element if not already selected
  if (!store.selectedElementIds.includes(props.element.id)) {
    store.selectElement(props.element.id);
  }
  
  // Set flag to prevent dragging during resize
  store.isResizing = true;
  
  const startX = e.clientX;
  const startY = e.clientY;
  const startElementX = props.element.x;
  const startElementY = props.element.y;
  const startWidth = props.element.width;
  const startHeight = props.element.height;
  
  // Calculate snap targets once at start of drag
  const snapTargets = flattenElements(
    store.elements, 
    (item) => store.getGlobalPosition(item),
    props.element.id
  );
  
  const handleMouseMove = (me: MouseEvent) => {
    me.preventDefault();
    me.stopPropagation();
    
    const dx = (me.clientX - startX) / store.scale;
    const dy = (me.clientY - startY) / store.scale;
    
    let newX = startElementX;
    let newY = startElementY;
    let newWidth = startWidth;
    let newHeight = startHeight;
    
    // Handle horizontal resize
    if (edge.includes('left')) {
      newX = startElementX + dx;
      newWidth = startWidth - dx;
    } else if (edge.includes('right')) {
      newWidth = startWidth + dx;
    }
    
    // Handle vertical resize
    if (edge.includes('top')) {
      newY = startElementY + dy;
      newHeight = startHeight - dy;
    } else if (edge.includes('bottom')) {
      newHeight = startHeight + dy;
    }

    // Maintain aspect ratio if Shift is pressed (only for corner resizing)
      if (me.shiftKey && (edge.includes('left') || edge.includes('right')) && (edge.includes('top') || edge.includes('bottom'))) {
        
        // Calculate the ratio based on the larger change
        const scaleX = newWidth / startWidth;
        const scaleY = newHeight / startHeight;
        
        // Use the larger scale factor (in absolute terms) to drive both dimensions
        // Note: we use Math.max to find the "most moved" axis
        const scale = Math.abs(scaleX - 1) > Math.abs(scaleY - 1) ? scaleX : scaleY;
        
        const adjustedWidth = startWidth * scale;
        const adjustedHeight = startHeight * scale;
        
        // Adjust position if resizing from top or left
        if (edge.includes('left')) {
          newX = startElementX + (startWidth - adjustedWidth);
        }
        if (edge.includes('top')) {
          newY = startElementY + (startHeight - adjustedHeight);
        }
        
        newWidth = adjustedWidth;
        newHeight = adjustedHeight;
      }
    
    // Minimum size constraint
    const minSize = 10;
    if (newWidth < minSize) {
      if (edge.includes('left')) {
        newX = startElementX + startWidth - minSize;
      }
      newWidth = minSize;
    }
    if (newHeight < minSize) {
      if (edge.includes('top')) {
        newY = startElementY + startHeight - minSize;
      }
      newHeight = minSize;
    }
    
    // Snapping Logic
    // We need to convert local coordinates to global for snapping calculation
    // Then convert back to local after snapping
    
    // 1. Get current global position of parent (if any)
    let parentGlobalX = 0;
    let parentGlobalY = 0;
    if (props.parent) {
      const parentGlobal = store.getGlobalPosition(props.parent);
      parentGlobalX = parentGlobal.x;
      parentGlobalY = parentGlobal.y;
    }
    
    // 2. Calculate proposed global position
    const proposedGlobalX = parentGlobalX + newX;
    const proposedGlobalY = parentGlobalY + newY;
    
    // 3. Perform snapping
    // We only snap edges that are being moved
    // Construct a "proposed rect" for snapping
    const snapResult = calculateSnapping(
      { x: proposedGlobalX, y: proposedGlobalY, width: newWidth, height: newHeight },
      snapTargets
    );
    
    // 4. Apply snap adjustments only to the edges being moved
    // Calculate how much the snap moved the rect globally
    const snapDeltaX = snapResult.x - proposedGlobalX;
    const snapDeltaY = snapResult.y - proposedGlobalY;
    
    // Apply X snap
    if (snapDeltaX !== 0) {
      if (edge.includes('left')) {
        // If moving left edge, the x changes and width changes
        newX += snapDeltaX;
        newWidth -= snapDeltaX;
      } else if (edge.includes('right')) {
        // If moving right edge, only width changes (x stays same, but we need to check if right edge snapped)
        // calculateSnapping returns new x/y for the top-left corner.
        // But if we are resizing right, we want the right edge to snap.
        // calculateSnapping logic snaps based on left/center/right.
        // If the right edge snapped, snapResult.x might have changed to accommodate width?
        // Actually calculateSnapping assumes moving the whole rect. 
        // We need a resizing-specific snap or adapt the result.
        
        // Let's refine: calculateSnapping returns the new TOP-LEFT x/y that aligns BEST.
        // If we are resizing RIGHT, we want the RIGHT edge to align.
        // If calculateSnapping suggests a new X, it implies a shift. 
        // But we are anchored at left.
        
        // Better approach: Check which guides are active and apply manually
        // But for simplicity, let's look at the result.
        // If we resizing right, we shouldn't change X.
        // So we should only look at guides that affect the Right edge.
        // But calculateSnapping is generic.
        
        // Let's try to infer:
        // If resizing right, we want newWidth = snapTarget - startX
        
        // Re-implement simplified snapping for resize to avoid fighting the generic mover
        
        // Actually, let's use the guides from snapResult to determine.
        // But snapResult.x/y is "best position". 
        
        // Alternative: Use snapResult.x/y but only if it matches our moving edge intent.
        
        // If resizing right:
        // We want (newX + newWidth) to snap. 
        // The generic snapper tries to snap Left, Center, Right.
        // If it snapped Right, then snapResult.x + width (if width kept same) would align.
        // But here width is changing.
        
        // Let's do a custom resize snap using the helper logic but applied to edges.
        
        // For now, let's just apply the delta to the moving side.
        newWidth += snapDeltaX;
      }
    }
    
    // Re-do snapping properly for Resize
    // We need to snap specific edges
    
    // --- Custom Resize Snapping ---
    let snappedX = newX;
    let snappedY = newY;
    let snappedWidth = newWidth;
    let snappedHeight = newHeight;
    const guides: SnapGuide[] = [];
    const threshold = 5;
    
    // Global edges
    const currentGlobalLeft = parentGlobalX + newX;
    const currentGlobalTop = parentGlobalY + newY;
    const currentGlobalRight = currentGlobalLeft + newWidth;
    const currentGlobalBottom = currentGlobalTop + newHeight;
    
    // Helper to check alignment
    const checkAlign = (val: number, isVertical: boolean) => {
      let bestVal = val;
      let minDiff = threshold + 1;
      let matchedGuide: SnapGuide | null = null;
      
      for (const target of snapTargets) {
        const tLeft = target.x;
        const tRight = target.x + target.width;
        const tTop = target.y;
        const tBottom = target.y + target.height;
        const tCenter = isVertical ? target.x + target.width/2 : target.y + target.height/2;
        
        const compareValues = isVertical 
          ? [tLeft, tRight, tCenter] 
          : [tTop, tBottom, tCenter];
          
        for (const targetVal of compareValues) {
          const diff = Math.abs(val - targetVal);
          if (diff < minDiff) {
            minDiff = diff;
            bestVal = targetVal;
            
            // Create guide
            const start = isVertical ? Math.min(currentGlobalTop, tTop) : Math.min(currentGlobalLeft, tLeft);
            const end = isVertical ? Math.max(currentGlobalBottom, tBottom) : Math.max(currentGlobalRight, tRight);
            
            matchedGuide = {
              type: isVertical ? 'vertical' : 'horizontal',
              position: targetVal,
              start,
              end
            };
          }
        }
      }
      return { val: bestVal, guide: matchedGuide, snapped: minDiff <= threshold };
    };
    
    // Snap X Axis
    if (edge.includes('left')) {
      const res = checkAlign(currentGlobalLeft, true);
      if (res.snapped) {
        const delta = res.val - currentGlobalLeft;
        snappedX += delta;
        snappedWidth -= delta;
        if (res.guide) guides.push(res.guide);
      }
    } else if (edge.includes('right')) {
      const res = checkAlign(currentGlobalRight, true);
      if (res.snapped) {
        const delta = res.val - currentGlobalRight;
        snappedWidth += delta;
        if (res.guide) guides.push(res.guide);
      }
    }
    
    // Snap Y Axis
    if (edge.includes('top')) {
      const res = checkAlign(currentGlobalTop, false);
      if (res.snapped) {
        const delta = res.val - currentGlobalTop;
        snappedY += delta;
        snappedHeight -= delta;
        if (res.guide) guides.push(res.guide);
      }
    } else if (edge.includes('bottom')) {
      const res = checkAlign(currentGlobalBottom, false);
      if (res.snapped) {
        const delta = res.val - currentGlobalBottom;
        snappedHeight += delta;
        if (res.guide) guides.push(res.guide);
      }
    }
    
    // Update active guides in store or component
    // We need to access the parent component (InfiniteCanvas) to update guides
    // For now, we can try to emit or use a shared store state if available.
    // InfiniteCanvas uses a local ref for guides. We might need to move guides to store.
    // Or dispatch a custom event.
    
    // Let's assume we can't easily update guides visualization without store change.
    // But the snapping effect itself (jumping to value) will work.
    // To show guides, we should probably add `alignmentGuides` to the store.
    
    // Update local variables with snapped values
    newX = snappedX;
    newY = snappedY;
    newWidth = snappedWidth;
    newHeight = snappedHeight;
    
    // --- End Custom Resize Snapping ---

    // Update element directly without triggering history on every move
    const el = store.findElement(store.elements, props.element.id);
    if (el) {
      el.x = newX;
      el.y = newY;
      el.width = newWidth;
      el.height = newHeight;
    }
    
    // Emit guides event? 
    // Since we are inside a component, we can use an event bus or store.
    // Let's add setGuides to store for simplicity in this refactor.
    if (store.setAlignmentGuides) {
        store.setAlignmentGuides(guides);
    }
  };
  
  const handleMouseUp = (me: MouseEvent) => {
    me.preventDefault();
    me.stopPropagation();
    
    // Clear resize flag
    store.isResizing = false;
    
    // Clear guides
    if (store.setAlignmentGuides) {
        store.setAlignmentGuides([]);
    }
    
    // Record final state to history
    store.updateElement(props.element.id, {
      x: props.element.x,
      y: props.element.y,
      width: props.element.width,
      height: props.element.height
    });
    
    window.removeEventListener('mousemove', handleMouseMove, true);
    window.removeEventListener('mouseup', handleMouseUp, true);
    document.body.style.cursor = '';
    document.body.style.userSelect = '';
  };
  
  // Set cursor style and disable text selection
  const cursorMap: Record<string, string> = {
    'top': 'ns-resize',
    'bottom': 'ns-resize',
    'left': 'ew-resize',
    'right': 'ew-resize',
    'top-left': 'nwse-resize',
    'top-right': 'nesw-resize',
    'bottom-left': 'nesw-resize',
    'bottom-right': 'nwse-resize',
  };
  document.body.style.cursor = cursorMap[edge] || 'default';
  document.body.style.userSelect = 'none';
  
  // Use capture phase to ensure we get all events
  window.addEventListener('mousemove', handleMouseMove, true);
  window.addEventListener('mouseup', handleMouseUp, true);
}

// Auto-play for carousel
let intervalId: number | null = null;

function startCarousel() {
  stopCarousel();
  if (props.element.type === 'carousel' && props.element.images?.length > 1) {
    const interval = props.element.interval || 3000;
    intervalId = window.setInterval(() => {
      const nextIndex = ((props.element.currentIndex || 0) + 1) % props.element.images.length;
      store.updateElement(props.element.id, { currentIndex: nextIndex });
    }, interval);
  }
}

function stopCarousel() {
  if (intervalId !== null) {
    clearInterval(intervalId);
    intervalId = null;
  }
}

watch(() => [props.element.type, props.element.interval, props.element.images?.length], () => {
  if (props.element.type === 'carousel') {
    startCarousel();
  } else {
    stopCarousel();
  }
});

onMounted(() => {
  if (props.element.type === 'carousel') {
    startCarousel();
  }
});

onUnmounted(() => {
  stopCarousel();
});

watch(() => props.element.interval, () => {
  if (props.element.type === 'carousel') {
    startCarousel();
  }
});

watch(() => props.element.images, () => {
    if (props.element.type === 'carousel') {
        startCarousel();
    }
}, { deep: true });
</script>

<template>
  <div 
    class="element-wrapper"
    :style="wrapperStyle"
    @mousedown="onMouseDown"
    @dblclick="onDblClick"
    @mousemove="onWrapperMouseMove"
    @mouseleave="onMouseLeave"
    :class="{ 
      selected: store.selectedElementIds.includes(element.id), 
      locked: element.locked,
      hovered: store.hoveredElementId === element.id && !store.selectedElementIds.includes(element.id)
    }"
  >
    <!-- Resize handles overlay - always visible when hovered or selected -->
    <div 
      v-if="(store.hoveredElementId === element.id || store.selectedElementIds.includes(element.id)) && !element.locked"
      class="resize-handles"
    >
      <div class="resize-handle top" @mousedown.stop="onResizeMouseDown($event, 'top')"></div>
      <div class="resize-handle right" @mousedown.stop="onResizeMouseDown($event, 'right')"></div>
      <div class="resize-handle bottom" @mousedown.stop="onResizeMouseDown($event, 'bottom')"></div>
      <div class="resize-handle left" @mousedown.stop="onResizeMouseDown($event, 'left')"></div>
      <div class="resize-handle top-left" @mousedown.stop="onResizeMouseDown($event, 'top-left')"></div>
      <div class="resize-handle top-right" @mousedown.stop="onResizeMouseDown($event, 'top-right')"></div>
      <div class="resize-handle bottom-left" @mousedown.stop="onResizeMouseDown($event, 'bottom-left')"></div>
      <div class="resize-handle bottom-right" @mousedown.stop="onResizeMouseDown($event, 'bottom-right')"></div>
    </div>

    <!-- Element Content (with optional overflow clipping) -->
    <div class="element-content" :style="contentStyle">
      <!-- Image Element -->
      <img 
        v-if="element.type === 'image'" 
        :src="element.imageUrl" 
        class="image-element"
        style="width: 100%; height: 100%; object-fit: fill; pointer-events: none;"
        draggable="false"
      />

      <!-- Carousel Element -->
      <div v-if="element.type === 'carousel'" class="carousel-element" style="width: 100%; height: 100%; position: relative; overflow: hidden;">
        <div 
          class="carousel-track"
          :style="{
            transform: `translateX(-${(element.currentIndex || 0) * 100}%)`,
            transition: 'transform 0.3s ease-in-out',
            display: 'flex',
            height: '100%'
          }"
        >
          <div 
            v-for="(img, index) in element.images" 
            :key="index"
            class="carousel-slide"
            style="min-width: 100%; height: 100%;"
          >
            <img 
              :src="img" 
              style="width: 100%; height: 100%; object-fit: cover; pointer-events: none;"
              draggable="false"
            />
          </div>
        </div>
        
        <!-- Navigation Dots -->
        <div class="carousel-dots" style="position: absolute; bottom: 10px; left: 0; right: 0; display: flex; justify-content: center; gap: 6px;">
          <div 
            v-for="(_, index) in element.images" 
            :key="index"
            class="dot"
            :style="{
              width: '8px',
              height: '8px',
              borderRadius: '50%',
              backgroundColor: (element.currentIndex || 0) === index ? '#fff' : 'rgba(255,255,255,0.5)',
              cursor: 'pointer'
            }"
            @click.stop="store.updateElement(element.id, { currentIndex: Number(index) })"
          ></div>
        </div>
      </div>

      <!-- Text Element -->
      <div v-if="element.type === 'text'" class="text-container" style="width: 100%; height: 100%;">
        <textarea 
          v-if="isEditing"
          ref="textareaRef"
          :value="element.content"
          @input="updateContent"
          @blur="onBlur"
          @mousedown.stop
          class="text-editor"
          :style="textStyle"
        ></textarea>
        <div v-else class="text-display" :style="textStyle">
          {{ element.content }}
        </div>
      </div>

      <!-- Recursive rendering -->
      <CanvasElement 
        v-for="child in element.children" 
        :key="child.id" 
        :element="child" 
        :parent="element"
      />
    </div>

    <!-- UI Elements (Labels) - Outside content clipping -->
    <div v-if="element.type === 'frame'" class="frame-label">
      <span v-if="!isRenamingFrame" @dblclick="startFrameRename">{{ element.name }}</span>
      <input 
        v-else
        ref="frameNameInput"
        v-model="tempFrameName"
        class="frame-name-input"
        @blur="finishFrameRename"
        @keydown="handleFrameRenameKeydown"
        @click.stop
        @mousedown.stop
      />
    </div>

    <!-- Copy Code Button - Top Right Corner -->
    <div 
      v-if="element.type === 'frame' && store.selectedElementIds.includes(element.id)" 
      class="copy-code-btn"
      @click="copyFrameCode"
      @mousedown.stop
      title="复制 uvue 代码"
    >
      <Code :size="16" />
    </div>

    <!-- Size Label -->
    <div v-if="store.selectedElementIds.includes(element.id)" class="size-label">
      {{ Math.round(element.width) }} × {{ Math.round(element.height) }}
    </div>
  </div>
</template>

<style scoped>
.element-wrapper {
  pointer-events: auto;
  user-select: none;
}

.element-wrapper.selected {
  outline: 2px solid #007acc;
  /* Removed z-index to preserve original stacking order */
}

.element-wrapper.locked {
  pointer-events: none;
}

.element-wrapper.hovered {
  outline: 2px solid #007acc;
}

/* Ensure content clipping works for frames */
.element-content {
  box-sizing: border-box;
}

.resize-handles {
  position: absolute;
  top: -4px;
  left: -4px;
  width: calc(100% + 8px);
  height: calc(100% + 8px);
  pointer-events: none;
  z-index: 1000;
}

.resize-handle {
  position: absolute;
  pointer-events: auto;
  background: transparent;
  z-index: 1001;
}

/* Edge handles - larger hit areas */
.resize-handle.top {
  top: 0;
  left: 8px;
  width: calc(100% - 16px);
  height: 8px;
  cursor: ns-resize;
}

.resize-handle.bottom {
  bottom: 0;
  left: 8px;
  width: calc(100% - 16px);
  height: 8px;
  cursor: ns-resize;
}

.resize-handle.left {
  left: 0;
  top: 8px;
  width: 8px;
  height: calc(100% - 16px);
  cursor: ew-resize;
}

.resize-handle.right {
  right: 0;
  top: 8px;
  width: 8px;
  height: calc(100% - 16px);
  cursor: ew-resize;
}

/* Corner handles - priority over edges */
.resize-handle.top-left {
  top: 0;
  left: 0;
  width: 8px;
  height: 8px;
  cursor: nwse-resize;
  z-index: 1002;
  background: white;
  border: 1px solid #007acc;
  box-sizing: border-box;
}

.resize-handle.top-right {
  top: 0;
  right: 0;
  width: 8px;
  height: 8px;
  cursor: nesw-resize;
  z-index: 1002;
  background: white;
  border: 1px solid #007acc;
  box-sizing: border-box;
}

.resize-handle.bottom-left {
  bottom: 0;
  left: 0;
  width: 8px;
  height: 8px;
  cursor: nesw-resize;
  z-index: 1002;
  background: white;
  border: 1px solid #007acc;
  box-sizing: border-box;
}

.resize-handle.bottom-right {
  bottom: 0;
  right: 0;
  width: 8px;
  height: 8px;
  cursor: nwse-resize;
  z-index: 1002;
  background: white;
  border: 1px solid #007acc;
  box-sizing: border-box;
}

.frame-label {
  position: absolute;
  top: -24px;
  left: 0;
  color: #666;
  font-size: 12px;
  background: rgba(255,255,255,0.8);
  padding: 2px 4px;
  border-radius: 2px;
  white-space: nowrap;
}

.frame-label span {
  cursor: pointer;
}

.frame-name-input {
  background: white;
  border: 1px solid #007acc;
  color: #333;
  padding: 2px 4px;
  font-size: 12px;
  outline: none;
  border-radius: 2px;
  min-width: 100px;
}

.copy-code-btn {
  position: absolute;
  top: -24px;
  right: 0;
  background: #22c55e;
  color: white;
  padding: 4px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.copy-code-btn:hover {
  background: #16a34a;
  transform: scale(1.05);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.copy-code-btn:active {
  transform: scale(0.95);
}

.size-label {
  position: absolute;
  bottom: -24px;
  left: 50%;
  transform: translateX(-50%);
  background-color: #007acc;
  color: white;
  padding: 2px 6px;
  border-radius: 2px;
  font-size: 10px;
  white-space: nowrap;
  pointer-events: none;
  z-index: 200; /* Ensure label is on top */
}
</style>
