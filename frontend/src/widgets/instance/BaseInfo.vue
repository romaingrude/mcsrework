<script setup lang="ts">
import { useInstanceInfo } from "@/hooks/useInstance";
import { t } from "@/lang/i18n";
import type { LayoutCard } from "@/types";
import { getInstanceServerStatus } from "@/services/apis/instance";
import { CheckCircleOutlined, ExclamationCircleOutlined } from "@ant-design/icons-vue";
import { computed, onMounted, onUnmounted, ref, watch } from "vue";
import { GLOBAL_INSTANCE_NAME } from "../../config/const";
import { useLayoutCardTools } from "../../hooks/useCardTools";
import { parseTimestamp } from "../../tools/time";
import DockerInfo from "./dialogs/DockerInfo.vue";

const props = defineProps<{
  card: LayoutCard;
}>();

const DockerInfoDialog = ref<InstanceType<typeof DockerInfo>>();
const { getMetaOrRouteValue } = useLayoutCardTools(props.card);

const instanceId = getMetaOrRouteValue("instanceId");
const daemonId = getMetaOrRouteValue("daemonId");

const { statusText, isRunning, isStopped, instanceTypeText, instanceInfo, execute } =
  useInstanceInfo({
    instanceId,
    daemonId,
    autoRefresh: true
  });

const { state: serverStatusState, execute: fetchServerStatus } = getInstanceServerStatus();
const serverStatusTimer = ref<ReturnType<typeof setInterval>>();

const getInstanceName = computed(() => {
  if (instanceInfo.value?.config.nickname === GLOBAL_INSTANCE_NAME) {
    return t("TXT_CODE_5bdaf23d");
  } else {
    return instanceInfo.value?.config.nickname;
  }
});

const refreshServerStatus = async () => {
  if (!instanceId || !daemonId) return;
  try {
    await fetchServerStatus({
      params: {
        uuid: instanceId,
        daemonId: daemonId
      }
    });
  } catch (error) {
    // ignore status errors
  }
};

const clearServerStatusTimer = () => {
  if (serverStatusTimer.value) {
    clearInterval(serverStatusTimer.value);
    serverStatusTimer.value = undefined;
  }
};

watch(
  () => isRunning.value,
  async (running) => {
    clearServerStatusTimer();
    if (running) {
      await refreshServerStatus();
      serverStatusTimer.value = setInterval(refreshServerStatus, 15000);
    } else {
      serverStatusState.value = undefined;
    }
  },
  { immediate: true }
);

const serverStatusText = computed(() => {
  if (!isRunning.value) return t("TXT_CODE_15f2e564");
  if (serverStatusState.value?.online) return t("TXT_CODE_bdb620b9");
  if (serverStatusState.value) return t("TXT_CODE_15f2e564");
  return t("TXT_CODE_c8333afa");
});

const serverStatusColor = computed(() => {
  if (!isRunning.value) return "default";
  if (serverStatusState.value?.online) return "green";
  if (serverStatusState.value) return "red";
  return "orange";
});

const instanceGameServerInfo = computed(() => {
  if (serverStatusState.value?.online) {
    return {
      players: `${serverStatusState.value.currentPlayers} / ${serverStatusState.value.maxPlayers}`,
      version: serverStatusState.value.version
    };
  }
  if (instanceInfo.value?.info?.mcPingOnline) {
    return {
      players: `${instanceInfo.value?.info.currentPlayers} / ${instanceInfo.value?.info.maxPlayers}`,
      version: instanceInfo.value?.info.version
    };
  }
  return null;
});

onMounted(async () => {
  if (instanceId && daemonId) {
    await execute({
      params: {
        uuid: instanceId,
        daemonId: daemonId
      }
    });
  }
});

onUnmounted(() => {
  clearServerStatusTimer();
});
</script>

