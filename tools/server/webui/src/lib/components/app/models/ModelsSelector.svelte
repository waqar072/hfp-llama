<script lang="ts">
	import { onMount, tick } from 'svelte';
	import {
		ChevronDown,
		Loader2,
		Package,
		Check,
		Server,
		Monitor
	} from '@lucide/svelte';
	import * as Popover from '$lib/components/ui/popover';
	import { cn } from '$lib/components/ui/utils';
	import {
		modelsStore,
		modelsLoading,
		modelsUpdating,
	} from '$lib/stores/models.svelte';
	import { targetNode } from '$lib/stores/server.svelte';
	import { SearchInput } from '$lib/components/app';

	interface Props {
		class?: string;
		disabled?: boolean;
		forceForegroundText?: boolean;
	}

	let {
		class: className = '',
		disabled = false,
		forceForegroundText = false,
	}: Props = $props();

	// --- Constants ---
	const IGNORED_NODES = ['digital-ocean-server', 'server2-ritesh'];

	// --- State ---
	let loading = $derived(modelsLoading());
	let updating = $derived(modelsUpdating());

	interface ClusterNode {
		given_name: string;
		status: string;
		model_name?: string;
		model_status?: string;
		address: string;
	}

	let nodes = $state<ClusterNode[]>([]);
	let liveModelName = $state<string | null>(null);
	let isOpen = $state(false);
	
	// Search State
	let searchTerm = $state('');
	let searchInputRef = $state<HTMLInputElement | null>(null);

	// --- Helper Functions ---
	function formatModelName(name: string | null | undefined): string {
		if (!name) return 'Loading...';
		return name.replace(/\.gguf$/i, '');
	}

	function itemSize(modelName: string | undefined): string {
		if (!modelName) return '';
		const parts = modelName.split('-');
		const lastPart = parts[parts.length - 1];
		
		if (lastPart.includes('B')) {
			return lastPart;
		}
		
		const sizeMatch = modelName.match(/(\d+B|\d+\.\d+B)/i);
		return sizeMatch ? sizeMatch[1] : '';
	}

	// --- Filter Logic ---
	let sortedNodes = $derived(
		nodes
			.filter((n) => !IGNORED_NODES.includes(n.given_name))
			.filter((n) => {
				const nodeStatus = (n.status || '').toLowerCase();
				const modelStatus = (n.model_status || '').toLowerCase();
				const modelName = (n.model_name || '');

				const isNodeUp = nodeStatus === 'online' || nodeStatus === 'healthy';
				const isModelUp = modelStatus === 'online' || modelStatus === 'healthy';
				const hasValidModel = modelName !== 'N/A' && modelName !== '';

				return isNodeUp && isModelUp && hasValidModel;
			})
			.filter((n) => {
				const term = searchTerm.toLowerCase();
				return n.given_name.toLowerCase().includes(term) || 
					   (n.model_name || '').toLowerCase().includes(term);
			})
			.sort((a, b) => a.given_name.localeCompare(b.given_name))
	);

	function toggleNodeSelection(address: string) {
		if (targetNode.address === address) {
			targetNode.address = null;
		} else {
			targetNode.address = address;
		}
		isOpen = false; 
		fetchLiveModel(); 
	}

	async function fetchNodes() {
		try {
			const response = await fetch('https://ai.nomineelife.com/api/nodes');
			if (!response.ok) throw new Error('Failed to fetch nodes');
			nodes = await response.json();
		} catch (error) {
			console.error('Fetch nodes error:', error);
		}
	}

	async function fetchLiveModel() {
		try {
			const headers: Record<string, string> = {};
			if (targetNode.address) {
				headers['X-Target-Node'] = targetNode.address;
			}

			const response = await fetch('https://ai.nomineelife.com/v1/models', { headers });
			if (!response.ok) return; 
			
			const data = await response.json();
			if (data?.data?.length > 0) {
				liveModelName = data.data[0].id;
			}
		} catch (error) {
			console.error('Fetch model error:', error);
		}
	}

	function handleOpenChange(open: boolean) {
		if (loading || updating) return;
		isOpen = open;
		if (open) {
			searchTerm = '';
			fetchNodes();
			tick().then(() => {
				requestAnimationFrame(() => searchInputRef?.focus());
			});
		}
	}

	onMount(() => {
		modelsStore.fetch().catch(() => {});
		fetchNodes();
		fetchLiveModel();

		const interval = setInterval(() => {
			fetchLiveModel();
			if (isOpen) fetchNodes();
		}, 5000);

		return () => clearInterval(interval);
	});

	$effect(() => {
		void targetNode.address; 
		fetchLiveModel();
	});
