<template>
  <NodeContainer :nodeModel="nodeModel">
    <h5 class="title-decoration-1 mb-8">节点设置</h5>
    <el-card shadow="never" class="card-never" style="--el-card-padding: 12px">
      <el-form
        @submit.prevent
        :model="form_data"
        label-position="top"
        require-asterisk-position="right"
        class="mb-24"
        label-width="auto"
        ref="questionNodeFormRef"
        hide-required-asterisk
      >
        <el-form-item
          label="AI 模型"
          prop="model_id"
          :rules="{
            required: true,
            message: '请选择 AI 模型',
            trigger: 'change'
          }"
        >
          <template #label>
            <div class="flex-between">
              <div>
                <span>AI 模型<span class="danger">*</span></span>
              </div>
              <el-button
                type="primary"
                link
                :disabled="!form_data.model_id"
                @click="openAIParamSettingDialog(form_data.model_id)"
                @refreshForm="refreshParam"
              >
                {{ $t('views.application.applicationForm.form.paramSetting') }}
              </el-button>
            </div>
          </template>
          <el-select
            @change="model_change"
            @wheel="wheel"
            :teleported="false"
            v-model="form_data.model_id"
            placeholder="请选择 AI 模型"
            class="w-full"
            popper-class="select-model"
            :clearable="true"
          >
            <el-option-group
              v-for="(value, label) in modelOptions"
              :key="value"
              :label="relatedObject(providerOptions, label, 'provider')?.name"
            >
              <el-option
                v-for="item in value.filter((v: any) => v.status === 'SUCCESS')"
                :key="item.id"
                :label="item.name"
                :value="item.id"
                class="flex-between"
              >
                <div class="flex align-center">
                  <span
                    v-html="relatedObject(providerOptions, label, 'provider')?.icon"
                    class="model-icon mr-8"
                  ></span>
                  <span>{{ item.name }}</span>
                  <el-tag v-if="item.permission_type === 'PUBLIC'" type="info" class="info-tag ml-8"
                    >公用
                  </el-tag>
                </div>
                <el-icon class="check-icon" v-if="item.id === form_data.model_id"
                  ><Check
                /></el-icon>
              </el-option>
              <!-- 不可用 -->
              <el-option
                v-for="item in value.filter((v: any) => v.status !== 'SUCCESS')"
                :key="item.id"
                :label="item.name"
                :value="item.id"
                class="flex-between"
                disabled
              >
                <div class="flex">
                  <span
                    v-html="relatedObject(providerOptions, label, 'provider')?.icon"
                    class="model-icon mr-8"
                  ></span>
                  <span>{{ item.name }}</span>
                  <span class="danger">（不可用）</span>
                </div>
                <el-icon class="check-icon" v-if="item.id === form_data.model_id"
                  ><Check
                /></el-icon>
              </el-option>
            </el-option-group>
            <template #footer>
              <div class="w-full text-left cursor" @click="openCreateModel()">
                <el-button type="primary" link>
                  <el-icon class="mr-4"><Plus /></el-icon>
                  添加模型
                </el-button>
              </div>
            </template>
          </el-select>
        </el-form-item>
        <el-form-item label="角色设定">
          <MdEditorMagnify
            title="角色设定"
            v-model="form_data.system"
            style="height: 100px"
            @submitDialog="submitSystemDialog"
            placeholder="角色设定"
          />
        </el-form-item>
        <el-form-item
          label="提示词"
          prop="prompt"
          :rules="{
            required: true,
            message: '请输入提示词',
            trigger: 'blur'
          }"
        >
          <template #label>
            <div class="flex align-center">
              <div class="mr-4">
                <span>提示词<span class="danger">*</span></span>
              </div>
              <el-tooltip effect="dark" placement="right" popper-class="max-w-200">
                <template #content
                  >通过调整提示词内容，可以引导大模型聊天方向，该提示词会被固定在上下文的开头，可以使用变量。</template
                >
                <AppIcon iconName="app-warning" class="app-warning-icon"></AppIcon>
                <el-icon><EditPen /></el-icon>
              </el-tooltip>
            </div>
          </template>
          <MdEditorMagnify
            @wheel="wheel"
            title="提示词"
            v-model="form_data.prompt"
            style="height: 150px"
            @submitDialog="submitDialog"
          />
        </el-form-item>
        <el-form-item label="历史聊天记录">
          <el-input-number
            v-model="form_data.dialogue_number"
            :min="0"
            :value-on-clear="0"
            controls-position="right"
            class="w-full"
            :step="1"
            :step-strictly="true"
          />
        </el-form-item>
        <el-form-item label="返回内容" @click.prevent>
          <template #label>
            <div class="flex align-center">
              <div class="mr-4">
                <span>返回内容<span class="danger">*</span></span>
              </div>
              <el-tooltip effect="dark" placement="right" popper-class="max-w-200">
                <template #content>
                  关闭后该节点的内容则不输出给用户。
                  如果你想让用户看到该节点的输出内容，请打开开关。
                </template>
                <AppIcon iconName="app-warning" class="app-warning-icon"></AppIcon>
              </el-tooltip>
            </div>
          </template>
          <el-switch size="small" v-model="form_data.is_result" />
        </el-form-item>
      </el-form>
    </el-card>

    <!-- 添加模版 -->
    <CreateModelDialog
      ref="createModelRef"
      @submit="getModel"
      @change="openCreateModel($event)"
    ></CreateModelDialog>
    <SelectProviderDialog ref="selectProviderRef" @change="openCreateModel($event)" />
    <AIModeParamSettingDialog ref="AIModeParamSettingDialogRef" @refresh="refreshParam" />
  </NodeContainer>
