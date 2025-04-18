<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import { Checkbox } from '@/components/ui/checkbox'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue
} from '@/components/ui/select'
import { ScrollArea } from '@/components/ui/scroll-area'
import { MCPServerConfig } from '@shared/presenter'
import { EmojiPicker } from '@/components/ui/emoji-picker'
import { useToast } from '@/components/ui/toast'
import { Icon } from '@iconify/vue'
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover'
import { ChevronDown } from 'lucide-vue-next'
import ModelSelect from '@/components/ModelSelect.vue'
import ModelIcon from '@/components/icons/ModelIcon.vue'
import { useSettingsStore } from '@/stores/settings'
import type { RENDERER_MODEL_META } from '@shared/presenter'

const { t } = useI18n()
const { toast } = useToast()
const settingsStore = useSettingsStore()

const props = defineProps<{
  serverName?: string
  initialConfig?: MCPServerConfig
  editMode?: boolean
  defaultJsonConfig?: string
}>()

const emit = defineEmits<{
  submit: [serverName: string, config: MCPServerConfig]
}>()

// 表单状态
const name = ref(props.serverName || '')
const command = ref(props.initialConfig?.command || 'npx')
const args = ref(props.initialConfig?.args?.join(' ') || '')
const env = ref(JSON.stringify(props.initialConfig?.env || {}, null, 2))
const descriptions = ref(props.initialConfig?.descriptions || '')
const icons = ref(props.initialConfig?.icons || '📁')
const type = ref<'sse' | 'stdio' | 'inmemory' | 'http'>(props.initialConfig?.type || 'stdio')
const baseUrl = ref(props.initialConfig?.baseUrl || '')

// 模型选择相关
const modelSelectOpen = ref(false)
const selectedImageModel = ref<RENDERER_MODEL_META | null>(null)
const selectedImageModelProvider = ref('')

// 判断是否是inmemory类型
const isInMemoryType = computed(() => type.value === 'inmemory')
// 判断是否是imageServer
const isImageServer = computed(() => isInMemoryType.value && name.value === 'imageServer')
// 判断字段是否只读(inmemory类型除了args和env外都是只读的)
const isFieldReadOnly = computed(() => props.editMode && isInMemoryType.value)

// 处理模型选择
const handleImageModelSelect = (model: RENDERER_MODEL_META, providerId: string) => {
  selectedImageModel.value = model
  selectedImageModelProvider.value = providerId
  // 将provider和modelId以空格分隔拼接成args的值
  args.value = `${providerId} ${model.id}`
  modelSelectOpen.value = false
}

// 获取内置服务器的本地化名称和描述
const getLocalizedName = computed(() => {
  const name = props.serverName
  if (isInMemoryType.value && name) {
    return t(`mcp.inmemory.${name}.name`, name)
  }
  return name
})

const getLocalizedDesc = computed(() => {
  if (isInMemoryType.value && name.value) {
    return t(`mcp.inmemory.${name.value}.desc`, descriptions.value)
  }
  return descriptions.value
})

// 权限设置
const autoApproveAll = ref(props.initialConfig?.autoApprove?.includes('all') || false)
const autoApproveRead = ref(
  props.initialConfig?.autoApprove?.includes('read') ||
    props.initialConfig?.autoApprove?.includes('all') ||
    false
)
const autoApproveWrite = ref(
  props.initialConfig?.autoApprove?.includes('write') ||
    props.initialConfig?.autoApprove?.includes('all') ||
    false
)

// 简单表单状态
const currentStep = ref(props.editMode ? 'detailed' : 'simple')
const jsonConfig = ref('')

// 当type变更时处理baseUrl的显示逻辑
const showBaseUrl = computed(() => type.value === 'sse' || type.value === 'http')
// 添加计算属性来控制命令相关字段的显示
const showCommandFields = computed(() => type.value === 'stdio')

// 当选择 all 时，自动选中其他权限
const handleAutoApproveAllChange = (checked: boolean) => {
  if (checked) {
    autoApproveRead.value = true
    autoApproveWrite.value = true
  }
}

