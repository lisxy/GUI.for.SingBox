<script setup lang="ts">
import { computed, h, inject, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

import { HttpGet, Readfile, Writefile } from '@/bridge'
import { RulesetFormat } from '@/enums/kernel'
import { useRulesetsStore } from '@/stores'
import { ignoredError, message, alert } from '@/utils'

import Button from '@/components/Button/index.vue'
import Pagination from '@/components/Pagination/index.vue'

type RulesetHub = {
  geosite: string
  geoip: string
  list: { name: string; type: 'geosite' | 'geoip'; description: string; count: number }[]
}

const pageSize = 27
const loading = ref(false)
const currentPage = ref(1)
const rulesetHub = ref<RulesetHub>({ geosite: '', geoip: '', list: [] })
const cacheFile = 'data/.cache/ruleset-list.json'
const hubUrl =
  'https://github.com/GUI-for-Cores/Ruleset-Hub/releases/download/latest/sing-full.json'

const { t } = useI18n()
const rulesetsStore = useRulesetsStore()

const keywords = ref('')
const handleCancel = inject('cancel') as any

watch(keywords, () => (currentPage.value = 1))

const filteredList = computed(() => {
  if (!keywords.value) return rulesetHub.value.list
  return rulesetHub.value.list.filter((ruleset) => ruleset.name.includes(keywords.value))
})

const currentList = computed(() => {
  return filteredList.value.slice(
    (currentPage.value - 1) * pageSize,
    (currentPage.value - 1) * pageSize + pageSize,
  )
})

const updateList = async () => {
  loading.value = true
  try {
    const { body } = await HttpGet<string>(hubUrl)
    rulesetHub.value = JSON.parse(body)
    await Writefile(cacheFile, body)
    message.success('rulesets.updateSuccess')
  } catch (error: any) {
    message.error(error)
  }
  loading.value = false
}

const getList = async () => {
  const body = await ignoredError(Readfile, cacheFile)
  if (body) {
    rulesetHub.value = JSON.parse(body)
    return
  }

  updateList()
}

const getRulesetUrlAndSuffix = (ruleset: RulesetHub['list'][number], format: RulesetFormat) => {
  const suffix = { [RulesetFormat.Binary]: '.srs', [RulesetFormat.Source]: '.json' }[format]
  const basrUrl = { geosite: rulesetHub.value.geosite, geoip: rulesetHub.value.geoip }[ruleset.type]
  return [basrUrl + ruleset.name + suffix, suffix]
}

const handleAddRuleset = async (ruleset: RulesetHub['list'][number], format: RulesetFormat) => {
  const [url, suffix] = getRulesetUrlAndSuffix(ruleset, format)
  const id = ruleset.type + '_' + ruleset.name + '.' + format
  const file = ruleset.type + '_' + ruleset.name + suffix
  try {
    await rulesetsStore.addRuleset({
      id,
      tag: `${ruleset.name}-${ruleset.type}${suffix}`,
      updateTime: 0,
      disabled: false,
      type: 'Http',
      format,
      path: 'data/rulesets/' + file,
      url,
      count: ruleset.count,
    })
    const { success } = message.info('rulesets.updating')
    await rulesetsStore.updateRuleset(id)
    success('common.success')
  } catch (error: any) {
    console.error(error)
    message.error(error.message || error)
  }
}

const handlePreview = async (ruleset: RulesetHub['list'][number], format: RulesetFormat) => {
  const { destroy, error } = message.info('rulesets.fetching', 15_000)
  try {
    const { body } = await HttpGet(getRulesetUrlAndSuffix(ruleset, format)[0])
    destroy()
    await alert(ruleset.name, JSON.stringify(body, null, 2))
  } catch (err: any) {
    error(err.message || err)
    setTimeout(destroy, 2000)
  }
}

const isAlreadyAdded = (id: string) => rulesetsStore.getRulesetById(id)

getList()

const modalSlots = {
  action: () =>
    h(Pagination, {
      current: currentPage.value,
      'onUpdate:current': (current: number) => (currentPage.value = current),
      total: filteredList.value.length,
      pageSize: pageSize,
      size: 'small',
      class: 'mr-auto',
      style: {
        display: loading.value ? 'block' : '',
      },
    }),
  close: () =>
    h(
      Button,
      {
        type: 'text',
        onClick: handleCancel,
      },
      () => t('common.close'),
    ),
}

defineExpose({ modalSlots })
</script>

<template>
  <div class="ruleset-hub">
    <div v-if="loading" class="loading"><Button type="text" loading /></div>
    <template v-else>
      <div class="header">
        <Button type="text">{{ t('rulesets.total') }} : {{ rulesetHub.list.length }}</Button>
        <Input
          v-model="keywords"
          size="small"
          clearable
          auto-size
          :placeholder="t('common.keywords')"
          class="ml-8 flex-1"
        />
        <Button @click="updateList" type="link" class="ml-auto">
          {{ t('plugins.update') }}
        </Button>
      </div>

      <div class="list">
        <Card
          v-for="ruleset in currentList"
          :key="ruleset.name + ruleset.type"
          :title="ruleset.name"
          class="ruleset-item"
        >
          <template #extra>
            <Tag size="small" color="cyan">{{ ruleset.type }}</Tag>
          </template>
          <div class="count">
            {{ t('rulesets.rulesetCount') }} : {{ ruleset.count }}
            <Button
              @click="handlePreview(ruleset, RulesetFormat.Source)"
              icon="preview"
              size="small"
              type="text"
            />
          </div>
          <div class="description">
            {{ ruleset.description || t('rulesets.noDesc') }}
          </div>
          <div class="action">
            <template
              v-if="isAlreadyAdded(ruleset.type + '_' + ruleset.name + '.' + RulesetFormat.Source)"
            >
              <Button type="text" size="small">
                {{ t('ruleset.format.source') }} {{ t('common.added') }}
              </Button>
            </template>
            <template v-else>
              <Button
                @click="handleAddRuleset(ruleset, RulesetFormat.Source)"
                type="link"
                size="small"
              >
                {{ t('common.add') }} {{ t('ruleset.format.source') }}
              </Button>
            </template>
            <template
              v-if="isAlreadyAdded(ruleset.type + '_' + ruleset.name + '.' + RulesetFormat.Binary)"
            >
              <Button type="text" size="small">
                {{ t('ruleset.format.binary') }} {{ t('common.added') }}
              </Button>
            </template>
            <template v-else>
              <Button
                @click="handleAddRuleset(ruleset, RulesetFormat.Binary)"
                type="link"
                size="small"
              >
                {{ t('common.add') }} {{ t('ruleset.format.binary') }}
              </Button>
            </template>
          </div>
        </Card>
      </div>
    </template>
  </div>
</template>

<style lang="less" scoped>
.ruleset-hub {
  height: 100%;
  display: flex;
  flex-direction: column;

  .ruleset-item {
    display: inline-block;
    margin: 4px;
    font-size: 12px;
    width: calc(33.333% - 8px);
    .count {
      display: flex;
      align-items: center;
      justify-content: space-between;
    }
    .description {
      margin: 4px 0;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    .action {
      text-align: right;
    }
  }
}

.loading {
  display: flex;
  justify-content: center;
  height: 98%;
}

.header {
  display: flex;
  align-items: center;
}

.list {
  flex: 1;
  padding-bottom: 16px;
  overflow: auto;
}
</style>
