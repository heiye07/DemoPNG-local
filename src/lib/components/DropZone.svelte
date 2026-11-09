<script lang="ts">
	import { Upload, X, Image } from 'lucide-svelte';

	let {
		screenshotUrl = null,
		index = 0,
		onupload,
		onremove,
		onuploadmultiple
	}: {
		screenshotUrl: string | null;
		index?: number;
		onupload: (file: File) => void;
		onremove: () => void;
		onuploadmultiple?: (files: File[]) => void;
	} = $props();

	let dragging = $state(false);
	let fileInput: HTMLInputElement;

	function uploadFiles(files: File[]) {
		if (files.length > 1 && onuploadmultiple) {
			onuploadmultiple(files);
		} else if (files.length === 1) {
			onupload(files[0]);
		}
	}

	function handlePaste(event: ClipboardEvent) {
		if (event.defaultPrevented) return;
		const target = event.target;
		if (
			target instanceof HTMLElement &&
			target.closest('input, textarea, [contenteditable]:not([contenteditable="false"])')
		) return;

		const files = Array.from(event.clipboardData?.items ?? [])
			.filter((item) => item.kind === 'file' && item.type.startsWith('image/'))
			.map((item) => item.getAsFile())
			.filter((file): file is File => file !== null);
		if (files.length === 0) return;

		event.preventDefault();
		uploadFiles(files);
	}

	function handleDrop(e: DragEvent) {
		e.preventDefault();
		dragging = false;
		const files = Array.from(e.dataTransfer?.files ?? []).filter((f) => f.type.startsWith('image/'));
		uploadFiles(files);
	}

	function handleDragOver(e: DragEvent) {
		e.preventDefault();
		dragging = true;
	}

	function handleDragLeave() {
		dragging = false;
	}

	function handleFileSelect(e: Event) {
		const input = e.target as HTMLInputElement;
		const files = Array.from(input.files ?? []).filter((f) => f.type.startsWith('image/'));
		uploadFiles(files);
		input.value = '';
	}

	function handleClick() {
		fileInput?.click();
	}
</script>

<svelte:window onpaste={handlePaste} />

<div class="relative">
	{#if screenshotUrl}
		<div class="group relative overflow-hidden rounded-xl border border-zinc-700 bg-zinc-800">
			<img
				src={screenshotUrl}
				alt="Screenshot {index + 1}"
				class="h-28 w-full object-cover"
			/>
			<button
				onclick={onremove}
				class="absolute right-1.5 top-1.5 rounded-full bg-zinc-900/80 p-1 opacity-0 transition-opacity group-hover:opacity-100 hover:bg-red-600"
			>
				<X size={14} />
			</button>
		</div>
	{:else}
		<button
			class="flex h-28 w-full flex-col items-center justify-center gap-2 rounded-xl border-2 border-dashed transition-colors
				{dragging ? 'border-pink-500 bg-pink-500/10' : 'border-zinc-700 bg-zinc-800/50 hover:border-zinc-600 hover:bg-zinc-800'}"
			ondrop={handleDrop}
			ondragover={handleDragOver}
			ondragleave={handleDragLeave}
			onclick={handleClick}
		>
			{#if dragging}
				<Image size={20} class="text-pink-400" />
				<span class="text-xs text-pink-400">Drop image(s)</span>
			{:else}
				<Upload size={20} class="text-zinc-500" />
				<span class="text-xs text-zinc-500">拖入或点击选择图片</span>
			{/if}
		</button>
	{/if}
	<p class="mt-2 text-[11px] text-zinc-500">也可按 Ctrl+V / ⌘V 粘贴图片，替换当前截图</p>
	<input
		bind:this={fileInput}
		type="file"
		accept="image/*"
		multiple
		class="hidden"
		onchange={handleFileSelect}
	/>
</div>
