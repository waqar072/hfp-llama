<script lang="ts">
	import { onMount, tick } from 'svelte';
	import {
		ChevronDown,
		EyeOff,
		Loader2,
		MicOff,
		Package,
		Power,
		Server
	} from '@lucide/svelte';
	import * as Tooltip from '$lib/components/ui/tooltip';
	import * as Popover from '$lib/components/ui/popover';
	import { cn } from '$lib/components/ui/utils';
	import {
		modelsStore,
		modelOptions,
		modelsLoading,
		modelsUpdating,
		selectedModelId,
		routerModels,
		propsCacheVersion,
		singleModelName
	} from '$lib/stores/models.svelte';
	import { usedModalities, conversationsStore } from '$lib/stores/conversations.svelte';
	import { ServerModelStatus } from '$lib/enums';
	import { isRouterMode } from '$lib/stores/server.svelte';
	import { DialogModelInformation, SearchInput } from '$lib/components/app';
	import type { ModelOption } from '$lib/types/models';

	interface Props {
		class?: string;
		currentModel?: string | null;
		onModelChange?: (modelId: string, modelName: string) => Promise<boolean> | boolean | void;
		disabled?: boolean;
		forceForegroundText?: boolean;
		useGlobalSelection?: boolean;
		upToMessageId?: string;
	}

	let {
		class: className = '',
		currentModel = null,
		onModelChange,
		disabled = false,
		forceForegroundText = false,
		useGlobalSelection = false,
		upToMessageId
	}: Props = $props();

	// --- Existing State ---
	let options = $derived(modelOptions());
	let loading = $derived(modelsLoading());
	let updating = $derived(modelsUpdating());
	let activeId = $derived(selectedModelId());
	let isRouter = $derived(isRouterMode());
	let serverModel = $derived(singleModelName());
	let currentRouterModels = $derived(routerModels());

	let requiredModalities = $derived(
		upToMessageId ? conversationsStore.getModalitiesUpToMessage(upToMessageId) : usedModalities()
	);

	// --- Cluster Node Logic (UPDATED) ---
	interface ClusterNode {
		given_name: string;
		status: string;
		model_name?: string;
		model_status?: string;
	}

	let nodes = $state<ClusterNode[]>([]);
	let clusterSearchTerm = $state('');
	let clusterOpen = $state(false);
	let clusterSearchInputRef = $state<HTMLInputElement | null>(null);

	// Filter nodes based on hidden list and search term (Checks Name AND Model Name now)
	let filteredNodes = $derived(
		nodes
			.filter((n) => !['digital-ocean-server', 'server2-ritesh'].includes(n.given_name))
			.filter((n) => {
				const term = clusterSearchTerm.toLowerCase();
				const nameMatch = n.given_name.toLowerCase().includes(term);
				const modelMatch = n.model_name?.toLowerCase().includes(term);
				return nameMatch || modelMatch;
			})
			// Sort: Online/Healthy first, then others
			.sort((a, b) => {
				const isAOnline = ['online', 'healthy'].includes(a.status);
				const isBOnline = ['online', 'healthy'].includes(b.status);
				if (isAOnline === isBOnline) return 0;
				return isAOnline ? -1 : 1;
			})
	);

	// Count includes both 'online' and 'healthy' statuses
	let onlineCount = $derived(
		nodes
			.filter((n) => !['digital-ocean-server', 'server2-ritesh'].includes(n.given_name))
			.filter((n) => n.status === 'online' || n.status === 'healthy').length
	);

	// --- Existing Functions ---
	function getModelStatus(modelId: string): ServerModelStatus | null {
		const model = currentRouterModels.find((m) => m.id === modelId);
		return (model?.status?.value as ServerModelStatus) ?? null;
	}

	function isModelCompatible(option: ModelOption): boolean {
		void propsCacheVersion();
		const modelModalities = modelsStore.getModelModalities(option.model);
		if (!modelModalities) {
			const status = getModelStatus(option.model);
			if (status === ServerModelStatus.LOADED) {
				if (requiredModalities.vision || requiredModalities.audio) return false;
			}
			return true;
		}
		if (requiredModalities.vision && !modelModalities.vision) return false;
		if (requiredModalities.audio && !modelModalities.audio) return false;
		return true;
	}

	function getMissingModalities(option: ModelOption): { vision: boolean; audio: boolean } | null {
		void propsCacheVersion();
		const modelModalities = modelsStore.getModelModalities(option.model);
		if (!modelModalities) {
			const status = getModelStatus(option.model);
			if (status === ServerModelStatus.LOADED) {
				const missing = {
					vision: requiredModalities.vision,
					audio: requiredModalities.audio
				};
				if (missing.vision || missing.audio) return missing;
			}
			return null;
		}
		const missing = {
			vision: requiredModalities.vision && !modelModalities.vision,
			audio: requiredModalities.audio && !modelModalities.audio
		};
		if (!missing.vision && !missing.audio) return null;
		return missing;
	}

	let isHighlightedCurrentModelActive = $derived(
		!isRouter || !currentModel
			? false
			: (() => {
					const currentOption = options.find((option) => option.model === currentModel);
					return currentOption ? currentOption.id === activeId : false;
				})()
	);

	let isCurrentModelInCache = $derived(() => {
		if (!isRouter || !currentModel) return true;
		return options.some((option) => option.model === currentModel);
	});

	let searchTerm = $state('');
	let searchInputRef = $state<HTMLInputElement | null>(null);
	let highlightedIndex = $state<number>(-1);

	let filteredOptions: ModelOption[] = $derived(
		(() => {
			const term = searchTerm.trim().toLowerCase();
			if (!term) return options;
			return options.filter(
				(option) =>
					option.model.toLowerCase().includes(term) || option.name?.toLowerCase().includes(term)
			);
		})()
	);

	let compatibleIndices = $derived(
		filteredOptions
			.map((option, index) => (isModelCompatible(option) ? index : -1))
			.filter((i) => i !== -1)
	);

	$effect(() => {
		void searchTerm;
		highlightedIndex = -1;
	});

	let isOpen = $state(false);
	let showModelDialog = $state(false);

	onMount(() => {
		// Load Models
		modelsStore.fetch().catch((error) => {
			console.error('Unable to load models:', error);
		});

		// Load Nodes
		fetchNodes();
	});

	async function fetchNodes() {
		try {
			const response = await fetch('https://ai.nomineelife.com/api/nodes');
			if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
			nodes = await response.json();
		} catch (error) {
			console.error('Failed to fetch nodes:', error);
		}
	}

	function handleClusterOpenChange(open: boolean) {
		if (open) {
			clusterOpen = true;
			clusterSearchTerm = '';
			fetchNodes(); // Refresh on open
			tick().then(() => {
				requestAnimationFrame(() => clusterSearchInputRef?.focus());
			});
		} else {
			clusterOpen = false;
		}
	}

	function handleOpenChange(open: boolean) {
		if (loading || updating) return;

		if (isRouter) {
			if (open) {
				isOpen = true;
				searchTerm = '';
				highlightedIndex = -1;
				tick().then(() => {
					requestAnimationFrame(() => searchInputRef?.focus());
				});
				modelsStore.fetchRouterModels().then(() => {
					modelsStore.fetchModalitiesForLoadedModels();
				});
			} else {
				isOpen = false;
				searchTerm = '';
				highlightedIndex = -1;
			}
		} else {
			showModelDialog = open;
		}
	}

	export function open() {
		handleOpenChange(true);
	}

	function handleSearchKeyDown(event: KeyboardEvent) {
		if (event.isComposing) return;
		if (event.key === 'ArrowDown') {
			event.preventDefault();
			if (compatibleIndices.length === 0) return;
			const currentPos = compatibleIndices.indexOf(highlightedIndex);
			if (currentPos === -1 || currentPos === compatibleIndices.length - 1) {
				highlightedIndex = compatibleIndices[0];
			} else {
				highlightedIndex = compatibleIndices[currentPos + 1];
			}
		} else if (event.key === 'ArrowUp') {
			event.preventDefault();
			if (compatibleIndices.length === 0) return;
			const currentPos = compatibleIndices.indexOf(highlightedIndex);
			if (currentPos === -1 || currentPos === 0) {
				highlightedIndex = compatibleIndices[compatibleIndices.length - 1];
			} else {
				highlightedIndex = compatibleIndices[currentPos - 1];
			}
		} else if (event.key === 'Enter') {
			event.preventDefault();
			if (highlightedIndex >= 0 && highlightedIndex < filteredOptions.length) {
				const option = filteredOptions[highlightedIndex];
				if (isModelCompatible(option)) {
					handleSelect(option.id);
				}
			} else if (compatibleIndices.length > 0) {
				highlightedIndex = compatibleIndices[0];
			}
		}
	}

	async function handleSelect(modelId: string) {
		const option = options.find((opt) => opt.id === modelId);
		if (!option) return;

		let shouldCloseMenu = true;

		if (onModelChange) {
			const result = await onModelChange(option.id, option.model);
			if (result === false) {
				shouldCloseMenu = false;
			}
		} else {
			await modelsStore.selectModelById(option.id);
			if (isRouter && getModelStatus(option.model) !== ServerModelStatus.LOADED) {
				try {
					await modelsStore.loadModel(option.model);
				} catch (error) {
					console.error('Failed to load model:', error);
				}
			}
		}

		if (shouldCloseMenu) {
			handleOpenChange(false);
			requestAnimationFrame(() => {
				const textarea = document.querySelector<HTMLTextAreaElement>(
					'[data-slot="chat-form"] textarea'
				);
				textarea?.focus();
			});
		}
	}

	function getDisplayOption(): ModelOption | undefined {
		if (!isRouter) {
			if (serverModel) {
				return {
					id: 'current',
					model: serverModel,
					name: serverModel.split('/').pop() || serverModel,
					capabilities: []
				};
			}
			return undefined;
		}

		if (useGlobalSelection && activeId) {
			const selected = options.find((option) => option.id === activeId);
			if (selected) return selected;
		}

		if (currentModel) {
			if (!isCurrentModelInCache()) {
				return {
					id: 'not-in-cache',
					model: currentModel,
					name: currentModel.split('/').pop() || currentModel,
					capabilities: []
				};
			}
			return options.find((option) => option.model === currentModel);
		}

		if (activeId) {
			return options.find((option) => option.id === activeId);
		}
		return undefined;
	}
