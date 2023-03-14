<template>
  <div class="node">
    <div class="node__body" :class="{ 'node__body--file': node.child.length === 0 }">
      <img :src="node.child.length > 0 ? folderImgUrl: fileImgUrl">
      <p class="node__name" @click="handleClick(node)">{{ node.name }}</p>
    </div>
    <div class="node__child">
      <FolderTreeNode v-for="child in node.child" :node="child" :key="child.path" @click-node="handleClick($event)" />
    </div>
  </div>
</template>

<script setup lang="ts">
import folderImgUrl from '@/assets/folder.png'
import fileImgUrl from '@/assets/files.png'

import { PropType } from 'vue'

export type DirSys = {
  path: string
  name: string
  child: DirSys[]
}

const emits = defineEmits(['click-node'])

function handleClick (node: DirSys) {
  if (node.child.length > 0) return
  // console.log('why', node)
  emits('click-node', node)
}

defineProps({
  node: {
    type: Object as PropType<DirSys>,
    required: true,
  },
})
</script>

<style lang="scss">
.node {
  .node__name {
    font-weight: bold;
    margin: 0;
  }

  .node__body {
    display: flex;
    align-items: center;
    gap: 8px;
    
    &.node__body--file {
      cursor: pointer;
    }

    img {
      width: 16px;
      height: 16px;
    }
  }

  .node__child {
    margin-left: 1rem;
  }
}
</style>