<template>
  <div
    :class="[!showTab ? 'h-48' : 'h-56']"
    class="bg-accent rounded-b-2xl absolute top-0 left-0 w-full"
  ></div>
  <div
    :class="[!showTab ? 'mb-6' : 'mb-2']"
    class="dark placeholder-black relative z-10 text-white px-5 pt-8"
  >
    <div class="flex items-center mb-4">
      <h1 class="text-xl font-semibold text-white">Automa</h1>
      <div class="flex-grow"></div>
      <ui-button
        v-tooltip.group="t('home.record.title')"
        icon
        class="mr-2"
        @click="state.newRecordingModal = true"
      >
        <v-remixicon name="riRecordCircleLine" />
      </ui-button>
      <ui-button
        v-tooltip.group="
          t(`home.elementSelector.${state.haveAccess ? 'name' : 'noAccess'}`)
        "
        icon
        class="mr-2"
        @click="initElementSelector"
      >
        <v-remixicon name="riFocus3Line" />
      </ui-button>
      <ui-button
        v-tooltip.group="t('common.dashboard')"
        icon
        :title="t('common.dashboard')"
        @click="openDashboard('')"
      >
        <v-remixicon name="riHome5Line" />
      </ui-button>
    </div>
    <div class="flex">
      <ui-input
        v-model="state.query"
        :placeholder="`${t('common.search')}...`"
        autocomplete="off"
        prepend-icon="riSearch2Line"
        class="w-full search-input"
      />
    </div>
    <ui-tabs
      v-if="showTab"
      v-model="state.activeTab"
      fill
      class="mt-1"
      @change="onTabChange"
    >
      <ui-tab value="local">
        {{ t(`home.workflow.type.local`) }}
      </ui-tab>
      <ui-tab v-if="hostedWorkflowStore.toArray.length > 0" value="host">
        {{ t(`home.workflow.type.host`) }}
      </ui-tab>
      <ui-tab v-if="userStore.user?.teams" value="team"> Teams </ui-tab>
    </ui-tabs>
  </div>
  <home-team-workflows
    v-if="state.retrieved"
    v-show="state.activeTab === 'team'"
    :search="state.query"
  />
  <div v-if="state.activeTab !== 'team'" class="px-5 pb-5 space-y-2">
    <ui-card v-if="workflowStore.getWorkflows.length === 0" class="text-center">
      <img src="@/assets/svg/alien.svg" />
      <p class="font-semibold">{{ t('message.empty') }}</p>
      <ui-button
        variant="accent"
        class="mt-6"
        @click="openDashboard('/workflows')"
      >
        {{ t('home.workflow.new') }}
      </ui-button>
    </ui-card>
    <home-workflow-card
      v-for="workflow in workflows"
      :key="workflow.id"
      :workflow="workflow"
      :tab="state.activeTab"
      @details="openWorkflowPage"
      @update="updateWorkflow(workflow.id, $event)"
      @execute="executeWorkflow"
      @rename="renameWorkflow"
      @delete="deleteWorkflow"
    />
  </div>
  <ui-modal v-model="state.newRecordingModal" custom-content>
    <ui-card
      :style="{ height: `${state.cardHeight}px` }"
      class="w-full recording-card overflow-hidden rounded-b-none"
      padding="p-0"
    >
      <div class="flex items-center px-4 pt-4 pb-2">
        <p class="flex-1 font-semibold">
          {{ t('home.record.title') }}
        </p>
        <v-remixicon
          class="text-gray-600 dark:text-gray-300 cursor-pointer"
          name="riCloseLine"
          size="20"
          @click="state.newRecordingModal = false"
        ></v-remixicon>
      </div>
      <home-start-recording
        @record="recordWorkflow"
        @close="state.newRecordingModal = false"
        @update="state.cardHeight = recordingCardHeight[$event] || 255"
      />
    </ui-card>
  </ui-modal>
</template>
<script setup>
import { computed, onMounted, shallowReactive } from 'vue';
import { useI18n } from 'vue-i18n';
import browser from 'webextension-polyfill';
import { useUserStore } from '@/stores/user';
import { useDialog } from '@/composable/dialog';
import { sendMessage } from '@/utils/message';
import { useWorkflowStore } from '@/stores/workflow';
import { useGroupTooltip } from '@/composable/groupTooltip';
import { useTeamWorkflowStore } from '@/stores/teamWorkflow';
import { useHostedWorkflowStore } from '@/stores/hostedWorkflow';
import HomeWorkflowCard from '@/components/popup/home/HomeWorkflowCard.vue';
import HomeTeamWorkflows from '@/components/popup/home/HomeTeamWorkflows.vue';
import HomeStartRecording from '@/components/popup/home/HomeStartRecording.vue';

