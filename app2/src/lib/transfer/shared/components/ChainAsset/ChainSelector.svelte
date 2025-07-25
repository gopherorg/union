<script lang="ts">
import { chainLogoMap } from "$lib/constants/chain-logos.ts"
import { DISABLED_CHAINS } from "$lib/constants/disabled-chains.ts"
import { tokensStore } from "$lib/stores/tokens.svelte.ts"
import { uiStore } from "$lib/stores/ui.svelte.ts"
import { transferData } from "$lib/transfer/shared/data/transfer-data.svelte.ts"
import { signingMode } from "$lib/transfer/signingMode.svelte"
import { cn } from "$lib/utils"
import type { Chain, Token, TokenWrapping } from "@unionlabs/sdk/schema"
import { Match, Option, pipe, Tuple } from "effect"

type Props = {
  type: "source" | "destination"
  onSelect: () => void
}

const { type, onSelect }: Props = $props()

type ChainWithAvailability = ReturnType<typeof Tuple.make<[Chain, boolean]>>

const updateSelectedChain = (chain: Chain) => {
  pipe(
    Match.value(type).pipe(
      Match.when("destination", () => {
        if (chain.chain_id === transferData.raw.source) {
          return
        }
        transferData.raw.updateField(type, chain.chain_id)
      }),
      Match.when("source", () => {
        transferData.raw.updateField(type, chain.chain_id)
        if (transferData.raw.destination === chain.chain_id) {
          transferData.raw.updateField("destination", "")
        }
      }),
      Match.exhaustive,
    ),
  )
  onSelect()
}

const filterBySigningMode = (chains: Array<Chain>) =>
  signingMode.mode === "multi" && type === "source"
    ? chains.filter(chain => chain.rpc_type === "cosmos")
    : chains

const isValidRoute = (chain: Chain) =>
  type === "source"
  || pipe(
    transferData.destinationChains,
    Option.map(goodXs => goodXs.map(x => x.chain_id).includes(chain.chain_id)),
    Option.getOrElse(() => false),
  )

const getChainStatus = (chain: Chain, hasBucket: boolean) => {
  const isSourceChain = type === "destination" && transferData.raw.source === chain.chain_id
  const isDisabledChain = chain.universal_chain_id
    && DISABLED_CHAINS.includes(chain.universal_chain_id)

  return pipe(
    Match.value(type).pipe(
      Match.when("source", () => ({
        isSelected: transferData.raw.source === chain.chain_id,
        isSourceChain: false,
        isDisabled: isDisabledChain,
        hasBucket,
        hasRoute: true,
      })),
      Match.when("destination", () => ({
        isSelected: transferData.raw.destination === chain.chain_id,
        isSourceChain,
        isDisabled: isSourceChain || !isValidRoute(chain) || !hasBucket || isDisabledChain,
        hasBucket,
        hasRoute: isValidRoute(chain),
      })),
      Match.exhaustive,
    ),
  )
}

const findTokenWithBucket = (
  tokenList: ReadonlyArray<Token>,
  predicate: (token: Token) => boolean,
) =>
  pipe(
    tokenList.find(predicate),
    Option.fromNullable,
    Option.map(token => token.bucket != null),
    Option.getOrElse(() => false),
  )

const hasTokenBucket = (
  destinationChain: Chain,
  tokenList: ReadonlyArray<Token>,
  baseToken: Token,
  sourceChain: Chain,
) => {
  const baseDenom = baseToken.denom.toLowerCase()

  const maybeUnwrapped = baseToken.wrapping.find(
    (w: TokenWrapping) =>
      w.wrapped_chain.universal_chain_id === sourceChain.universal_chain_id
      && w.unwrapped_chain.universal_chain_id === destinationChain.universal_chain_id,
  )

  return pipe(
    Option.fromNullable(maybeUnwrapped),
    Option.match({
      onSome: unwrapped =>
        findTokenWithBucket(
          tokenList,
          t => t.denom.toLowerCase() === unwrapped.unwrapped_denom.toLowerCase(),
        ),
      onNone: () =>
        findTokenWithBucket(tokenList, t =>
          t.wrapping.some(
            (w: TokenWrapping) =>
              w.unwrapped_denom.toLowerCase() === baseDenom
              && w.unwrapped_chain.universal_chain_id === sourceChain.universal_chain_id
              && w.wrapped_chain.universal_chain_id === destinationChain.universal_chain_id,
          )),
    }),
  )
}

