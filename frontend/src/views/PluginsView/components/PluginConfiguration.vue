<script setup lang="ts">
import { ref, inject } from 'vue'
import { useI18n } from 'vue-i18n'

import { PluginTriggerEvent } from '@/enums/app'
import { usePluginsStore, useAppSettingsStore } from '@/stores'
import { deepClone, message, sampleID } from '@/utils'

import type { PluginConfiguration } from '@/types/app'

interface Props {
  id: string
}

const props = defineProps<Props>()

const key = ref(sampleID())
const loading = ref(false)
const settings = ref<Record<string, any>>({})
const oldSettings = ref<Record<string, any>>({})
const configuration = ref<PluginConfiguration[]>([])

const { t } = useI18n()
const pluginsStore = usePluginsStore()
const appSettingsStore = useAppSettingsStore()

const handleCancel = inject('cancel') as any

const handleSubmit = async () => {
  loading.value = true
  try {
    await pluginsStore.manualTrigger(
      props.id,
      PluginTriggerEvent.OnConfigure,
      settings.value,
      oldSettings.value,
    )
  } catch (error: any) {
    const errors = [
      props.id + ' Not Found',
      'is Missing source code',
      'Disabled',
      "Can't find variable: " + PluginTriggerEvent.OnConfigure,
      PluginTriggerEvent.OnConfigure + ' is not defined',
    ]
    if (errors.every((v) => !error.includes(v))) {
      message.error(error)
      return
    }
  } finally {
    loading.value = false
  }
  appSettingsStore.app.pluginSettings[props.id] = settings.value
  handleCancel()
  message.success('common.success')
}

const getOptions = (val: string[]) => {
  return val.map((v) => {
    const arr = v.split(',')
    return { label: arr[0], value: arr[1] || arr[0] }
  })
}

const handleRestoreConfiguration = (showMessage = false) => {
  settings.value = {}
  configuration.value.forEach(({ key, value }) => (settings.value[key] = deepClone(value)))
  key.value = sampleID()
  showMessage && message.success('common.success')
}

const p = pluginsStore.getPluginById(props.id)
if (p) {
  configuration.value = p.configuration
  const _settings = appSettingsStore.app.pluginSettings[p.id]
  if (_settings) {
    settings.value = deepClone(_settings)
  } else {
    // Fill with default value
    handleRestoreConfiguration()
  }
  oldSettings.value = deepClone(settings.value)
}
</script>

<template>
  <div class="form">
    <Card
      v-for="(conf, index) in configuration"
      :key="conf.id"
      :title="index + 1 + '、' + conf.title"
      class="mb-8"
    >
      <div class="mb-8" style="font-size: 12px">{{ conf.description }}</div>
      <Component
        v-model="settings[conf.key]"
        :key="key"
        :is="conf.component"
        :options="getOptions(conf.options)"
        :autofocus="false"
        editable
      />
    </Card>
  </div>
  <div class="form-action">
    <Button @click="handleRestoreConfiguration(true)" type="link" class="mr-auto">
      {{ t('plugin.restore') }}
    </Button>
    <Button @click="handleCancel" :disabled="loading">{{ t('common.cancel') }}</Button>
    <Button @click="handleSubmit" :loading="loading" type="primary">
      {{ t('common.save') }}
    </Button>
  </div>
</template>

<style lang="less" scoped>
.form {
  padding: 0 8px;
  overflow-y: auto;
  max-height: 70vh;
  .name {
    font-size: 14px;
    padding: 8px 0;
    white-space: nowrap;
  }
}
</style>
