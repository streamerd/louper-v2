<script lang="ts">
  import { fade } from 'svelte/transition'
  import Loading from '$lib/components/Loading.svelte'
  import type { Facet } from '../../types/entities'
  import { constants } from 'ethers'
  import { initWeb3W } from 'web3w'
  import { onDestroy } from 'svelte'
  import { NETWORKS } from '$lib/config'
  import { getExplorerTxUrl } from '../utils'
  import { utils } from 'ethers'

  export let facet: Facet | undefined = undefined
  export let address: string
  export let network: string
  export let showModal = false

  let error: any = null
  let args: any = {}

  const FacetCutAction = {
    Add: 0,
    Replace: 1,
    Remove: 2,
  }

  const { wallet, builtin, flow, transactions, chain } = initWeb3W({})

  const iface = new utils.Interface([
    'function diamondCut(tuple(address facetAddress, uint8 action, bytes4[] functionSelectors)[],address initAddress, bytes callData) external',
  ])

  $: if (facet) {
    args = [
      [
        {
          facetAddress: constants.AddressZero,
          action: FacetCutAction.Remove,
          functionSelectors: facet.methods.map((m) => m.selector),
        },
      ],
      constants.AddressZero,
      '0x',
    ]
  }

  const connect = async (option = 'builtin') => {
    try {
      await wallet.connect(option)
      await chain.updateContracts({
        chainId: NETWORKS[network].chainId,
        contracts: {
          facet: {
            address,
            abi: iface.fragments.map((f) => f),
          },
        },
      })
    } catch (e) {
      wallet.acknowledgeError()
      await wallet.disconnect()
    }
  }

  const closeModal = async () => {
    showModal = false
    error = null
    wallet.disconnect()
    $transactions.forEach((t) => transactions.acknowledge(t.hash, t.status))
    chainUnsub()
  }

  $: if ($flow.executionError) {
    error = $flow.executionError
    flow.cancel()
    wallet.acknowledgeError()
  }

  let chainUnsub = chain.subscribe(async (c) => {
    if (!$wallet.disconnecting && c.chainId && NETWORKS[network].chainId !== c.chainId) {
      await wallet.disconnect()
      alert(`Invalid network. Pleae connect to ${network}.`)
    }
  })
  onDestroy(() => {
    error = null
    chainUnsub()
  })
</script>

{#if showModal && facet}
  <div
    transition:fade={{ duration: 250 }}
    class="fixed mt-0 z-10 inset-0 overflow-y-auto flex items-center justify-center w-full h-full bg-black bg-opacity-75"
  >
    <!-- A basic modal dialog with title, body and one button to close -->
    <div
      class="h-auto text-left min-w-full fixed  md:min-w-0 md:w-1/2 rounded shadow-xl p-8 mx-12 bg-base-100 text-base-content"
    >
      <div class="mt-3 text-center sm:mt-0 sm:ml-4 sm:text-left">
        <h3 class="text-2xl font-medium leading-6 mb-5">
          Remove {facet.name}
        </h3>

        <div class="alert alert-error">
          <div class="flex-1">
            <span class="text-2xl mr-2">💀</span>
            <label for="">
              This is a BETA feature and may break your diamond contract. This will remove this
              facet and all selectors from this contract!
            </label>
          </div>
        </div>
      </div>

      <div class="container flex justify-center mt-5 gap-2">
        {#if $wallet.state !== 'Ready'}
          {#if $builtin.available}
            <button class="btn btn-sm glass bg-primary" on:click={() => connect()}>
              Connect
            </button>
          {/if}
          <button class="btn btn-sm glass bg-primary" on:click={() => connect('walletconnect')}>
            Connect w/ WalletConnect
          </button>
        {/if}
      </div>

      {#if $wallet.state === 'Ready'}
        <div class="mb-2">
          <p
            class="leading-5 bg-neutral-focus text-neutral-content w-full p-5 rounded-box overflow-auto"
          >
            {#if $wallet.pendingUserConfirmation}
              Please check and approve the transaction in your wallet.
            {/if}

            {#if $flow.inProgress}
              <div class="self-center">
                <Loading />
              </div>
            {/if}

            {#if error}
              <div class="self-center">
                <p class="text-red-500 font-semibold">
                  {error.message}
                </p>
              </div>
            {/if}
            {#each $transactions as transaction}
              TX Hash: <a
                class="text-info"
                href={getExplorerTxUrl(transaction.hash, network)}
                target="_blank"
              >
                {transaction.hash}
              </a>
              <span class="uppercase font-semibold">
                {transaction.status}
              </span>
            {/each}
          </p>
        </div>
      {/if}

      {#if $wallet.state === 'Ready'}
        <div class="flex justify-center">
          <button
            class="btn btn-xl glass bg-error"
            on:click={() => flow.execute((contracts) => contracts.facet['diamondCut'](...args))}
          >
            <svg
              class="w-6 h-6 mr-1"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
              />
            </svg>
            REMOVE
          </button>
        </div>
      {/if}

      <!-- One big close button.  --->
      <div class="mt-5 sm:mt-6">
        <div class="flex rounded-md w-full justify-center">
          <button class="btn btn-sm glass bg-primary" on:click={closeModal}>Close</button>
        </div>
      </div>
    </div>
  </div>
{/if}
