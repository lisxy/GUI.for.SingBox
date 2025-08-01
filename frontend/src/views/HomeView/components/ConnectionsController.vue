<script lang="ts" setup>
import { ref, computed, onUnmounted } from 'vue'
import { useI18n } from 'vue-i18n'

import { deleteConnection } from '@/api/kernel'
import { DraggableOptions } from '@/constant/app'
import { DefaultConnections } from '@/constant/kernel'
import { useBool } from '@/hooks'
import { useAppSettingsStore, useKernelApiStore } from '@/stores'
import { addToRuleSet, formatBytes, formatRelativeTime, message, picker } from '@/utils'

import type { PickerItem } from '@/components/Picker/index.vue'
import type { Column } from '@/components/Table/index.vue'
import type { Menu } from '@/types/app'
import type { CoreApiConnectionsData } from '@/types/kernel'

type TrafficCacheType = { up: number; down: number }
const TrafficCache: Record<string, TrafficCacheType> = {}

const appSettingsStore = useAppSettingsStore()

const columns = computed(() =>
  (
    [
      {
        title: 'home.connections.type',
        align: 'center',
        key: 'metadata.type',
        hidden: !appSettingsStore.app.connections.visibility['metadata.type'],
        sort: (a, b) => b.metadata.type.localeCompare(a.metadata.type),
        customRender: ({ value, record }) => {
          return value + '(' + record.metadata.network + ')'
        },
      },
      {
        title: 'home.connections.processPath',
        key: 'metadata.processPath',
        hidden: !appSettingsStore.app.connections.visibility['metadata.processPath'],
        sort: (a, b) => b.metadata.processPath.localeCompare(a.metadata.processPath),
      },
      {
        title: 'home.connections.host',
        key: 'metadata.host',
        hidden: !appSettingsStore.app.connections.visibility['metadata.host'],
        sort: (a, b) => b.metadata.host.localeCompare(a.metadata.host),
        customRender: ({ value, record }) => {
          return value || record.metadata.destinationIP
        },
      },
      {
        title: 'home.connections.sourceIP',
        align: 'center',
        key: 'metadata.sourceIP',
        hidden: !appSettingsStore.app.connections.visibility['metadata.sourceIP'],
        sort: (a, b) => b.metadata.sourceIP.localeCompare(a.metadata.sourceIP),
        customRender: ({ value, record }) => {
          return value + ':' + record.metadata.sourcePort
        },
      },
      {
        title: 'home.connections.destinationIP',
        align: 'center',
        key: 'metadata.destinationIP',
        hidden: !appSettingsStore.app.connections.visibility['metadata.destinationIP'],
        sort: (a, b) => b.metadata.destinationIP.localeCompare(a.metadata.destinationIP),
        customRender: ({ value, record }) => {
          return value + ':' + record.metadata.destinationPort
        },
      },
      {
        title: 'home.connections.rule',
        align: 'center',
        key: 'rule',
        hidden: !appSettingsStore.app.connections.visibility['rule'],
        sort: (a, b) => b.rule.localeCompare(a.rule),
        customRender: ({ value, record }) => {
          return value + (record.rulePayload ? '::' + record.rulePayload : '')
        },
      },
      {
        title: 'home.connections.chains',
        key: 'chains',
        hidden: !appSettingsStore.app.connections.visibility['chains'],
        sort: (a, b) => b.chains[0].localeCompare(a.chains[0]),
        customRender: ({ value }) => value.slice().reverse().join(' :: '),
      },
      {
        title: 'home.connections.uploadSpeed',
        align: 'center',
        key: 'up',
        minWidth: '90px',
        hidden: !appSettingsStore.app.connections.visibility['up'],
        sort: (a, b) => b.upload - b.up - (a.upload - a.up),
        customRender: ({ value, record }) => formatBytes(record.upload - value) + '/s',
      },
      {
        title: 'home.connections.downSpeed',
        align: 'center',
        key: 'down',
        minWidth: '90px',
        hidden: !appSettingsStore.app.connections.visibility['down'],
        sort: (a, b) => b.download - b.down - (a.download - a.down),
        customRender: ({ value, record }) => formatBytes(record.download - value) + '/s',
      },
      {
        title: 'home.connections.upload',
        align: 'center',
        key: 'upload',
        hidden: !appSettingsStore.app.connections.visibility['upload'],
        sort: (a, b) => b.upload - a.upload,
        customRender: ({ value }) => formatBytes(value),
      },
      {
        title: 'home.connections.download',
        align: 'center',
        key: 'download',
        hidden: !appSettingsStore.app.connections.visibility['download'],
        sort: (a, b) => b.download - a.download,
        customRender: ({ value }) => formatBytes(value),
      },
      {
        title: 'home.connections.time',
        align: 'center',
        key: 'start',
        hidden: !appSettingsStore.app.connections.visibility['start'],
        sort: (a, b) => new Date(a.start).getTime() - new Date(b.start).getTime(),
        customRender: ({ value }) => formatRelativeTime(value),
      },
    ] as Column[]
  ).sort(
    (a, b) =>
      appSettingsStore.app.connections.order.indexOf(a.key) -
      appSettingsStore.app.connections.order.indexOf(b.key),
  ),
)

