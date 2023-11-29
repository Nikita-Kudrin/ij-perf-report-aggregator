<template>
  <div class="flex flex-col gap-5">
    <Toolbar class="customToolbar">
      <template #start>
        <CopyLink :timerange-configurator="timeRangeConfigurator" />
        <TimeRangeSelect
          :ranges="timeRangeConfigurator.timeRanges"
          :value="timeRangeConfigurator.value.value"
          :on-change="onChangeRange"
        />
        <BranchSelect
          :branch-configurator="branchConfigurator"
          :triggered-by-configurator="triggeredByConfigurator"
        />
        <DimensionSelect
          label="Project"
          :value-to-label="getProjectName"
          :dimension="projectConfigurator"
        />
        <MachineSelect :machine-configurator="machineConfigurator" />
        <slot name="toolbar" />
      </template>
      <template #end>
        <PlotSettings @update:configurators="updateConfigurators" />
      </template>
    </Toolbar>
    <main class="flex">
      <div
        ref="container"
        class="flex flex-1 flex-col gap-5 overflow-hidden pt-5"
      >
        <Divider label="Bootstrap" />
        <section class="grid grid-cols-2 gap-x-6">
          <PerformanceLineChart
            :measures="['appInit_d', 'app initialization.end']"
            title="App Initialization"
            :configurators="configurators"
          />
          <PerformanceLineChart
            :measures="['bootstrap_d']"
            title="Bootstrap"
            :configurators="configurators"
            :with-measure-name="true"
          />
        </section>

        <section class="grid grid-cols-2 gap-x-6">
          <PerformanceLineChart
            :measures="['classLoadingPreparedCount', 'classLoadingLoadedCount']"
            title="Class Loading (Count)"
            :configurators="configurators"
          />
          <PerformanceLineChart
            :configurators="configurators"
            :measures="['editorRestoring']"
            title="Editor restoring"
            :with-measure-name="true"
          />
        </section>

        <span v-if="highlightingPasses">
          <Divider label="Highlighting Passes" />
          <PerformanceLineChart
            title="Highlighting Passes"
            :measures="highlightingPasses"
            :configurators="configurators"
          />
          <PerformanceLineChart
            title="Code Analysis"
            :measures="['metrics.codeAnalysisDaemon/fusExecutionTime', 'metrics.runDaemon/executionTime']"
            :configurators="configurators"
          />
        </span>

        <Divider label="Notifications" />
        <PerformanceLineChart
          title="Notifications"
          :measures="['metrics.notifications/number']"
          :skip-zero-values="false"
          :configurators="configurators"
        />

        <Divider label="Exit" />
        <PerformanceLineChart
          title="Exit Metrics"
          :measures="['metrics.exitMetrics/application.exit', 'metrics.exitMetrics/saveSettingsOnExit', 'metrics.exitMetrics/disposeProjects']"
          :configurators="configurators"
        />
      </div>
      <InfoSidebar />
    </main>
  </div>
</template>
<script setup lang="ts">
import { provide, ref } from "vue"
import { useRouter } from "vue-router"
import { createBranchConfigurator } from "../../configurators/BranchConfigurator"
import { dimensionConfigurator } from "../../configurators/DimensionConfigurator"
import { MachineConfigurator } from "../../configurators/MachineConfigurator"
import { privateBuildConfigurator } from "../../configurators/PrivateBuildConfigurator"
import { ServerConfigurator } from "../../configurators/ServerConfigurator"
import { TimeRange, TimeRangeConfigurator } from "../../configurators/TimeRangeConfigurator"
import { getDBType } from "../../shared/dbTypes"
import { configuratorListKey } from "../../shared/injectionKeys"
import { containerKey, serverConfiguratorKey, sidebarVmKey } from "../../shared/keys"
import DimensionSelect from "../charts/DimensionSelect.vue"
import PerformanceLineChart from "../charts/PerformanceLineChart.vue"
import CopyLink from "../settings/CopyLink.vue"
import PlotSettings from "../settings/PlotSettings.vue"
import { createProjectConfigurator, getProjectName } from "../startup/projectNameMapping"
import { fetchHighlightingPasses } from "../startup/utils"
import BranchSelect from "./BranchSelect.vue"
import Divider from "./Divider.vue"
import MachineSelect from "./MachineSelect.vue"
import { PersistentStateManager } from "./PersistentStateManager"
import TimeRangeSelect from "./TimeRangeSelect.vue"
import { DataQueryConfigurator } from "./dataQuery"
import { provideReportUrlProvider } from "./lineChartTooltipLinkProvider"
import { InfoSidebarImpl } from "./sideBar/InfoSidebar"

import { InfoDataPerformance } from "./sideBar/InfoSidebarPerformance"
import InfoSidebar from "./sideBar/InfoSidebarPerformance.vue"

interface StartupProductDashboard {
  product: string
  defaultProject: string
}

const props = defineProps<StartupProductDashboard>()
provideReportUrlProvider()

const container = ref<HTMLElement>()

const dbName = "ij"
const dbTable = "report"

const sidebarVm = new InfoSidebarImpl<InfoDataPerformance>(getDBType(dbName, dbTable))

provide(containerKey, container)
provide(sidebarVmKey, sidebarVm)

const serverConfigurator = new ServerConfigurator(dbName, dbTable)
provide(serverConfiguratorKey, serverConfigurator)
const persistentStateManager = new PersistentStateManager(
  `${props.product}-startup-dashboard`,
  {
    project: props.defaultProject,
    machine: "Windows Munich i7-3770, 32Gb",
    branch: "master",
  },
  useRouter()
)

const timeRangeConfigurator = new TimeRangeConfigurator(persistentStateManager)
const branchConfigurator = createBranchConfigurator(serverConfigurator, persistentStateManager, [timeRangeConfigurator])
const machineConfigurator = new MachineConfigurator(serverConfigurator, persistentStateManager, [timeRangeConfigurator, branchConfigurator])
const productConfigurator = dimensionConfigurator("product", serverConfigurator, persistentStateManager, false, [timeRangeConfigurator, branchConfigurator])
productConfigurator.selected.value = props.product
const projectConfigurator = createProjectConfigurator(productConfigurator, serverConfigurator, persistentStateManager, [
  productConfigurator,
  timeRangeConfigurator,
  branchConfigurator,
])
const triggeredByConfigurator = privateBuildConfigurator(serverConfigurator, persistentStateManager, [branchConfigurator, timeRangeConfigurator])
const configurators = [
  serverConfigurator,
  machineConfigurator,
  timeRangeConfigurator,
  productConfigurator,
  projectConfigurator,
  branchConfigurator,
  triggeredByConfigurator,
] as DataQueryConfigurator[]

provide(configuratorListKey, configurators)

const updateConfigurators = (configurator: DataQueryConfigurator) => {
  configurators.push(configurator)
}

function onChangeRange(value: TimeRange) {
  timeRangeConfigurator.value.value = value
}

const highlightingPasses = fetchHighlightingPasses()
</script>

<style>
.customToolbar {
  background-color: transparent;
  border: none;
  padding: 0;
}
</style>