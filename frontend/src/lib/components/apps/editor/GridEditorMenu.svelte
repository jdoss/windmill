<script context="module" lang="ts">
	import { writable } from 'svelte/store'

	interface ContextMenuRegistry {
		id: string
		close: () => void
	}

	// Using a Svelte store for global state management
	const openedContextMenus = writable<Set<ContextMenuRegistry>>(new Set())
</script>

<script lang="ts">
	import { getModifierKey } from '$lib/utils'

	import { Copy, Paintbrush2, Scissors, Trash } from 'lucide-svelte'
	import ComponentCallbacks from './component/ComponentCallbacks.svelte'
	import { getContext } from 'svelte'
	import type { AppEditorContext, AppViewerContext } from '../types'
	import DeleteComponent from './settingsPanel/DeleteComponent.svelte'
	import { secondaryMenuLeft } from './settingsPanel/secondaryMenu'
	import StylePanel from './settingsPanel/StylePanel.svelte'
	import { clickOutside } from '$lib/utils'
	import Portal from 'svelte-portal'
	import { twMerge } from 'tailwind-merge'

	let contextMenuVisible = false
	let menuX = 0
	let menuY = 0

	function handleRightClick(event: MouseEvent) {
		event.preventDefault()
		contextMenuVisible = true
		menuX = event.clientX
		menuY = event.clientY

		openedContextMenus.update((menus) => {
			menus.forEach((menu) => menu.id !== id && menu.close())
			menus.clear()
			menus.add({ id, close: () => (contextMenuVisible = false) })
			return menus
		})
	}

	function closeContextMenu() {
		contextMenuVisible = false
		openedContextMenus.update((menus) => {
			menus.clear()
			return menus
		})
	}

	function handleClickOutside(event: MouseEvent) {
		if (contextMenuVisible) {
			closeContextMenu()
		}
	}

	export let id: string
	let componentCallbacks: ComponentCallbacks | undefined = undefined

	const { selectedComponent } = getContext<AppViewerContext>('AppViewerContext')
	const { movingcomponents } = getContext<AppEditorContext>('AppEditorContext')

	let deleteComponent: DeleteComponent | undefined = undefined

	const menuItems = [
		{
			label: 'Cut',
			onClick: () => {
				componentCallbacks?.handleCut(new KeyboardEvent('keydown'))
			},
			icon: Scissors,
			shortcut: `${getModifierKey()}X`,
			disabled: $movingcomponents?.includes($selectedComponent?.[0] ?? '')
		},
		{
			label: 'Copy',
			onClick: () => {
				componentCallbacks?.handleCopy(new KeyboardEvent('keydown'))
			},
			icon: Copy,
			shortcut: `${getModifierKey()}C`
		},
		{
			label: 'Show style panel',
			onClick: () => {
				secondaryMenuLeft?.toggle(StylePanel, { type: 'style' })
			},
			icon: Paintbrush2,
			disabled: $secondaryMenuLeft.isOpen
		},

		{
			label: 'Delete',
			onClick: () => {
				deleteComponent?.removeGridElement()
			},
			icon: Trash,
			shortcut: `Del`,
			color: 'red'
		}
	]
</script>

<ComponentCallbacks bind:this={componentCallbacks} />
<DeleteComponent bind:this={deleteComponent} />

<svelte:window on:click={handleClickOutside} />

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div on:click={closeContextMenu} on:contextmenu={handleRightClick} class="h-full w-full">
	<!-- svelte-ignore a11y-no-static-element-interactions -->
	<slot />

	{#if contextMenuVisible}
		<Portal>
			<div style="position: fixed; top: {menuY}px; left: {menuX}px; z-index:6000;">
				<div class="rounded-md bg-surface border shadow-md divide-y w-64">
					<div class="p-1" use:clickOutside={false}>
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						{#each menuItems as item}
							<!-- svelte-ignore a11y-no-static-element-interactions -->
							<!-- svelte-ignore a11y-click-events-have-key-events -->
							<button
								class={twMerge(
									'flex items-center p-2 hover:bg-surface-hover cursor-pointer transition-all rounded-md  w-full',
									item.color === 'red' && 'text-red-500',
									item.color === 'green' && 'text-green-500',
									item.color === 'blue' && 'text-blue-500',
									item.disabled && 'opacity-50 cursor-not-allowed'
								)}
								on:click={() => {
									item.onClick()
									closeContextMenu()
								}}
								disabled={item.disabled}
							>
								<!-- svelte-ignore missing-declaration -->
								<svelte:component this={item.icon} class="w-4 h-4" />

								<span class="ml-2 text-xs">{item.label}</span>
								{#if item.shortcut}
									<span class="ml-auto text-xs text-gray-400">
										{item.shortcut}
									</span>
								{/if}
							</button>
						{/each}
					</div>
				</div>
			</div>
		</Portal>
	{/if}
</div>
