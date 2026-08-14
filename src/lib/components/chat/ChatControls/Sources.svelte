<script lang="ts">
	import { tick } from 'svelte';
	import { showControls, showSources, sourcesPanel } from '$lib/stores';

	import Markdown from '$lib/components/chat/Messages/Markdown.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';

	export let overlay = false;

	let panelEl: HTMLElement | null = null;
	let expanded: Set<number> = new Set();
	let focusIdx: number | null = null;

	const decodeString = (str: string) => {
		try {
			return decodeURIComponent(str);
		} catch {
			return str;
		}
	};

	// 유형 뱃지 — 외부 favicon 요청 없이 URL로만 판별(망분리)
	const typeBadge = (url: string | undefined) => {
		if (!url) return '문서';
		try {
			const host = new URL(url).hostname;
			return host.includes('law.go.kr') ? '법령' : host;
		} catch {
			return '문서';
		}
	};

	const toggle = (idx: number) => {
		if (expanded.has(idx)) {
			expanded.delete(idx);
		} else {
			expanded.add(idx);
		}
		expanded = expanded;
	};

	// 패널이 열리거나 대상(메시지·focus)이 바뀔 때만 접힘 상태를 리셋하고 스크롤한다.
	// $sourcesPanel 자체를 조건으로 쓰면 스트리밍 동기화(citations 갱신)마다
	// 리셋·재스크롤이 일어나므로 identity 키로 가드한다.
	let panelKey = '';
	$: if ($sourcesPanel) {
		const key = `${$sourcesPanel.messageId}#${$sourcesPanel.focusIdx ?? ''}`;
		if (key !== panelKey) {
			panelKey = key;
			focusIdx = $sourcesPanel.focusIdx ?? null;
			expanded = new Set(focusIdx !== null ? [focusIdx] : []);
			tick().then(() => {
				if (focusIdx !== null) {
					panelEl
						?.querySelector(`#source-card-${focusIdx}`)
						?.scrollIntoView({ block: 'start', behavior: 'smooth' });
				}
			});
		}
	}
</script>

{#if $sourcesPanel}
	<div class="h-full w-full flex flex-col" bind:this={panelEl}>
		<div class="flex justify-between items-center py-3 px-4 shrink-0">
			<div class="font-medium text-gray-900 dark:text-white">
				출처 {$sourcesPanel.citations.length}건
			</div>
			<button
				class="self-center pointer-events-auto p-1 rounded-full bg-white dark:bg-gray-850"
				aria-label="출처 패널 닫기"
				on:click={() => {
					showControls.set(false);
					showSources.set(false);
					sourcesPanel.set(null);
				}}
			>
				<XMark className="size-3.5 text-gray-900 dark:text-white" />
			</button>
		</div>

		<div class="flex-1 min-h-0 overflow-y-auto px-3 pb-4 flex flex-col gap-2 relative">
			{#if overlay}
				<div class="absolute inset-0 z-10"></div>
			{/if}

			{#each $sourcesPanel.citations as citation, idx}
				<div
					id="source-card-{idx}"
					class="rounded-xl border p-3 shrink-0 {focusIdx === idx
						? 'border-gray-300 dark:border-gray-600'
						: 'border-gray-100 dark:border-gray-850'}"
				>
					<div class="flex items-center gap-2 mb-1.5 min-w-0">
						<div
							class="text-xs font-medium bg-gray-50 dark:bg-gray-850 rounded-md px-1.5 py-0.5 shrink-0"
						>
							{idx + 1}
						</div>
						{#if citation.source?.url}
							<a
								class="text-sm font-medium truncate hover:underline dark:text-gray-100"
								href={citation.source.url}
								target="_blank"
								rel="noopener noreferrer"
							>
								{decodeString(citation.source?.name ?? '출처')}
							</a>
						{:else}
							<div class="text-sm font-medium truncate dark:text-gray-100">
								{decodeString(citation.source?.name ?? '출처')}
							</div>
						{/if}
						<div class="text-xs text-gray-500 dark:text-gray-400 shrink-0 ml-auto">
							{typeBadge(citation.source?.url)}
						</div>
					</div>

					<div
						class="text-sm prose dark:prose-invert markdown-prose-sm min-w-full max-w-full overflow-hidden {expanded.has(
							idx
						)
							? ''
							: 'max-h-24'}"
					>
						{#each citation.document as doc, docIdx}
							{#if docIdx > 0}
								<hr class="my-2" />
							{/if}
							<Markdown content={doc} id="source-panel-{idx}-{docIdx}" />
						{/each}
					</div>
					<button
						class="mt-1 text-xs text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition"
						on:click={() => toggle(idx)}
					>
						{expanded.has(idx) ? '접기' : '더 보기'}
					</button>
				</div>
			{/each}
		</div>
	</div>
{/if}
