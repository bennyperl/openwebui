<script lang="ts">
	import { getContext } from 'svelte';
	import { toast } from 'svelte-sonner';
	import Sparkles from '$lib/components/icons/Sparkles.svelte';
	import { user as _user, settings } from '$lib/stores';

	const i18n = getContext('i18n');

	export let params: Record<string, any> = {};
	export let showFileSystemPrompt = false;

	$: canUseSystemPrompt =
		$_user?.role === 'admin' || ($_user?.permissions?.chat?.system_prompt ?? true);
</script>

{#if canUseSystemPrompt}
	<div class="mx-2 mt-1 pb-1 flex items-center flex-wrap gap-1.5" dir={$settings?.chatDirection ?? 'auto'}>
		<button
			type="button"
			class="flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium transition cursor-pointer select-none {params?.system
				? 'bg-emerald-500/15 text-emerald-700 dark:text-emerald-300 border border-emerald-500/30'
				: 'bg-gray-100 dark:bg-gray-800 text-gray-700 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-700 border border-transparent'}"
			on:click={() => {
				showFileSystemPrompt = !showFileSystemPrompt;
			}}
		>
			<Sparkles className="size-3.5 text-emerald-500" />
			<span>
				{params?.system
					? $i18n.t('System Prompt Active')
					: $i18n.t('+ Add File System Prompt')}
			</span>
			{#if params?.system}
				<span class="size-2 rounded-full bg-emerald-500 animate-pulse" />
			{/if}
		</button>
	</div>

	{#if showFileSystemPrompt}
		<div
			class="mx-2 my-1.5 p-3 rounded-xl bg-gray-50/80 dark:bg-gray-850 border border-emerald-500/30 text-xs space-y-2"
			dir={$settings?.chatDirection ?? 'auto'}
		>
			<div class="flex items-center justify-between">
				<div class="flex items-center gap-1.5 font-semibold text-emerald-600 dark:text-emerald-400">
					<Sparkles className="size-4" />
					<span>{$i18n.t('File System Prompt')}</span>
				</div>
				<button
					type="button"
					class="text-gray-400 hover:text-gray-600 dark:hover:text-gray-200 transition"
					on:click={() => {
						showFileSystemPrompt = false;
					}}
				>
					✕
				</button>
			</div>

			<textarea
				bind:value={params.system}
				rows="3"
				class="w-full text-xs p-2.5 rounded-lg bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 focus:ring-1 focus:ring-emerald-500 focus:outline-none dark:text-gray-100"
				placeholder={$i18n.t('Enter system prompt for file analysis...')}
			/>

			<div
				class="flex items-center justify-between flex-wrap gap-2 pt-1 border-t border-gray-200/60 dark:border-gray-800"
			>
				<div class="flex items-center gap-1.5 flex-wrap">
					<span class="text-[10px] uppercase font-semibold text-gray-400"
						>{$i18n.t('Quick Templates')}:</span
					>
					<button
						type="button"
						class="px-2 py-0.5 rounded bg-gray-200/70 dark:bg-gray-800 hover:bg-gray-300 dark:hover:bg-gray-700 text-[11px] transition"
						on:click={() => {
							params.system = $i18n.t(
								'You are an executive summary assistant. Read the attached document and provide a bulleted summary highlighting key takeaways and actionable insights.'
							);
						}}
					>
						{$i18n.t('Summarize')}
					</button>
					<button
						type="button"
						class="px-2 py-0.5 rounded bg-gray-200/70 dark:bg-gray-800 hover:bg-gray-300 dark:hover:bg-gray-700 text-[11px] transition"
						on:click={() => {
							params.system = $i18n.t(
								'You are a data extraction specialist. Extract all tables, numbers, dates, and key metrics from the attached document.'
							);
						}}
					>
						{$i18n.t('Extract Data')}
					</button>
				</div>

				{#if params?.system}
					<div class="flex items-center gap-2">
						<button
							type="button"
							class="text-[11px] text-gray-400 hover:text-red-500 transition"
							on:click={() => {
								params.system = '';
								toast.info($i18n.t('File system prompt cleared.'));
							}}
						>
							{$i18n.t('Clear')}
						</button>
						<button
							type="button"
							class="px-2.5 py-1 rounded bg-emerald-600 hover:bg-emerald-500 text-white text-[11px] font-medium transition"
							on:click={() => {
								showFileSystemPrompt = false;
								toast.success($i18n.t('File system prompt saved and applied!'));
							}}
						>
							{$i18n.t('Apply & Save')}
						</button>
					</div>
				{:else}
					<button
						type="button"
						class="px-2.5 py-1 rounded bg-emerald-600 hover:bg-emerald-500 text-white text-[11px] font-medium transition"
						on:click={() => {
							showFileSystemPrompt = false;
						}}
					>
						{$i18n.t('Done')}
					</button>
				{/if}
			</div>
		</div>
	{/if}
{/if}
