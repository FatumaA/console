<script lang="ts">
    import { onMount } from 'svelte';
    import { goto, invalidate } from '$app/navigation';
    import { base } from '$app/paths';
    import { Button, Form, FormItem, FormList, InputDigits, InputText } from '$lib/elements/forms';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { Unauthenticated } from '$lib/layout';
    import { Dependencies } from '$lib/constants';
    import { Submit, trackEvent, trackError } from '$lib/actions/analytics';
    import { page } from '$app/stores';
    import { AuthenticationFactor, type Models } from '@appwrite.io/console';

    export let data;

    let code: string;
    let disabled: boolean;
    let challenge: Models.MfaChallenge;
    let challengeType: AuthenticationFactor;

    const factors = data.factors as Models.MfaFactors & { recoveryCode: boolean };
    const enabledFactors = Object.entries(factors).filter(([_, enabled]) => enabled);

    async function createChallenge(factor: AuthenticationFactor) {
        challengeType = factor;
        challenge = await sdk.forConsole.account.createMfaChallenge(factor);
    }

    onMount(async () => {
        const enabledNonRecoveryFactors = enabledFactors.filter(
            ([factor, _]) => factor != 'recoveryCode'
        );
        if (enabledNonRecoveryFactors.length == 1) {
            if (factors.phone) {
                createChallenge(AuthenticationFactor.Phone);
            } else if (factors.email) {
                createChallenge(AuthenticationFactor.Email);
            }
        }
    });

    async function back() {
        await sdk.forConsole.account.deleteSession('current');
        await goto(`${base}/`);
    }

    async function verify() {
        try {
            disabled = true;
            if (challenge == null) {
                await createChallenge(AuthenticationFactor.Totp);
            }
            await sdk.forConsole.account.updateMfaChallenge(challenge.$id, code);
            await invalidate(Dependencies.ACCOUNT);
            trackEvent(Submit.AccountCreate);
            if ($page.url.searchParams) {
                const redirect = $page.url.searchParams.get('redirect');
                $page.url.searchParams.delete('redirect');
                if (redirect) {
                    await goto(`${base}${redirect}${$page.url.search}`);
                } else {
                    await goto(`${base}/console${$page.url.search ?? ''}`);
                }
            } else {
                await goto(`${base}/console`);
            }
        } catch (error) {
            disabled = false;
            addNotification({
                type: 'error',
                message: error.message
            });
            trackError(error, Submit.AccountCreate);
        }
    }
</script>

<svelte:head>
    <title>Verify - Appwrite</title>
</svelte:head>

<Unauthenticated>
    <svelte:fragment slot="top">
        <div class="top u-flex u-position-absolute u-main-center">
            <div class="flex u-width-full-line">
                <Button text noMargin class="u-border-width-0" on:click={back}>
                    <span class="icon-cheveron-left u-font-size-20" aria-hidden="true" />
                    Back</Button>
            </div>
        </div>
    </svelte:fragment>
    <svelte:fragment slot="title">Verify your identity</svelte:fragment>
    <svelte:fragment>
        <Form onSubmit={verify}>
            <FormList>
                {#if factors.totp && (challengeType == AuthenticationFactor.Totp || challengeType == null)}
                    <p class="body-text-1">
                        Enter below a 6-digit one-time code generated by your authentication app.
                    </p>
                    <InputDigits bind:value={code} required autofocus />
                {:else if challengeType == AuthenticationFactor.Email}
                    <p class="body-text-1">
                        A 6-digit verification code was sent to your email, enter it below.
                    </p>
                    <InputDigits bind:value={code} required autofocus />
                {:else if challengeType == AuthenticationFactor.Phone}
                    <p class="body-text-1">
                        A 6-digit verification code was sent to your phone, enter it below.
                    </p>
                    <InputDigits bind:value={code} required autofocus />
                {:else if challengeType == AuthenticationFactor.Recoverycode}
                    <p class="body-text-1">
                        Enter below one of the recovery codes you received when enabling MFA for
                        this account.
                    </p>
                    <InputText id="recovery-code" bind:value={code} required autofocus />
                {/if}
                <FormItem>
                    <Button fullWidth submit {disabled}>Verify</Button>
                </FormItem>
                {#if enabledFactors.length > 1}
                    <span class="with-separators eyebrow-heading-3">or</span>
                    <div class="u-flex-vertical u-gap-8">
                        {#if factors.totp && challengeType != null && challengeType != AuthenticationFactor.Totp}
                            <FormItem>
                                <Button
                                    secondary
                                    fullWidth
                                    {disabled}
                                    on:click={() => createChallenge(AuthenticationFactor.Totp)}>
                                    <span
                                        class="icon-device-mobile u-font-size-20"
                                        aria-hidden="true" />
                                    <span class="text">Authenticator app</span>
                                </Button>
                            </FormItem>
                        {/if}
                        {#if factors.email && challengeType != AuthenticationFactor.Email}
                            <FormItem>
                                <Button
                                    secondary
                                    fullWidth
                                    {disabled}
                                    on:click={() => createChallenge(AuthenticationFactor.Email)}>
                                    <span class="icon-mail u-font-size-20" aria-hidden="true" />
                                    <span class="text">Email verification</span>
                                </Button>
                            </FormItem>
                        {/if}
                        {#if factors.phone && challengeType != AuthenticationFactor.Phone}
                            <FormItem>
                                <Button
                                    secondary
                                    fullWidth
                                    {disabled}
                                    on:click={() => createChallenge(AuthenticationFactor.Phone)}>
                                    <span class="icon-chat-alt u-font-size-20" aria-hidden="true" />
                                    <span class="text">Phone verification</span>
                                </Button>
                            </FormItem>
                        {/if}
                        {#if factors.recoveryCode && challengeType != AuthenticationFactor.Recoverycode}
                            <FormItem>
                                <Button
                                    text
                                    fullWidth
                                    {disabled}
                                    on:click={() =>
                                        createChallenge(AuthenticationFactor.Recoverycode)}>
                                    <span class="text">Use recovery code</span>
                                </Button>
                            </FormItem>
                        {/if}
                    </div>
                {/if}
            </FormList>
        </Form>
    </svelte:fragment>
</Unauthenticated>

<style lang="scss">
    @import '@appwrite.io/pink/src/abstract/variables/_devices.scss';
    .top {
        inset-block-start: 5.85rem;
        inline-size: 100%;
        max-inline-size: 27.5rem;

        div {
            max-inline-size: 27.5rem;
        }
    }

    @media (max-width: 440px) {
        .top {
            inset-block-start: 0rem;
            padding-inline: 0.5rem;
        }
    }

    /* for smaller screens */
    @media #{$break2open} {
        .top {
            inset-block-start: 5.85rem;
            padding-inline: 0;
        }
    }
</style>