const recordingCardHeight = {
  new: 255,
  existing: 480,
};

const { t } = useI18n();
const dialog = useDialog();
const userStore = useUserStore();
const workflowStore = useWorkflowStore();
const teamWorkflowStore = useTeamWorkflowStore();
const hostedWorkflowStore = useHostedWorkflowStore();

useGroupTooltip();

const state = shallowReactive({
  query: '',
  teams: [],
  cardHeight: 255,
  retrieved: false,
  haveAccess: true,
  activeTab: 'local',
  newRecordingModal: false,
});

const hostedWorkflows = computed(() => {
  if (state.activeTab !== 'host') return [];

  return hostedWorkflowStore.toArray.filter((workflow) =>
    workflow.name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
  );
});
const localWorkflows = computed(() => {
  if (state.activeTab !== 'local') return [];

  return workflowStore.getWorkflows
    .filter(({ name }) =>
      name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
    )
    .sort((a, b) => (a.createdAt > b.createdAt ? -1 : 1));
});
const workflows = computed(() =>
  state.activeTab === 'local' ? localWorkflows.value : hostedWorkflows.value
);
const showTab = computed(
  () => hostedWorkflowStore.toArray.length > 0 || userStore.user?.teams
);

function executeWorkflow(workflow) {
  sendMessage('workflow:execute', workflow, 'background');
}
function updateWorkflow(id, data) {
  return workflowStore.update({
    id,
    data,
  });
}
function renameWorkflow({ id, name }) {
  dialog.prompt({
    title: t('home.workflow.rename'),
    placeholder: t('common.name'),
    okText: t('common.rename'),
    inputValue: name,
    onConfirm: (newName) => {
      updateWorkflow(id, { name: newName });
    },
  });
}
function deleteWorkflow({ id, name }) {
  dialog.confirm({
    title: t('home.workflow.delete'),
    okVariant: 'danger',
    body: t('message.delete', { name }),
    onConfirm: () => {
      if (state.activeTab === 'local') {
        workflowStore.delete(id);
      } else {
        hostedWorkflowStore.delete(id);
      }
    },
  });
}
function openDashboard(url) {
  sendMessage('open:dashboard', url, 'background');
}
async function initElementSelector() {
  const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
  const result = await browser.tabs.sendMessage(tab.id, {
    type: 'automa-element-selector',
  });

  if (!result) {
    await browser.tabs.executeScript(tab.id, {
      allFrames: true,
      file: './elementSelector.bundle.js',
    });
  }

  window.close();
}
async function recordWorkflow(options = {}) {
  try {
    const flows = [];
    const [activeTab] = await browser.tabs.query({
      active: true,
      currentWindow: true,
    });

    if (activeTab && activeTab.url.startsWith('http')) {
      flows.push({
        id: 'new-tab',
        description: activeTab.url,
        data: { url: activeTab.url },
      });
    }

    await browser.storage.local.set({
      isRecording: true,
      recording: {
        flows,
        name: 'unnamed',
        activeTab: {
          id: activeTab.id,
          url: activeTab.url,
        },
        ...options,
      },
    });
    await browser.browserAction.setBadgeBackgroundColor({ color: '#ef4444' });
    await browser.browserAction.setBadgeText({ text: 'rec' });

    const tabs = await browser.tabs.query({});
    for (const tab of tabs) {
      if (
        tab.url.startsWith('http') &&
        !tab.url.includes('chrome.google.com')
      ) {
        await browser.tabs.executeScript(tab.id, {
          allFrames: true,
          file: 'recordWorkflow.bundle.js',
        });
      }
    }

    window.close();
  } catch (error) {
    console.error(error);
  }
}
function openWorkflowPage({ id, hostId }) {
  let url = `/workflows/${id}`;

  if (state.activeTab === 'host') {
    url = `/workflows/${hostId}/host`;
  }

  openDashboard(url);
}
function onTabChange(value) {
  localStorage.setItem('popup-tab', value);
}

onMounted(async () => {
  const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
  state.haveAccess = /^(https?)/.test(tab.url);

  await userStore.loadUser({ storage: localStorage, ttl: 1000 * 60 * 5 });
  await teamWorkflowStore.loadData();

  let activeTab = localStorage.getItem('popup-tab') || 'local';

  if (activeTab === 'team' && !userStore.user?.teams) activeTab = 'local';
  else if (activeTab === 'host' && hostedWorkflowStore.toArray.length < 0)
    activeTab = 'local';

  state.retrieved = true;
  state.activeTab = activeTab;
});
</script>
<style>
.recording-card {
  transition: height 300ms cubic-bezier(0.4, 0, 0.2, 1) !important;
}
</style>