</template>
<script setup lang="ts">
import { set, groupBy } from 'lodash'
import { app } from '@/main'
import NodeContainer from '@/workflow/common/NodeContainer.vue'
import CreateModelDialog from '@/views/template/component/CreateModelDialog.vue'
import SelectProviderDialog from '@/views/template/component/SelectProviderDialog.vue'
import AIModeParamSettingDialog from '@/views/application/component/AIModeParamSettingDialog.vue'
import type { FormInstance } from 'element-plus'
import { ref, computed, onMounted } from 'vue'
import applicationApi from '@/api/application'
import useStore from '@/stores'
import { relatedObject } from '@/utils/utils'
import type { Provider } from '@/api/type/model'
import { isLastNode } from '@/workflow/common/data'
const AIModeParamSettingDialogRef = ref<InstanceType<typeof AIModeParamSettingDialog>>()

const { model } = useStore()

const wheel = (e: any) => {
  if (e.ctrlKey === true) {
    e.preventDefault()
    return true
  } else {
    e.stopPropagation()
    return true
  }
}

const model_change = (model_id?: string) => {
  if (model_id) {
    AIModeParamSettingDialogRef.value?.reset_default(model_id, id)
  } else {
    refreshParam({})
  }
}
function submitDialog(val: string) {
  set(props.nodeModel.properties.node_data, 'prompt', val)
}

function submitSystemDialog(val: string) {
  set(props.nodeModel.properties.node_data, 'system', val)
}
const {
  params: { id }
} = app.config.globalProperties.$route as any

// @ts-ignore
const defaultPrompt = `根据上下文优化和完善用户问题：{{开始.question}}
请输出一个优化后的问题。`
const form = {
  model_id: '',
  system: '你是一个问题优化大师',
  prompt: defaultPrompt,
  dialogue_number: 1,
  is_result: false
}
function refreshParam(data: any) {
  set(props.nodeModel.properties.node_data, 'model_params_setting', data)
}

const openAIParamSettingDialog = (modelId: string) => {
  if (modelId) {
    AIModeParamSettingDialogRef.value?.open(modelId, id, form_data.value.model_params_setting)
  }
}
const form_data = computed({
  get: () => {
    if (props.nodeModel.properties.node_data) {
      return props.nodeModel.properties.node_data
    } else {
      set(props.nodeModel.properties, 'node_data', form)
    }
    return props.nodeModel.properties.node_data
  },
  set: (value) => {
    set(props.nodeModel.properties, 'node_data', value)
  }
})
const props = defineProps<{ nodeModel: any }>()

const questionNodeFormRef = ref<FormInstance>()
const createModelRef = ref<InstanceType<typeof CreateModelDialog>>()
const selectProviderRef = ref<InstanceType<typeof SelectProviderDialog>>()

const modelOptions = ref<any>(null)
const providerOptions = ref<Array<Provider>>([])

const validate = () => {
  return questionNodeFormRef.value?.validate().catch((err) => {
    return Promise.reject({ node: props.nodeModel, errMessage: err })
  })
}

function getModel() {
  if (id) {
    applicationApi.getApplicationModel(id).then((res: any) => {
      modelOptions.value = groupBy(res?.data, 'provider')
    })
  } else {
    model.asyncGetModel().then((res: any) => {
      modelOptions.value = groupBy(res?.data, 'provider')
    })
  }
}

function getProvider() {
  model.asyncGetProvider().then((res: any) => {
    providerOptions.value = res?.data
  })
}

const openCreateModel = (provider?: Provider) => {
  if (provider && provider.provider) {
    createModelRef.value?.open(provider)
  } else {
    selectProviderRef.value?.open()
  }
}

onMounted(() => {
  getProvider()
  getModel()
  if (typeof props.nodeModel.properties.node_data?.is_result === 'undefined') {
    if (isLastNode(props.nodeModel)) {
      set(props.nodeModel.properties.node_data, 'is_result', true)
    }
  }
  set(props.nodeModel, 'validate', validate)
})
</script>
<style lang="scss" scoped></style>
