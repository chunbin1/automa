<template>
  <template v-if="retrieved">
    <app-sidebar />
    <main class="pl-16">
      <router-view />
    </main>
    <ui-dialog>
      <template #auth>
        <div class="text-center">
          <p class="font-semibold text-xl">Oops!! 😬</p>
          <p class="mt-2 text-gray-600 dark:text-gray-200">
            {{ t('auth.text') }}
          </p>
          <ui-button
            tag="a"
            href="https://www.automa.site/auth"
            class="mt-6 w-full block"
            variant="accent"
          >
            {{ t('auth.signIn') }}
          </ui-button>
        </div>
      </template>
    </ui-dialog>
    <div
      v-if="isUpdated"
      class="p-4 shadow-2xl z-50 fixed bottom-8 left-1/2 -translate-x-1/2 rounded-lg bg-accent text-white dark:text-gray-900 flex items-center"
    >
      <v-remixicon name="riInformationLine" class="mr-3" />
      <p>
        {{ t('updateMessage.text1', { version: currentVersion }) }}
      </p>
      <a
        :href="`https://github.com/AutomaApp/automa/releases/latest`"
        target="_blank"
        rel="noopener"
        class="underline ml-1"
      >
        {{ t('updateMessage.text2') }}
      </a>
      <button
        class="ml-6 text-gray-200 dark:text-gray-600"
        @click="isUpdated = false"
      >
        <v-remixicon size="20" name="riCloseLine" />
      </button>
    </div>
  </template>
  <div v-else class="py-8 text-center">
    <ui-spinner color="text-accent" size="28" />
  </div>
  <app-survey />
</template>
<script setup>
import iconFirefox from '@/assets/svg/logoFirefox.svg';
import iconChrome from '@/assets/svg/logo.svg';
import { ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRouter } from 'vue-router';
import { compare } from 'compare-versions';
import browser from 'webextension-polyfill';
import { useStore } from '@/stores/main';
import { useUserStore } from '@/stores/user';
import { useFolderStore } from '@/stores/folder';
import { useWorkflowStore } from '@/stores/workflow';
import { useTeamWorkflowStore } from '@/stores/teamWorkflow';
import { useTheme } from '@/composable/theme';
import { parseJSON } from '@/utils/helper';
import { useHostedWorkflowStore } from '@/stores/hostedWorkflow';
import { useSharedWorkflowStore } from '@/stores/sharedWorkflow';
import { loadLocaleMessages, setI18nLanguage } from '@/lib/vueI18n';
import { getUserWorkflows } from '@/utils/api';
import dbLogs from '@/db/logs';
import dayjs from '@/lib/dayjs';
import AppSurvey from '@/components/newtab/app/AppSurvey.vue';
import AppSidebar from '@/components/newtab/app/AppSidebar.vue';
import dataMigration from '@/utils/dataMigration';

let icon;
if (window.location.protocol === 'moz-extension:') {
  icon = iconFirefox;
} else {
  icon = iconChrome;
}

const iconElement = document.createElement('link');
iconElement.rel = 'icon';
iconElement.href = icon;
document.head.appendChild(iconElement);

const { t } = useI18n();
const store = useStore();
const theme = useTheme();
const router = useRouter();
const userStore = useUserStore();
const folderStore = useFolderStore();
const workflowStore = useWorkflowStore();
const teamWorkflowStore = useTeamWorkflowStore();
const sharedWorkflowStore = useSharedWorkflowStore();
const hostedWorkflowStore = useHostedWorkflowStore();

theme.init();

const retrieved = ref(false);
const isUpdated = ref(false);

const currentVersion = browser.runtime.getManifest().version;
const prevVersion = localStorage.getItem('ext-version') || '0.0.0';

