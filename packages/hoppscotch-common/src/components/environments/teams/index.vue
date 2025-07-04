<template>
  <div>
    <input
      v-model="filterText"
      type="search"
      autocomplete="off"
      class="flex w-full bg-transparent px-4 py-2 h-8 border-b border-dividerLight"
      :placeholder="t('action.search')"
      :disabled="loading"
    />
    <div
      class="sticky top-upperPrimaryStickyFold z-10 flex flex-1 flex-shrink-0 justify-between overflow-x-auto border-b border-dividerLight bg-primary"
    >
      <HoppButtonSecondary
        v-if="team === undefined || team.role === 'VIEWER'"
        v-tippy="{ theme: 'tooltip' }"
        disabled
        class="!rounded-none"
        :icon="IconPlus"
        :title="t('team.no_access')"
        :label="t('action.new')"
      />
      <HoppButtonSecondary
        v-else
        :icon="IconPlus"
        :label="`${t('action.new')}`"
        class="!rounded-none"
        @click="displayModalAdd(true)"
      />
      <div class="flex">
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          to="https://docs.hoppscotch.io/documentation/features/environments"
          blank
          :title="t('app.wiki')"
          :icon="IconHelpCircle"
        />
        <HoppButtonSecondary
          v-if="team !== undefined && team.role === 'VIEWER'"
          v-tippy="{ theme: 'tooltip' }"
          disabled
          :icon="IconImport"
          :title="t('modal.import_export')"
        />
        <HoppButtonSecondary
          v-else
          v-tippy="{ theme: 'tooltip' }"
          :icon="IconImport"
          :title="t('modal.import_export')"
          @click="displayModalImportExport(true)"
        />
      </div>
    </div>

    <div v-if="loading" class="flex flex-col items-center justify-center p-4">
      <HoppSmartSpinner class="my-4" />
      <span class="text-secondaryLight">{{ t("state.loading") }}</span>
    </div>

    <div v-else-if="adapterError" class="flex flex-col items-center py-4">
      <icon-lucide-help-circle class="svg-icons mb-4" />
      {{ t(getEnvActionErrorMessage(adapterError)) }}
    </div>

    <HoppSmartPlaceholder
      v-else-if="filteredAndAlphabetizedTeamEnvs.length === 0"
      :alt="
        filterText
          ? `${t('empty.search_environment')}`
          : t('empty.environments')
      "
      :text="
        filterText
          ? `${t('empty.search_environment')} '${filterText}'`
          : t('empty.environments')
      "
      :src="
        filterText
          ? undefined
          : `/images/states/${colorMode.value}/blockchain.svg`
      "
    >
      <template v-if="filterText" #icon>
        <icon-lucide-search class="svg-icons opacity-75" />
      </template>

      <template v-else #body>
        <div class="flex flex-col items-center space-y-4">
          <span class="text-center text-secondaryLight">
            {{ t("environment.import_or_create") }}
          </span>
          <div class="flex flex-col items-stretch gap-4">
            <HoppButtonPrimary
              :icon="IconImport"
              :label="t('import.title')"
              filled
              outline
              :title="isTeamViewer ? t('team.no_access') : ''"
              :disabled="isTeamViewer"
              @click="isTeamViewer ? null : displayModalImportExport(true)"
            />
            <HoppButtonSecondary
              :label="`${t('add.new')}`"
              filled
              outline
              :icon="IconPlus"
              :title="isTeamViewer ? t('team.no_access') : ''"
              :disabled="isTeamViewer"
              @click="isTeamViewer ? null : displayModalAdd(true)"
            />
          </div>
        </div>
      </template>
    </HoppSmartPlaceholder>

    <div v-else>
      <EnvironmentsTeamsEnvironment
        v-for="{ env, index } in JSON.parse(
          JSON.stringify(filteredAndAlphabetizedTeamEnvs)
        )"
        :key="`environment-${index}`"
        :environment="env"
        :is-viewer="team?.role === 'VIEWER'"
        :selected="isEnvironmentSelected(env.id)"
        @edit-environment="editEnvironment(env)"
        @select-environment="selectEnvironment(env)"
        @show-environment-properties="
          showEnvironmentProperties(env.environment.id)
        "
      />
    </div>

    <EnvironmentsTeamsDetails
      :show="showModalDetails"
      :action="action"
      :editing-environment="editingEnvironment"
      :editing-team-id="team?.teamID"
      :editing-variable-name="editingVariableName"
      :is-secret-option-selected="secretOptionSelected"
      :is-viewer="team?.role === 'VIEWER'"
      @hide-modal="displayModalEdit(false)"
    />
    <EnvironmentsImportExport
      v-if="showModalImportExport"
      :team-environments="filteredAndAlphabetizedTeamEnvs.map(({ env }) => env)"
      :team-id="team?.teamID"
      environment-type="TEAM_ENV"
      @hide-modal="displayModalImportExport(false)"
    />
    <EnvironmentsProperties
      v-if="showEnvironmentsPropertiesModal"
      v-model="environmentsPropertiesModalActiveTab"
      :environment-i-d="selectedEnvironmentID!"
      @hide-modal="showEnvironmentsPropertiesModal = false"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from "vue"
