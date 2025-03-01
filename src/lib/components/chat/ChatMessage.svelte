<script lang="ts">
	import { marked } from "marked";
	import markedKatex from "marked-katex-extension";
	import type { Message } from "$lib/types/Message";
	import { afterUpdate, createEventDispatcher } from "svelte";
	import { deepestChild } from "$lib/utils/deepestChild";
	import { page } from "$app/stores";

	import CodeBlock from "../CodeBlock.svelte";
	import CopyToClipBoardBtn from "../CopyToClipBoardBtn.svelte";
	import IconLoading from "../icons/IconLoading.svelte";
	import CarbonRotate360 from "~icons/carbon/rotate-360";
	import CarbonDownload from "~icons/carbon/download";
	import CarbonThumbsUp from "~icons/carbon/thumbs-up";
	import CarbonThumbsDown from "~icons/carbon/thumbs-down";
	import { PUBLIC_SEP_TOKEN } from "$lib/constants/publicSepToken";
	import type { Model } from "$lib/types/Model";

	import OpenWebSearchResults from "../OpenWebSearchResults.svelte";
	import type { WebSearchUpdate } from "$lib/types/MessageUpdate";

	function sanitizeMd(md: string) {
		let ret = md
			.replace(/<\|[a-z]*$/, "")
			.replace(/<\|[a-z]+\|$/, "")
			.replace(/<$/, "")
			.replaceAll(PUBLIC_SEP_TOKEN, " ")
			.replaceAll(/<\|[a-z]+\|>/g, " ")
			.replaceAll(/<br\s?\/?>/gi, "\n")
			.replaceAll("<", "&lt;")
			.trim();

		for (const stop of [...(model.parameters?.stop ?? []), "<|endoftext|>"]) {
			if (ret.endsWith(stop)) {
				ret = ret.slice(0, -stop.length).trim();
			}
		}

		return ret;
	}
	function unsanitizeMd(md: string) {
		return md.replaceAll("&lt;", "<");
	}

	export let model: Model;
	export let question = "";
	export let message: Message;
	export let loading = false;
	export let isAuthor = true;
	export let readOnly = false;
	export let isTapped = false;

	export let webSearchMessages: WebSearchUpdate[];

	const dispatch = createEventDispatcher<{
		retry: { content: string; id: Message["id"] };
		vote: { score: Message["score"]; id: Message["id"] };
	}>();

	let contentEl: HTMLElement;
	let loadingEl: IconLoading;
	let pendingTimeout: ReturnType<typeof setTimeout>;
	let isCopied = false;

	const renderer = new marked.Renderer();
	// For code blocks with simple backticks
	renderer.codespan = (code) => {
		// Unsanitize double-sanitized code
		return `<code>${code.replaceAll("&amp;", "&")}</code>`;
	};

	// eslint-disable-next-line @typescript-eslint/no-unused-vars
	const { extensions, ...defaults } = marked.getDefaults() as marked.MarkedOptions & {
		// eslint-disable-next-line @typescript-eslint/no-explicit-any
		extensions: any;
	};
	const options: marked.MarkedOptions = {
		...defaults,
		gfm: true,
		breaks: true,
		renderer,
	};

	marked.use(
		markedKatex({
			throwOnError: false,
			// output: "html",
		})
	);

	$: tokens = marked.lexer(sanitizeMd(message.content));

	afterUpdate(() => {
		loadingEl?.$destroy();
		clearTimeout(pendingTimeout);

		// Add loading animation to the last message if update takes more than 600ms
		if (loading) {
			pendingTimeout = setTimeout(() => {
				if (contentEl) {
					loadingEl = new IconLoading({
						target: deepestChild(contentEl),
						props: { classNames: "loading inline ml-2" },
					});
				}
			}, 600);
		}
	});

	let searchUpdates: WebSearchUpdate[] = [];

	$: searchUpdates = ((webSearchMessages.length > 0
		? webSearchMessages
		: message.updates?.filter(({ type }) => type === "webSearch")) ?? []) as WebSearchUpdate[];

	$: downloadLink =
		message.from === "user" ? `${$page.url.pathname}/message/${message.id}/prompt` : undefined;

	let webSearchIsDone = true;

	$: webSearchIsDone =
		searchUpdates.length > 0 && searchUpdates[searchUpdates.length - 1].messageType === "sources";

	$: webSearchSources =
		searchUpdates &&
		searchUpdates?.filter(({ messageType }) => messageType === "sources")?.[0]?.sources;

	$: if (isCopied) {
		setTimeout(() => {
			isCopied = false;
		}, 1000);
	}
