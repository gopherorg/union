<script lang="ts">
import { page } from "$app/state"
import ExternalLinkIcon from "$lib/components/icons/ExternalLinkIcon.svelte"
import ConnectWalletButton from "$lib/components/ui/ConnectWalletButton.svelte"
import ProfileCard from "$lib/dashboard/components/SideCard.svelte"
import { uiStore } from "$lib/stores/ui.svelte"
import { cn } from "$lib/utils"
import { onMount } from "svelte"
import { navigation } from "./navigation.js"

const isCurrentPath = (path: string) => {
  // Exact match
  if (page.url.pathname === path) {
    return true
  }
  // Check if current path is a subroute of the navigation item
  // For example, /explorer/packets/123 should highlight /explorer/packets
  if (path !== "/" && page.url.pathname.startsWith(`${path}/`)) {
    return true
  }

  return false
}

let highlightElement: HTMLElement
let overflowContainer: HTMLDivElement

const updateHighlightPosition = () => {
  const yOffset = overflowContainer?.scrollTop ?? 0
  if (page.url.pathname && highlightElement) {
    // Find the best matching navigation item
    let bestMatch: HTMLElement | null = null
    let bestMatchLength = 0

    // Check all navigation items to find the best match
    const allNavItems = document.querySelectorAll("[data-path]")
    allNavItems.forEach((item) => {
      const itemPath = item.getAttribute("data-path")
      if (
        itemPath
        && (page.url.pathname === itemPath
          || (page.url.pathname.startsWith(`${itemPath}/`)
            && itemPath.length > bestMatchLength))
      ) {
        bestMatch = item as HTMLElement
        bestMatchLength = itemPath.length
      }
    })

    if (bestMatch) {
      const rect = (bestMatch as HTMLElement).getBoundingClientRect()
      highlightElement.style.top = `${rect.top + yOffset}px`
      highlightElement.style.left = `${rect.left}px`
      highlightElement.style.width = `${rect.width}px`
      highlightElement.style.height = `${rect.height}px`
      highlightElement.style.opacity = "1"
    } else {
      highlightElement.style.opacity = "0"
    }
  }
}

$effect(() => {
  if (page.url.pathname) {
    updateHighlightPosition()
  }
})

onMount(() => {
  window.addEventListener("resize", updateHighlightPosition)
  // Use requestAnimationFrame to ensure DOM is fully rendered
  requestAnimationFrame(() => {
    updateHighlightPosition()
  })

  return () => {
    window.removeEventListener("resize", updateHighlightPosition)
  }
})
</script>

<div
  class="relative h-full overflow-y-auto"
  bind:this={overflowContainer}
>
  <div
    bind:this={highlightElement}
    class="absolute -z-10 bg-accent rounded-lg transition-all duration-300"
  >
  </div>

  <div class="min-h-full flex flex-col overflow-y-auto">
    <a
      class="px-6 flex items-center gap-2 border-b-1 h-16 border-zinc-900"
      href="/"
    >
      <img
        class="h-10"
        src="/images/union-logo.svg"
        alt="Union"
      />
      {#key uiStore.edition}
        <div class="bg-accent px-2 py rounded text-sm font-mono font-bold">
          {uiStore.theme.label}
        </div>
      {/key}
    </a>
    <div class="flex flex-col flex-1">
      <ProfileCard />
      {#each navigation as section}
        {#if section.title !== "Developer" || uiStore.showDeveloperPages}
          <section class="border-zinc-900 p-6 last:flex-1 flex flex-col justify-end">
            {#if section.title}
              <h2 class="font-bold text-sm mb-2.5 text-center uppercase text-zinc-600">
                {section.title}
              </h2>
            {/if}
            <ul class="flex flex-col gap-1">
              {#each section.items as item}
                <li>
                  <a
                    href={item.path}
                    data-path={item.path}
                    target={item.external ? "_blank" : undefined}
                    rel={item.external ? "noopener noreferrer" : undefined}
                    class={cn(
                      "relative flex items-center gap-2 px-3 py-2 rounded-lg transition-colors",
                      isCurrentPath(item.path) ? "" : "dark:hover:bg-zinc-900",
                    )}
                  >
                    <item.icon
                      class={cn(
                        isCurrentPath(item.path)
                          ? "size-5 text-white"
                          : "size-5 zinc-500",
                      )}
                    />
                    {item.title}
                    {#if item.external}
                      <ExternalLinkIcon class="size-4 ml-auto text-zinc-500" />
                    {/if}
                  </a>

                  {#if item.subroutes && item.subroutes.length > 0}
                    <ul class="flex flex-col border-zinc-800 gap-1 pt-2 border-l-1 ml-5 pl-2">
                      {#each item.subroutes as subroute}
                        {#if !subroute.editions
                  || subroute.editions.includes(uiStore.edition)}
                          <li>
                            <a
                              href={subroute.path}
                              data-path={subroute.path}
                              class={cn(
                                "relative flex items-center gap-2 px-3 py-1 rounded-lg transition-colors",
                                isCurrentPath(subroute.path)
                                  ? ""
                                  : "dark:hover:bg-zinc-900",
                              )}
                            >
                              {subroute.title}
                            </a>
                          </li>
                        {/if}
                      {/each}
                    </ul>
                  {/if}
                </li>
              {/each}
            </ul>
          </section>
        {/if}
      {/each}
    </div>

    <div class="flex flex-col gap-2 p-6 border-t border-zinc-900">
      <ConnectWalletButton />
      <!--
    <Button variant="secondary" onclick={() => uiStore.openSettingsModal()}>
      <SharpSettingsIcon class="size-5"/>
      Settings
    </Button>
    !-->
    </div>
  </div>
</div>
