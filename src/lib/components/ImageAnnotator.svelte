<script lang="ts">
	import { onMount } from 'svelte';

	type Tool = 'rectangle' | 'arrow' | 'mosaic';
	type Point = { x: number; y: number };
	type Mark = { tool: Tool; start: Point; end: Point; color: string; width: number };
	let { source, onsave, onclose }: {
		source: string;
		onsave: (file: File) => void;
		onclose: () => void;
	} = $props();
	let dialog: HTMLDialogElement;
	let canvas: HTMLCanvasElement;
	let image: HTMLImageElement;
	let tool = $state<Tool>('rectangle');
	let color = $state('#ef4444');
	let width = $state(6);
	let marks = $state<Mark[]>([]);
	let undone = $state<Mark[]>([]);
	let draft: Mark | null = null;
	let pointerId: number | null = null;
	let ready = $state(false);
	let saving = $state(false);
	let error = $state('');
	const tools: { id: Tool; label: string }[] = [
		{ id: 'rectangle', label: '矩形框' },
		{ id: 'arrow', label: '箭头' },
		{ id: 'mosaic', label: '马赛克' }
	];

	onMount(() => {
		dialog.showModal();
		image = new Image();
		let active = true;
		image.onload = () => {
			if (!active) return;
			canvas.width = image.naturalWidth;
			canvas.height = image.naturalHeight;
			ready = true;
			render();
		};
		image.onerror = () => { if (active) error = '图片加载失败，请关闭后重试。'; };
		image.src = source;
		return () => { active = false; };
	});

	function draw(context: CanvasRenderingContext2D, mark: Mark) {
		const { start, end } = mark;
		const left = Math.min(start.x, end.x);
		const top = Math.min(start.y, end.y);
		const w = Math.abs(end.x - start.x);
		const h = Math.abs(end.y - start.y);
		context.strokeStyle = mark.color;
		context.fillStyle = mark.color;
		context.lineWidth = mark.width;
		context.lineJoin = 'round';
		context.lineCap = 'round';
		if (mark.tool === 'rectangle') {
			context.strokeRect(left, top, w, h);
		} else if (mark.tool === 'arrow') {
			const angle = Math.atan2(end.y - start.y, end.x - start.x);
			const head = Math.min(Math.hypot(w, h) * 0.45, mark.width * 4);
			context.beginPath();
			context.moveTo(start.x, start.y);
			context.lineTo(end.x, end.y);
			context.stroke();
			context.beginPath();
			context.moveTo(end.x, end.y);
			context.lineTo(end.x - head * Math.cos(angle - Math.PI / 6), end.y - head * Math.sin(angle - Math.PI / 6));
			context.lineTo(end.x - head * Math.cos(angle + Math.PI / 6), end.y - head * Math.sin(angle + Math.PI / 6));
			context.closePath();
			context.fill();
		} else if (w > 0 && h > 0) {
			const sample = document.createElement('canvas');
			const blockSize = mark.width * 4;
			sample.width = Math.max(1, Math.ceil(w / blockSize));
			sample.height = Math.max(1, Math.ceil(h / blockSize));
			const small = sample.getContext('2d')!;
			small.drawImage(image, left, top, w, h, 0, 0, sample.width, sample.height);
			context.save();
			context.imageSmoothingEnabled = false;
			context.clearRect(left, top, w, h);
			context.drawImage(sample, 0, 0, sample.width, sample.height, left, top, w, h);
			context.restore();
		}
	}

	function render() {
		if (!ready) return;
		const context = canvas.getContext('2d')!;
		context.clearRect(0, 0, canvas.width, canvas.height);
		context.drawImage(image, 0, 0);
		for (const mark of marks) draw(context, mark);
		if (draft) draw(context, draft);
	}

	function point(event: PointerEvent): Point {
		const rect = canvas.getBoundingClientRect();
		return {
			x: Math.max(0, Math.min(canvas.width, (event.clientX - rect.left) * canvas.width / rect.width)),
			y: Math.max(0, Math.min(canvas.height, (event.clientY - rect.top) * canvas.height / rect.height))
		};
	}

	function start(event: PointerEvent) {
		if (!ready || saving || event.button !== 0 || pointerId !== null) return;
		event.preventDefault();
		pointerId = event.pointerId;
		canvas.setPointerCapture(event.pointerId);
		const position = point(event);
		draft = { tool, start: position, end: position, color, width };
	}

	function move(event: PointerEvent) {
		if (!draft || event.pointerId !== pointerId) return;
		draft.end = point(event);
		render();
	}

	function finish(event: PointerEvent) {
		if (!draft || event.pointerId !== pointerId) return;
		draft.end = point(event);
		if (Math.hypot(draft.end.x - draft.start.x, draft.end.y - draft.start.y) >= 3) {
			marks = [...marks, draft];
			undone = [];
		}
		cancelStroke();
	}

	function cancelStroke() {
		draft = null;
		const captured = pointerId;
		pointerId = null;
		if (captured !== null && canvas.hasPointerCapture(captured)) canvas.releasePointerCapture(captured);
		render();
	}

	function undo() {
		const last = marks.at(-1);
		if (!last) return;
		undone = [...undone, last];
		marks = marks.slice(0, -1);
		render();
	}

	function redo() {
		const last = undone.at(-1);
		if (!last) return;
		marks = [...marks, last];
		undone = undone.slice(0, -1);
		render();
	}

	async function save() {
		if (!ready || saving) return;
		saving = true;
		error = '';
		cancelStroke();
		try {
			const blob = await new Promise<Blob>((resolve, reject) => {
				canvas.toBlob((result) => result ? resolve(result) : reject(new Error('PNG encoding failed')), 'image/png');
			});
			onsave(new File([blob], 'annotated-image.png', { type: 'image/png' }));
			onclose();
		} catch {
			error = '保存失败，请重试。';
		} finally {
			saving = false;
		}
	}
