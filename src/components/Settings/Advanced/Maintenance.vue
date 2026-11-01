<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import { useToast } from 'vue-toastification'

import { useConfirmDialog } from '~/composables/useConfirmDialog'
import { importSettingsStorage } from '~/composables/useSettingsStorage'
import { HomeSubPage } from '~/contentScripts/views/Home/types'
import { AppPage } from '~/enums/appEnums'
import type { MomentsWantedUser } from '~/logic'
import { momentsPinnedUsers, momentsWantedUsers, normalizeRecommendationModeSwitcherOptions, originalSettings, RECOMMENDATION_MODE_OPTIONS, settings, videoCardContextMenuKeys } from '~/logic'
import { applyPendingSettingsMigrations, formatSettingsMigrationConfirmMessage, getPendingSettingsMigrationChoices, hasPendingSettingsMigrations } from '~/utils/settingsMigration'

import SettingsItem from '../components/SettingsItem.vue'
import SettingsItemGroup from '../components/SettingsItemGroup.vue'

const { t } = useI18n()
const toast = useToast()
const { confirm: showConfirmDialog } = useConfirmDialog()
const importSettingsRef = ref<HTMLInputElement>()

interface NormalizedImportValue {
  compatible: boolean
  value?: unknown
}

const videoCardContextMenuKeySet = new Set<string>(videoCardContextMenuKeys)
const topBarComponentKeySet = new Set(originalSettings.topBarComponentsConfig.map(item => item.key))
const topBarBadgeTypes = new Set(['number', 'dot', 'none'])
const structuredArrayKeys = new Set([
  'dockItemsConfig',
  'filterByTitle',
  'filterByUser',
  'homePageTabVisibilityList',
  'recommendationModeSwitcherOptions',
  'topBarComponentsConfig',
  'topBarPinnedChannels',
  'videoCardContextMenuConfig',
  'videoCardShadowCurve',
])

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value)
}

function normalizeFilterRules(value: unknown): NormalizedImportValue {
  if (!Array.isArray(value))
    return { compatible: false }

  const normalized = value.flatMap((item) => {
    if (!isRecord(item) || typeof item.keyword !== 'string' || typeof item.remark !== 'string')
      return []

    const keyword = item.keyword.trim()
    if (!keyword)
      return []

    return [{ keyword, remark: item.remark.trim() }]
  })

  return value.length > 0 && normalized.length === 0
    ? { compatible: false }
    : { compatible: true, value: normalized }
}

function normalizeStructuredArray(key: string, value: unknown): NormalizedImportValue | null {
  if (!structuredArrayKeys.has(key))
    return null

  if (key === 'filterByTitle' || key === 'filterByUser')
    return normalizeFilterRules(value)

  if (!Array.isArray(value))
    return { compatible: false }

  if (key === 'topBarPinnedChannels') {
    return value.every(item => typeof item === 'string')
      ? { compatible: true, value: [...value] }
      : { compatible: false }
  }

  if (key === 'videoCardContextMenuConfig') {
    const compatible = value.every(item => isRecord(item)
      && typeof item.key === 'string'
      && videoCardContextMenuKeySet.has(item.key)
      && typeof item.visible === 'boolean')
    return compatible ? { compatible: true, value: structuredClone(value) } : { compatible: false }
  }

  if (key === 'topBarComponentsConfig') {
    const compatible = value.every(item => isRecord(item)
      && typeof item.key === 'string'
      && topBarComponentKeySet.has(item.key)
      && typeof item.visible === 'boolean'
      && typeof item.badgeType === 'string'
      && topBarBadgeTypes.has(item.badgeType))
    return compatible ? { compatible: true, value: structuredClone(value) } : { compatible: false }
  }

  if (key === 'dockItemsConfig') {
    const compatible = value.every(item => isRecord(item)
      && Object.values(AppPage).includes(item.page as AppPage)
      && typeof item.visible === 'boolean'
      && typeof item.openInNewTab === 'boolean'
      && typeof item.useOriginalBiliPage === 'boolean')
    return compatible ? { compatible: true, value: structuredClone(value) } : { compatible: false }
  }

  if (key === 'homePageTabVisibilityList') {
    const compatible = value.every(item => isRecord(item)
      && Object.values(HomeSubPage).includes(item.page as HomeSubPage)
      && typeof item.visible === 'boolean')
    return compatible ? { compatible: true, value: structuredClone(value) } : { compatible: false }
  }

  if (key === 'recommendationModeSwitcherOptions') {
    // 兼容旧版备份中按固定顺序保存的可见项字符串数组
    const isMode = (mode: unknown) => (RECOMMENDATION_MODE_OPTIONS as readonly unknown[]).includes(mode)
    const compatible = value.every(item => isMode(item)) || value.every(item => isRecord(item)
      && isMode(item.mode)
      && typeof item.visible === 'boolean')
    return compatible
      ? { compatible: true, value: normalizeRecommendationModeSwitcherOptions(value) }
      : { compatible: false }
  }

  if (key === 'videoCardShadowCurve') {
    const compatible = value.every(item => isRecord(item)
      && Number.isFinite(item.position)
      && Number.isFinite(item.opacity))
    return compatible ? { compatible: true, value: structuredClone(value) } : { compatible: false }
  }

  return null
}

