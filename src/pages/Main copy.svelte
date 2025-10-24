<script>
	import { onMount } from 'svelte';
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

	const analyzeImage = async () => {
		if (!imageUrl || processing) return;
		processing = true;
		await new Promise((resolve) => setTimeout(resolve, 50)); // yield to UI

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

			// Simple sharpness: average gradient magnitude
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

			// Brightness profile along center row
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

	onMount(() => {
		ctx = canvasEl.getContext('2d');
	});
</script>

<div class="p-4 space-y-4">
	<h1 class="text-2xl font-bold">Camera Tester (Web version)</h1>

	<input type="file" accept="image/*" on:change={loadImage} />
	{#if imageUrl}
		<div class="flex gap-4 mt-2">
			<canvas bind:this={canvasEl} class="border rounded"></canvas>
			<div>
				<button on:click={analyzeImage} disabled={processing} class="bg-blue-500 text-white px-4 py-2 rounded">
					{processing ? 'Processing...' : 'Analyze Image'}
				</button>
				{#if results.mean}
					<div class="mt-4 space-y-1 text-sm">
						<p>Mean Brightness: {results.mean}</p>
						<p>Std Deviation: {results.std}</p>
						<p>Sharpness: {results.sharpness}</p>
					</div>
				{/if}
			</div>
		</div>
	{/if}

	<div id="graph" class="mt-6"></div>
</div>

<style>
	canvas {
		max-width: 500px;
		height: auto;
	}
</style>
