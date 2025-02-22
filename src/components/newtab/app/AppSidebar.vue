<template>
  <aside
    class="fixed flex flex-col items-center h-screen left-0 top-0 w-16 py-6 bg-white dark:bg-gray-800 z-50"
  >
    <img
      :title="`v${extensionVersion}`"
      src="@/assets/svg/logo.svg"
      class="w-10 mb-4 mx-auto"
    />
    <div
      class="space-y-2 w-full relative text-center"
      @mouseleave="showHoverIndicator = false"
    >
      <div
        v-show="showHoverIndicator"
        ref="hoverIndicator"
        class="rounded-lg h-10 w-10 absolute left-1/2 bg-box-transparent transition-transform duration-200"
        style="transform: translate(-50%, 0)"
      ></div>
      <router-link
        v-for="tab in tabs"
        v-slot="{ href, navigate, isActive }"
        :key="tab.id"
        :to="tab.path"
        custom
      >
        <a
          v-tooltip:right.group="
            `${t(`common.${tab.id}`, 2)} (${tab.shortcut.readable})`
          "
          :class="{ 'is-active': isActive }"
          :href="href"
          class="z-10 relative w-full flex items-center justify-center tab relative"
          @click="navigate"
          @mouseenter="hoverHandler"
        >
          <div class="p-2 rounded-lg transition-colors inline-block">
            <v-remixicon :name="tab.icon" />
          </div>
          <span
            v-if="tab.id === 'log' && runningWorkflowsLen > 0"
            class="absolute h-4 w-4 text-xs dark:text-black text-white rounded-full bg-accent -top-1 right-2"
          >
            {{ runningWorkflowsLen }}
          </span>
        </a>
      </router-link>
    </div>
    <div class="flex-grow"></div>
    <ui-popover
      v-if="userStore.user"
      trigger="mouseenter click"
      placement="right"
    >
      <template #trigger>
        <span class="inline-block p-1 bg-box-transparent rounded-full">
          <img
            :src="userStore.user.avatar_url"
            height="32"
            width="32"
            class="rounded-full"
          />
        </span>
      </template>
      <div class="w-44">
        <div class="flex items-center">
          <p class="flex-1 text-overflow">
            {{ userStore.user.username }}
          </p>
          <span
            title="Subscription"
            :class="subColors[userStore.user.subscription]"
            class="px-2 py-1 rounded-md text-sm capitalize"
          >
            {{ userStore.user.subscription }}
          </span>
        </div>
        <p
          v-if="userStore.user.level"
          class="text-gray-600 dark:text-gray-200 mt-1"
        >
          Level: {{ userStore.user.level }}
        </p>
      </div>
    </ui-popover>
    <ui-popover trigger="mouseenter" placement="right" class="my-4">
      <template #trigger>
        <v-remixicon name="riGroupLine" />
      </template>
      <p class="mb-2">{{ t('home.communities') }}</p>
      <ui-list class="w-40">
        <ui-list-item
          v-for="item in communities"
          :key="item.name"
          :href="item.url"
          small
          tag="a"
          target="_blank"
          rel="noopener"
        >
          <v-remixicon :name="item.icon" class="mr-2" />
          {{ item.name }}
        </ui-list-item>
      </ui-list>
    </ui-popover>
    <router-link v-tooltip:right.group="t('settings.menu.about')" to="/about">
      <v-remixicon class="cursor-pointer" name="riInformationLine" />
    </router-link>
  </aside>
</template>
<script setup>
import { ref, computed } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';
import browser from 'webextension-polyfill';
import { useUserStore } from '@/stores/user';
import { useWorkflowStore } from '@/stores/workflow';
import { useShortcut, getShortcut } from '@/composable/shortcut';
import { useGroupTooltip } from '@/composable/groupTooltip';
import { communities } from '@/utils/shared';

useGroupTooltip();

const { t } = useI18n();
const router = useRouter();
const userStore = useUserStore();
const workflowStore = useWorkflowStore();

const extensionVersion = browser.runtime.getManifest().version;
const subColors = {
  free: 'bg-box-transparent',
  pro: 'bg-accent text-white',
  business: 'bg-accent text-white',
};
const tabs = [
  {
    id: 'workflow',
    icon: 'riFlowChart',
    path: '/workflows',
    shortcut: getShortcut('page:workflows', '/workflows'),
  },
  {
    id: 'schedule',
    icon: 'riTimeLine',
    path: '/schedule',
    shortcut: getShortcut('page:schedule', '/triggers'),
  },
  {
    id: 'storage',
    icon: 'riHardDrive2Line',
    path: '/storage',
    shortcut: getShortcut('page:storage', '/storage'),
  },
  {
    id: 'log',
    icon: 'riHistoryLine',
    path: '/logs',
    shortcut: getShortcut('page:logs', '/logs'),
  },
  {
    id: 'settings',
    icon: 'riSettings3Line',
    path: '/settings',
    shortcut: getShortcut('page:settings', '/settings'),
  },
];
const hoverIndicator = ref(null);
const showHoverIndicator = ref(false);
const runningWorkflowsLen = computed(() => workflowStore.states.length);

useShortcut(
  tabs.map(({ shortcut }) => shortcut),
  ({ data }) => {
    if (!data) return;

    router.push(data);
  }
);

function hoverHandler({ target }) {
  showHoverIndicator.value = true;
  hoverIndicator.value.style.transform = `translate(-50%, ${target.offsetTop}px)`;
}
</script>
<style scoped>
.tab.is-active:after {
  content: '';
  position: absolute;
  right: 0;
  top: 0;
  height: 100%;
  width: 4px;
  @apply bg-accent dark:bg-gray-100;
}
</style>
