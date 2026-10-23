<script lang="ts">
	import { tick } from 'svelte';
	import { showControls, showSources, sourcesPanel } from '$lib/stores';

	import XMark from '$lib/components/icons/XMark.svelte';
	import CitationModal from '$lib/components/chat/Messages/Citations/CitationModal.svelte';

	let panelEl: HTMLElement | null = null;
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

	// 발췌는 평문 4줄 미리보기로만 — 마크다운 렌더링은 출처마다(법령 조문 제목·목록, 웹 발췌)
	// 타이포를 제각각 흔들어 카드 통일감을 깨므로, 패널은 스니펫 평문이고
	// 전문 마크다운은 카드를 눌러 여는 출처 창(모달)이 담당한다.
	// 모달의 CONTENT_PREVIEW_LIMIT와 같은 값 — 아주 긴 발췌로 DOM이 비대해지는 것을
	// 막는다. 잘리면 말줄임만 붙인다(전문은 모달에서 본다).
	const EXCERPT_LIMIT = 10000;

	const excerptText = (citation: any) => {
		const text = (citation.document ?? [])
			.join('\n\n')
			.replace(/\n{3,}/g, '\n\n')
			.trim();
		return text.length > EXCERPT_LIMIT ? `${text.slice(0, EXCERPT_LIMIT)} …` : text;
	};

	// 카드 클릭 → 그 출처의 출처 창. 메시지별 CitationModal(Citations.svelte)은 메시지 안에 있어
	// 패널에서 닿지 않으므로 패널이 자기 인스턴스를 둔다. 패널은 그대로 두고, 누른 카드를 표시한다.
	let showCitationModal = false;
	let selectedCitation: any = null;

	const openCitation = (idx: number) => {
		selectedCitation = $sourcesPanel?.citations?.[idx] ?? null;
		if (!selectedCitation) return;
		focusIdx = idx;
		showCitationModal = true;
	};

	// 패널이 열리거나 대상(메시지·focus)이 바뀔 때만 강조 카드를 리셋하고 스크롤한다.
	// $sourcesPanel 자체를 조건으로 쓰면 스트리밍 동기화(citations 갱신)마다
	// 리셋·재스크롤이 일어나므로 identity 키로 가드한다.
	let panelKey = '';
	$: if ($sourcesPanel) {
		const key = `${$sourcesPanel.messageId}#${$sourcesPanel.focusIdx ?? ''}`;
		if (key !== panelKey) {
			panelKey = key;
			focusIdx = $sourcesPanel.focusIdx ?? null;
			showCitationModal = false;
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
			{#each $sourcesPanel.citations as citation, idx}
				<!-- 카드 전체가 버튼 — 어디를 눌러도(Enter·Space 포함) 출처 창이 열린다. 웹·법령 출처의
				     외부 링크는 출처 창 제목에서 연다(카드 안에 링크를 두면 두 동작이 겹친다). -->
				<div
					id="source-card-{idx}"
					role="button"
					tabindex="0"
					aria-label="출처 {idx + 1} 자세히 보기"
					class="rounded-xl border p-3 shrink-0 cursor-pointer transition hover:bg-gray-50 dark:hover:bg-gray-850/50 {focusIdx ===
					idx
						? 'border-gray-300 dark:border-gray-600'
						: 'border-gray-100 dark:border-gray-850 hover:border-gray-200 dark:hover:border-gray-700'}"
					on:click={() => openCitation(idx)}
					on:keydown={(e) => {
						if (e.key === 'Enter' || e.key === ' ') {
							e.preventDefault();
							openCitation(idx);
						}
					}}
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
					<div
						class="mt-1 text-sm font-medium leading-snug line-clamp-2 text-gray-900 dark:text-gray-100"
					>
						{decodeString(citation.source?.name ?? '출처')}
					</div>
					<div
						class="mt-1 text-xs leading-relaxed text-gray-500 dark:text-gray-400 whitespace-pre-line line-clamp-4"
					>
						{excerptText(citation)}
					</div>
				</div>
			{/each}
		</div>
	</div>

	<!-- 「패널로 보기」(onOpenPanel)는 넘기지 않는다 — 이미 패널에서 열었다. -->
	<CitationModal bind:show={showCitationModal} citation={selectedCitation} showRelevance={false} />
{/if}
