<script setup lang="ts">
import { useGetClaimsBySlug, useGetMe, usePostClaimsBySlugStart } from '~/api'

definePageMeta({
  layout: 'account',
})

useHead({ title: 'Take over your place' })

const route = useRoute()
const router = useRouter()
const toast = useToast()
const lastOrganization = useLocalStorage<string | null>('fbs:last-organization', null)

const slug = computed(() => String(route.params.slug))
const { data: claim, isPending, error } = useGetClaimsBySlug({ path: computed(() => ({ slug: slug.value })) }, { query: { retry: false } })

const link = ref<{ url: string, expiresAt: Date } | undefined>()
const { mutateAsync: start, isPending: starting } = usePostClaimsBySlugStart()

async function makeLink() {
  try {
    const started = await start({ path: { slug: slug.value } })
    link.value = { url: started.url, expiresAt: started.expiresAt }
  }
  catch {
    toast.add({ title: 'Unable to generate link', description: 'Please try again in a moment.', color: 'error' })
  }
}

// Once the link has been opened in Telegram, the place is theirs, and the app can go in
const waiting = computed(() => link.value !== undefined)
const { data: me } = useGetMe({ query: { refetchInterval: computed(() => waiting.value ? 3000 : false) } })
watch(me, async (value) => {
  if (waiting.value && value?.memberships.some(m => m.tenantSlug === slug.value && m.status === 'Active')) {
    lastOrganization.value = slug.value
    await router.push(`/t/${slug.value}`)
  }
})
</script>

<template>
  <div class="space-y-6">
    <div
      v-if="isPending"
      class="space-y-3"
    >
      <USkeleton class="h-24 w-full" />
    </div>

    <UPageCard
      v-else-if="error || !claim"
      title="There is nothing to take over here"
      description="This organization is no longer accepting claims, or you are already an active member."
      variant="subtle"
      icon="i-lucide-circle-slash"
    >
      <UButton
        to="/"
        label="Back to organizations"
        color="neutral"
        variant="subtle"
      />
    </UPageCard>

    <UPageCard
      v-else
      :title="`Take over your place in ${claim.organizationName}`"
      description="Your previous account was linked to your phone number. Open the link from the same Telegram account you used previously."
      variant="subtle"
      icon="i-lucide-user-check"
    >
      <template v-if="!link">
        <UButton
          label="Get my link"
          icon="i-lucide-link"
          :loading="starting"
          @click="makeLink"
        />
      </template>

      <template v-else>
        <div class="space-y-3">
          <UButton
            :to="link.url"
            target="_blank"
            label="Open in Telegram"
            icon="i-simple-icons-telegram"
          />
          <p class="text-sm text-muted">
            The link is valid for 10 minutes and can only be used once. Once verified in Telegram, you will be redirected automatically.
          </p>
          <p class="text-sm text-muted">
            If Telegram reports that your chat is not linked to any account, ensure you are signed in with the same Telegram account as before.
          </p>
        </div>
      </template>
    </UPageCard>
  </div>
</template>
