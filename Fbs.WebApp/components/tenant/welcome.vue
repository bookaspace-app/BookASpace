<script setup lang="ts">
// For somebody new to booking: where to learn how, before they go and try. It goes once they have been through the walkthrough of the
// booking page, or have picked a time there, or say they don't want it. Admins have the setup checklist in its place.
const { path } = useTenant()
const { toured, picked } = useBookingHelpMemory()
const hidden = useLocalStorage('fbs:welcome-hidden', false)
const shown = computed(() => !hidden.value && !toured.value && !picked.value)

function hide() {
  hidden.value = true
}
</script>

<template>
  <section
    v-if="shown"
    aria-label="New here?"
    data-testid="welcome"
  >
    <UPageCard
      title="New here?"
      description="Select and drag time slots directly on the calendar to book facilities. Take a quick interactive tour to get started."
      icon="i-lucide-sparkles"
      variant="subtle"
    >
      <div class="flex flex-wrap gap-2">
        <UButton
          :to="{ path: path('book'), state: { tour: true } }"
          label="Walk me through it"
          icon="i-lucide-route"
          size="sm"
        />
        <UButton
          :to="path('guide')"
          label="How to book"
          icon="i-lucide-book-open"
          size="sm"
          color="neutral"
          variant="subtle"
        />
        <UButton
          label="Hide this"
          size="sm"
          color="neutral"
          variant="ghost"
          @click="hide"
        />
      </div>
    </UPageCard>
  </section>
</template>
