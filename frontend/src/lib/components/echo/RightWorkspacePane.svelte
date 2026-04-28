<script lang="ts">
	import { Cloud, Database, Leaf, RefreshCw } from '@lucide/svelte';
	import type { CacheEntry, CacheStatsResponse, CacheUse } from '$lib/cache';
	import { BACKEND_URL } from '$lib/constants';

	let localRamCache = $state<CacheEntry[]>([]);
	let s3CacheUsed = $state<CacheUse[]>([]);
	let metrics = $state<CacheStatsResponse['metrics']>({
		cacheHits: 0,
		localCacheHits: 0,
		s3CacheHits: 0,
		estimatedTokensSaved: 0,
		energySavedWh: 0,
		co2SavedG: 0
	});
	let constants = $state<CacheStatsResponse['constants']>({
		defaultKWhPer1KTokens: 0.00035,
		gridCO2gPerKWh: 475,
		modelKWhPer1KTokens: {}
	});
	let uploading = $state(false);
	let lastUploadAt = $state('');
	let lastDownloadAt = $state('');
	let loading = $state(false);
	let interval = $state<ReturnType<typeof setInterval> | undefined>(undefined);

	// Maximum words to display for questions/answers
	let { maxWords = 15 }: { maxWords?: number } = $props();

	function truncateWords(text: string, max: number) {
		if (!text) return '';
		const parts = text.trim().split(/\s+/);
		if (parts.length <= max) return text;
		return parts.slice(0, max).join(' ') + '…';
	}

	async function fetchCacheStats() {
		loading = true;
		try {
			const res = await fetch(`${BACKEND_URL}/cache-stats`);
			if (!res.ok) {
				throw new Error(`Cache stats request failed: ${res.status}`);
			}
			const data = (await res.json()) as CacheStatsResponse;

			uploading = data.uploading;
			metrics = data.metrics;
			constants = data.constants ?? constants;
			localRamCache = Array.isArray(data.localRamCache) ? data.localRamCache : [];
			s3CacheUsed = Array.isArray(data.s3CacheUsed) ? data.s3CacheUsed : [];
			lastUploadAt = data.lastUploadAt
				? new Date(data.lastUploadAt).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
				: '—';
			lastDownloadAt = data.lastDownloadAt
				? new Date(data.lastDownloadAt).toLocaleTimeString([], {
						hour: '2-digit',
						minute: '2-digit'
					})
				: '—';
		} catch (e) {
			console.error('Failed to fetch cache stats', e);
		} finally {
			loading = false;
		}
	}

	$effect(() => {
		fetchCacheStats();
		interval = setInterval(fetchCacheStats, 5000);
		return () => {
			if (interval) {
				clearInterval(interval);
			}
		};
	});
</script>

<aside class="flex h-full w-[85vw] shrink-0 flex-col border-l border-white/5 bg-card lg:w-[360px]">
	<div class="min-h-0 flex-1 border-b dark:border-white/5">
		<div class="px-4 pt-3 pb-2 text-xs text-gray-400">Cache Saved</div>
		<div class="h-[calc(100%-34px)] space-y-2 overflow-y-auto px-3 pb-3">
			{#if localRamCache.length === 0}
				<div
					class="flex h-full flex-col items-center justify-center space-y-2 text-gray-600 opacity-60"
				>
					<Database size={28} strokeWidth={1} />
					<p class="text-xs">No local entries yet</p>
				</div>
			{/if}

			{#each localRamCache as item}
				<div class="rounded-lg border bg-card p-3 dark:border-white/6">
					<h3 class="line-clamp-2 text-xs leading-relaxed dark:text-gray-200">
						"{truncateWords(item.question, maxWords)}"
					</h3>
					<div class="mt-2 text-[10px] text-gray-500">↳ {truncateWords(item.answer, maxWords)}</div>
				</div>
			{/each}
		</div>
	</div>

	<div class="min-h-0 flex-1">
		<div class="px-4 pt-3 pb-2 text-xs text-gray-400">S3 Cache Used</div>
		<div class="h-[calc(100%-34px)] space-y-2 overflow-y-auto px-3 pb-3">
			{#if s3CacheUsed.length === 0}
				<div
					class="flex h-full flex-col items-center justify-center space-y-2 text-gray-600 opacity-60"
				>
					<Cloud size={26} strokeWidth={1} />
					<p class="text-xs">No S3 cache hits yet</p>
				</div>
			{/if}

			{#each s3CacheUsed as item}
				<div class="rounded-lg border bg-card p-3 dark:border-white/6">
					<h3 class="line-clamp-2 pr-4 text-xs leading-relaxed dark:text-gray-200">
						"{truncateWords(item.question, maxWords)}"
					</h3>
					<div class="mt-2 text-[10px] text-gray-500">↳ {truncateWords(item.answer, maxWords)}</div>
					<div class="mt-2 flex items-center justify-between text-[10px] text-gray-500">
						<span
							>{new Date(item.timestamp).toLocaleTimeString([], {
								hour: '2-digit',
								minute: '2-digit'
							})}</span
						>
						<span>{item.co2SavedG.toFixed(3)} g CO₂</span>
					</div>
				</div>
			{/each}
		</div>
	</div>
</aside>
