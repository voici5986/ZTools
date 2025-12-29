<template>
  <DetailPanel :title="editingShortcut ? '编辑全局快捷键' : '添加全局快捷键'" @back="emit('back')">
    <div class="editor-wrapper">
      <div class="editor-content">
        <!-- 快捷键录制 -->
        <div class="form-item">
          <label class="form-label">快捷键</label>
          <HotkeyInput
            v-model="recordedShortcut"
            :platform="platform"
            placeholder="点击录制快捷键"
          />
          <span class="form-hint">点击上方输入框录制快捷键</span>
        </div>

        <!-- 目标指令输入 -->
        <div class="form-item">
          <label class="form-label">目标指令</label>
          <input
            v-model="targetCommand"
            type="text"
            class="input"
            placeholder="格式: 插件名称/指令名称，例如: 翻译/translate"
          />
          <span class="form-hint">格式: 插件名称/指令名称（支持动态指令）</span>
        </div>
      </div>

      <div class="editor-footer">
        <button class="btn" @click="emit('back')">取消</button>
        <button class="btn" :disabled="!recordedShortcut || !targetCommand" @click="handleSave">
          确定
        </button>
      </div>
    </div>
  </DetailPanel>
</template>

<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import DetailPanel from '../common/DetailPanel.vue'
import HotkeyInput from '../common/HotkeyInput.vue'

interface GlobalShortcut {
  id: string
  shortcut: string
  target: string
  enabled: boolean
}

const props = defineProps<{
  editingShortcut?: GlobalShortcut | null
}>()

const emit = defineEmits<{
  (e: 'back'): void
  (e: 'save', shortcut: string, target: string): void
}>()

// 平台信息（用于区分 Alt/Option 显示）
const platform = ref<'darwin' | 'win32' | 'linux'>('darwin')

// 录制状态
const recordedShortcut = ref('')
const targetCommand = ref('')

// 初始化编辑数据
watch(
  () => props.editingShortcut,
  (newVal) => {
    if (newVal) {
      recordedShortcut.value = newVal.shortcut
      targetCommand.value = newVal.target
    } else {
      recordedShortcut.value = ''
      targetCommand.value = ''
    }
  },
  { immediate: true }
)

// 保存
function handleSave(): void {
  if (!recordedShortcut.value || !targetCommand.value) {
    return
  }
  emit('save', recordedShortcut.value, targetCommand.value)
}

// 初始化平台信息
onMounted(() => {
  const pf = window.ztools.internal.getPlatform()
  if (pf === 'darwin' || pf === 'win32' || pf === 'linux') {
    platform.value = pf
  }
})
</script>

<style scoped>
.editor-wrapper {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.editor-content {
  flex: 1;
  padding: 24px;
}

.form-item {
  margin-bottom: 24px;
}

.form-item:last-child {
  margin-bottom: 0;
}

.form-label {
  display: block;
  font-size: 14px;
  font-weight: 500;
  color: var(--text-color);
  margin-bottom: 8px;
}

.form-hint {
  display: block;
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 6px;
}

.editor-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding: 16px 24px;
  border-top: 1px solid var(--divider-color);
  background: var(--bg-color);
}
</style>
