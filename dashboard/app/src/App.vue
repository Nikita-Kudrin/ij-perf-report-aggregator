<template>
  <router-view v-slot="{ Component, route }">
    <ScrollTop />
    <template v-if="route.path.startsWith('/degradation') || route.path.startsWith('/metrics') || route.path.startsWith('/bisect')">
      <keep-alive
        :key="route.path"
        max="4"
      >
        <component :is="Component" />
      </keep-alive>
    </template>
    <template v-else>
      <PageLayout>
        <keep-alive
          :key="route.path"
          max="4"
        >
          <component :is="Component" />
        </keep-alive>
      </PageLayout>
    </template>
  </router-view>
  <Toast />
</template>
<script setup lang="ts">
import Toast from "primevue/toast"
import PageLayout from "new-dashboard/src/PageLayout.vue"
import { PersistentStateManager } from "new-dashboard/src/components/common/PersistentStateManager"
import { ServerWithCompressConfigurator } from "new-dashboard/src/configurators/ServerWithCompressConfigurator"
import { limit, refToObservable } from "new-dashboard/src/configurators/rxjs"
import { serverUrlObservableKey } from "new-dashboard/src/shared/injectionKeys"
import { filter, shareReplay } from "rxjs"
import { provide, shallowRef, watch } from "vue"
import { useRoute } from "vue-router"

const serverUrl = shallowRef(ServerWithCompressConfigurator.DEFAULT_SERVER_URL)
// shallow ref doesn't work - items are modified by primevue
const serverUrlObservable = refToObservable(serverUrl).pipe(
  filter((it: string | null): it is string => it !== null && it.length > 0),
  shareReplay(1)
)
provide(serverUrlObservableKey, serverUrlObservable)

const activePath = shallowRef("")
const _route = useRoute()
watch(
  () => _route.path,
  (p) => {
    activePath.value = p
    limit.clearQueue()
  }
)

const persistentStateManager = new PersistentStateManager("common", { serverUrl: ServerWithCompressConfigurator.DEFAULT_SERVER_URL })
persistentStateManager.add("serverUrl", serverUrl)
</script>