async function fetchUserData() {
  try {
    if (!useRouter.user) return;

    const { backup, hosted } = await getUserWorkflows();
    userStore.hostedWorkflows = hosted || {};

    if (backup && backup.length > 0) {
      const { lastBackup } = browser.storage.local.get('lastBackup');
      if (!lastBackup) {
        const backupIds = backup.map(({ id }) => id);

        userStore.backupIds = backupIds;
        await browser.storage.local.set({
          backupIds,
          lastBackup: new Date().toISOString(),
        });
      }

      await workflowStore.insertOrUpdate(backup, { checkUpdateDate: true });
    }

    userStore.retrieved = true;
  } catch (error) {
    console.error(error);
  }
}
/* eslint-disable-next-line */
function autoDeleteLogs() {
  const deleteAfter = store.settings.deleteLogAfter;
  if (deleteAfter === 'never') return;

  const lastCheck =
    +localStorage.getItem('checkDeleteLogs') || Date.now() - 8.64e7;
  const dayDiff = dayjs().diff(dayjs(lastCheck), 'day');

  if (dayDiff < 1) return;

  const aDayInMs = 8.64e7;
  const maxLogAge = Date.now() - aDayInMs * deleteAfter;

  dbLogs.items
    .where('endedAt')
    .below(maxLogAge)
    .toArray()
    .then((values) => {
      const ids = values.map(({ id }) => id);

      dbLogs.items.bulkDelete(ids);
      dbLogs.ctxData.where('logId').anyOf(ids).delete();
      dbLogs.logsData.where('logId').anyOf(ids).delete();
      dbLogs.histories.where('logId').anyOf(ids).delete();

      localStorage.setItem('checkDeleteLogs', Date.now());
    });
}
async function syncHostedWorkflows() {
  const hostIds = [];
  const userHosted = userStore.getHostedWorkflows;
  const hostedWorkflows = hostedWorkflowStore.workflows;

  Object.keys(hostedWorkflows).forEach((hostId) => {
    const isItsOwn = userHosted.find((item) => item.hostId === hostId);
    if (isItsOwn) return;

    hostIds.push({ hostId, updatedAt: hostedWorkflows[hostId].updatedAt });
  });

  await hostedWorkflowStore.fetchWorkflows(hostIds);
}

window.addEventListener('storage', ({ key, newValue }) => {
  if (key !== 'workflowState') return;

  const states = parseJSON(newValue, {});
  workflowStore.states = Object.values(states).filter(
    ({ isDestroyed }) => !isDestroyed
  );
});
browser.runtime.onMessage.addListener(({ type, data }) => {
  if (type === 'workflow:added') {
    if (data.source === 'team') {
      teamWorkflowStore.loadData().then(() => {
        router.push(
          `/teams/${data.teamId}/workflows/${data.workflowId}?permission=true`
        );
      });
    } else {
      workflowStore.loadData().then(() => {
        router.push(`/workflows/${data.workflowId}?permission=true`);
      });
    }
  }
});

(async () => {
  try {
    const { isFirstTime } = await browser.storage.local.get('isFirstTime');
    isUpdated.value = !isFirstTime && compare(currentVersion, prevVersion, '>');

    await Promise.allSettled([
      folderStore.load(),
      store.loadSettings(),
      workflowStore.loadData(),
      teamWorkflowStore.loadData(),
      hostedWorkflowStore.loadData(),
    ]);

    await loadLocaleMessages(store.settings.locale, 'newtab');
    await setI18nLanguage(store.settings.locale);

    await dataMigration();
    await userStore.loadUser({ useCache: false, ttl: 2 });

    retrieved.value = true;

    await Promise.allSettled([
      sharedWorkflowStore.fetchWorkflows(),
      fetchUserData(),
      syncHostedWorkflows(),
    ]);

    autoDeleteLogs();
  } catch (error) {
    retrieved.value = true;
    console.error(error);
  }

  localStorage.setItem('ext-version', currentVersion);
})();
</script>
<style>
html,
body {
  @apply bg-gray-50 dark:bg-gray-900 text-black dark:text-gray-100;
}

body {
  min-height: 100vh;
}

#app {
  height: 100%;
}

h1,
h2,
h3 {
  @apply dark:text-white;
}
</style>
