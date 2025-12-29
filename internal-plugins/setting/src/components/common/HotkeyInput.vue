<template>
  <div class="hotkey-input" :class="{ recording: isRecording }" @click="startRecording">
    {{ displayHotkey }}
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from 'vue'

interface Props {
  modelValue: string
  placeholder?: string
  platform?: 'darwin' | 'win32' | 'linux'
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '点击录制快捷键',
  platform: 'darwin'
})

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void
  (e: 'change', value: string): void
}>()

// 录制状态
const isRecording = ref(false)
const recordedKeys = ref<string[]>([])

// 显示的快捷键文本
const displayHotkey = computed(() => {
  if (isRecording.value) {
    return recordedKeys.value.length > 0 ? recordedKeys.value.join('+') : '请按下快捷键...'
  }
  return props.modelValue || props.placeholder
})

// 开始录制快捷键
async function startRecording(): Promise<void> {
  isRecording.value = true
  recordedKeys.value = []

  // 请求后端注册临时快捷键监听
  try {
    const result = await window.ztools.internal.startHotkeyRecording()
    if (result.success) {
      console.log('已启动后端快捷键监听')
    } else {
      console.warn('启动后端快捷键监听失败，使用前端监听:', result.error)
    }
  } catch (error) {
    console.error('启动后端快捷键监听异常，使用前端监听:', error)
  }

  // 同时监听前端键盘事件作为备用
  document.addEventListener('keydown', handleKeyDown)
  document.addEventListener('keyup', handleKeyUp)
}

// 停止录制
function stopRecording(): void {
  isRecording.value = false
  document.removeEventListener('keydown', handleKeyDown)
  document.removeEventListener('keyup', handleKeyUp)
}

// 处理按键
function handleKeyDown(e: KeyboardEvent): void {
  e.preventDefault()
  e.stopPropagation()

  const keys: string[] = []

  // 修饰键（根据平台区分 Alt 文案）
  if (e.metaKey) keys.push('Command')
  if (e.ctrlKey) keys.push('Ctrl')
  if (e.altKey) keys.push(props.platform === 'win32' ? 'Alt' : 'Option')
  if (e.shiftKey) keys.push('Shift')

  // 主键 - 使用 e.code 避免 Option 键产生特殊字符
  if (
    e.code &&
    ![
      'MetaLeft',
      'MetaRight',
      'ControlLeft',
      'ControlRight',
      'AltLeft',
      'AltRight',
      'ShiftLeft',
      'ShiftRight'
    ].includes(e.code)
  ) {
    // 处理 e.code 转换为友好格式
    let mainKey = ''

    if (e.code.startsWith('Key')) {
      // KeyA -> A, KeyB -> B
      mainKey = e.code.replace('Key', '')
    } else if (e.code.startsWith('Digit')) {
      // Digit1 -> 1, Digit2 -> 2
      mainKey = e.code.replace('Digit', '')
    } else if (e.code.startsWith('Numpad')) {
      // Numpad1 -> Numpad1
      mainKey = e.code
    } else {
      // Space, Enter, Tab 等
      mainKey = e.code
    }

    if (mainKey) {
      keys.push(mainKey)
    }
  }

  recordedKeys.value = keys
}

// 按键抬起时确认快捷键
function handleKeyUp(e: KeyboardEvent): void {
  e.preventDefault()
  e.stopPropagation()

  if (recordedKeys.value.length > 1) {
    // 至少需要一个修饰键 + 一个主键
    const newHotkey = recordedKeys.value.join('+')
    emit('update:modelValue', newHotkey)
    emit('change', newHotkey)
  }

  stopRecording()
}

// 处理后端传来的快捷键（立即确认）
function handleHotkeyRecorded(shortcut: string): void {
  if (isRecording.value) {
    console.log('收到后端快捷键录制事件:', shortcut)
    // 直接设置快捷键
    recordedKeys.value = shortcut.split('+')
    emit('update:modelValue', shortcut)
    emit('change', shortcut)
    stopRecording()
  }
}

// 组件挂载时监听后端快捷键事件
onMounted(() => {
  if (window.ztools.internal.onHotkeyRecorded) {
    window.ztools.internal.onHotkeyRecorded(handleHotkeyRecorded)
  }
})

// 组件卸载时清理监听
onUnmounted(() => {
  stopRecording()
})

// 暴露方法供父组件调用
defineExpose({
  stopRecording
})
</script>

<style scoped>
.hotkey-input {
  min-width: 150px;
  padding: 6px 16px;
  border: 2px solid var(--control-border);
  border-radius: 6px;
  background: var(--control-bg);
  color: var(--text-color);
  font-size: 14px;
  font-weight: 500;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
  user-select: none;
}

.hotkey-input:hover {
  background: var(--hover-bg);
  border-color: color-mix(in srgb, var(--primary-color), black 15%);
}

.hotkey-input.recording {
  border-color: color-mix(in srgb, var(--primary-color), black 15%);
  background: var(--primary-light-bg);
  color: var(--primary-color);
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.7;
  }
}
</style>