// JSON配置解析
const parseJsonConfig = () => {
  try {
    const parsedConfig = JSON.parse(jsonConfig.value)
    if (!parsedConfig.mcpServers || typeof parsedConfig.mcpServers !== 'object') {
      throw new Error('Invalid MCP server configuration format')
    }

    // 获取第一个服务器的配置
    const serverEntries = Object.entries(parsedConfig.mcpServers)
    if (serverEntries.length === 0) {
      throw new Error('No MCP servers found in configuration')
    }

    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const [serverName, serverConfig] = serverEntries[0] as [string, any]

    // 填充表单数据
    name.value = serverName
    command.value = serverConfig.command || 'npx'
    args.value = serverConfig.args?.join(' ') || ''
    env.value = JSON.stringify(serverConfig.env || {}, null, 2)
    descriptions.value = serverConfig.descriptions || ''
    icons.value = serverConfig.icons || '📁'
    type.value = serverConfig.type || ''
    baseUrl.value = serverConfig.url || ''
    console.log('type', type.value, baseUrl.value)
    if (type.value !== 'stdio' && type.value !== 'sse' && type.value !== 'http') {
      if (baseUrl.value) {
        type.value = 'http'
      } else {
        type.value = 'stdio'
      }
    }

    // 权限设置
    autoApproveAll.value = serverConfig.autoApprove?.includes('all') || false
    autoApproveRead.value =
      serverConfig.autoApprove?.includes('read') ||
      serverConfig.autoApprove?.includes('all') ||
      false
    autoApproveWrite.value =
      serverConfig.autoApprove?.includes('write') ||
      serverConfig.autoApprove?.includes('all') ||
      false

    // 切换到详细表单
    currentStep.value = 'detailed'

    toast({
      title: t('settings.mcp.serverForm.parseSuccess'),
      description: t('settings.mcp.serverForm.configImported')
    })
  } catch (error) {
    console.error('解析JSON配置失败:', error)
    toast({
      title: t('settings.mcp.serverForm.parseError'),
      description: error instanceof Error ? error.message : String(error),
      variant: 'destructive'
    })
  }
}

// 切换到详细表单
const goToDetailedForm = () => {
  currentStep.value = 'detailed'
}

// 验证
const isNameValid = computed(() => name.value.trim().length > 0)
const isCommandValid = computed(() => {
  // 对于SSE类型，命令不是必需的
  if (type.value === 'sse' || type.value === 'http') return true
  // 对于STDIO类型，命令是必需的
  return command.value.trim().length > 0
})
const isEnvValid = computed(() => {
  try {
    JSON.parse(env.value)
    return true
  } catch (error) {
    return false
  }
})
const isBaseUrlValid = computed(() => {
  if (type.value !== 'sse' && type.value !== 'http') return true
  return baseUrl.value.trim().length > 0
})

const isFormValid = computed(() => {
  // 基本验证：名称必须有效
  if (!isNameValid.value) return false

  // 对于SSE类型，只需要名称和baseUrl有效
  if (type.value === 'sse' || type.value === 'http') {
    return isNameValid.value && isBaseUrlValid.value
  }

  // 对于STDIO类型，需要名称和命令有效，以及环境变量格式正确
  return isNameValid.value && isCommandValid.value && isEnvValid.value
})

