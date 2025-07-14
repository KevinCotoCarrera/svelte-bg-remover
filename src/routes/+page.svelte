<script lang="ts">
	import Input from '$lib/components/Input/input.svelte';
	import { onMount } from 'svelte';
	import { fade, fly, slide } from 'svelte/transition';

	// State management
	let imageName = $state('');
	let inputImage: string | null = $state(null);
	let outputImage: string | null = $state(null);
	let isProcessing = $state(false);
	let threshold = $state(60);
	let imageQuality = $state(0.9);
	let format = $state('webp');
	let originalSize = $state({ width: 0, height: 0 });
	let activeTab = $state('remove');
	let showAdvancedOptions = $state(false);
	let edgeHighlight = $state(false);
	let smoothEdges = $state(true);
	let dragActive = $state(false);
	let errorMessage = $state<string | null>(null);
	let showTour = $state(false);

	onMount(() => {
		const hasSeenTour = localStorage.getItem('bgRemover_hasSeenTour');
		if (!hasSeenTour) {
			showTour = true;
			localStorage.setItem('bgRemover_hasSeenTour', 'true');
		}
	});

	// Handle file drop and upload
	const handleFileDrop = (event: DragEvent) => {
		event.preventDefault();
		dragActive = false;

		const file = event.dataTransfer?.files?.[0];
		if (!file) return;
		processFile(file);
	};

	const handleDragOver = (event: DragEvent) => {
		event.preventDefault();
		dragActive = true;
	};

	const handleDragLeave = () => {
		dragActive = false;
	};

	const handleImageUpload = async (event: Event) => {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (!file) return;

		// Make sure we're in a fresh state before processing
		outputImage = null;
		errorMessage = null;

		processFile(file);
	};

	const processFile = async (file: File) => {
		errorMessage = null;
		if (!file.type.startsWith('image/')) {
			errorMessage = 'Please select a valid image file';
			return;
		}

		if (file.size > 10 * 1024 * 1024) {
			errorMessage = 'File size exceeds 10MB limit';
			return;
		}

		// Set default filename (without extension)
		if (!imageName) {
			imageName = file.name.replace(/\.[^/.]+$/, '');
		}

		const reader = new FileReader();
		reader.onload = async (e: ProgressEvent<FileReader>) => {
			try {
				inputImage = e.target?.result as string;
				await processImage();
			} catch (error) {
				console.error('Error processing image:', error);
				errorMessage = 'Failed to process image. Please try another file.';
			}
		};
		reader.readAsDataURL(file);
	};

	const processImage = async () => {
		if (!inputImage) return;
		isProcessing = true;

		try {
			const img = new Image();
			img.src = inputImage;
			await img.decode();

			originalSize = { width: img.width, height: img.height };

			if (activeTab === 'remove') {
				await removeBackground(inputImage);
			}
			// Other tabs can be implemented as needed
		} catch (error) {
			console.error('Error:', error);
			errorMessage = 'An error occurred during processing';
		} finally {
			isProcessing = false;
		}
	};

	const removeBackground = async (imageSrc: string) => {
		const img = new Image();
		img.src = imageSrc;
		await img.decode();

		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d');
		if (!ctx) return;
		canvas.width = img.width;
		canvas.height = img.height;
		ctx.drawImage(img, 0, 0);

		const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
		const data = imageData.data;
		const width = canvas.width;
		const height = canvas.height;

		// Get background colors from edges
		const edgeColors = getEdgeColors(imageData);
		const backgroundColors = identifyBackgroundColors(edgeColors);

		// Create alpha mask
		const alphaMask = new Uint8Array(width * height);
		createAlphaMask(imageData, alphaMask, backgroundColors, width, height);

		// Apply edge refinements if selected
		if (smoothEdges) {
			smoothAlphaMask(alphaMask, width, height);
		}

		// Apply edge highlighting if selected
		if (edgeHighlight) {
			highlightEdges(alphaMask, imageData, width, height);
		}

		// Apply alpha mask to image
		applyAlphaMask(data, alphaMask, width, height);

		ctx.putImageData(imageData, 0, 0);
		outputImage = canvas.toDataURL(`image/${format}`, imageQuality);
	};

	// Function to get colors from the edges of the image
	const getEdgeColors = (imageData: ImageData) => {
		const { width, height, data } = imageData;
		const edgeColors: number[][] = [];
		const samples = 100; // Number of samples to take from each edge
		const step = Math.max(1, Math.floor(width / samples));

		// Sample from top and bottom edges
		for (let x = 0; x < width; x += step) {
			// Top edge
			const topIdx = (0 * width + x) * 4;
			edgeColors.push([data[topIdx], data[topIdx + 1], data[topIdx + 2]]);

			// Bottom edge
			const bottomIdx = ((height - 1) * width + x) * 4;
			edgeColors.push([data[bottomIdx], data[bottomIdx + 1], data[bottomIdx + 2]]);
		}

		// Sample from left and right edges
		for (let y = 0; y < height; y += step) {
			// Left edge
			const leftIdx = (y * width + 0) * 4;
			edgeColors.push([data[leftIdx], data[leftIdx + 1], data[leftIdx + 2]]);

			// Right edge
			const rightIdx = (y * width + (width - 1)) * 4;
			edgeColors.push([data[rightIdx], data[rightIdx + 1], data[rightIdx + 2]]);
		}

		return edgeColors;
	};

	// Identify likely background colors using clustering
	const identifyBackgroundColors = (colors: number[][]) => {
		// Basic clustering of similar colors
		const clusters: { color: number[]; count: number }[] = [];

		for (const color of colors) {
			let foundCluster = false;

			for (const cluster of clusters) {
				if (colorDistance(color, cluster.color) < threshold) {
					// Update cluster average
					const count = cluster.count;
					const newCount = count + 1;
					cluster.color = cluster.color.map((c, i) =>
						Math.round((c * count + color[i]) / newCount)
					);
					cluster.count = newCount;
					foundCluster = true;
					break;
				}
			}

			if (!foundCluster) {
				clusters.push({ color: [...color], count: 1 });
			}
		}

		// Sort clusters by count (most frequent first)
		clusters.sort((a, b) => b.count - a.count);

		// Return top clusters (likely background colors)
		return clusters.slice(0, 3).map((c) => c.color);
	};

	// Calculate Euclidean distance between colors
	const colorDistance = (c1: number[], c2: number[]) => {
		return Math.sqrt(
			Math.pow(c1[0] - c2[0], 2) + Math.pow(c1[1] - c2[1], 2) + Math.pow(c1[2] - c2[2], 2)
		);
	};

	// Create initial alpha mask using flood fill from edges
	const createAlphaMask = (
		imageData: ImageData,
		alphaMask: Uint8Array,
		backgroundColors: number[][],
		width: number,
		height: number
	) => {
		const data = imageData.data;
		const visited = new Uint8Array(width * height);
		const queue: { x: number; y: number }[] = [];

		// Helper to check if a color matches background
		const isBackgroundColor = (idx: number) => {
			const r = data[idx];
			const g = data[idx + 1];
			const b = data[idx + 2];

			for (const bgColor of backgroundColors) {
				if (colorDistance([r, g, b], bgColor) < threshold) {
					return true;
				}
			}
			return false;
		};

		// Initialize from edges
		for (let x = 0; x < width; x++) {
			const topIdx = x * 4;
			const bottomIdx = ((height - 1) * width + x) * 4;

			if (isBackgroundColor(topIdx)) {
				queue.push({ x, y: 0 });
				visited[0 * width + x] = 1;
			}

			if (isBackgroundColor(bottomIdx)) {
				queue.push({ x, y: height - 1 });
				visited[(height - 1) * width + x] = 1;
			}
		}

		for (let y = 0; y < height; y++) {
			const leftIdx = y * width * 4;
			const rightIdx = (y * width + width - 1) * 4;

			if (isBackgroundColor(leftIdx)) {
				queue.push({ x: 0, y });
				visited[y * width] = 1;
			}

			if (isBackgroundColor(rightIdx)) {
				queue.push({ x: width - 1, y });
				visited[y * width + width - 1] = 1;
			}
		}

		// Flood fill (8-connected neighbors for better coverage)
		const directions = [
			{ dx: -1, dy: 0 },
			{ dx: 1, dy: 0 },
			{ dx: 0, dy: -1 },
			{ dx: 0, dy: 1 },
			{ dx: -1, dy: -1 },
			{ dx: 1, dy: 1 },
			{ dx: -1, dy: 1 },
			{ dx: 1, dy: -1 }
		];

		while (queue.length) {
			const { x, y } = queue.shift()!;
			const idx = y * width + x;
			alphaMask[idx] = 1; // Mark as background

			for (const { dx, dy } of directions) {
				const nx = x + dx;
				const ny = y + dy;

				if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
					const nidx = ny * width + nx;
					const pixelIdx = nidx * 4;

					if (!visited[nidx] && isBackgroundColor(pixelIdx)) {
						visited[nidx] = 1;
						queue.push({ x: nx, y: ny });
					}
				}
			}
		}
	};

	// Smooth alpha mask edges
	const smoothAlphaMask = (alphaMask: Uint8Array, width: number, height: number) => {
		const tempMask = new Uint8Array(alphaMask);
		const kernel = [1, 2, 1, 2, 4, 2, 1, 2, 1]; // Gaussian-like kernel
		const kernelSum = kernel.reduce((a, b) => a + b, 0);

		for (let y = 1; y < height - 1; y++) {
			for (let x = 1; x < width - 1; x++) {
				let sum = 0;

				for (let ky = -1; ky <= 1; ky++) {
					for (let kx = -1; kx <= 1; kx++) {
						const idx = (y + ky) * width + (x + kx);
						const kernelIdx = (ky + 1) * 3 + (kx + 1);
						sum += tempMask[idx] * kernel[kernelIdx];
					}
				}

				// If the weighted average is > 0.5, consider it background
				alphaMask[y * width + x] = sum > kernelSum / 2 ? 1 : 0;
			}
		}
	};

	// Highlight edges for better object definition
	const highlightEdges = (
		alphaMask: Uint8Array,
		imageData: ImageData,
		width: number,
		height: number
	) => {
		const data = imageData.data;

		for (let y = 1; y < height - 1; y++) {
			for (let x = 1; x < width - 1; x++) {
				const idx = y * width + x;

				if (alphaMask[idx] === 0) {
					// Check if this foreground pixel is next to background
					let isEdge = false;

					for (let ky = -1; ky <= 1 && !isEdge; ky++) {
						for (let kx = -1; kx <= 1 && !isEdge; kx++) {
							if (kx === 0 && ky === 0) continue;

							const nidx = (y + ky) * width + (x + kx);
							if (alphaMask[nidx] === 1) {
								isEdge = true;
							}
						}
					}

					if (isEdge) {
						// Enhance contrast at edges
						const pixelIdx = idx * 4;
						data[pixelIdx] = Math.min(255, data[pixelIdx] * 1.2);
						data[pixelIdx + 1] = Math.min(255, data[pixelIdx + 1] * 1.2);
						data[pixelIdx + 2] = Math.min(255, data[pixelIdx + 2] * 1.2);
					}
				}
			}
		}
	};

	// Apply alpha mask to image data
	const applyAlphaMask = (
		data: Uint8ClampedArray,
		alphaMask: Uint8Array,
		width: number,
		height: number
	) => {
		for (let y = 0; y < height; y++) {
			for (let x = 0; x < width; x++) {
				const idx = y * width + x;
				if (alphaMask[idx] === 1) {
					const i = idx * 4;
					data[i + 3] = 0; // Set alpha to 0 (transparent)
				}
			}
		}
	};

	const downloadImage = () => {
		if (!outputImage) return;

		// Use default name if not provided
		const filename = imageName ? imageName : 'background-removed';

		const link = document.createElement('a');
		link.href = outputImage;
		link.download = `${filename}.${format}`;
		link.click();
	};

	const resetImage = () => {
		inputImage = null;
		outputImage = null;
		errorMessage = null;
		isProcessing = false;
		// Reset file input by creating a new event to simulate a form reset
		setTimeout(() => {
			// Find and reset all file inputs
			document.querySelectorAll('input[type="file"]').forEach((input) => {
				(input as HTMLInputElement).value = '';
			});
		}, 100);
	};

	const toggleAdvancedOptions = () => {
		showAdvancedOptions = !showAdvancedOptions;
	};

	const closeTour = () => {
		showTour = false;
	};
