<script lang="ts">
  import { chainId, connected, selectedAccount } from 'svelte-web3';

  import { AmountRangeInput, Button } from '$components';

  import { NotificationType, showNotification } from '$lib/notification';
  import {
    connectToWallet,
    withdrawStateStore,
    addWSMRToMetamask,
  } from '$lib/withdraw';
    import { GAS_PRICE } from '$lib/wsmr';
    import { L2_NATIVE_GAS_TOKEN_DECIMALS, wSMR_TOKEN_DECIMALS } from '$lib/constants';

  type WrapFormInput = {
    smrTokensToWrap: number;
    wsmrTokensToUnwrap: number;
  }
  const formInput: WrapFormInput = {
    smrTokensToWrap: 0,
    wsmrTokensToUnwrap: 0,
  };

  let isWraping: boolean = false;
  let isUnwraping: boolean = false;
  let canSetAmountToWrap = true;
  let canSetAmountToUnwrap = true;
  let estimatedTxFee: number = 0;
  let balanceWSMR: number = 0;

  $: updateCanWrap($withdrawStateStore.availableBaseTokens);
  $: formattedBalanceSMR = (
    $withdrawStateStore.availableBaseTokens /
    10 ** L2_NATIVE_GAS_TOKEN_DECIMALS
  ).toFixed(6);
  $: canWrap =
    $withdrawStateStore.availableBaseTokens > 0 &&
    formInput.smrTokensToWrap > 0;
  $: canUnwrap =
    balanceWSMR > 0 &&
    formInput.wsmrTokensToUnwrap > 0;
  $: $withdrawStateStore.isMetamaskConnected = window.ethereum
    ? window.ethereum.isMetamaskConnected
    : false;

  $: $withdrawStateStore, updateFormInput();

  function updateFormInput() {
    if (formInput.smrTokensToWrap > $withdrawStateStore.availableBaseTokens) {
      formInput.smrTokensToWrap = 0;
    }
  }

  async function updateCanWrap(_baseTokens: number,) {
    if (!$withdrawStateStore.wsmrContractObj) {
      return (canSetAmountToWrap = false);
    }

    try {
      const estimatedGas = await ($withdrawStateStore.wsmrContractObj as any).estimateGasDeposit(0);
      estimatedTxFee = +estimatedGas.toString() * GAS_PRICE;
      estimatedTxFee += 0.01; // to avoid not enough txFee
      estimatedTxFee *= 10 ** L2_NATIVE_GAS_TOKEN_DECIMALS;

      canSetAmountToWrap = $withdrawStateStore.availableBaseTokens > estimatedTxFee;
      balanceWSMR = await ($withdrawStateStore.wsmrContractObj as any).balanceOf($selectedAccount);
      balanceWSMR = Number(balanceWSMR);
      balanceWSMR -= 1000000000000; // to avoid exceed the balance due to rounding
      canSetAmountToUnwrap = balanceWSMR > 0;
    } catch (ex) {
      console.log('updateCanWrap - Error:', ex);
      canSetAmountToWrap = false;
      canSetAmountToUnwrap = false;
    }
  }

  async function wrapOrUnwrap(
    smrTokens: number,
    wsmrTokens: number,
  ) {
    if (!$selectedAccount) {
      return;
    }

    let result: any;

    const wrapText = smrTokens > 0 ? 'wrap' : 'unwrap';

    try {
      smrTokens > 0 ? isWraping = true : isUnwraping = true;
      result = smrTokens > 0 ? await $withdrawStateStore.wsmrContractObj.deposit(
        smrTokens,
      ) : await $withdrawStateStore.wsmrContractObj.withdraw(
        BigInt(wsmrTokens),
      );
    } catch (ex) {
      smrTokens > 0 ? isWraping = false : isUnwraping = false;
      showNotification({
        type: NotificationType.Error,
        message: `Failed to send ${wrapText} request: ${ex?.data?.message || ex?.message}`,
        duration: 8000,
      });
      return;
    }

    if (result.status) {
      showNotification({
        type: NotificationType.Success,
        message: `${wrapText} request sent. Transaction: ${result.transactionHash}`,
        duration: 4000,
      });
    } else {
      showNotification({
        type: NotificationType.Error,
        message: `Failed to send ${wrapText} request: ${JSON.stringify(
          result,
          null,
          4,
        )}`,
        duration: 8000,
      });
    }
    isWraping = false;
    isUnwraping = false;
    smrTokens > 0 ? formInput.smrTokensToWrap = 0 : formInput.wsmrTokensToUnwrap = 0;
  }

  async function onWrapClick() {
    await wrapOrUnwrap(
      formInput.smrTokensToWrap,
      0
    );
  }

  async function onUnwrapClick() {
    await wrapOrUnwrap(
      0,
      formInput.wsmrTokensToUnwrap,
    );
  }

