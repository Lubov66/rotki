<script setup lang="ts">
const anonymousUsageAnalytics = ref<boolean>(false);
const { submitUsageAnalytics } = storeToRefs(useGeneralSettingsStore());

onMounted(() => {
  set(anonymousUsageAnalytics, get(submitUsageAnalytics));
});

const { t } = useI18n();
</script>

<template>
  <SettingsOption
    setting="submitUsageAnalytics"
    :error-message="t('general_settings.usage_analytics.validation.error')"
  >
    <template #title>
      {{ t('general_settings.usage_analytics.title') }}
    </template>
    <template #default="{ error, success, updateImmediate }">
      <RuiSwitch
        v-model="anonymousUsageAnalytics"
        class="general-settings__fields__anonymous-usage-statistics"
        color="primary"
        :label="t('general_settings.usage_analytics.label')"
        :success-messages="success"
        :error-messages="error"
        @update:model-value="updateImmediate($event)"
      />
    </template>
  </SettingsOption>
</template>
