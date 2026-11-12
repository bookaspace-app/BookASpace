<script setup lang="ts">
import { useQueryClient } from '@tanstack/vue-query'
import { getMeQueryKey, useGetInvitesByToken, useGetMe, usePostInvitesByTokenAccept } from '~/api'

definePageMeta({
  layout: 'account',
})

useHead({ title: 'Join' })

const route = useRoute()
const router = useRouter()
const queryClient = useQueryClient()
const toast = useToast()
const lastOrganization = useLocalStorage<string | null>('fbs:last-organization', null)

const token = computed(() => String(route.params.token))
const { data: invite, isPending, error } = useGetInvitesByToken({ path: computed(() => ({ token: token.value })) }, { query: { retry: false } })
const { data: me } = useGetMe()

const displayName = ref('')
watch(me, (value) => {
  if (value && !displayName.value) {
    displayName.value = value.name ?? ''
  }
}, { immediate: true })

const outcome = ref<{ slug: string, status: string, name: string } | undefined>()
const removed = ref(false)
const full = ref(false)
const { mutateAsync: accept, isPending: joining } = usePostInvitesByTokenAccept()

async function join() {
  try {
    const joined = await accept({ path: { token: token.value }, body: { displayName: displayName.value.trim() || undefined } })
    await queryClient.invalidateQueries({ queryKey: getMeQueryKey() })
    if (joined.status === 'Active') {
      lastOrganization.value = joined.slug
      await router.push(`/t/${joined.slug}`)
    }
    else {
      outcome.value = { slug: joined.slug, status: joined.status, name: joined.organizationName }
    }
  }
  catch (e) {
    if (getErrorCodes(e).includes('removed')) {
      removed.value = true
    }
    else if (getErrorCodes(e).includes('member-limit')) {
      full.value = true
    }
    else if (getErrorStatus(e) === 404) {
      await queryClient.invalidateQueries({ queryKey: [{ url: '/Invites/:token' }] })
    }
    else {
      toast.add({ title: 'Couldn\'t join', description: 'Something went wrong. Try again.', color: 'error' })
    }
  }
}
</script>

<template>
  <div class="space-y-6">
    <UAlert
      v-if="full"
      title="This organization is full"
      description="The organization has reached its member limit. An admin needs to remove someone before new members can join."
      color="warning"
      variant="subtle"
      icon="i-lucide-users-round"
    />

    <UPageCard
      v-if="outcome"
      :title="`You asked to join ${outcome.name}`"
      description="An admin needs to approve your request before you can start booking. You can safely leave this page while waiting."
      variant="subtle"
      icon="i-lucide-hourglass"
    >
      <UButton
        to="/orgs"
        label="Back to organizations"
        color="neutral"
        variant="subtle"
      />
    </UPageCard>

    <UAlert
      v-else-if="removed"
      title="You've been removed from this organization"
      description="Ask an admin if you need access restored."
      color="warning"
      variant="subtle"
      icon="i-lucide-triangle-alert"
    />

    <div
      v-else-if="isPending"
      class="space-y-3"
    >
      <USkeleton class="h-24 w-full" />
    </div>

    <UPageCard
      v-else-if="error || !invite"
      title="This link doesn't work"
      description="This invite link has expired, was used up, or was stopped. Ask an admin for a new link."
      variant="subtle"
      icon="i-lucide-link-2-off"
    >
      <UButton
        to="/onboarding"
        label="Create an organization instead"
        color="neutral"
        variant="subtle"
      />
    </UPageCard>

    <UPageCard
      v-else
      :title="`Join ${invite.organizationName}`"
      :description="invite.requiresApproval ? 'An admin will need to approve you before you can start booking.' : 'You will be added to the organization right away.'"
      variant="subtle"
      icon="i-lucide-user-plus"
    >
      <form
        class="space-y-4"
        @submit.prevent="join"
      >
        <UFormField
          label="Your name"
          description="How your name will appear to other members. Defaults to your account name."
        >
          <UInput
            v-model="displayName"
            class="w-full"
            autocomplete="name"
          />
        </UFormField>

        <UButton
          type="submit"
          :label="invite.requiresApproval ? 'Ask to join' : 'Join'"
          icon="i-lucide-user-plus"
          :loading="joining"
        />
      </form>
    </UPageCard>
  </div>
</template>