function normalizeImportedValue(current: unknown, imported: unknown, key = ''): NormalizedImportValue {
  const structuredArray = normalizeStructuredArray(key, imported)
  if (structuredArray)
    return structuredArray

  if (Array.isArray(current)) {
    if (!Array.isArray(imported))
      return { compatible: false }
    if (!current.length)
      return { compatible: true, value: structuredClone(imported) }

    const normalized: unknown[] = []
    for (const item of imported) {
      const result = normalizeImportedValue(current[0], item)
      if (!result.compatible)
        return { compatible: false }
      normalized.push(result.value)
    }
    return { compatible: true, value: normalized }
  }

  if (key === 'savedVideoQuality') {
    const compatible = imported === null || (typeof imported === 'number' && Number.isSafeInteger(imported) && imported >= 0)
    return compatible ? { compatible: true, value: imported } : { compatible: false }
  }

  if (current === null) {
    if (key === 'savedVideoAspectRatio') {
      const compatible = imported === null || imported === '0:0' || imported === '4:3' || imported === '16:9'
      return compatible ? { compatible: true, value: imported } : { compatible: false }
    }
    return imported === null ? { compatible: true, value: null } : { compatible: false }
  }

  if (isRecord(current)) {
    if (!isRecord(imported))
      return { compatible: false }

    const normalized = structuredClone(current)
    for (const [nestedKey, nestedValue] of Object.entries(imported)) {
      if (!Object.prototype.hasOwnProperty.call(current, nestedKey))
        continue

      const result = normalizeImportedValue(current[nestedKey], nestedValue, nestedKey)
      if (!result.compatible)
        return { compatible: false }
      normalized[nestedKey] = result.value
    }
    return { compatible: true, value: normalized }
  }

  if (typeof current === 'number') {
    return typeof imported === 'number' && Number.isFinite(imported)
      ? { compatible: true, value: imported }
      : { compatible: false }
  }

  return typeof imported === typeof current
    ? { compatible: true, value: imported }
    : { compatible: false }
}

const SETTINGS_EXPORT_FORMAT_VERSION = 2

/**
 * 校验导出文件里的动态页用户列表。
 * 这些列表保存在独立 storage 键里，不在 settings 结构中，需要单独导入。
 */
function normalizeMomentsUsers(value: unknown): { users: MomentsWantedUser[], dropped: number } | null {
  if (!Array.isArray(value))
    return null

  const users: MomentsWantedUser[] = []
  let dropped = 0
  value.forEach((item) => {
    if (!isRecord(item)
      || typeof item.mid !== 'string'
      || !item.mid.trim()
      || typeof item.name !== 'string'
      || typeof item.face !== 'string') {
      dropped++
      return
    }

    users.push({ mid: item.mid.trim(), name: item.name, face: item.face })
  })

  return users.length > 0 || value.length === 0
    ? { users, dropped }
    : null
}

function handleImportSettings() {
  importSettingsRef.value?.click()
}

