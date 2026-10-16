<script lang="ts">
  import { authManager } from '$lib/managers/auth-manager.svelte';
  import { serverConfigManager } from '$lib/managers/server-config-manager.svelte';
  import { OnboardingRole } from '$lib/types';
  import Logo from '$lib/components/shared-components/Logo.svelte';
  import { t } from 'svelte-i18n';

  let userRole = $derived(
    authManager.user.isAdmin && !serverConfigManager.value.isOnboarded ? OnboardingRole.SERVER : OnboardingRole.USER,
  );
</script>

<div class="gap-4">
  <Logo variant="icon" size="giant" class="mb-2" />
  <p class="mb-6 text-4xl font-medium text-primary sm:text-6xl">
    {$t('onboarding_welcome_user', { values: { user: authManager.user.name } })}
  </p>
  <p class="pb-6 text-2xl font-light sm:text-3xl">
    {userRole === OnboardingRole.SERVER
      ? $t('onboarding_server_welcome_description')
      : $t('onboarding_user_welcome_description')}
  </p>
</div>
