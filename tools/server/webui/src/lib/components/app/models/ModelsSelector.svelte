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

	// --- Filter Logic ---
	let sortedNodes = $derived(
		nodes
			// 1. Remove specific ignored servers
			.filter((n) => !['digital-ocean-server', 'server2-ritesh'].includes(n.given_name))
			
			// 2. Strict Status Filter (Online/Healthy Only)
			.filter((n) => {
				const nodeStatus = (n.status || '').toLowerCase();
				const modelStatus = (n.model_status || '').toLowerCase();
				const modelName = (n.model_name || '');

				const isNodeUp = nodeStatus === 'online' || nodeStatus === 'healthy';
				const isModelUp = modelStatus === 'online' || modelStatus === 'healthy';
				const hasValidModel = modelName !== 'N/A' && modelName !== '';

				return isNodeUp && isModelUp && hasValidModel;
			})

			// 3. Search Filter
			.filter((n) => {
				const term = searchTerm.toLowerCase();
				return n.given_name.toLowerCase().includes(term) || 
					   (n.model_name || '').toLowerCase().includes(term);
			})

			// 4. Sort Alphabetically
			.sort((a, b) => a.given_name.localeCompare(b.given_name))
	);

	function formatModelName(name: string | null | undefined): string {
		if (!name) return 'Loading...';
		return name.replace(/\.gguf$/i, '');
	}

	function toggleNodeSelection(address: string) {
		if (targetNode.address === address) {
			targetNode.address = null; // Toggle off (Auto)
		} else {
			targetNode.address = address; // Toggle on
		}
		isOpen = false; // Close dropdown
		fetchLiveModel(); // Update model name immediately
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
		if (targetNode.address || !targetNode.address) {
			fetchLiveModel();
		}
	});
</script>

<div class={cn('relative inline-flex flex-row items-center gap-1', className)}>
	<Popover.Root bind:open={isOpen} onOpenChange={handleOpenChange}>
		<Popover.Trigger
			class={cn(
				`inline-flex cursor-pointer items-center gap-2 rounded-lg border px-3 py-2 text-sm transition-colors focus:outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50`,
				targetNode.address 
					? 'bg-blue-50 border-blue-200 text-blue-700 dark:bg-blue-900/20 dark:border-blue-800 dark:text-blue-300' 
					: 'bg-background border-border hover:bg-muted/50',
				forceForegroundText ? 'text-foreground' : '',
			)}
			disabled={disabled || updating}
		>
			{#if targetNode.address}
				<Server class="h-4 w-4 shrink-0" />
			{:else}
				<Package class="h-4 w-4 shrink-0" />
			{/if}

			<span class="font-medium truncate max-w-[400px]">
				{formatModelName(liveModelName)}
			</span>

			{#if updating}
				<Loader2 class="h-4 w-4 shrink-0 animate-spin opacity-50" />
			{:else}
				<ChevronDown class="h-4 w-4 shrink-0 opacity-50" />
			{/if}
		</Popover.Trigger>

		<Popover.Content
			class="w-[320px] p-0 shadow-lg rounded-lg overflow-hidden"
			align="end"
			sideOffset={5}
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
					/>
				</div>

				<div class="max-h-[300px] overflow-y-auto p-1">
					{#each sortedNodes as node}
						{@const isSelected = targetNode.address === node.address}
						
						<button
							type="button"
							class={cn(
								"flex w-full items-center justify-between rounded-md px-2 py-2 text-sm transition-colors text-left",
								isSelected 
									? "bg-blue-50 text-blue-700 dark:bg-blue-900/30 dark:text-blue-300" 
									: "hover:bg-muted text-foreground"
							)}
							onclick={() => toggleNodeSelection(node.address)}
						>
							<div class="flex items-center gap-3 min-w-0 overflow-hidden">
								<Monitor class={cn("h-4 w-4 shrink-0 text-green-500")} />
								<div class="flex flex-col min-w-0">
									<span class="font-medium truncate block">{node.given_name}</span>
									{#if node.model_name}
										<span class="text-[10px] opacity-70 truncate block">
											{formatModelName(node.model_name)}
										</span>
									{/if}
								</div>
							</div>
							
							{#if isSelected}
								<Check class="h-4 w-4 shrink-0 text-blue-600 dark:text-blue-400" />
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