</script>

<dialog bind:this={dialog} aria-label="批注图片" class="m-auto max-h-[95vh] w-[min(1100px,95vw)] rounded-xl border border-zinc-700 bg-zinc-900 p-5 text-zinc-100 backdrop:bg-black/75" oncancel={(event) => { event.preventDefault(); if (!saving) onclose(); }} onpaste={(event) => event.stopPropagation()}>
	<div class="mb-4 flex flex-wrap items-center gap-3">
		<h2 class="mr-auto font-semibold">批注图片</h2>
		<button class="rounded bg-zinc-800 px-3 py-2 text-sm disabled:opacity-40" disabled={saving} onclick={onclose}>取消</button>
		<button class="rounded bg-pink-600 px-3 py-2 text-sm disabled:opacity-40" disabled={!ready || saving} onclick={save}>{saving ? '保存中…' : '保存批注'}</button>
	</div>
	<div class="mb-3 flex flex-wrap items-center gap-2 text-sm">
		{#each tools as option}
			<button class="rounded px-3 py-2 {tool === option.id ? 'bg-pink-600' : 'bg-zinc-800'}" aria-pressed={tool === option.id} onclick={() => tool = option.id}>{option.label}</button>
		{/each}
		<label class="flex items-center gap-2 px-2">颜色 <input type="color" bind:value={color} disabled={tool === 'mosaic'} class="h-8 w-9 disabled:opacity-40" /></label>
		<label class="flex items-center gap-2">{tool === 'mosaic' ? '色块大小' : '线条粗细'} <input type="range" min="2" max="24" step="1" bind:value={width} class="w-24" /></label>
		<button class="rounded bg-zinc-800 px-3 py-2 disabled:opacity-40" disabled={!marks.length || saving} onclick={undo}>撤销</button>
		<button class="rounded bg-zinc-800 px-3 py-2 disabled:opacity-40" disabled={!undone.length || saving} onclick={redo}>重做</button>
	</div>
	<p class="mb-3 text-xs text-zinc-400">在图片上按住并拖动进行批注，保存后应用到当前截图。</p>
	{#if error}<p role="alert" class="mb-3 text-sm text-red-400">{error}</p>{/if}
	{#if !ready && !error}<p class="py-4 text-sm">正在加载图片…</p>{/if}
	<div class="flex justify-center overflow-auto rounded bg-zinc-950 p-2">
		<canvas bind:this={canvas} aria-label="图片批注画布" class="block max-h-[62vh] max-w-full touch-none {ready ? 'cursor-crosshair' : 'invisible'}" onpointerdown={start} onpointermove={move} onpointerup={finish} onpointercancel={cancelStroke} onlostpointercapture={cancelStroke}></canvas>
	</div>
</dialog>