function handleImportFile(event: Event) {
  const input = event.target as HTMLInputElement
  const selectedFile = input.files?.[0]
  input.value = ''

  if (!selectedFile)
    return

  const reader = new FileReader()
  reader.onload = async () => {
    try {
      const importedSettings = JSON.parse(String(reader.result)) as Record<string, unknown>
      if (!importedSettings || Array.isArray(importedSettings) || typeof importedSettings !== 'object')
        throw new TypeError('Invalid settings backup')

      // v2 导出会在 settings 扁平字段之外额外带上动态页的两个用户列表；
      // 旧版导出没有这些字段，保持原有导入行为。
      const {
        bewlyExportVersion: exportVersion,
        momentsPinnedUsers: importedPinnedUsers,
        momentsWantedUsers: importedWantedUsers,
        ...settingsPayload
      } = importedSettings

      if (hasPendingSettingsMigrations(settingsPayload)) {
        const message = formatSettingsMigrationConfirmMessage(
          settingsPayload,
          t,
          'settings.maintenance.migrate_legacy_import_confirm',
        )
        const toggleFields = getPendingSettingsMigrationChoices(settingsPayload).map(choice => ({
          id: choice.id,
          label: String(t(choice.titleKey)),
          value: choice.value,
          enabledLabel: String(t('settings.chk_box.show')),
          disabledLabel: String(t('settings.chk_box.hidden')),
        }))
        const shouldMigrate = await showConfirmDialog(message ?? t('settings.maintenance.migrate_legacy_import_confirm'), {
          title: t('settings.maintenance.migrate_legacy_title'),
          confirmLabel: t('settings.maintenance.migrate_legacy_action'),
          toggleFields,
        })
        if (shouldMigrate) {
          applyPendingSettingsMigrations(settingsPayload, Object.fromEntries(
            toggleFields.map(field => [field.id, field.value]),
          ))
        }
      }

      const currentSettings = originalSettings as unknown as Record<string, unknown>
      const recognizedSettings: Record<string, unknown> = {}
      let importedCount = 0
      let ignoredCount = 0
      Object.entries(settingsPayload).forEach(([key, value]) => {
        if (!Object.prototype.hasOwnProperty.call(currentSettings, key)) {
          ignoredCount++
          return
        }

        const result = normalizeImportedValue(currentSettings[key], value, key)
        if (!result.compatible) {
          ignoredCount++
          return
        }

        recognizedSettings[key] = result.value
        importedCount++
      })

      const hasAuxiliaryPayload = typeof exportVersion === 'number' && exportVersion >= SETTINGS_EXPORT_FORMAT_VERSION
      const wantedUsers = hasAuxiliaryPayload && importedWantedUsers !== undefined
        ? normalizeMomentsUsers(importedWantedUsers)
        : null
      const pinnedUsers = hasAuxiliaryPayload && importedPinnedUsers !== undefined
        ? normalizeMomentsUsers(importedPinnedUsers)
        : null
      const auxiliaryImportedCount = (wantedUsers ? 1 : 0) + (pinnedUsers ? 1 : 0)
      const auxiliaryIgnoredCount = (hasAuxiliaryPayload && importedWantedUsers !== undefined && !wantedUsers ? 1 : 0)
        + (hasAuxiliaryPayload && importedPinnedUsers !== undefined && !pinnedUsers ? 1 : 0)
        + (wantedUsers?.dropped ?? 0)
        + (pinnedUsers?.dropped ?? 0)

      if (importedCount + auxiliaryImportedCount === 0) {
        toast.warning(t('settings.maintenance.import_no_matches'))
        return
      }

      if (importedCount > 0)
        await importSettingsStorage(recognizedSettings)

      if (wantedUsers)
        momentsWantedUsers.value = wantedUsers.users
      if (pinnedUsers)
        momentsPinnedUsers.value = pinnedUsers.users

      toast.success(t('settings.maintenance.import_success', {
        imported: importedCount + auxiliaryImportedCount,
        ignored: ignoredCount + auxiliaryIgnoredCount,
      }))
    }
    catch {
      toast.error(t('settings.maintenance.import_failed'))
    }
  }
  reader.readAsText(selectedFile)
}

function handleExportSettings() {
  // 动态页的两个用户列表是独立 storage 键，必须显式并入导出负载。
  const payload = {
    ...settings.value,
    bewlyExportVersion: SETTINGS_EXPORT_FORMAT_VERSION,
    momentsPinnedUsers: momentsPinnedUsers.value,
    momentsWantedUsers: momentsWantedUsers.value,
  }
  const jsonStr = JSON.stringify(payload, null, 2)
  const blob = new Blob([jsonStr], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  const dateTimeStr = new Date().toLocaleString('sv-SE').replace(/[- :]/g, '')

  link.href = url
  link.download = `bewly-settings-${dateTimeStr}.json`
  link.click()
  URL.revokeObjectURL(url)
}

async function handleResetSettings() {
  if (!await showConfirmDialog(t('settings.reset_settings_confirm')))
    return

  // 重置时保留用户当前使用的语言
  const resetSettings = structuredClone(originalSettings)
  resetSettings.language = settings.value.language
  settings.value = resetSettings
}
</script>

<template>
  <div>
    <SettingsItemGroup
      :title="$t('settings.maintenance.backup_title')"
      :desc="$t('settings.maintenance.backup_desc')"
    >
      <SettingsItem
        :title="$t('settings.import_settings')"
        :desc="$t('settings.maintenance.import_desc')"
        right-width="auto"
      >
        <input
          ref="importSettingsRef"
          type="file"
          accept=".json"
          hidden
          @change="handleImportFile"
        >
        <Button @click="handleImportSettings">
          <template #left>
            <div i-uil:import />
          </template>
          {{ $t('settings.import_settings') }}
        </Button>
      </SettingsItem>
      <SettingsItem
        :title="$t('settings.export_settings')"
        :desc="$t('settings.export_settings_desc')"
        right-width="auto"
      >
        <Button @click="handleExportSettings">
          <template #left>
            <div i-uil:export />
          </template>
          {{ $t('settings.export_settings') }}
        </Button>
      </SettingsItem>
    </SettingsItemGroup>

    <SettingsItemGroup
      :title="$t('settings.maintenance.reset_title')"
      :desc="$t('settings.maintenance.reset_desc')"
    >
      <SettingsItem
        :title="$t('settings.reset_settings')"
        :desc="$t('settings.maintenance.reset_warning')"
        :badge="$t('settings.badge_irreversible')"
        right-width="auto"
      >
        <Button class="danger-button" @click="handleResetSettings">
          <template #left>
            <i i-mingcute:back-line />
          </template>
          {{ $t('settings.reset_settings') }}
        </Button>
      </SettingsItem>
    </SettingsItemGroup>
  </div>
</template>

<style lang="scss" scoped>
.danger-button {
  color: var(--bew-error-color);
}
</style>
