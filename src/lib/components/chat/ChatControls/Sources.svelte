<script lang="ts">
	import { tick } from 'svelte';
	import { showControls, showSources, sourcesPanel } from '$lib/stores';

	import XMark from '$lib/components/icons/XMark.svelte';

	export let overlay = false;

	let panelEl: HTMLElement | null = null;
	let expanded: Set<number> = new Set();
	let focusIdx: number | null = null;
	// favicon 로드 실패(내부망에서 구글 favicon 서비스 차단 등) 시 글자 아바타 폴백
	let iconFailed: Set<number> = new Set();

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

	// 발췌는 평문으로만 — 마크다운 렌더링은 출처마다(법령 조문 제목·목록, 웹 발췌)
	// 타이포를 제각각 흔들어 카드 통일감을 깨므로, 패널은 스니펫 평문이고
	// 전문 마크다운은 모달이 담당한다.
	const excerptText = (citation: any) =>
		(citation.document ?? [])
			.join('\n\n')
			.replace(/\n{3,}/g, '\n\n')
			.trim();

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
			iconFailed = new Set();
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
					<div class="flex items-center gap-1.5 text-xs text-gray-500 dark:text-gray-400 min-w-0">
						<span class="font-medium bg-gray-50 dark:bg-gray-850 rounded px-1 shrink-0">
							{idx + 1}
						</span>
						{#if citation.source?.url}
							{#if !iconFailed.has(idx)}
								<img
									src="https://www.google.com/s2/favicons?sz=32&domain={citation.source.url}"
									alt=""
									class="size-4 rounded-sm shrink-0"
									on:error={() => {
										iconFailed.add(idx);
										iconFailed = iconFailed;
									}}
								/>
							{:else}
								<span
									class="size-4 rounded-sm shrink-0 bg-gray-100 dark:bg-gray-800 text-[9px] font-semibold flex items-center justify-center"
								>
									{typeBadge(citation.source?.url).charAt(0).toUpperCase()}
								</span>
							{/if}
						{/if}
						<span class="truncate">{typeBadge(citation.source?.url)}</span>
					</div>
					{#if citation.source?.url}
						<a
							class="mt-1 block text-sm font-medium leading-snug line-clamp-2 text-gray-900 dark:text-gray-100 hover:underline"
							href={citation.source.url}
							target="_blank"
							rel="noopener noreferrer"
						>
							{decodeString(citation.source?.name ?? '출처')}
						</a>
					{:else}
						<div
							class="mt-1 text-sm font-medium leading-snug line-clamp-2 text-gray-900 dark:text-gray-100"
						>
							{decodeString(citation.source?.name ?? '출처')}
						</div>
					{/if}
					<button
						class="mt-1 w-full text-left"
						aria-expanded={expanded.has(idx)}
						on:click={() => toggle(idx)}
					>
						<div
							class="text-xs leading-relaxed text-gray-500 dark:text-gray-400 whitespace-pre-line {expanded.has(
								idx
							)
								? ''
								: 'line-clamp-4'}"
						>
							{excerptText(citation)}
						</div>
					</button>
				</div>
			{/each}
		</div>
	</div>
{/if}