// 提交表单
const handleSubmit = () => {
  if (!isFormValid.value) return

  // 处理自动授权设置
  const autoApprove: string[] = []
  if (autoApproveAll.value) {
    autoApprove.push('all')
  } else {
    if (autoApproveRead.value) autoApprove.push('read')
    if (autoApproveWrite.value) autoApprove.push('write')
  }

  // 创建基本配置（必需的字段）
  const baseConfig = {
    descriptions: descriptions.value.trim(),
    icons: icons.value.trim(),
    autoApprove,
    type: type.value
  }

  // 创建符合MCPServerConfig接口的配置对象
  let serverConfig: MCPServerConfig

  if (type.value === 'sse') {
    // SSE类型的服务器
    serverConfig = {
      ...baseConfig,
      command: '', // 提供空字符串作为默认值
      args: [], // 提供空数组作为默认值
      env: {}, // 提供空对象作为默认值
      baseUrl: baseUrl.value.trim()
    }
  } else if (type.value === 'http') {
    // HTTP类型的服务器
    serverConfig = {
      ...baseConfig,
      command: '', // 提供空字符串作为默认值
      args: [], // 提供空数组作为默认值
      env: {}, // 提供空对象作为默认值
      baseUrl: baseUrl.value.trim()
    }
  } else {
    // STDIO类型的服务器或者inmemory类型
    try {
      serverConfig = {
        ...baseConfig,
        command: command.value.trim(),
        args: args.value.split(/\s+/).filter(Boolean),
        env: JSON.parse(env.value)
      }
    } catch (error) {
      // 如果JSON解析失败，使用空对象
      serverConfig = {
        ...baseConfig,
        command: command.value.trim(),
        args: args.value.split(/\s+/).filter(Boolean),
        env: {}
      }
      toast({
        title: t('settings.mcp.serverForm.jsonParseError'),
        description: String(error),
        variant: 'destructive'
      })
    }
  }

  emit('submit', name.value.trim(), serverConfig)
}

const placeholder = `mcp配置示例
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        ...
      ]
    },
    "sseServer":{
      "url": "https://your-sse-server-url"
    }
  },

}`

// 监听 defaultJsonConfig 变化
watch(
  () => props.defaultJsonConfig,
  (newConfig) => {
    if (newConfig) {
      jsonConfig.value = newConfig
      parseJsonConfig()
    }
  },
  { immediate: true }
)

// 初始化时解析args中的provider和modelId（针对imageServer）
watch(
  [() => name.value, () => args.value, () => type.value],
  ([newName, newArgs, newType]) => {
    if (newType === 'inmemory' && newName === 'imageServer' && newArgs) {
      // 从args中解析出provider和modelId
      const argsParts = newArgs.split(/\s+/)
      if (argsParts.length >= 2) {
        const providerId = argsParts[0]
        const modelId = argsParts[1]
        // 查找对应的模型
        const foundModel = settingsStore.findModelByIdOrName(modelId)
        if (foundModel && foundModel.providerId === providerId) {
          selectedImageModel.value = foundModel.model
          selectedImageModelProvider.value = providerId
        } else {
          console.warn(`未找到匹配的模型: ${providerId} ${modelId}`)
        }
      }
    }
  },
  { immediate: true }
)

// 打开MCP Marketplace
const openMcpMarketplace = () => {
  window.open('https://mcp.deepchatai.cn', '_blank')
}
</script>