</script>

{#if message.from === "assistant"}
	<div
		class="group relative -mb-8 flex items-start justify-start gap-4 pb-8 leading-relaxed"
		role="presentation"
		on:click={() => (isTapped = !isTapped)}
		on:keypress={() => (isTapped = !isTapped)}
	>
		<img
			alt=""
			src="https://huggingface.co/avatars/2edb18bd0206c16b433841a47f53fa8e.svg"
			class="mt-5 h-3 w-3 flex-none select-none rounded-full shadow-lg"
		/>
		<div
			class="relative min-h-[calc(2rem+theme(spacing[3.5])*2)] min-w-[60px] break-words rounded-2xl border border-gray-100 bg-gradient-to-br from-gray-50 px-5 py-3.5 text-gray-600 prose-pre:my-2 dark:border-gray-800 dark:from-gray-800/40 dark:text-gray-300"
		>
			{#if searchUpdates && searchUpdates.length > 0}
				<OpenWebSearchResults
					classNames={tokens.length ? "mb-3.5" : ""}
					webSearchMessages={searchUpdates}
					loading={!(searchUpdates[searchUpdates.length - 1]?.messageType === "sources")}
				/>
			{/if}
			{#if !message.content && (webSearchIsDone || (webSearchMessages && webSearchMessages.length === 0))}
				<IconLoading />
			{/if}

			<div
				class="prose max-w-none max-sm:prose-sm dark:prose-invert prose-headings:font-semibold prose-h1:text-lg prose-h2:text-base prose-h3:text-base prose-pre:bg-gray-800 dark:prose-pre:bg-gray-900"
				bind:this={contentEl}
			>
				{#each tokens as token}
					{#if token.type === "code"}
						<CodeBlock lang={token.lang} code={unsanitizeMd(token.text)} />
					{:else}
						<!-- eslint-disable-next-line svelte/no-at-html-tags -->
						{@html marked.parse(token.raw, options)}
					{/if}
				{/each}
			</div>
			<!-- Web Search sources -->
			{#if webSearchSources?.length}
				<div class="mt-4 flex flex-wrap items-center gap-x-2 gap-y-1.5 text-sm">
					<div class="text-gray-400">Sources:</div>
					{#each webSearchSources as { link, title, hostname }}
						<a
							class="flex items-center gap-2 whitespace-nowrap rounded-lg border bg-white px-2 py-1.5 leading-none hover:border-gray-300 dark:border-gray-800 dark:bg-gray-900 dark:hover:border-gray-700"
							href={link}
							target="_blank"
						>
							<img
								class="h-3.5 w-3.5 rounded"
								src="https://www.google.com/s2/favicons?sz=64&domain_url={hostname}"
								alt="{title} favicon"
							/>
							<div>{hostname.replace(/^www\./, "")}</div>
						</a>
					{/each}
				</div>
			{/if}
		</div>
		{#if isAuthor && !loading && message.content}
			<div
				class="absolute bottom-1 right-0 flex max-md:transition-all md:bottom-0 md:group-hover:visible md:group-hover:opacity-100
					{message.score ? 'visible opacity-100' : 'invisible max-md:-translate-y-4 max-md:opacity-0'}
					{isTapped || isCopied ? 'max-md:visible max-md:translate-y-0 max-md:opacity-100' : ''}
				"
			>
				<button
					class="btn rounded-sm p-1 text-sm text-gray-400 focus:ring-0 hover:text-gray-500 dark:text-gray-400 dark:hover:text-gray-300
					{message.score && message.score > 0
						? 'text-green-500 hover:text-green-500 dark:text-green-400 hover:dark:text-green-400'
						: ''}"
					title={message.score === 1 ? "Remove +1" : "+1"}
					type="button"
					on:click={() => {
							const { content, id } = message;
							const score = message.score === 1 ? 0 : 1

							dispatch("vote", { score , id })
							const messagePack = JSON.stringify({
									event: 'chat-ui_vote',
									detail: {
										id,
										question,
										answer: content,
										feedback: score === 1 ? "UP": "NONE"
									}
							});
							window.parent.postMessage(messagePack, '*');
						}
					}
				>
					<CarbonThumbsUp class="h-[1.14em] w-[1.14em]" />
				</button>
				<button
					class="btn rounded-sm p-1 text-sm text-gray-400 focus:ring-0 hover:text-gray-500 dark:text-gray-400 dark:hover:text-gray-300
					{message.score && message.score < 0
						? 'text-red-500 hover:text-red-500 dark:text-red-400 hover:dark:text-red-400'
						: ''}"
					title={message.score === -1 ? "Remove -1" : "-1"}
					type="button"
					on:click={() => {
							const { content, id } = message;
							const score = message.score === -1 ? 0 : -1;

							dispatch("vote", { score, id })
							const messagePack = JSON.stringify({
									event: 'chat-ui_vote',
									detail: {
										id,
										question,
										answer: content,
										feedback: score === -1 ? "DOWN": "NONE"
									}
							});
							window.parent.postMessage(messagePack, '*');
						}
					}
				>
					<CarbonThumbsDown class="h-[1.14em] w-[1.14em]" />
				</button>
				<CopyToClipBoardBtn
					on:click={() => {
						isCopied = true;
					}}
					classNames="ml-1.5 !rounded-sm !p-1 !text-sm !text-gray-400 focus:!ring-0 hover:!text-gray-500 dark:!text-gray-400 dark:hover:!text-gray-300 !border-none !shadow-none"
					value={message.content}
				/>
			</div>
		{/if}
	</div>
{/if}
{#if message.from === "user"}
	<div class="group relative flex items-start justify-start gap-4 max-sm:text-sm">
		<div class="flex flex-col">
			{#if message.files && message.files.length > 0}
				<div class="mx-auto grid w-fit grid-cols-2 gap-5 px-5">
					{#each message.files as file}
						<!-- handle the case where this is a hash that points to an image in the db, hash is always 64 char long -->
						{#if file.length === 64}
							<img
								src={$page.url.pathname + "/output/" + file}
								alt="input from user"
								class="my-2 aspect-auto max-h-48 rounded-lg shadow-lg"
							/>
						{:else}
							<!-- handle the case where this is a base64 encoded image -->
							<img
								src={"data:image/*;base64," + file}
								alt="input from user"
								class="my-2 aspect-auto max-h-48 rounded-lg shadow-lg"
							/>
						{/if}
					{/each}
				</div>
			{/if}

			<div
				class="max-w-full whitespace-break-spaces break-words rounded-2xl px-5 py-3.5 text-gray-500 dark:text-gray-400"
			>
				{message.content.trim()}
			</div>
			{#if !loading}
				<div class="absolute right-0 top-3.5 flex gap-2 lg:-right-2">
					{#if downloadLink}
						<a
							class="rounded-lg border border-gray-100 p-1 text-xs text-gray-400 group-hover:block hover:text-gray-500 md:hidden dark:border-gray-800 dark:text-gray-400 dark:hover:text-gray-300"
							title="Download prompt and parameters"
							type="button"
							target="_blank"
							href={downloadLink}
						>
							<CarbonDownload />
						</a>
					{/if}
					{#if !readOnly}
						<button
							class="cursor-pointer rounded-lg border border-gray-100 p-1 text-xs text-gray-400 group-hover:block hover:text-gray-500 md:hidden lg:-right-2 dark:border-gray-800 dark:text-gray-400 dark:hover:text-gray-300"
							title="Retry"
							type="button"
							on:click={() => dispatch("retry", { content: message.content, id: message.id })}
						>
							<CarbonRotate360 />
						</button>
					{/if}
				</div>
			{/if}
		</div>
	</div>
{/if}