</script>

<div class={cn('relative inline-flex flex-row items-center gap-1', className)}>
	
	<Popover.Root bind:open={clusterOpen} onOpenChange={handleClusterOpenChange}>
		<Popover.Trigger
			class={cn(
				`inline-flex h-8 w-8 cursor-pointer items-center justify-center rounded-md text-muted-foreground transition hover:bg-muted hover:text-foreground focus:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2`,
				clusterOpen ? 'bg-muted text-foreground' : ''
			)}
		>
			<div class="relative">
				<Server class="h-4 w-4" />
				<span
					class={cn(
						'absolute -right-0.5 -top-0.5 h-2 w-2 rounded-full border border-background',
						onlineCount > 0 ? 'bg-green-500' : 'bg-muted-foreground/30'
					)}
				></span>
			</div>
		</Popover.Trigger>

		<Popover.Content
			class="w-72 p-0"
			align="start"
			sideOffset={8}
		>
			<div class="flex flex-col">
				<div class="flex items-center justify-between border-b px-3 py-2">
					<span class="text-xs font-semibold">Tailscale Cluster</span>
					<div class="flex items-center gap-1.5">
						<span
							class={cn(
								'h-1.5 w-1.5 rounded-full',
								onlineCount > 0 ? 'bg-green-500' : 'bg-muted-foreground/30'
							)}
						></span>
						<span class="text-xs text-muted-foreground">{onlineCount} online</span>
					</div>
				</div>

				<div class="border-b p-2">
					<SearchInput
						id="cluster-search"
						placeholder="Search nodes..."
						bind:value={clusterSearchTerm}
						bind:ref={clusterSearchInputRef}
					/>
				</div>

				<div class="max-h-60 overflow-y-auto py-1">
					{#if filteredNodes.length === 0}
						<div class="px-4 py-3 text-center text-sm text-muted-foreground">
							No active nodes.
						</div>
					{:else}
						{#each filteredNodes as node}
							<div 
								class="flex items-center justify-between px-3 py-2 text-sm hover:bg-muted/50"
								title={node.model_name ? `Running: ${node.model_name}` : 'No model info'}
							>
								
								
							<div class="flex flex-col min-w-0 pr-2">
    							<span class="truncate font-medium">{node.given_name}</span>
   		     					  {#if node.model_name}
  						            <span class="truncate text-[10px] text-muted-foreground opacity-80">
           						      {node.model_name}
       							    </span>
    							  {/if}
							</div>



								<div class="flex items-center gap-2">
									<span
										class={cn(
											'text-[10px] capitalize',
											node.status === 'online' || node.status === 'healthy'
												? 'text-green-500'
												: 'text-muted-foreground'
										)}
									>
										{node.status}
									</span>
									<span
										class={cn(
											'h-2 w-2 rounded-full',
											node.status === 'online' || node.status === 'healthy'
												? 'bg-green-500'
												: 'bg-muted-foreground/30'
										)}
									></span>
								</div>
							</div>
						{/each}
					{/if}
				</div>
			</div>
		</Popover.Content>
	</Popover.Root>

	{#if loading && options.length === 0 && isRouter}
		<div class="flex items-center gap-2 text-xs text-muted-foreground">
			<Loader2 class="h-3.5 w-3.5 animate-spin" />
			Loading models…
		</div>
	{:else if options.length === 0 && isRouter}
		<p class="text-xs text-muted-foreground">No models available.</p>
	{:else}
		{@const selectedOption = getDisplayOption()}

		{#if isRouter}
			<Popover.Root bind:open={isOpen} onOpenChange={handleOpenChange}>
				<Popover.Trigger
					class={cn(
						`inline-flex cursor-pointer items-center gap-1.5 rounded-sm bg-muted-foreground/10 px-1.5 py-1 text-xs transition hover:text-foreground focus:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-60`,
						!isCurrentModelInCache()
							? 'bg-red-400/10 !text-red-400 hover:bg-red-400/20 hover:text-red-400'
							: forceForegroundText
								? 'text-foreground'
								: isHighlightedCurrentModelActive
									? 'text-foreground'
									: 'text-muted-foreground',
						isOpen ? 'text-foreground' : ''
					)}
					style="max-width: min(calc(100cqw - 6.5rem), 32rem)"
					disabled={disabled || updating}
				>
					<Package class="h-3.5 w-3.5" />

					<span class="truncate font-medium">
						{selectedOption?.model || 'Select model'}
					</span>

					{#if updating}
						<Loader2 class="h-3 w-3.5 animate-spin" />
					{:else}
						<ChevronDown class="h-3 w-3.5" />
					{/if}
				</Popover.Trigger>

				<Popover.Content
					class="group/popover-content w-96 max-w-[calc(100vw-2rem)] p-0"
					align="end"
					sideOffset={8}
					collisionPadding={16}
				>
					<div class="flex max-h-[50dvh] flex-col overflow-hidden">
						<div
							class="order-1 shrink-0 border-b p-4 group-data-[side=top]/popover-content:order-2 group-data-[side=top]/popover-content:border-t group-data-[side=top]/popover-content:border-b-0"
						>
							<SearchInput
								id="model-search"
								placeholder="Search models..."
								bind:value={searchTerm}
								bind:ref={searchInputRef}
								onClose={() => handleOpenChange(false)}
								onKeyDown={handleSearchKeyDown}
							/>
						</div>
						<div
							class="models-list order-2 min-h-0 flex-1 overflow-y-auto group-data-[side=top]/popover-content:order-1"
						>
							{#if !isCurrentModelInCache() && currentModel}
								<button
									type="button"
									class="flex w-full cursor-not-allowed items-center bg-red-400/10 px-4 py-2 text-left text-sm text-red-400"
									role="option"
									aria-selected="true"
									aria-disabled="true"
									disabled
								>
									<span class="truncate">{selectedOption?.name || currentModel}</span>
									<span class="ml-2 text-xs whitespace-nowrap opacity-70">(not available)</span>
								</button>
								<div class="my-1 h-px bg-border"></div>
							{/if}
							{#if filteredOptions.length === 0}
								<p class="px-4 py-3 text-sm text-muted-foreground">No models found.</p>
							{/if}
							{#each filteredOptions as option, index (option.id)}
								{@const status = getModelStatus(option.model)}
								{@const isLoaded = status === ServerModelStatus.LOADED}
								{@const isLoading = status === ServerModelStatus.LOADING}
								{@const isSelected = currentModel === option.model || activeId === option.id}
								{@const isCompatible = isModelCompatible(option)}
								{@const isHighlighted = index === highlightedIndex}
								{@const missingModalities = getMissingModalities(option)}

								<div
									class={cn(
										'group flex w-full items-center gap-2 px-4 py-2 text-left text-sm transition focus:outline-none',
										isCompatible
											? 'cursor-pointer hover:bg-muted focus:bg-muted'
											: 'cursor-not-allowed opacity-50',
										isSelected || isHighlighted
											? 'bg-accent text-accent-foreground'
											: isCompatible
												? 'hover:bg-accent hover:text-accent-foreground'
												: '',
										isLoaded ? 'text-popover-foreground' : 'text-muted-foreground'
									)}
									role="option"
									aria-selected={isSelected || isHighlighted}
									aria-disabled={!isCompatible}
									tabindex={isCompatible ? 0 : -1}
									onclick={() => isCompatible && handleSelect(option.id)}
									onmouseenter={() => (highlightedIndex = index)}
									onkeydown={(e) => {
										if (isCompatible && (e.key === 'Enter' || e.key === ' ')) {
											e.preventDefault();
											handleSelect(option.id);
										}
									}}
								>
									<span class="min-w-0 flex-1 truncate">{option.model}</span>

									{#if missingModalities}
										<span class="flex shrink-0 items-center gap-1 text-muted-foreground/70">
											{#if missingModalities.vision}
												<Tooltip.Root>
													<Tooltip.Trigger>
														<EyeOff class="h-3.5 w-3.5" />
													</Tooltip.Trigger>
													<Tooltip.Content class="z-[9999]">
														<p>No vision support</p>
													</Tooltip.Content>
												</Tooltip.Root>
											{/if}
											{#if missingModalities.audio}
												<Tooltip.Root>
													<Tooltip.Trigger>
														<MicOff class="h-3.5 w-3.5" />
													</Tooltip.Trigger>
													<Tooltip.Content class="z-[9999]">
														<p>No audio support</p>
													</Tooltip.Content>
												</Tooltip.Root>
											{/if}
										</span>
									{/if}

									{#if isLoading}
										<Tooltip.Root>
											<Tooltip.Trigger>
												<Loader2 class="h-4 w-4 shrink-0 animate-spin text-muted-foreground" />
											</Tooltip.Trigger>
											<Tooltip.Content class="z-[9999]">
												<p>Loading model...</p>
											</Tooltip.Content>
										</Tooltip.Root>
									{:else if isLoaded}
										<Tooltip.Root>
											<Tooltip.Trigger>
												<button
													type="button"
													class="relative ml-2 flex h-4 w-4 shrink-0 items-center justify-center"
													onclick={(e) => {
														e.stopPropagation();
														modelsStore.unloadModel(option.model);
													}}
												>
													<span
														class="mr-2 h-2 w-2 rounded-full bg-green-500 transition-opacity group-hover:opacity-0"
													></span>
													<Power
														class="absolute mr-2 h-4 w-4 text-red-500 opacity-0 transition-opacity group-hover:opacity-100 hover:text-red-600"
													/>
												</button>
											</Tooltip.Trigger>
											<Tooltip.Content class="z-[9999]">
												<p>Unload model</p>
											</Tooltip.Content>
										</Tooltip.Root>
									{:else}
										<span class="mx-2 h-2 w-2 rounded-full bg-muted-foreground/50"></span>
									{/if}
								</div>
							{/each}
						</div>
					</div>
				</Popover.Content>
			</Popover.Root>
		{:else}
			<button
				class={cn(
					`inline-flex cursor-pointer items-center gap-1.5 rounded-sm bg-muted-foreground/10 px-1.5 py-1 text-xs transition hover:text-foreground focus:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-60`,
					!isCurrentModelInCache()
						? 'bg-red-400/10 !text-red-400 hover:bg-red-400/20 hover:text-red-400'
						: forceForegroundText
							? 'text-foreground'
							: isHighlightedCurrentModelActive
								? 'text-foreground'
								: 'text-muted-foreground',
					isOpen ? 'text-foreground' : ''
				)}
				style="max-width: min(calc(100cqw - 6.5rem), 32rem)"
				onclick={() => handleOpenChange(true)}
				disabled={disabled || updating}
			>
				<Package class="h-3.5 w-3.5" />

				<span class="truncate font-medium">
					{selectedOption?.model}
				</span>

				{#if updating}
					<Loader2 class="h-3 w-3.5 animate-spin" />
				{/if}
			</button>
		{/if}
	{/if}
</div>

{#if showModelDialog && !isRouter}
	<DialogModelInformation bind:open={showModelDialog} />
{/if}