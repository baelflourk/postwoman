<template>
  <div class="flex flex-col">
    <div v-if="fetching" class="flex justify-center">
      <HoppSmartSpinner />
    </div>

    <div v-if="team" class="flex flex-col">
      <div class="flex items-center space-x-4">
        <button
          class="p-2 rounded-3xl bg-divider hover:bg-dividerDark transition flex justify-center items-center"
          @click="router.push('/teams')"
        >
          <icon-lucide-arrow-left class="text-xl" />
        </button>
        <div class="flex justify-center items-center space-x-3">
          <h1 class="text-lg text-accentContrast">
            {{ team.name }}
          </h1>
          <span>/</span>
          <h2 class="text-lg text-accentContrast">
            {{ currentTabName }}
          </h2>
        </div>
      </div>

      <div class="py-8">
        <HoppSmartTabs v-model="selectedOptionTab" render-inactive-tabs>
          <HoppSmartTab :id="'details'" label="Details">
            <TeamsDetails
              :team="team"
              :teamName="teamName"
              v-model:showRenameInput="showRenameInput"
              @rename-team="renameTeamName"
              @delete-team="deleteTeam"
              class="py-8 px-4"
            />
          </HoppSmartTab>
          <HoppSmartTab :id="'members'" label="Members">
            <TeamsMembers @update-team="updateTeam()" class="py-8 px-4" />
          </HoppSmartTab>
          <HoppSmartTab :id="'invites'" label="Invites">
            <TeamsPendingInvites :editingTeamID="team.id" class="py-8 px-4" />
          </HoppSmartTab>
        </HoppSmartTabs>

        <HoppSmartConfirmModal
          :show="confirmDeletion"
          :title="`Confirm Deletion of ${team.name} team?`"
          @hide-modal="confirmDeletion = false"
          @resolve="deleteTeamMutation(deleteTeamUID)"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useClientHandle, useMutation } from '@urql/vue';
import { computed, onMounted, ref, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useToast } from '../../composables/toast';
import {
  RemoveTeamDocument,
  RenameTeamDocument,
  TeamInfoDocument,
  TeamMemberRole,
  TeamInfoQuery,
} from '../../helpers/backend/graphql';
import { HoppSmartTabs } from '@hoppscotch/ui';

const toast = useToast();

type OptionTabs = 'details' | 'members' | 'invites';

const selectedOptionTab = ref<OptionTabs>('details');

const currentTabName = computed(() => {
  switch (selectedOptionTab.value) {
    case 'details':
      return 'Team details';
    case 'members':
      return 'Team members';
    case 'invites':
      return 'Pending invites';
    default:
      return '';
  }
});

// Get the details of the team
const team = ref<TeamInfoQuery['admin']['teamInfo'] | undefined>();
const teamName = ref('');
const route = useRoute();
const fetching = ref(true);
const { client } = useClientHandle();

const getTeamInfo = async () => {
  fetching.value = true;
  const result = await client
    .query(TeamInfoDocument, { teamID: route.params.id.toString() })
    .toPromise();
  if (result.error) {
    return toast.error('Unable to load team info..');
  }
  if (result.data?.admin.teamInfo) {
    team.value = result.data.admin.teamInfo;
    teamName.value = team.value.name;
  }
  fetching.value = false;
};

onMounted(async () => await getTeamInfo());

const updateTeam = async () => await getTeamInfo();

// Rename the team name
const showRenameInput = ref(false);
const teamRename = useMutation(RenameTeamDocument);

const renameTeamName = async (teamName: string) => {
  if (!team.value) return;

  if (team.value.name === teamName) {
    showRenameInput.value = false;
    return;
  }
  const variables = { uid: team.value.id, name: teamName };
  await teamRename.executeMutation(variables).then((result) => {
    if (result.error) {
      toast.error('Failed to rename team!!');
    } else {
      showRenameInput.value = false;
      if (team.value) {
        team.value.name = teamName;
        toast.success('Team renamed successfully!!');
      }
    }
  });
};

// Delete team from the infra
const router = useRouter();
const confirmDeletion = ref(false);
const teamDeletion = useMutation(RemoveTeamDocument);
const deleteTeamUID = ref<string | null>(null);

const deleteTeam = (id: string) => {
  confirmDeletion.value = true;
  deleteTeamUID.value = id;
};

const deleteTeamMutation = async (id: string | null) => {
  if (!id) {
    confirmDeletion.value = false;
    toast.error('Team deletion failed!!');
    return;
  }
  const variables = { uid: id };
  await teamDeletion.executeMutation(variables).then((result) => {
    if (result.error) {
      toast.error('Team deletion failed!!');
    } else {
      toast.success('Team deleted successfully!!');
    }
  });
  confirmDeletion.value = false;
  deleteTeamUID.value = null;
  router.push('/teams');
};

// Update Roles of Members
const roleUpdates = ref<
  {
    userID: string;
    role: TeamMemberRole;
  }[]
>([]);

watch(
  () => team.value,
  (teamDetails) => {
    const members = teamDetails?.teamMembers ?? [];

    // Remove deleted members
    roleUpdates.value = roleUpdates.value.filter(
      (update) =>
        members.findIndex(
          (y: { user: { uid: string } }) => y.user.uid === update.userID
        ) !== -1
    );
  }
);
</script>