const columnTitleMap = computed(() => {
  const map: Record<string, string | undefined> = {}
  appSettingsStore.app.connections.order.forEach(
    (field) => (map[field] = columns.value.find((column) => column.key === field)?.title),
  )
  return map
})

const menu: Menu[] = [
  {
    label: 'common.details',
    handler: (record: Record<string, any>) => {
      details.value = JSON.stringify(record, null, 2)
      toggleDetails()
    },
  },
  {
    label: 'home.connections.close',
    handler: async (record: Record<string, any>) => {
      if (!isActive.value) return
      try {
        await deleteConnection(record.id)
      } catch (error: any) {
        console.log(error)
        message.error(error)
      }
    },
  },
  ...[
    ['home.connections.addToDirect', 'direct'],
    ['home.connections.addToProxy', 'proxy'],
    ['home.connections.addToReject', 'block'],
  ].map(([label, ruleset]) => {
    return {
      label,
      handler: async (record: Record<string, any>) => {
        const options: PickerItem<Record<string, any>[]>[] = []
        if (record.metadata.host) {
          options.push({
            label: t('kernel.rules.type.domain'),
            value: { domain: record.metadata.host } as any,
            description: record.metadata.host,
          })
          const domain_suffix = '.' + record.metadata.host.split('.').slice(1).join('.')
          options.push({
            label: t('kernel.rules.type.domain_suffix'),
            value: {
              domain_suffix: domain_suffix,
            } as any,
            description: domain_suffix,
          })
        }
        if (record.metadata.destinationIP) {
          options.push({
            label: t('kernel.rules.type.ip_cidr'),
            value: { ip_cidr: record.metadata.destinationIP + '/32' } as any,
            description: record.metadata.destinationIP,
          })
        }
        if (record.metadata.processPath) {
          options.push({
            label: t('kernel.rules.type.process_path'),
            value: { process_path: record.metadata.processPath } as any,
            description: record.metadata.processPath,
          })
        }
        const payloads = await picker.multi('rulesets.selectRuleType', options)
        try {
          await addToRuleSet(ruleset as any, payloads)
          message.success('common.success')
        } catch (error: any) {
          message.error(error)
          console.log(error)
        }
      },
    }
  }),
]

const details = ref()
const isActive = ref(true)
const keywords = ref('')
const dataSource = ref<(CoreApiConnectionsData['connections'][0] & TrafficCacheType)[]>([])
const disconnectedData = ref<CoreApiConnectionsData['connections']>([])
const [showDetails, toggleDetails] = useBool(false)
const [showSettings, toggleSettings] = useBool(false)
const [isPause, togglePause] = useBool(false)
const { t } = useI18n()
const kernelApiStore = useKernelApiStore()

