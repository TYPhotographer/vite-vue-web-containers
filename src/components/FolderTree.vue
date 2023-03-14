<template>
  <FolderTreeNode v-for="child in list" :node="child" :key="child.path" @click-node="$emit('click-node', $event)" />
</template>

<script setup lang="ts">
import { PropType } from 'vue'
import FolderTreeNode, { type DirSys } from './FolderTreeNode.vue';

defineEmits(['click-node'])

defineProps({
  list: {
    type: Array as PropType<DirSys[]>,
    required: true,
  },
})
</script>