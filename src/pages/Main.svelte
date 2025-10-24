<script>
	import { onMount, tick } from 'svelte';
	import * as d3 from 'd3';

	let imageFile = null;
	let imageUrl = '';
	let canvasEl;
	let ctx;
	let results = {};
	let processing = false;

	const loadImage = (event) => {
		const file = event.target.files[0];
		if (!file) return;
		imageFile = file;
		imageUrl = URL.createObjectURL(file);
	};

	async function analyzeImage() {
		if (!imageUrl || processing) return;
		processing = true;

		await tick() // небольшой костыль чтобы анализ изображения не прерывал работу UI

		const img = new Image();
		img.src = imageUrl;
		img.onload = () => {
			canvasEl.width = img.width;
			canvasEl.height = img.height;
			ctx.drawImage(img, 0, 0);

			const imageData = ctx.getImageData(0, 0, img.width, img.height);
			const { data, width, height } = imageData;
			const gray = new Float32Array(width * height);

			for (let i = 0; i < data.length; i += 4) {
				gray[i / 4] = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
			}

			const mean = d3.mean(gray);
			const std = d3.deviation(gray);

			// Простая резкость: средняя величина градиента
			let gradSum = 0;
			let gradCount = 0;
			for (let y = 1; y < height - 1; y++) {
				for (let x = 1; x < width - 1; x++) {
					const idx = y * width + x;
					const gx = gray[idx + 1] - gray[idx - 1];
					const gy = gray[idx + width] - gray[idx - width];
					gradSum += Math.sqrt(gx * gx + gy * gy);
					gradCount++;
				}
			}
			const sharpness = gradSum / gradCount;

			// график яркости по центральной строке
			const centerY = Math.floor(height / 2);
			const line = gray.slice(centerY * width, (centerY + 1) * width);

			results = { mean: mean.toFixed(2), std: std.toFixed(2), sharpness: sharpness.toFixed(2), line };
			drawGraph(line);
			processing = false;
		};
	};

	const drawGraph = (line) => {
		const graph = d3.select('#graph');
		graph.selectAll('*').remove();

		const width = 500, height = 200;
		const svg = graph.append('svg').attr('width', width).attr('height', height);

		const x = d3.scaleLinear().domain([0, line.length]).range([0, width]);
		const y = d3.scaleLinear().domain([0, 255]).range([height, 0]);

		const lineGen = d3.line().x((d, i) => x(i)).y((d) => y(d));

		svg.append('path')
		.datum(line)
		.attr('fill', 'none')
		.attr('stroke', 'steelblue')
		.attr('stroke-width', 1.5)
		.attr('d', lineGen);
	};

	async function getCTX(imageUrlFormal) {
		if(!imageUrlFormal) return

		await tick() // Ждем пока инициализируется (и присвоится переменной canvasEl) элемент канвы, чтобы на его основе иниц-ть переменную ctx

		ctx = canvasEl.getContext('2d')
	}

	$: getCTX(imageUrl)
</script>

<div class="p-4 space-y-4">
	<h1 class="text-2xl font-bold">Camera rating</h1>

	<input type="file" accept="image/*" on:change={loadImage} />
	{#if imageUrl}
		<div class="flex gap-4 mt-2">
			<canvas bind:this={canvasEl} class="border rounded"></canvas>
			<div>
				<button on:click={analyzeImage} disabled={processing} class="bg-blue-500 text-white px-4 py-2 rounded">
					{processing ? 'Идет анализ...' : 'Проанализировать'}
				</button>
				{#if results.mean}
					<div class="mt-4 space-y-1 text-sm">
						<p>Яркость: {results.mean}</p>
						<p>Стандартное отклонение: {results.std}</p>
						<p>Резкость: {results.sharpness}</p>
					</div>
				{/if}
			</div>
		</div>
	{/if}
	<div class="graphBlock rounded-lg px-15 py-5">
		<p>Яркость по центральной строке</p>
		<div id="graph" class="mt-6"></div>
	</div>
</div>

<style>
	canvas {
		max-width: 500px;
		height: auto;
	}
	.graphBlock {
		background-color: var(--bg-main-accent);
	}
</style>