const filteredConnections = computed(() => {
  if (!keywords.value) return isActive.value ? dataSource.value : disconnectedData.value
  return (isActive.value ? dataSource.value : disconnectedData.value).filter((connection) =>
    Object.values(connection.metadata).some((v) =>
      String(v).toLocaleLowerCase().includes(keywords.value.toLocaleLowerCase()),
    ),
  )
})

const handleCloseAll = async () => {
  try {
    await Promise.all(
      filteredConnections.value.map((connection) => deleteConnection(connection.id)),
    )
    disconnectedData.value.push(...filteredConnections.value)
    dataSource.value = dataSource.value.filter(
      (connection) => !filteredConnections.value.find((c) => c.id === connection.id),
    )
  } catch (error: any) {
    message.error(error.message || error)
  }
}

const handleClearClosedConns = () => {
  disconnectedData.value.splice(0)
}

const handleResetConnections = () => {
  appSettingsStore.app.connections = DefaultConnections()
  message.success('common.success')
}

const unregisterConnectionsHandler = kernelApiStore.onConnections((data) => {
  if (isPause.value) return
  const connections = data.connections || []

  dataSource.value.forEach((connection) => {
    // Record Disconnected Connections
    const exist = connections.some((v) => v.id === connection.id)
    !exist && disconnectedData.value.push(connection)
  })

  dataSource.value = connections.map((connection) => {
    // Record Previous Traffic Information
    const result = { ...connection, up: 0, down: 0 }
    const cache = TrafficCache[connection.id]
    result.up = cache?.up || connection.upload
    result.down = cache?.down || connection.download
    TrafficCache[connection.id] = {
      down: connection.download,
      up: connection.upload,
    }
    return result
  })
})

onUnmounted(() => {
  unregisterConnectionsHandler()
})
</script>

<template>
  <div class="flex flex-col h-full">
    <div class="flex items-center">
      <Radio
        v-model="isActive"
        :options="[
          { label: 'home.connections.active', value: true },
          { label: 'home.connections.closed', value: false },
        ]"
        size="small"
      />
      <Input
        v-model="keywords"
        clearable
        auto-size
        size="small"
        placeholder="Search"
        class="ml-8 flex-1"
      />
      <Button
        @click="togglePause"
        :icon="isPause ? 'play' : 'pause'"
        size="small"
        type="text"
        class="ml-8"
      />
      <Button
        v-if="isActive"
        @click="handleCloseAll"
        v-tips="'home.connections.closeAll'"
        icon="close"
        size="small"
        type="text"
      />
      <Button
        v-else
        @click="handleClearClosedConns"
        v-tips="'common.clear'"
        icon="clear"
        size="small"
        type="text"
      />
      <Button @click="toggleSettings" icon="settings" size="small" type="text" />
    </div>
    <Table
      class="flex-1 mt-8"
      :columns="columns"
      :menu="menu"
      :data-source="filteredConnections"
      sort="start"
    />
  </div>

  <Modal
    v-model:open="showDetails"
    :submit="false"
    cancel-text="common.close"
    title="home.connections.details"
    max-height="80"
    max-width="80"
    mask-closable
  >
    <CodeViewer v-model="details" />
  </Modal>

  <Modal
    v-model:open="showSettings"
    :submit="false"
    mask-closable
    max-height="80"
    cancel-text="common.close"
    title="home.connections.sort"
  >
    <template #action>
      <Button @click="handleResetConnections" type="text" class="mr-auto">
        {{ t('common.reset') }}
      </Button>
    </template>
    <div v-draggable="[appSettingsStore.app.connections.order, DraggableOptions]">
      <Card v-for="column in appSettingsStore.app.connections.order" :key="column" class="mb-2">
        <div class="flex items-center justify-between py-2">
          <span class="font-bold">{{ t(columnTitleMap[column] || column) }}</span>
          <Switch v-model="appSettingsStore.app.connections.visibility[column]" />
        </div>
      </Card>
    </div>
  </Modal>
</template>
