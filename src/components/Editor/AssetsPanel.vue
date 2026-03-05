<script setup lang="ts">
import { useEditorStore } from '../../stores/editor';
import { Square, Type, Image as ImageIcon, Layout, Box, GalleryHorizontal } from 'lucide-vue-next';

const store = useEditorStore();

const basicComponents = [
  { type: 'rect', name: '矩形', icon: Square },
  { type: 'text', name: '文本', icon: Type },
  { type: 'image', name: '图片', icon: ImageIcon },
  { type: 'frame', name: '画框', icon: Layout },
  { type: 'carousel', name: '轮播图', icon: GalleryHorizontal },
];

const containerComponents = [
  { type: 'flex', name: 'Flex布局', icon: Box },
];

function handleDragStart(e: DragEvent, type: string) {
  e.dataTransfer?.setData('application/vizcraft-component', type);
  if (e.dataTransfer) {
    e.dataTransfer.effectAllowed = 'copy';
    
    // Create an empty image to hide the default drag ghost
    const img = new Image();
    img.src = 'data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7'; // Transparent 1x1 pixel
    e.dataTransfer.setDragImage(img, 0, 0);
  }
  // Set global variable for dragover access (since dataTransfer is protected in dragover)
  (window as any).__draggedComponentType = type;
}

function handleDragEnd() {
  (window as any).__draggedComponentType = null;
}
</script>

<template>
  <div class="assets-panel">
    <div class="panel-content">
      <div class="section-title">容器组件</div>
      <div class="component-grid">
        <div 
          v-for="item in containerComponents" 
          :key="item.type"
          class="component-item"
          draggable="true"
          @dragstart="handleDragStart($event, item.type)"
          @dragend="handleDragEnd"
        >
          <div class="icon-wrapper">
            <component :is="item.icon" :size="24" />
          </div>
          <span class="component-name">{{ item.name }}</span>
        </div>
      </div>

      <div class="section-title" style="margin-top: 24px;">基础组件</div>
      <div class="component-grid">
        <div 
          v-for="item in basicComponents" 
          :key="item.type"
          class="component-item"
          draggable="true"
          @dragstart="handleDragStart($event, item.type)"
          @dragend="handleDragEnd"
        >
          <div class="icon-wrapper">
            <component :is="item.icon" :size="24" />
          </div>
          <span class="component-name">{{ item.name }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.assets-panel {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.section-title {
  height: 40px;
  display: flex;
  align-items: center;
  /* padding: 0 16px; Remove horizontal padding as it is inside panel-content which has padding */
  padding-bottom: 8px;
  border-bottom: 1px solid #333;
  font-weight: 600;
  font-size: 12px;
  color: #eeeeee;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
}

.panel-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.component-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
}

.component-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: #333;
  border-radius: 6px;
  padding: 16px 8px;
  cursor: grab;
  transition: all 0.2s;
  border: 1px solid transparent;
}

.component-item:hover {
  background-color: #444;
  border-color: #666;
  transform: translateY(-2px);
}

.component-item:active {
  cursor: grabbing;
}

.icon-wrapper {
  margin-bottom: 8px;
  color: #ccc;
}

.component-name {
  font-size: 12px;
  color: #aaa;
}
</style>