</script>

<div
	class="flex min-h-screen flex-col items-center gap-4 bg-gradient-to-b from-blue-50 to-purple-50 p-6 dark:from-gray-900 dark:to-blue-950"
>
	<header class="w-full max-w-4xl">
		<div class="mb-6 flex items-center justify-between">
			<h1
				class="bg-gradient-to-r from-blue-600 to-purple-600 bg-clip-text text-3xl font-bold text-transparent md:text-4xl"
			>
				Background Remover
			</h1>
			<button
				class="rounded-full p-2 text-gray-600 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-800"
				onclick={toggleAdvancedOptions}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="h-6 w-6"
					fill="none"
					viewBox="0 0 24 24"
					stroke="currentColor"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"
					/>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"
					/>
				</svg>
			</button>
		</div>

		<div
			class="w-full rounded-2xl bg-white p-8 shadow-lg transition-all dark:bg-gray-800 dark:text-gray-200"
		>
			<!-- Upload area -->
			<div
				class={`flex flex-col items-center justify-center rounded-xl border-2 border-dashed p-6 transition-all ${dragActive ? 'border-blue-500 bg-blue-50 dark:bg-blue-900/20' : 'border-gray-300 dark:border-gray-600'}`}
				role="region"
				ondragover={handleDragOver}
				ondragleave={handleDragLeave}
				ondrop={handleFileDrop}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="mb-4 h-12 w-12 text-blue-500"
					fill="none"
					viewBox="0 0 24 24"
					stroke="currentColor"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="1.5"
						d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"
					/>
				</svg>
				<p class="mb-2 text-lg font-medium text-gray-800 dark:text-gray-200">
					Drag & drop your image here
				</p>
				<p class="mb-4 text-sm text-gray-500 dark:text-gray-400">or</p>
				<!-- We use 'key' to force re-render of the input element after reset -->
				<label
					class="cursor-pointer rounded-lg bg-blue-500 px-6 py-3 text-white transition hover:bg-blue-600"
				>
					<input
						type="file"
						accept="image/*"
						onchange={handleImageUpload}
						class="hidden"
						id="file-upload-input"
					/>
					/>
				</label>
				<p class="mt-2 text-xs text-gray-500 dark:text-gray-400">Maximum file size: 10MB</p>
			</div>

			<!-- Input fields -->
			<div class="mt-6">
				<Input
					name="imageName"
					label="Image filename"
					placeholder="Enter filename (without extension)"
					size="md"
					bind:value={imageName}
				/>
				<p class="mt-1 text-xs text-gray-600 dark:text-gray-300">
					Enter a name for your downloaded file
				</p>

				<!-- Advanced options dropdown -->
				{#if showAdvancedOptions}
					<div
						class="mt-4 rounded-lg border bg-gray-50 p-4 dark:border-gray-600 dark:bg-gray-700/50"
						transition:slide
					>
						<h3 class="mb-3 text-lg font-medium text-gray-800 dark:text-gray-200">
							Advanced Options
						</h3>

						<div class="grid grid-cols-1 gap-4 md:grid-cols-2">
							<div>
								<label class="mb-1 block text-sm font-medium text-gray-800 dark:text-gray-200"
									>Threshold</label
								>
								<div class="flex items-center gap-2">
									<input type="range" min="20" max="100" bind:value={threshold} class="w-full" />
									<span class="w-8 text-center text-sm text-gray-800 dark:text-gray-200"
										>{threshold}</span
									>
								</div>
								<p class="mt-1 text-xs text-gray-600 dark:text-gray-400">
									Higher values remove more colors
								</p>
							</div>

							<div>
								<label class="mb-1 block text-sm font-medium text-gray-800 dark:text-gray-200"
									>Quality</label
								>
								<div class="flex items-center gap-2">
									<input
										type="range"
										min="0.5"
										max="1"
										step="0.1"
										bind:value={imageQuality}
										class="w-full"
									/>
									<span class="w-8 text-center text-sm text-gray-800 dark:text-gray-200"
										>{Math.round(imageQuality * 100)}%</span
									>
								</div>
							</div>

							<div>
								<label class="mb-2 block text-sm font-medium text-gray-800 dark:text-gray-200"
									>Format</label
								>
								<div class="flex gap-4">
									<label class="inline-flex items-center text-gray-800 dark:text-gray-200">
										<input
											type="radio"
											class="form-radio"
											name="format"
											value="webp"
											bind:group={format}
										/>
										<span class="ml-2">WebP</span>
									</label>
									<label class="inline-flex items-center text-gray-800 dark:text-gray-200">
										<input
											type="radio"
											class="form-radio"
											name="format"
											value="png"
											bind:group={format}
										/>
										<span class="ml-2">PNG</span>
									</label>
								</div>
							</div>

							<div class="flex flex-col gap-2">
								<label class="inline-flex items-center text-gray-800 dark:text-gray-200">
									<input type="checkbox" bind:checked={smoothEdges} />
									<span class="ml-2">Smooth Edges</span>
								</label>
								<label class="inline-flex items-center text-gray-800 dark:text-gray-200">
									<input type="checkbox" bind:checked={edgeHighlight} />
									<span class="ml-2">Highlight Edges</span>
								</label>
							</div>
						</div>
					</div>
				{/if}
			</div>
		</div>
	</header>

	<!-- Processing indicator -->
	{#if isProcessing}
		<div class="mt-8 flex flex-col items-center" transition:fade>
			<div class="loader mb-4"></div>
			<p>Processing your image...</p>
		</div>
	{/if}

	<!-- Results display -->
	{#if inputImage && !isProcessing}
		<div
			class="mt-8 grid w-full max-w-6xl grid-cols-1 gap-8 md:grid-cols-2"
			transition:fade={{ duration: 300 }}
		>
			<div class="rounded-2xl bg-white p-6 shadow-lg dark:bg-gray-800">
				<h2 class="mb-4 text-xl font-semibold text-gray-800 dark:text-gray-200">Original Image</h2>
				<div class="relative aspect-square w-full overflow-hidden rounded-lg">
					<img src={inputImage} alt="Original" class="h-full w-full object-contain" />
				</div>
				<p class="mt-2 text-sm text-gray-500 dark:text-gray-400">
					Size: {originalSize.width} × {originalSize.height}px
				</p>
			</div>

			{#if outputImage}
				<div class="rounded-2xl bg-white p-6 shadow-lg dark:bg-gray-800">
					<h2 class="mb-4 text-xl font-semibold text-gray-800 dark:text-gray-200">
						Background Removed
					</h2>
					<div class="relative aspect-square w-full overflow-hidden rounded-lg">
						<img src={outputImage} alt="No Background" class="h-full w-full object-contain" />
					</div>

					<div class="mt-4 flex gap-2">
						<button
							onclick={downloadImage}
							class="flex flex-1 items-center justify-center gap-2 rounded-lg bg-blue-500 px-4 py-3 text-white transition hover:bg-blue-600"
						>
							<svg
								xmlns="http://www.w3.org/2000/svg"
								class="h-5 w-5"
								viewBox="0 0 20 20"
								fill="currentColor"
							>
								<path
									fill-rule="evenodd"
									d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L9 10.586V3a1 1 0 112 0v7.586l1.293-1.293a1 1 0 111.414 1.414l-3 3a1 1 0 01-1.414 0l-3-3a1 1 0 010-1.414z"
									clip-rule="evenodd"
								/>
							</svg>
							Download {format.toUpperCase()}
						</button>
						<button
							onclick={resetImage}
							class="rounded-lg border border-gray-300 px-4 py-3 text-gray-700 transition hover:bg-gray-100 dark:border-gray-600 dark:text-gray-300 dark:hover:bg-gray-700"
						>
							Reset
						</button>
					</div>
				</div>
			{/if}
		</div>
	{/if}

	<!-- Error message -->
	{#if errorMessage}
		<div
			class="mt-4 rounded-lg border border-red-200 bg-red-50 p-4 text-red-700 dark:border-red-800 dark:bg-red-900/20 dark:text-red-400"
			transition:fade
		>
			<div class="flex items-center">
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="mr-2 h-5 w-5"
					viewBox="0 0 20 20"
					fill="currentColor"
				>
					<path
						fill-rule="evenodd"
						d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z"
						clip-rule="evenodd"
					/>
				</svg>
				{errorMessage}
			</div>
		</div>
	{/if}

	<!-- Quick tutorial/tour -->
	{#if showTour}
		<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50" transition:fade>
			<div
				class="max-w-md rounded-2xl bg-white p-6 shadow-2xl dark:bg-gray-800 dark:text-gray-200"
				transition:fly={{ y: 20 }}
			>
				<h3 class="mb-4 text-xl font-bold">Welcome to Background Remover</h3>

				<div class="mb-6 space-y-4">
					<div class="flex items-start">
						<div class="mr-3 flex-shrink-0 rounded-full bg-blue-100 p-2 dark:bg-blue-900/30">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								class="h-5 w-5 text-blue-600 dark:text-blue-400"
								viewBox="0 0 20 20"
								fill="currentColor"
							>
								<path
									fill-rule="evenodd"
									d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
						<p>Upload any image by dragging and dropping or clicking the upload area.</p>
					</div>
					<div class="flex items-start">
						<div class="mr-3 flex-shrink-0 rounded-full bg-blue-100 p-2 dark:bg-blue-900/30">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								class="h-5 w-5 text-blue-600 dark:text-blue-400"
								viewBox="0 0 20 20"
								fill="currentColor"
							>
								<path
									fill-rule="evenodd"
									d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
						<p>Our AI automatically removes the background of your image.</p>
					</div>
					<div class="flex items-start">
						<div class="mr-3 flex-shrink-0 rounded-full bg-blue-100 p-2 dark:bg-blue-900/30">
							<svg
								xmlns="http://www.w3.org/2000/svg"
								class="h-5 w-5 text-blue-600 dark:text-blue-400"
								viewBox="0 0 20 20"
								fill="currentColor"
							>
								<path
									fill-rule="evenodd"
									d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
									clip-rule="evenodd"
								/>
							</svg>
						</div>
						<p>Use the advanced settings to fine-tune the results and then download your image!</p>
					</div>
				</div>

				<div class="flex justify-end">
					<button
						onclick={closeTour}
						class="rounded-lg bg-blue-500 px-5 py-2 text-white transition hover:bg-blue-600"
					>
						Got it!
					</button>
				</div>
			</div>
		</div>
	{/if}
</div>

<style>
	:root {
		--checkerboard-size: 16px;
		--primary: #3b82f6;
		--primary-dark: #2563eb;
		--primary-light: #60a5fa;
		--neutral-50: #f9fafb;
		--neutral-100: #f3f4f6;
		--neutral-200: #e5e7eb;
		--neutral-800: #1f2937;
		--neutral-900: #111827;
	}

	@keyframes checkerboard {
		0% {
			background-position:
				0 0,
				var(--checkerboard-size) var(--checkerboard-size);
		}
		100% {
			background-position:
				var(--checkerboard-size) var(--checkerboard-size),
				0 0;
		}
	}

	img {
		background: checkerboard;
		transition: all 0.3s ease;
	}

	img[alt='No Background'] {
		background-image:
			linear-gradient(45deg, rgba(156, 163, 175, 0.3) 25%, transparent 25%),
			linear-gradient(-45deg, rgba(156, 163, 175, 0.3) 25%, transparent 25%),
			linear-gradient(45deg, transparent 75%, rgba(156, 163, 175, 0.3) 75%),
			linear-gradient(-45deg, transparent 75%, rgba(156, 163, 175, 0.3) 75%);
		background-size: var(--checkerboard-size) var(--checkerboard-size);
		background-position:
			0 0,
			0 var(--checkerboard-size),
			var(--checkerboard-size) calc(-1 * var(--checkerboard-size)),
			calc(-1 * var(--checkerboard-size)) 0;
		animation: checkerboard 20s infinite linear;
	}

	/* Dark mode support for checkerboard */
	@media (prefers-color-scheme: dark) {
		img[alt='No Background'] {
			background-image:
				linear-gradient(45deg, rgba(75, 85, 99, 0.4) 25%, transparent 25%),
				linear-gradient(-45deg, rgba(75, 85, 99, 0.4) 25%, transparent 25%),
				linear-gradient(45deg, transparent 75%, rgba(75, 85, 99, 0.4) 75%),
				linear-gradient(-45deg, transparent 75%, rgba(75, 85, 99, 0.4) 75%);
		}
	}

	.loader {
		width: 48px;
		height: 48px;
		border: 5px solid var(--neutral-200);
		border-bottom-color: var(--primary);
		border-radius: 50%;
		display: inline-block;
		box-sizing: border-box;
		animation: rotation 1s linear infinite;
	}

	@keyframes rotation {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}

	/* Smooth transitions for all interactive elements */
	button,
	input,
	a,
	label {
		transition: all 0.2s ease;
	}

	/* Custom styling for range inputs */
	input[type='range'] {
		-webkit-appearance: none;
		appearance: none;
		width: 100%;
		height: 6px;
		background: var(--neutral-200);
		border-radius: 3px;
		outline: none;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 18px;
		height: 18px;
		background: var(--primary);
		border-radius: 50%;
		cursor: pointer;
		transition: all 0.2s;
	}

	input[type='range']::-webkit-slider-thumb:hover {
		background: var(--primary-dark);
		transform: scale(1.1);
	}

	/* Custom styling for checkboxes and radio buttons */
	input[type='checkbox'],
	input[type='radio'] {
		accent-color: var(--primary);
	}

	@keyframes slide-up {
		from {
			transform: translateY(20px);
			opacity: 0;
		}
		to {
			transform: translateY(0);
			opacity: 1;
		}
	}

	/* Animation for newly loaded elements */
	.bg-white,
	.bg-gray-50,
	.mt-8 {
		animation: slide-up 0.4s ease-out;
	}
</style>
