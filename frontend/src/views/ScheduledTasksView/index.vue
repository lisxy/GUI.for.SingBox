<script setup lang="ts">
import { useI18n, I18nT } from 'vue-i18n'

import { DraggableOptions } from '@/constant/app'
import { View } from '@/enums/app'
import { useAppSettingsStore, useScheduledTasksStore } from '@/stores'
import { debounce, formatRelativeTime, formatDate, message } from '@/utils'

import { useModal } from '@/components/Modal'

import ScheduledTaskForm from './components/ScheduledTaskForm.vue'
import ScheduledTasksLogs from './components/ScheduledTasksLogs.vue'

import type { Menu, ScheduledTask } from '@/types/app'

const menuList: Menu[] = [
  {
    label: 'scheduledtasks.run',
    handler: (id: string) => {
      scheduledTasksStore.runScheduledTask(id)
    },
  },
  {
    label: 'scheduledtasks.log',
    handler: (id: string) => {
      handleShowTaskLogs(id)
    },
  },
]

const { t } = useI18n()
const [Modal, modalApi] = useModal({})
const scheduledTasksStore = useScheduledTasksStore()
const appSettingsStore = useAppSettingsStore()

const handleShowTaskLogs = (id?: string) => {
  modalApi.setProps({
    title: 'scheduledtasks.logs',
    cancelText: 'common.close',
    maskClosable: true,
    submit: false,
    width: '90',
    height: '90',
  })
  modalApi.setContent(ScheduledTasksLogs, { id }).open()
}

const handleShowTaskForm = (id?: string) => {
  modalApi.setProps({
    title: id ? 'common.edit' : 'common.add',
    maxHeight: '90',
    minWidth: '70',
    maxWidth: '90',
  })
  modalApi.setContent(ScheduledTaskForm, { id }).open()
}

const handleDeleteTask = async (s: ScheduledTask) => {
  try {
    await scheduledTasksStore.deleteScheduledTask(s.id)
  } catch (error: any) {
    console.error('deleteSubscribe: ', error)
    message.error(error)
  }
}

const handleDisableTask = async (s: ScheduledTask) => {
  s.disabled = !s.disabled
  scheduledTasksStore.editScheduledTask(s.id, s)
}

const onSortUpdate = debounce(scheduledTasksStore.saveScheduledTasks, 1000)
</script>

<template>
  <div v-if="scheduledTasksStore.scheduledtasks.length === 0" class="grid-list-empty">
    <Empty>
      <template #description>
        <I18nT
          keypath="scheduledtasks.empty"
          tag="div"
          scope="global"
          class="flex items-center mt-12"
        >
          <template #action>
            <Button @click="handleShowTaskForm()" type="link">{{ t('common.add') }}</Button>
          </template>
        </I18nT>
      </template>
    </Empty>
  </div>

  <div v-else class="grid-list-header">
    <Radio
      v-model="appSettingsStore.app.scheduledtasksView"
      :options="[
        { label: 'common.grid', value: View.Grid },
        { label: 'common.list', value: View.List },
      ]"
    />
    <Button @click="handleShowTaskLogs()" type="text" class="ml-auto">
      {{ t('scheduledtasks.logs') }}
    </Button>
    <Button @click="handleShowTaskForm()" type="primary" icon="add" class="ml-16">
      {{ t('common.add') }}
    </Button>
  </div>

  <div
    v-draggable="[
      scheduledTasksStore.scheduledtasks,
      { ...DraggableOptions, onUpdate: onSortUpdate },
    ]"
    :class="'grid-list-' + appSettingsStore.app.scheduledtasksView"
  >
    <Card
      v-for="s in scheduledTasksStore.scheduledtasks"
      :key="s.id"
      :title="s.name"
      :disabled="s.disabled"
      v-menu="menuList.map((v) => ({ ...v, handler: () => v.handler?.(s.id) }))"
      class="grid-list-item"
    >
      <template v-if="appSettingsStore.app.scheduledtasksView === View.Grid" #extra>
        <Dropdown>
          <Button type="link" size="small" icon="more" />
          <template #overlay>
            <div class="flex flex-col gap-4 min-w-64 p-4">
              <Button type="text" size="small" @click="handleDisableTask(s)">
                {{ s.disabled ? t('common.enable') : t('common.disable') }}
              </Button>
              <Button type="text" size="small" @click="handleShowTaskForm(s.id)">
                {{ t('common.edit') }}
              </Button>
              <Button type="text" size="small" @click="handleDeleteTask(s)">
                {{ t('common.delete') }}
              </Button>
            </div>
          </template>
        </Dropdown>
      </template>

      <template v-else #extra>
        <Button type="text" size="small" @click="handleDisableTask(s)">
          {{ s.disabled ? t('common.enable') : t('common.disable') }}
        </Button>
        <Button type="text" size="small" @click="handleShowTaskForm(s.id)">
          {{ t('common.edit') }}
        </Button>
        <Button type="text" size="small" @click="handleDeleteTask(s)">
          {{ t('common.delete') }}
        </Button>
      </template>
      <div>
        {{ t('scheduledtask.type') }}
        :
        {{ t('scheduledtask.' + s.type) }}
      </div>
      <div>
        {{ t('scheduledtask.cron') }}
        :
        {{ s.cron }}
      </div>
      <div v-if="appSettingsStore.app.scheduledtasksView === View.Grid">
        {{ t('scheduledtask.lastTime') }}
        :
        {{ s.lastTime ? formatRelativeTime(s.lastTime) : '--' }}
      </div>
      <div v-else>
        {{ t('scheduledtask.lastTime') }}
        :
        {{ s.lastTime ? formatDate(s.lastTime, 'YYYY-MM-DD HH:mm:ss') : '--' }}
      </div>
    </Card>
  </div>

  <Modal />
</template>