</script>

<withdraw-component class="flex flex-col space-y-6 mt-6">
  {#if !$connected && !$selectedAccount}
    <div class="input_container">
      <button on:click={connectToWallet}>Connect to Wallet</button>
    </div>
  {:else if !$withdrawStateStore.isLoading}
    <info-box>
      <div class="flex flex-col space-y-2">
        <info-item-title>Chain ID</info-item-title>
        <info-item-value>{$chainId}</info-item-value>
      </div>
      <div class="flex flex-col space-y-2">
        <info-item-title>SMR Balance</info-item-title>
        <info-item-value>{formattedBalanceSMR}</info-item-value>
      </div>
      <div class="flex flex-col space-y-2">
        <info-item-title>wSMR Balance</info-item-title>
        <info-item-value>{(balanceWSMR / 10 ** wSMR_TOKEN_DECIMALS).toFixed(6)}</info-item-value>
      </div>
    </info-box>
    <info-box>
      <div class="flex flex-col space-y-2">
        <tokens-to-send-wrapper>
          <div class="mb-2">Wrap SMR</div>
          <info-box class="flex flex-col space-y-4 max-h-96 overflow-auto">
            <AmountRangeInput
              label="SMR Token:"
              bind:value={formInput.smrTokensToWrap}
              disabled={!canSetAmountToWrap}
              min={estimatedTxFee}
              max={Math.max(
                $withdrawStateStore.availableBaseTokens - estimatedTxFee,
                0,
              )}
              decimals={L2_NATIVE_GAS_TOKEN_DECIMALS}
            />
          </info-box>
        </tokens-to-send-wrapper>
      </div>
      <div class="flex flex-col space-y-2">
        <tokens-to-send-wrapper>
          <div class="mb-2">
            Unwrap wSMR {' '} &nbsp;
            <button on:click={addWSMRToMetamask}>
              <img src="/metamask-logo.svg" alt="Metamask logo" width="70%" />
            </button>
          </div>
          <info-box class="flex flex-col space-y-4 max-h-96 overflow-auto">
            <AmountRangeInput
              label="wSMR Token:"
              bind:value={formInput.wsmrTokensToUnwrap}
              disabled={!canSetAmountToUnwrap}
              min={0}
              max={Math.max(
                balanceWSMR,
                0,
              )}
              decimals={L2_NATIVE_GAS_TOKEN_DECIMALS}
            />
          </info-box>
        </tokens-to-send-wrapper>
      </div>
    </info-box>
    <info-box>
      <div class="flex flex-col space-y-2">
        <Button
          title="Wrap"
          onClick={onWrapClick}
          disabled={!canWrap}
          busy={isWraping}
        />
      </div>
      <div class="flex flex-col space-y-2">
        <Button
          title="Unwrap"
          onClick={onUnwrapClick}
          disabled={!canUnwrap}
          busy={isUnwraping}
          ghost
        />
      </div>
    </info-box>
  {/if}
</withdraw-component>

<style>
  info-box {
    @apply w-full;
    @apply flex;
    @apply justify-between;
    @apply bg-shimmer-background-tertiary;
    @apply rounded-xl;
    @apply p-4;
  }
  info-item-title {
    @apply text-xs;
    @apply text-shimmer-text-secondary;
  }

  info-item-value {
    @apply text-2xl;
  }
</style>
