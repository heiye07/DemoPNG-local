<script lang="ts">
	import { onMount, tick } from 'svelte';

	type Tool = 'rectangle' | 'arrow' | 'mosaic';
	type ArrowStyle = 'open' | 'solid' | 'double' | 'block';
	type Point = { x: number; y: number };
	type Mark = { tool: Tool; start: Point; end: Point; color: string; width: number; radius: number; arrowStyle: ArrowStyle };
	let { source, onsave, onclose }: {
		source: string;
		onsave: (file: File) => void;
		onclose: () => void;
	} = $props();
	let dialog: HTMLDialogElement;
	let canvas: HTMLCanvasElement;
	let viewport: HTMLDivElement;
	let image: HTMLImageElement;
	let tool = $state<Tool | 'pan'>('rectangle');
	let color = $state('#ef4444');
	let width = $state(6);
	let radius = $state(16);
	let arrowStyle = $state<ArrowStyle>('open');
	let maximized = $state(false);
	let imageSize = $state({ width: 1, height: 1 });
	let viewportSize = $state({ width: 1, height: 1 });
	let zoom = $state<number | 'fit'>('fit');
	let scale = $derived(zoom === 'fit'
		? Math.max(0.01, Math.min((viewportSize.width - 32) / imageSize.width, (viewportSize.height - 32) / imageSize.height, 1))
		: zoom);
	let pan: { x: number; y: number; left: number; top: number } | null = null;
	const widths = [2, 6, 12, 24, 40, 64];
	const arrowStyles: { id: ArrowStyle; label: string; symbol: string }[] = [
		{ id: 'open', label: '简洁箭头', symbol: '⟶' },
		{ id: 'solid', label: '实心箭头', symbol: '➜' },
		{ id: 'double', label: '双向箭头', symbol: '⟷' },
		{ id: 'block', label: '粗实心箭头', symbol: '⬆' }
	];
	let marks = $state<Mark[]>([]);
	let undone = $state<Mark[]>([]);
	let draft: Mark | null = null;
	let pointerId: number | null = null;
	let ready = $state(false);
	let saving = $state(false);
	let error = $state('');
	const tools: { id: Tool | 'pan'; label: string }[] = [
		{ id: 'rectangle', label: '矩形框' },
		{ id: 'arrow', label: '箭头' },
		{ id: 'mosaic', label: '马赛克' },
		{ id: 'pan', label: '拖动预览' }
	];

	onMount(() => {
		dialog.showModal();
		const observer = new ResizeObserver(() => {
			viewportSize = { width: viewport.clientWidth, height: viewport.clientHeight };
		});
		observer.observe(viewport);
		image = new Image();
		let active = true;
		image.onload = () => {
			if (!active) return;
			imageSize = { width: image.naturalWidth, height: image.naturalHeight };
			canvas.width = image.naturalWidth;
			canvas.height = image.naturalHeight;
			ready = true;
			render();
		};
		image.onerror = () => { if (active) error = '图片加载失败，请关闭后重试。'; };
		image.src = source;
		return () => { active = false; observer.disconnect(); };
	});

	async function changeZoom(next: number | 'fit') {
		const bounds = canvas.getBoundingClientRect();
		const view = viewport.getBoundingClientRect();
		const centerX = (view.left + viewport.clientWidth / 2 - bounds.left) / bounds.width;
		const centerY = (view.top + viewport.clientHeight / 2 - bounds.top) / bounds.height;
		zoom = next === 'fit' ? next : Math.max(0.1, Math.min(8, next));
		await tick();
		const updated = canvas.getBoundingClientRect();
		viewport.scrollLeft += updated.left + centerX * updated.width - view.left - viewport.clientWidth / 2;
		viewport.scrollTop += updated.top + centerY * updated.height - view.top - viewport.clientHeight / 2;
	}

	function drawArrowHead(context: CanvasRenderingContext2D, tip: Point, angle: number, head: number, filled: boolean) {
		context.beginPath();
		context.moveTo(tip.x - head * Math.cos(angle - Math.PI / 6), tip.y - head * Math.sin(angle - Math.PI / 6));
		context.lineTo(tip.x, tip.y);
		context.lineTo(tip.x - head * Math.cos(angle + Math.PI / 6), tip.y - head * Math.sin(angle + Math.PI / 6));
		if (filled) {
			context.closePath();
			context.fill();
		} else {
			context.stroke();
		}
	}

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
			context.beginPath();
			context.roundRect(left, top, w, h, Math.min(mark.radius, w / 2, h / 2));
			context.stroke();
		} else if (mark.tool === 'arrow') {
			const angle = Math.atan2(end.y - start.y, end.x - start.x);
			const length = Math.hypot(w, h);
			const head = Math.min(length * 0.4, mark.width * 4);
			if (mark.arrowStyle === 'block') {
				// Draw in arrow-local coordinates to keep the shaft and head joined.
				context.save();
				context.translate(start.x, start.y);
				context.rotate(angle);
				const half = Math.min(mark.width, length / 6);
				const neck = Math.max(0, length - head);
				context.beginPath();
				context.moveTo(0, -half);
				context.lineTo(neck, -half);
				context.lineTo(neck, -Math.max(half * 2, head / 2));
				context.lineTo(length, 0);
				context.lineTo(neck, Math.max(half * 2, head / 2));
				context.lineTo(neck, half);
				context.lineTo(0, half);
				context.closePath();
				context.fill();
				context.restore();
			} else {
				const inset = mark.arrowStyle === 'solid' ? head * 0.6 : 0;
				context.beginPath();
				context.moveTo(start.x, start.y);
				context.lineTo(end.x - inset * Math.cos(angle), end.y - inset * Math.sin(angle));
				context.stroke();
				drawArrowHead(context, end, angle, head, mark.arrowStyle === 'solid');
				if (mark.arrowStyle === 'double') drawArrowHead(context, start, angle + Math.PI, head, false);
			}
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
		if (draft) {
			draw(context, draft);
			if (draft.tool === 'mosaic') {
				const left = Math.min(draft.start.x, draft.end.x);
				const top = Math.min(draft.start.y, draft.end.y);
				const w = Math.abs(draft.end.x - draft.start.x);
				const h = Math.abs(draft.end.y - draft.start.y);
				context.save();
				context.lineWidth = 3 / scale;
				context.strokeStyle = '#ffffff';
				context.strokeRect(left, top, w, h);
				context.lineWidth = 2 / scale;
				context.strokeStyle = '#ec4899';
				context.setLineDash([6 / scale, 4 / scale]);
				context.strokeRect(left, top, w, h);
				context.restore();
			}
		}
	}

	function point(event: PointerEvent): Point {
		const rect = canvas.getBoundingClientRect();
		return {
			x: Math.max(0, Math.min(canvas.width, (event.clientX - rect.left) * canvas.width / rect.width)),
			y: Math.max(0, Math.min(canvas.height, (event.clientY - rect.top) * canvas.height / rect.height))
		};
	}

	function start(event: PointerEvent) {
		if (!ready || saving || (event.button !== 0 && event.button !== 1) || pointerId !== null) return;
		event.preventDefault();
		pointerId = event.pointerId;
		canvas.setPointerCapture(event.pointerId);
		if (tool === 'pan' || event.button === 1) {
			pan = { x: event.clientX, y: event.clientY, left: viewport.scrollLeft, top: viewport.scrollTop };
			return;
		}
		const position = point(event);
		draft = { tool, start: position, end: position, color, width, radius, arrowStyle };
	}

	function move(event: PointerEvent) {
		if (event.pointerId !== pointerId) return;
		if (pan) {
			viewport.scrollLeft = pan.left - (event.clientX - pan.x);
			viewport.scrollTop = pan.top - (event.clientY - pan.y);
			return;
		}
		if (!draft) return;
		draft.end = point(event);
		render();
	}

	function finish(event: PointerEvent) {
		if (pan && event.pointerId === pointerId) { cancelStroke(); return; }
		if (!draft || event.pointerId !== pointerId) return;
		draft.end = point(event);
		if (Math.hypot(draft.end.x - draft.start.x, draft.end.y - draft.start.y) >= 3) {
			marks = [...marks, draft];
			undone = [];
		}
		cancelStroke();
	}

	function cancelStroke() {
		pan = null;
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

<dialog bind:this={dialog} aria-label="批注图片" class="m-auto border border-zinc-700 bg-zinc-900 p-0 text-zinc-100 backdrop:bg-black/75 {maximized ? 'h-[100dvh] max-h-[100dvh] w-screen max-w-none rounded-none' : 'h-[90dvh] max-h-[95dvh] w-[min(1200px,95vw)] max-w-[95vw] rounded-xl'}" oncancel={(event) => { event.preventDefault(); if (!saving) onclose(); }} onpaste={(event) => event.stopPropagation()}>
	<div class="flex h-full min-h-0 flex-col">
		<header class="flex shrink-0 flex-wrap items-center gap-2 border-b border-zinc-800 px-5 py-3">
			<h2 class="mr-auto font-semibold">批注图片</h2>
			<button class="control" onclick={() => maximized = !maximized}>{maximized ? '还原窗口' : '最大化'}</button>
			<button class="control" disabled={saving} onclick={onclose}>取消</button>
			<button class="rounded bg-pink-600 px-3 py-2 text-sm disabled:opacity-40" disabled={!ready || saving} onclick={save}>{saving ? '保存中…' : '保存批注'}</button>
		</header>
		<div class="flex shrink-0 flex-wrap items-center gap-2 border-b border-zinc-800 px-5 py-3 text-sm">
			{#each tools as option}
				<button class="rounded px-3 py-2 {tool === option.id ? 'bg-pink-600' : 'bg-zinc-800'}" aria-pressed={tool === option.id} onclick={() => tool = option.id}>{option.label}</button>
			{/each}
			<label class="flex items-center gap-2 px-2">颜色 <input type="color" bind:value={color} disabled={tool === 'mosaic' || tool === 'pan'} class="h-8 w-9 disabled:opacity-40" /></label>
			{#if tool !== 'pan'}
				<label class="flex items-center gap-2">{tool === 'mosaic' ? '色块大小' : '线条粗细'}
					<select aria-label={tool === 'mosaic' ? '色块大小' : '线条粗细'} bind:value={width} class="rounded bg-zinc-800 px-2 py-2">
						{#each widths as value}<option value={value}>{tool === 'mosaic' ? value * 4 : value} px</option>{/each}
					</select>
				</label>
			{/if}
			{#if tool === 'rectangle'}
				<label class="flex items-center gap-2">圆角
					<select aria-label="圆角" bind:value={radius} class="rounded bg-zinc-800 px-2 py-2">
						{#each [0, 8, 16, 32, 64] as value}<option value={value}>{value === 0 ? '直角' : `${value} px`}</option>{/each}
					</select>
				</label>
			{:else if tool === 'arrow'}
				<label class="flex items-center gap-2">箭头样式
					<select aria-label="箭头样式" bind:value={arrowStyle} class="rounded bg-zinc-800 px-2 py-2">
						{#each arrowStyles as style}<option value={style.id}>{style.symbol} {style.label}</option>{/each}
					</select>
				</label>
			{/if}
			<button class="control" disabled={!marks.length || saving} onclick={undo}>撤销</button>
			<button class="control" disabled={!undone.length || saving} onclick={redo}>重做</button>
		</div>
		<div class="flex shrink-0 flex-wrap items-center gap-2 px-5 py-2 text-xs">
			<button class="control" aria-label="缩小" disabled={!ready || scale <= 0.1} onclick={() => changeZoom(scale / 1.25)}>−</button>
			<span class="min-w-12 text-center" aria-label="当前缩放比例">{Math.round(scale * 100)}%</span>
			<button class="control" aria-label="放大" disabled={!ready || scale >= 8} onclick={() => changeZoom(scale * 1.25)}>＋</button>
			<button class="control" disabled={!ready} onclick={() => changeZoom('fit')}>适应窗口</button>
			<button class="control" disabled={!ready} onclick={() => changeZoom(1)}>100%</button>
			<span class="ml-auto text-zinc-400">放大后可用“拖动预览”或鼠标中键移动图片</span>
		</div>
		{#if error}<p role="alert" class="px-5 py-2 text-sm text-red-400">{error}</p>{/if}
		{#if !ready && !error}<p class="px-5 py-2 text-sm">正在加载图片…</p>{/if}
		<div bind:this={viewport} aria-label="图片预览区域" class="min-h-0 flex-1 overflow-auto bg-zinc-950 overscroll-contain">
			<div class="grid place-items-center" style="width: max(100%, {imageSize.width * scale + 32}px); height: max(100%, {imageSize.height * scale + 32}px);">
				<canvas bind:this={canvas} aria-label="图片批注画布" class="block touch-none {ready ? (tool === 'pan' ? 'cursor-grab' : 'cursor-crosshair') : 'invisible'}" style="width: {imageSize.width * scale}px; height: {imageSize.height * scale}px;" onpointerdown={start} onpointermove={move} onpointerup={finish} onpointercancel={cancelStroke} onlostpointercapture={cancelStroke}></canvas>
			</div>
		</div>
	</div>
</dialog>

<style>
	.control {
		border-radius: 0.25rem;
		background: #27272a;
		padding: 0.5rem 0.75rem;
		font-size: 0.875rem;
	}
	.control:disabled { opacity: 0.4; }
	.control:hover:enabled { background: #3f3f46; }
</style>