const filterChainsByTokenAvailability = (
  chains: Array<Chain>,
  filterWhitelist: boolean,
): Array<ChainWithAvailability> =>
  pipe(
    Match.value(type).pipe(
      Match.when("source", () => chains.map(chain => Tuple.make(chain, false))),
      Match.when("destination", () =>
        pipe(
          Option.all({
            baseToken: transferData.baseToken,
            sourceChain: transferData.sourceChain,
          }),
          Option.match({
            onNone: () => chains.map(chain => Tuple.make(chain, false)),
            onSome: ({ baseToken, sourceChain }) =>
              chains.map(destinationChain => {
                // For testnet chains, we always mark them as available (hasBucket=true) to allow testing
                if (destinationChain.testnet === true) {
                  return Tuple.make(destinationChain, true)
                }
                if (!filterWhitelist) {
                  return Tuple.make(destinationChain, true)
                }
                const tokens = tokensStore.getData(destinationChain.universal_chain_id)
                return Option.match(tokens, {
                  onNone: () => Tuple.make(destinationChain, false),
                  onSome: tokenList =>
                    Tuple.make(
                      destinationChain,
                      hasTokenBucket(destinationChain, tokenList, baseToken, sourceChain),
                    ),
                })
              }),
          }),
        )),
      Match.exhaustive,
    ),
  )

const filteredChains = $derived(
  pipe(
    // Now use transferData.filteredChains which already includes edition filtering
    transferData.filteredChains,
    Option.map(allChains =>
      pipe(
        allChains,
        filterBySigningMode,
        chains => filterChainsByTokenAvailability(chains, uiStore.filterWhitelist),
        chainWithAvailability => {
          // Sort so disabled chains appear last for both source and destination
          return chainWithAvailability.sort((a, b) => {
            const [chainA] = a
            const [chainB] = b
            const isDisabledA = chainA.universal_chain_id
              && DISABLED_CHAINS.includes(chainA.universal_chain_id)
            const isDisabledB = chainB.universal_chain_id
              && DISABLED_CHAINS.includes(chainB.universal_chain_id)

            // If one is disabled and the other isn't, disabled goes last
            if (isDisabledA && !isDisabledB) {
              return 1
            }
            if (!isDisabledA && isDisabledB) {
              return -1
            }

            // If both have same disabled status, maintain original order
            return 0
          })
        },
      )
    ),
  ),
)
</script>

<div>
  {#if Option.isSome(filteredChains)}
    {@const chainss = filteredChains.value}
    <div class="relative">
      <div class="p-4 grid grid-cols-3 gap-2 max-h-[400px] overflow-y-auto">
        {#each chainss as chainWithAvailability}
          {@const [chain, hasBucket] = chainWithAvailability}
          {@const status = getChainStatus(chain, hasBucket)}
          {@const chainLogo = chain.universal_chain_id
          ? chainLogoMap.get(chain.universal_chain_id)
          : null}

          <button
            class={cn(
              "flex flex-col items-center gap-2 justify-start px-2 py-4 rounded-md transition-colors min-h-[120px]",
              status.isSelected
                ? "bg-zinc-900 hover:bg-zinc-800 ring-1 ring-accent"
                : status.isDisabled
                ? "bg-zinc-900 opacity-50 cursor-not-allowed"
                : "bg-zinc-900 hover:bg-zinc-800 cursor-pointer",
            )}
            onclick={() => !status.isDisabled && updateSelectedChain(chain)}
            disabled={status.isDisabled}
          >
            {#if chainLogo?.color}
              <span class="w-10 h-10 flex items-center justify-center overflow-hidden">
                <img
                  src={chainLogo.color}
                  alt=""
                />
              </span>
            {/if}

            <span class="text-xs text-center truncate w-fit">{chain.display_name}</span>

            <!-- Status labels container with consistent height -->
            <div class="text-xs -mt-2 h-4 flex items-center">
              {#if status.isSourceChain}
                <span class="text-sky-400">source chain</span>
              {:else if chain.universal_chain_id
              && DISABLED_CHAINS.includes(chain.universal_chain_id)}
                <span class="text-yellow-400">disabled</span>
              {:else if type === "destination" && !status.hasRoute && !status.isSourceChain}
                <span class="text-yellow-400">no route</span>
              {:else if type === "destination" && !status.hasBucket && status.hasRoute
              && !status.isSourceChain}
                <span class="text-yellow-400">not whitelisted</span>
              {/if}
            </div>
          </button>
        {/each}
      </div>
    </div>
    <div class="absolute bottom-0 left-0 right-0 h-16 bg-gradient-to-t from-zinc-925 to-transparent blur-fade-bottom-up pointer-events-none">
    </div>
  {:else}
    <div class="py-2 text-center text-zinc-500">
      <span class="inline-block animate-pulse">Loading chains...</span>
    </div>
  {/if}
</div>