<template>
  <!-- eslint-disable vue/html-indent -->
  <CardPanel class="containerWrapper" style="height: 100%">
    <template #title>
      {{ card.title }}
    </template>
    <template #body>
      <a-typography-paragraph>
        {{ t("TXT_CODE_7ec9c59c") }}{{ getInstanceName }}
      </a-typography-paragraph>
      <a-typography-paragraph>
        <div class="flex flex-wrap instance-tag">
          <!-- instance status -->
          <a-tag v-if="isRunning" color="green" class="tag">
            <CheckCircleOutlined />
            {{ statusText }}
          </a-tag>
          <a-tag v-else-if="isStopped" class="tag">
            <ExclamationCircleOutlined />
            {{ statusText }}
          </a-tag>
          <a-tag v-else class="tag" color="pink">
            {{ statusText }}
          </a-tag>

          <a-tag class="tag" :color="serverStatusColor">
            {{ t("TXT_CODE_server_status_label") }}: {{ serverStatusText }}
          </a-tag>

          <!-- instance type -->
          <a-tag class="tag" color="purple"> {{ instanceTypeText }}</a-tag>

          <a-tag color="purple" class="tag">
            {{ t("TXT_CODE_ad30f3c5") }}{{ instanceInfo?.started }}
          </a-tag>
          
          <a-tag color="purple" class="tag">
            {{ t("TXT_CODE_6420023d") }}{{ instanceInfo?.autoRestarted }}
          </a-tag>

          <!-- real tags -->
          <a-tag v-for="tag in instanceInfo?.config.tag" :key="tag" class="tag" color="blue">
            {{ tag }}
          </a-tag>
        </div>
      </a-typography-paragraph>

      <a-typography-paragraph v-if="instanceGameServerInfo">
        <span>{{ t("TXT_CODE_855c4a1c") }}</span>
        <span>{{ instanceGameServerInfo.players }}</span>
      </a-typography-paragraph>
      <a-typography-paragraph v-if="instanceGameServerInfo">
        <span>
          {{ t("TXT_CODE_e260a220") }}
        </span>
        <span>
          {{ instanceGameServerInfo.version }}
        </span>
      </a-typography-paragraph>

      <template v-if="instanceInfo?.config.processType === 'docker'">
        <a-typography-paragraph>
          {{ t("TXT_CODE_4f917a65") }}
          <a href="javascript:;" @click="DockerInfoDialog?.openDialog()">
            {{ t("TXT_CODE_530f5951") }}
          </a>
        </a-typography-paragraph>
      </template>

      <a-typography-paragraph v-if="Number(instanceInfo?.info?.allocatedPorts?.length) > 0">
        {{ t("TXT_CODE_2e4469f6") }}
        <div style="padding: 10px 0px 0px 16px">
          <div
            v-for="(item, index) in instanceInfo?.info?.allocatedPorts"
            :key="index"
            class="mb-4"
          >
            <span>
              <a-tag color="green">{{ item.protocol.toUpperCase() }}</a-tag>
            </span>
            <a-tag>
              <span>{{ t("TXT_CODE_8dfc41ef") }}: {{ item.host }}</span>
              <span class="ml-4"> {{ t("TXT_CODE_8f8103b7") }}: {{ item.container }} </span>
            </a-tag>
          </div>
        </div>
      </a-typography-paragraph>

      <a-typography-paragraph>
        <span>{{ t("TXT_CODE_ae747cc0") }}</span>
        <span>{{ parseTimestamp(instanceInfo?.config.endTime) || t("TXT_CODE_e3a77a77") }}</span>
      </a-typography-paragraph>
      <a-typography-paragraph v-if="!instanceGameServerInfo">
        {{ t("TXT_CODE_8b8e08a6") }}{{ parseTimestamp(instanceInfo?.config.createDatetime) }}
      </a-typography-paragraph>
      <a-typography-paragraph>
        {{ t("TXT_CODE_46f575ae") }}{{ parseTimestamp(instanceInfo?.config.lastDatetime) }}
      </a-typography-paragraph>
      <a-typography-paragraph v-if="!instanceGameServerInfo">
        <span>{{ t("TXT_CODE_cec321b4") }}{{ instanceInfo?.config.oe.toUpperCase() }} </span>
        <span class="ml-6">
          {{ t("TXT_CODE_400a4210") }}{{ instanceInfo?.config.ie.toUpperCase() }}
        </span>
      </a-typography-paragraph>
      <a-typography-paragraph>
        <a-typography-text :title="instanceInfo?.instanceUuid">
          {{ t("TXT_CODE_30051f9b") }}
        </a-typography-text>
        <a-typography-text :copyable="{ text: instanceInfo?.instanceUuid }"> </a-typography-text>
        <a-typography-text class="ml-20" :title="daemonId">
          {{ t("TXT_CODE_5f2d2e30") }}
        </a-typography-text>
        <a-typography-text :copyable="{ text: daemonId }"> </a-typography-text>
      </a-typography-paragraph>
    </template>
  </CardPanel>

  <DockerInfo ref="DockerInfoDialog" :docker-info="instanceInfo?.config.docker" />
</template>

<style lang="scss" scoped>
.instance-tag {
  margin-left: -4px;
  margin-right: -4px;
  .tag {
    margin: 4px;
  }
}
</style>