<template>
  <!-- 简单表单 -->
  <form v-if="currentStep === 'simple'" class="space-y-4 h-full flex flex-col">
    <ScrollArea class="h-0 flex-grow">
      <div class="space-y-4 px-4 pb-4">
        <div class="text-sm">
          {{ t('settings.mcp.serverForm.jsonConfigIntro') }}
        </div>

        <!-- MCP Marketplace 入口 -->
        <div class="my-4">
          <Button
            variant="outline"
            class="w-full flex items-center justify-center gap-2"
            @click="openMcpMarketplace"
          >
            <Icon icon="lucide:shopping-bag" class="w-4 h-4" />
            <span>{{ t('settings.mcp.serverForm.browseMarketplace') }}</span>
            <Icon icon="lucide:external-link" class="w-3.5 h-3.5 text-muted-foreground" />
          </Button>
        </div>

        <div class="space-y-2">
          <Label class="text-xs text-muted-foreground" for="json-config">
            {{ t('settings.mcp.serverForm.jsonConfig') }}
          </Label>
          <Textarea id="json-config" v-model="jsonConfig" rows="10" :placeholder="placeholder" />
        </div>
      </div>
    </ScrollArea>

    <div class="flex justify-between pt-2 border-t px-4">
      <Button type="button" variant="outline" size="sm" @click="goToDetailedForm">
        {{ t('settings.mcp.serverForm.skipToManual') }}
      </Button>
      <Button type="button" size="sm" @click="parseJsonConfig">
        {{ t('settings.mcp.serverForm.parseAndContinue') }}
      </Button>
    </div>
  </form>

  <!-- 详细表单 -->
  <form v-else class="space-y-2 h-full flex flex-col" @submit.prevent="handleSubmit">
    <ScrollArea class="h-0 flex-grow">
      <div class="space-y-2 px-4 pb-4">
        <!-- 服务器名称 -->
        <!-- 本地化名称 (针对inmemory类型) -->
        <div class="space-y-2" v-if="isInMemoryType && name">
          <Label class="text-xs text-muted-foreground" for="localized-name">{{
            t('settings.mcp.serverForm.name')
          }}</Label>

          <div
            class="flex h-9 items-center justify-between whitespace-nowrap rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm ring-offset-background opacity-50"
          >
            {{ getLocalizedName }}
          </div>
        </div>
        <div class="space-y-2" v-else>
          <Label class="text-xs text-muted-foreground" for="server-name">{{
            t('settings.mcp.serverForm.name')
          }}</Label>
          <Input
            id="server-name"
            v-model="name"
            :placeholder="t('settings.mcp.serverForm.namePlaceholder')"
            :disabled="editMode || isFieldReadOnly"
            required
          />
        </div>

        <!-- 图标 -->
        <div class="space-y-2">
          <Label class="text-xs text-muted-foreground" for="server-icon">{{
            t('settings.mcp.serverForm.icons')
          }}</Label>
          <div class="flex items-center space-x-2">
            <EmojiPicker v-model="icons" :disabled="isFieldReadOnly" />
          </div>
        </div>

        <!-- 服务器类型 -->
        <div class="space-y-2">
          <Label class="text-xs text-muted-foreground" for="server-type">{{
            t('settings.mcp.serverForm.type')
          }}</Label>
          <Select v-model="type" :disabled="isFieldReadOnly">
            <SelectTrigger class="w-full">
              <SelectValue :placeholder="t('settings.mcp.serverForm.typePlaceholder')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="stdio">{{ t('settings.mcp.serverForm.typeStdio') }}</SelectItem>
              <SelectItem value="sse">{{ t('settings.mcp.serverForm.typeSse') }}</SelectItem>
              <SelectItem value="http">{{ t('settings.mcp.serverForm.typeHttp') }}</SelectItem>
              <SelectItem
                value="inmemory"
                v-if="props.editMode && props.initialConfig?.type === 'inmemory'"
                >{{ t('settings.mcp.serverForm.typeInMemory') }}</SelectItem
              >
            </SelectContent>
          </Select>
        </div>

        <!-- 基础URL，仅在类型为SSE时显示 -->
        <div class="space-y-2" v-if="showBaseUrl">
          <Label class="text-xs text-muted-foreground" for="server-base-url">{{
            t('settings.mcp.serverForm.baseUrl')
          }}</Label>
          <Input
            id="server-base-url"
            v-model="baseUrl"
            :placeholder="t('settings.mcp.serverForm.baseUrlPlaceholder')"
            :disabled="isFieldReadOnly"
            required
          />
        </div>

        <!-- 命令 -->
        <div class="space-y-2" v-if="showCommandFields">
          <Label class="text-xs text-muted-foreground" for="server-command">{{
            t('settings.mcp.serverForm.command')
          }}</Label>
          <Input
            id="server-command"
            v-model="command"
            :placeholder="t('settings.mcp.serverForm.commandPlaceholder')"
            :disabled="isFieldReadOnly"
            required
          />
        </div>

        <!-- 参数 -->
        <div v-if="isImageServer" class="space-y-2">
          <Label class="text-xs text-muted-foreground" for="server-model">
            {{ t('settings.mcp.serverForm.imageModel') || '模型选择' }}
          </Label>
          <Popover v-model:open="modelSelectOpen">
            <PopoverTrigger as-child>
              <Button variant="outline" class="w-full justify-between">
                <div class="flex items-center gap-2">
                  <ModelIcon :model-id="selectedImageModel?.id || ''" class="h-4 w-4" />
                  <span class="truncate">{{
                    selectedImageModel?.name || t('settings.common.selectModel')
                  }}</span>
                </div>
                <ChevronDown class="h-4 w-4 opacity-50" />
              </Button>
            </PopoverTrigger>
            <PopoverContent class="w-80 p-0">
              <ModelSelect @update:model="handleImageModelSelect" />
            </PopoverContent>
          </Popover>
        </div>
        <div class="space-y-2" v-else-if="showCommandFields || isInMemoryType">
          <Label class="text-xs text-muted-foreground" for="server-args">{{
            t('settings.mcp.serverForm.args')
          }}</Label>
          <Input
            id="server-args"
            v-model="args"
            :placeholder="t('settings.mcp.serverForm.argsPlaceholder')"
          />
        </div>

        <!-- 环境变量 -->
        <div class="space-y-2" v-if="showCommandFields || isInMemoryType">
          <Label class="text-xs text-muted-foreground" for="server-env">{{
            t('settings.mcp.serverForm.env')
          }}</Label>
          <Textarea
            id="server-env"
            v-model="env"
            rows="5"
            :placeholder="t('settings.mcp.serverForm.envPlaceholder')"
            :class="{ 'border-red-500': !isEnvValid }"
          />
        </div>

        <!-- 描述 -->
        <!-- 本地化描述 (针对inmemory类型) -->
        <div class="space-y-2" v-if="isInMemoryType && name">
          <Label class="text-xs text-muted-foreground" for="localized-desc">{{
            t('settings.mcp.serverForm.descriptions')
          }}</Label>
          <div
            class="flex h-9 items-center justify-between whitespace-nowrap rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm ring-offset-background opacity-50"
          >
            {{ getLocalizedDesc }}
          </div>
        </div>
        <div class="space-y-2" v-else>
          <Label class="text-xs text-muted-foreground" for="server-description">{{
            t('settings.mcp.serverForm.descriptions')
          }}</Label>
          <Input
            id="server-description"
            v-model="descriptions"
            :placeholder="t('settings.mcp.serverForm.descriptionsPlaceholder')"
            :disabled="isFieldReadOnly"
          />
        </div>

        <!-- 自动授权选项 -->
        <div class="space-y-3">
          <Label class="text-xs text-muted-foreground">{{
            t('settings.mcp.serverForm.autoApprove')
          }}</Label>
          <div class="flex flex-col space-y-2">
            <div class="flex items-center space-x-2">
              <Checkbox
                id="auto-approve-all"
                v-model:checked="autoApproveAll"
                @update:checked="handleAutoApproveAllChange"
              />
              <label
                for="auto-approve-all"
                class="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
              >
                {{ t('settings.mcp.serverForm.autoApproveAll') }}
              </label>
            </div>

            <div class="flex items-center space-x-2">
              <Checkbox
                id="auto-approve-read"
                v-model:checked="autoApproveRead"
                :disabled="autoApproveAll"
              />
              <label
                for="auto-approve-read"
                class="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
              >
                {{ t('settings.mcp.serverForm.autoApproveRead') }}
              </label>
            </div>

            <div class="flex items-center space-x-2">
              <Checkbox
                id="auto-approve-write"
                v-model:checked="autoApproveWrite"
                :disabled="autoApproveAll"
              />
              <label
                for="auto-approve-write"
                class="text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
              >
                {{ t('settings.mcp.serverForm.autoApproveWrite') }}
              </label>
            </div>
          </div>
        </div>
      </div>
    </ScrollArea>

    <!-- 提交按钮 -->
    <div class="flex justify-end pt-2 border-t px-4">
      <Button type="submit" size="sm" :disabled="!isFormValid">
        {{ t('settings.mcp.serverForm.submit') }}
      </Button>
    </div>
  </form>
</template>
