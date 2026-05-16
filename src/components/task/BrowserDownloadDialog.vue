<script setup lang="ts">
/** @fileoverview 浏览器下载信息弹窗组件 */
import { ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { NModal, NCard, NForm, NFormItem, NInput, NButton, NSpace, NIcon, NText } from 'naive-ui'
import { open as openDialog } from '@tauri-apps/plugin-dialog'
import { downloadDir } from '@tauri-apps/api/path'
import { FolderOpenOutline } from '@vicons/ionicons5'

interface BrowserDownloadData {
  url: string
  referer?: string
  cookie?: string
  filename?: string
  fileSize?: string
}

const props = defineProps<{
  show: boolean
  data: BrowserDownloadData | null
}>()

const emit = defineEmits<{
  close: []
  confirm: [data: { dir: string; filename: string }]
}>()

const { t } = useI18n()

const form = ref({
  url: '',
  dir: '',
  filename: '',
})

watch(
  () => props.show,
  async (visible) => {
    if (visible && props.data) {
      form.value.url = props.data.url || ''
      form.value.filename = props.data.filename || ''
      try {
        if (!form.value.dir) {
          form.value.dir = await downloadDir()
        }
      } catch {
        form.value.dir = '~/Downloads'
      }
    }
  },
)

watch(
  () => props.data,
  (data) => {
    if (data) {
      form.value.url = data.url || ''
      form.value.filename = data.filename || ''
    }
  },
)

async function handleSelectDirectory() {
  try {
    const selected = await openDialog({ directory: true })
    if (typeof selected === 'string') {
      form.value.dir = selected
    }
  } catch {
    console.error('Failed to select directory')
  }
}

function handleConfirm() {
  emit('confirm', {
    dir: form.value.dir,
    filename: form.value.filename,
  })
  handleClose()
}

function handleClose() {
  emit('close')
  Object.assign(form.value, {
    url: '',
    dir: '',
    filename: '',
  })
}
</script>

<template>
  <NModal
    :show="show"
    transform-origin="center"
    :mask-closable="false"
    :close-on-esc="true"
    @update:show="
      (v: boolean) => {
        if (!v) handleClose()
      }
    "
  >
    <NCard
      :title="t('browser-download.title') || '下载文件信息'"
      closable
      class="browser-download-card"
      :style="{
        width: '580px',
        maxWidth: '90vw',
      }"
      :content-style="{ padding: '0 24px 24px' }"
      @close="handleClose"
    >
      <NForm label-placement="left" label-width="80px">
        <NFormItem :label="t('browser-download.url') || '下载链接:'" style="margin-bottom: 16px">
          <NInput
            v-model:value="form.url"
            type="textarea"
            :rows="2"
            readonly
            :placeholder="t('browser-download.url-placeholder') || '下载链接将自动填入...'"
          />
        </NFormItem>

        <NFormItem :label="t('browser-download.custom-path') || '自定义路径:'" style="margin-bottom: 16px">
          <div class="path-input-group">
            <NInput
              v-model:value="form.dir"
              :placeholder="t('browser-download.path-placeholder') || '留空自动分类'"
              style="flex: 1"
            />
            <NButton
              v-if="data?.fileSize"
              quaternary
              size="small"
              style="margin-left: 8px; color: var(--text-muted)"
              disabled
            >
              {{ data.fileSize }}
            </NButton>
            <NButton style="margin-left: 8px" @click="handleSelectDirectory">
              <template #icon>
                <NIcon><FolderOpenOutline /></NIcon>
              </template>
            </NButton>
          </div>
        </NFormItem>

        <div class="auto-category-hint" style="margin-left: 96px; margin-bottom: 16px">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            viewBox="0 0 24 24"
            width="13"
            height="13"
            style="flex-shrink: 0; vertical-align: middle; margin-right: 4px; color: var(--text-muted)"
          >
            <circle cx="12" cy="12" r="10" fill="none" stroke="currentColor" stroke-width="2" />
            <line x1="12" y1="7" x2="12" y2="13" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
            <circle cx="12" cy="16.5" r="1.5" fill="currentColor" />
          </svg>
          <NText depth="3" style="font-size: 12px">
            {{ t('browser-download.auto-category-hint') || '已开启自动分类，文件将保存到对应的目录' }}
          </NText>
        </div>

        <NFormItem :label="t('browser-download.filename') || '重命名:'">
          <NInput
            v-model:value="form.filename"
            :placeholder="t('browser-download.filename-placeholder') || '选填'"
          />
        </NFormItem>
      </NForm>

      <template #footer>
        <NSpace justify="end">
          <NButton @click="handleClose">
            {{ t('browser-download.cancel') || '取消' }}
          </NButton>
          <NButton type="primary" @click="handleConfirm">
            {{ t('browser-download.start-download') || '开始下载' }}
          </NButton>
        </NSpace>
      </template>
    </NCard>
  </NModal>
</template>

<style scoped>
.browser-download-card {
  border-radius: 12px;
}

.path-input-group {
  flex: 1;
  display: flex;
  align-items: center;
}

.auto-category-hint {
  display: flex;
  align-items: center;
}
</style>
