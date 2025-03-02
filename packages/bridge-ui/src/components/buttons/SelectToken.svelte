<script lang="ts">
  import { getProvider } from '@wagmi/core'
  import { token } from "../../store/token";
  import { bridgeType } from "../../store/bridge";
  import { ETH, tokens } from "../../domain/token";
  import type { Token } from "../../domain/token";
  import { BridgeType } from "../../domain/bridge";
  import { ChevronDown, PlusCircle } from "svelte-heros-v2";
  import { errorToast, successToast } from "../../utils/toast";
  import { ethers } from "ethers";
  import ERC20 from "../../constants/abi/ERC20";
  import { signer } from '../../store/signer';
  import { userTokens, tokenService } from '../../store/userToken';
  import { fromChain, toChain } from '../../store/chain';
  import Erc20 from '../icons/ERC20.svelte';
  import AddCustomErc20 from '../form/AddCustomERC20.svelte';

  let dropdownElement: HTMLDivElement;

  async function select(t: Token) {
    if (t === $token) return;
    token.set(t);
    if (t.symbol.toLowerCase() == ETH.symbol.toLowerCase()) {
      bridgeType.set(BridgeType.ETH);
    } else {
      bridgeType.set(BridgeType.ERC20);
    }
    successToast(`Token changed to ${t.symbol.toUpperCase()}`);

    // to close the dropdown on click
    dropdownElement?.classList.remove('dropdown-open');
    if (document.activeElement instanceof HTMLElement) {
      document.activeElement.blur();
    }
  }

  let showAddressField = false;

  let loading = false;

  async function addERC20(event: SubmitEvent) {
    try {
      loading = true;
      const eventTarget = event.target as HTMLFormElement;
      const { customTokenAddress } = eventTarget;
      const tokenAddress = customTokenAddress.value;
      if(!ethers.utils.isAddress(tokenAddress)) {
        throw new Error();
      }
      const provider = getProvider();
      const contract = new ethers.Contract(tokenAddress, ERC20, provider);
      
      const userAddress = await $signer.getAddress();
      const erc20Check = await contract.balanceOf(userAddress);
      const [tokenName, decimals, symbol] = await Promise.all([
        contract.name(),
        contract.decimals(),
        contract.symbol(),
      ])

      const token = {
        name: tokenName,
        addresses: [
          {
            chainId: $fromChain.id,
            address: tokenAddress,
          },
          {
            chainId: $toChain.id,
            address: "0x00",
          }
        ],
        decimals: decimals,
        symbol: symbol,
        logoComponent: null,
      }
      const updateTokensList = await $tokenService.StoreToken(token, userAddress);
      select(token);
      userTokens.set(updateTokensList);
      eventTarget.reset();
      showAddressField = false;
    } catch(error) {
      errorToast("Not a valid ERC20 address");
      console.error(error);
    } finally {
      loading = false;
    }
  }
  let customTokens = [];
  userTokens.subscribe(value => {
    if(value)
      customTokens = value;
  })
</script>

<div class="dropdown dropdown-bottom" bind:this={dropdownElement}>
  <label
    tabindex="0"
    class="flex items-center justify-center hover:cursor-pointer"
  >
    {#if $token.logoComponent}
      <svelte:component this={$token.logoComponent} />
    {:else}
      <Erc20 />
    {/if}
    <p class="px-2 text-sm">{$token.symbol.toUpperCase()}</p>
    <ChevronDown size='20' />
  </label>
  <ul
    tabindex="0"
    class="token-dropdown dropdown-content menu my-2 shadow-xl bg-dark-2 rounded-box p-2"
  >
    {#each tokens as t}
      <li class="cursor-pointer w-full hover:bg-dark-5 px-4 py-4 rounded-none">
        <button on:click={async () => await select(t)} class="flex hover:bg-dark-5">
          <svelte:component this={t.logoComponent} height={22} width={22} />
          <span class="text-sm font-medium bg-transparent px-2"
            >{t.symbol.toUpperCase()}</span
          >
        </button>
      </li>
    {/each}
    {#each customTokens as t}
      <li class="cursor-pointer w-full hover:bg-dark-5 px-4 py-4">
        <button on:click={async () => await select(t)} class="flex hover:bg-dark-5">
          <Erc20 height={22} width={22} />
          <span class="text-sm font-medium bg-transparent px-2"
            >{t.symbol.toUpperCase()}</span
          >
        </button>
      </li>
    {/each}
    <li class="cursor-pointer hover:bg-dark-5 px-4 py-4">
      <button on:click={() => showAddressField = true} class="flex hover:bg-dark-5 justify-between items-center">
        <PlusCircle size='25' />
        <span class="text-sm font-medium bg-transparent flex-1 w-[100px] px-0 pl-2"
          >Add Custom</span
        >
      </button>
    </li>
  </ul>

  {#if showAddressField}
    <AddCustomErc20 bind:showAddressField addERC20={addERC20} bind:loading />
  {/if}
</div>
