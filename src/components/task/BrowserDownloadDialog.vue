<script setup lang="ts">
/** @fileoverview 浏览器下载信息弹窗组件 - 完整功能版 */
import { ref, watch, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import {
  NModal,
  NCard,
  NForm,
  NFormItem,
  NInput,
  NButton,
  NSpace,
  NIcon,
  NText,
  NSelect,
  NCollapse,
  NCollapseItem,
  NCheckbox,
} from 'naive-ui'
import { open as openDialog } from '@tauri-apps/plugin-dialog'
import { downloadDir } from '@tauri-apps/api/path'
import { FolderOpenOutline } from '@vicons/ionicons5'
import { usePreferenceStore } from '@/stores/preference'
import { isGlobalProxyConfigured, isGlobalDownloadProxyActive } from '@/composables/useAddTaskSubmit'
import type { Aria2EngineOptions } from '@shared/types'

export interface BrowserDownloadData {
  url: string
  referer?: string
  cookie?: string
  filename?: string
  fileSize?: string
}

export interface BrowserDownloadConfirmData {
  url: string
  dir: string
  filename: string
  options: Aria2EngineOptions
}

const props = defineProps<{
  show: boolean
  data: BrowserDownloadData | null
}>()

const emit = defineEmits<{
  close: []
  confirm: [data: BrowserDownloadConfirmData]
}>()

const { t } = useI18n()
const preferenceStore = usePreferenceStore()

// 表单数据
const form = ref({
  url: '',
  dir: '',
  filename: '',
  userAgent: '',
  authorization: '',
  referer: '',
  cookie: '',
  proxyMode: 'none' as 'none' | 'global' | 'custom',
  customProxy: '',
})

// 自动分类状态
const autoCategoryEnabled = computed(() => preferenceStore.config.fileCategoryEnabled)
const useAutoCategory = ref(true)

// 高级选项展开状态
const showAdvanced = ref(false)

// 全局代理可用性
const globalProxyAvailable = computed(() => isGlobalProxyConfigured(preferenceStore.config.proxy))
const globalProxyServer = computed(() => preferenceStore.config.proxy?.server ?? '')

// 代理选项
const proxyOptions = [
  { label: t('browser-download.proxy-none') || '不使用代理', value: 'none' },
  { label: t('browser-download.proxy-global') || '使用全局代理', value: 'global' },
  { label: t('browser-download.proxy-custom') || '自定义代理', value: 'custom' },
]

// 监听弹窗显示状态
watch(
  () => props.show,
  async (visible) => {
    if (visible && props.data) {
      // 填充数据
      form.value.url = props.data.url || ''
      form.value.filename = props.data.filename || ''
      form.value.referer = props.data.referer || ''
      form.value.cookie = props.data.cookie || ''

      // 设置默认下载目录
      try {
        if (!form.value.dir) {
          form.value.dir = await downloadDir()
        }
      } catch {
        form.value.dir = '~/Downloads'
      }

      // 如果全局代理可用且启用了下载代理，默认选中全局代理
      if (isGlobalDownloadProxyActive(preferenceStore.config.proxy)) {
        form.value.proxyMode = 'global'
      }

      // 默认启用自动分类
      useAutoCategory.value = true
    }
  },
)

// 监听数据变化
watch(
  () => props.data,
  (data) => {
    if (data) {
      form.value.url = data.url || ''
      form.value.filename = data.filename || ''
      form.value.referer = data.referer || ''
      form.value.cookie = data.cookie || ''
    }
  },
)

// 选择目录
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

// 构建引擎选项
function buildOptions(): Aria2EngineOptions {
  const options: Aria2EngineOptions = {
    dir: form.value.dir,
    split: String(preferenceStore.config.split ?? 16),
  }

  // 重命名
  if (form.value.filename) {
    options.out = form.value.filename
  }

  // User-Agent
  if (form.value.userAgent) {
    options['user-agent'] = form.value.userAgent
  }

  // Referer
  if (form.value.referer) {
    options.referer = form.value.referer
  }

  // Cookie
  if (form.value.cookie) {
    options.header = [`Cookie: ${form.value.cookie}`]
  }

  // Authorization
  if (form.value.authorization) {
    if (!options.header) {
      options.header = [`Authorization: ${form.value.authorization}`]
    } else {
      options.header = [...options.header, `Authorization: ${form.value.authorization}`]
    }
  }

  // 代理
  if (form.value.proxyMode === 'global') {
    options['all-proxy'] = globalProxyServer.value
  } else if (form.value.proxyMode === 'custom' && form.value.customProxy) {
    options['all-proxy'] = form.value.customProxy
  } else {
    options['all-proxy'] = ''
  }

  return options
}

// 确认下载
function handleConfirm() {
  const options = buildOptions()

  emit('confirm', {
    url: form.value.url,
    dir: form.value.dir,
    filename: form.value.filename,
    options,
  })
  handleClose()
}

// 关闭弹窗
function handleClose() {
  emit('close')
  // 重置表单
  Object.assign(form.value, {
    url: '',
    dir: '',
    filename: '',
    userAgent: '',
    authorization: '',
    referer: '',
    cookie: '',
    proxyMode: 'none',
    customProxy: '',
  })
  showAdvanced.value = false
  useAutoCategory.value = true
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
        <!-- 下载链接 -->
        <NFormItem :label="t('browser-download.url') || '下载链接:'" style="margin-bottom: 16px">
          <NInput
            v-model:value="form.url"
            type="textarea"
            :rows="2"
            readonly
            :placeholder="t('browser-download.url-placeholder') || '下载链接将自动填入...'"
          />
        </NFormItem>

        <!-- 自动分类模式 -->
        <template v-if="autoCategoryEnabled">
          <!-- 自动分类开启时 -->
          <template v-if="useAutoCategory">
            <NFormItem :label="t('browser-download.custom-path') || '自定义路径:'" style="margin-bottom: 8px">
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

            <!-- 自动分类提示 -->
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
          </template>

          <!-- 手动模式 -->
          <template v-else>
            <NFormItem :label="t('browser-download.save-path') || '存储路径:'" style="margin-bottom: 16px">
              <div class="path-input-group">
                <NInput
                  v-model:value="form.dir"
                  :placeholder="t('browser-download.save-path-placeholder') || '选择存储路径'"
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
          </template>

          <!-- 自动分类切换 -->
          <div style="margin-left: 96px; margin-bottom: 16px">
            <NCheckbox v-model:checked="useAutoCategory">
              <NText depth="3" style="font-size: 12px">
                {{ t('browser-download.use-auto-category') || '使用自动分类' }}
              </NText>
            </NCheckbox>
          </div>
        </template>

        <!-- 非自动分类模式 -->
        <template v-else>
          <NFormItem :label="t('browser-download.save-path') || '存储路径:'" style="margin-bottom: 16px">
            <div class="path-input-group">
              <NInput
                v-model:value="form.dir"
                :placeholder="t('browser-download.save-path-placeholder') || '选择存储路径'"
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
        </template>

        <!-- 重命名 -->
        <NFormItem :label="t('browser-download.filename') || '重命名:'" style="margin-bottom: 16px">
          <NInput v-model:value="form.filename" :placeholder="t('browser-download.filename-placeholder') || '选填'" />
        </NFormItem>

        <!-- 高级选项 -->
        <NCollapse
          :expanded-names="showAdvanced ? ['advanced'] : []"
          @update:expanded-names="
            (val: string[]) => {
              showAdvanced = val.includes('advanced')
            }
          "
        >
          <NCollapseItem :title="t('browser-download.advanced-options') || '高级选项'" name="advanced">
            <div class="advanced-panel">
              <!-- User-Agent -->
              <NFormItem
                :label="t('browser-download.user-agent') || 'User-Agent:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput
                  v-model:value="form.userAgent"
                  :placeholder="t('browser-download.user-agent-placeholder') || '留空使用默认值'"
                  size="small"
                />
              </NFormItem>

              <!-- 认证信息 -->
              <NFormItem
                :label="t('browser-download.authorization') || '认证信息:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput
                  v-model:value="form.authorization"
                  :placeholder="t('browser-download.authorization-placeholder') || '用户名:密码'"
                  size="small"
                />
              </NFormItem>

              <!-- 来源页面 -->
              <NFormItem
                :label="t('browser-download.referer') || '来源页面:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput
                  v-model:value="form.referer"
                  :placeholder="t('browser-download.referer-placeholder') || 'Referer URL'"
                  size="small"
                />
              </NFormItem>

              <!-- Cookie -->
              <NFormItem
                :label="t('browser-download.cookie') || 'Cookie:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput
                  v-model:value="form.cookie"
                  type="textarea"
                  :rows="2"
                  :placeholder="t('browser-download.cookie-placeholder') || '留空使用默认值'"
                  size="small"
                />
              </NFormItem>

              <!-- 代理选择 -->
              <NFormItem :label="t('browser-download.proxy') || '代理:'" label-width="72px" style="margin-bottom: 12px">
                <NSelect v-model:value="form.proxyMode" :options="proxyOptions" size="small" style="width: 100%" />
              </NFormItem>

              <!-- 全局代理地址 -->
              <NFormItem
                v-if="form.proxyMode === 'global' && globalProxyAvailable"
                :label="t('browser-download.proxy-server') || '代理地址:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput :value="globalProxyServer" readonly size="small" />
              </NFormItem>

              <!-- 自定义代理 -->
              <NFormItem
                v-if="form.proxyMode === 'custom'"
                :label="t('browser-download.custom-proxy') || '自定义代理:'"
                label-width="72px"
                style="margin-bottom: 12px"
              >
                <NInput
                  v-model:value="form.customProxy"
                  :placeholder="t('browser-download.custom-proxy-placeholder') || 'http://127.0.0.1:7890'"
                  size="small"
                />
              </NFormItem>
            </div>
          </NCollapseItem>
        </NCollapse>
      </NForm>

      <!-- 底部按钮 -->
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

.advanced-panel {
  padding: 16px;
  background-color: rgba(0, 0, 0, 0.02);
  border-radius: 8px;
  border: 1px solid var(--n-border-color);
}
</style>