import { GQLError } from "~/helpers/backend/GQLClient"
import { TeamEnvironment } from "~/helpers/teams/TeamEnvironment"
import { useI18n } from "~/composables/i18n"
import { useColorMode } from "~/composables/theming"
import IconPlus from "~icons/lucide/plus"
import IconHelpCircle from "~icons/lucide/help-circle"
import IconImport from "~icons/lucide/folder-down"
import { defineActionHandler } from "~/helpers/actions"
import { TeamWorkspace } from "~/services/workspace.service"
import { sortTeamEnvironmentsAlphabetically } from "~/helpers/utils/sortEnvironmentsAlphabetically"
import { getEnvActionErrorMessage } from "~/helpers/error-messages"
import { HandleEnvChangeProp } from "../index.vue"
import { selectedEnvironmentIndex$ } from "~/newstore/environments"
import { useReadonlyStream } from "~/composables/stream"

const t = useI18n()

const colorMode = useColorMode()

const props = defineProps<{
  team: TeamWorkspace | undefined
  teamEnvironments: TeamEnvironment[]
  adapterError: GQLError<string> | null
  loading: boolean
}>()

const emit = defineEmits<{
  (e: "select-environment", data: HandleEnvChangeProp): void
}>()

const filterText = ref("")

// Sort environments alphabetically by default and filter by search text
const filteredAndAlphabetizedTeamEnvs = computed(() => {
  const envs = sortTeamEnvironmentsAlphabetically(props.teamEnvironments, "asc")
  const rawFilter = filterText.value

  // Ensure specifying whitespace characters alone result in the empty state for no search results
  const trimmedFilter = rawFilter.trim().toLowerCase()

  // Whitespace-only input results in an empty state
  if (rawFilter && !trimmedFilter) return []

  // No search text → Show all environments
  if (!trimmedFilter) return envs

  // Filter environments based on search text
  return envs.filter(({ env }) =>
    env.environment.name.toLowerCase().includes(trimmedFilter)
  )
})

const showModalImportExport = ref(false)
const showModalDetails = ref(false)
const action = ref<"new" | "edit">("edit")
const editingEnvironment = ref<TeamEnvironment | null>(null)
const editingVariableName = ref("")
const secretOptionSelected = ref(false)

const showEnvironmentsPropertiesModal = ref(false)
const environmentsPropertiesModalActiveTab = ref("details")
const selectedEnvironmentID = ref<string | null>(null)

const isTeamViewer = computed(() => props.team?.role === "VIEWER")

const displayModalAdd = (shouldDisplay: boolean) => {
  action.value = "new"
  showModalDetails.value = shouldDisplay
}
const displayModalEdit = (shouldDisplay: boolean) => {
  action.value = "edit"
  showModalDetails.value = shouldDisplay

  if (!shouldDisplay) resetSelectedData()
}
const displayModalImportExport = (shouldDisplay: boolean) => {
  showModalImportExport.value = shouldDisplay
}
const editEnvironment = (environment: TeamEnvironment | null) => {
  editingEnvironment.value = environment
  action.value = "edit"
  displayModalEdit(true)
}
const resetSelectedData = () => {
  editingEnvironment.value = null
  editingVariableName.value = ""
  secretOptionSelected.value = false
}

const showEnvironmentProperties = (environmentID: string) => {
  showEnvironmentsPropertiesModal.value = true
  selectedEnvironmentID.value = environmentID
}

const selectEnvironment = (environment: TeamEnvironment) => {
  emit("select-environment", {
    index: 1,
    env: {
      type: "team-environment",
      environment,
    },
  })
}

const selectedEnvironmentIndex = useReadonlyStream(selectedEnvironmentIndex$, {
  type: "NO_ENV_SELECTED",
})

const isEnvironmentSelected = (id: string) => {
  return (
    selectedEnvironmentIndex.value.type === "TEAM_ENV" &&
    selectedEnvironmentIndex.value.teamEnvID === id
  )
}

defineActionHandler(
  "modals.team.environment.edit",
  ({ envName, variableName, isSecret }) => {
    if (variableName) editingVariableName.value = variableName
    const teamEnvToEdit = filteredAndAlphabetizedTeamEnvs.value.find(
      ({ env }) => env.environment.name === envName
    )
    if (teamEnvToEdit) {
      const { env } = teamEnvToEdit
      editEnvironment(env)
      secretOptionSelected.value = isSecret ?? false
    }
  }
)
</script>