</script>

<div class={cn('relative inline-flex items-center', className)}>
	<Popover.Root bind:open={isOpen} onOpenChange={handleOpenChange}>
		<Popover.Trigger
			class={cn(
				// Base Layout: Taller (h-8) and standard text size (text-sm)
				"inline-flex h-8 items-center gap-2 rounded-md px-3 text-sm font-medium transition-colors",
				"focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50",
				
				// Colors
				targetNode.address 
					? "bg-blue-500/15 text-blue-600 dark:text-blue-400 hover:bg-blue-500/25" 
					: "bg-secondary text-secondary-foreground hover:bg-secondary/80",

				forceForegroundText ? "text-foreground" : "",
				
				// --- WIDTH CONTROLS ---
				// Mobile: Expanded to 200px to fill the red area you highlighted
				// Desktop: Large allowance (500px)
				"max-w-[200px] sm:max-w-[500px]"
			)}
			disabled={disabled || updating}
		>
			{#if targetNode.address}
				<Server class="h-4 w-4 shrink-0" />
			{:else}
				<Package class="h-4 w-4 shrink-0 opacity-70" />
			{/if}

			<span class="truncate pb-px block max-w-[150px] sm:max-w-[450px]">
				{formatModelName(liveModelName)}
			</span>

			{#if updating}
				<Loader2 class="h-3.5 w-3.5 shrink-0 animate-spin opacity-50" />
			{/if}
		</Popover.Trigger>

		<Popover.Content
			class="w-[calc(100vw-32px)] max-w-[400px] sm:w-[400px] p-0 shadow-lg rounded-lg overflow-hidden"
			align="end"
			sideOffset={8}
			side="top"
			collisionPadding={16}
		>
			<div class="flex flex-col">
				<div class="px-3 py-2 text-xs font-semibold text-muted-foreground uppercase tracking-wider bg-muted/30 border-b flex justify-between items-center">
					<span>Select Processing Node</span>
					<span class="bg-primary/10 text-primary px-1.5 py-0.5 rounded text-[10px] tabular-nums">
						{sortedNodes.length} Online
					</span>
				</div>

				<div class="p-2 border-b">
					<SearchInput
						id="node-search"
						placeholder="Search nodes..."
						bind:value={searchTerm}
						bind:ref={searchInputRef}
						class="w-full h-9 text-sm" 
					/>
				</div>

				<div class="max-h-[min(50vh,300px)] overflow-y-auto p-1">
					{#each sortedNodes as node}
						{@const isSelected = targetNode.address === node.address}
						{@const formattedModelName = formatModelName(node.model_name)}
						
						<button
							type="button"
							class={cn(
								// Increased text size to text-sm for better visibility
								"flex w-full items-center justify-between rounded-md px-2 py-2.5 text-sm transition-colors text-left group",
								isSelected 
									? "bg-blue-50 text-blue-700 dark:bg-blue-900/30 dark:text-blue-300" 
									: "hover:bg-muted text-foreground"
							)}
							onclick={() => toggleNodeSelection(node.address)}
						>
							<div class="flex items-center gap-3 min-w-0 flex-1">
								<Monitor class={cn("h-4 w-4 shrink-0 text-green-500")} />
								<div class="flex flex-col min-w-0 flex-1 overflow-hidden">
									<span class="font-medium truncate block">
										{node.given_name}
									</span>
									{#if formattedModelName}
										<div class="flex items-center gap-1 mt-0.5">
											<span class="text-xs bg-muted/50 px-1.5 py-0.5 rounded truncate flex-1 max-w-full">
												{formattedModelName}
											</span>
											<span class="text-[10px] text-muted-foreground opacity-70 shrink-0">
												{itemSize(formattedModelName)}
											</span>
										</div>
									{/if}
								</div>
							</div>
							
							{#if isSelected}
								<Check class="h-4 w-4 shrink-0 text-blue-600 dark:text-blue-400 ml-2" />
							{/if}
						</button>
					{/each}

					{#if sortedNodes.length === 0}
						<div class="p-4 text-center text-sm text-muted-foreground">
							No healthy nodes found.
						</div>
					{/if}
				</div>
			</div>
		</Popover.Content>
	</Popover.Root>
</div>

<style>
</style>