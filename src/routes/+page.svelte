<script lang="ts">
  import Input from '$lib/components/Input/input.svelte';

  let imageName = $state('');
  let inputImage: string | null = $state(null);
  let outputImage: string | null = $state(null);

  const handleImageUpload = async (event: Event) => {
    const target = event.target as HTMLInputElement;
    const file = target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = async (e: ProgressEvent<FileReader>) => {
      inputImage = e.target?.result as string;
      await removeBackground(inputImage);
    };
    reader.readAsDataURL(file);
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

    // Function to get the dominant color in the image
    const getDominantColor = (imageData: ImageData) => {
      const colorCount: { [key: string]: number } = {};
      const threshold = 200; // Threshold to consider color close enough to be dominant

      for (let i = 0; i < imageData.data.length; i += 4) {
        const r = imageData.data[i];
        const g = imageData.data[i + 1];
        const b = imageData.data[i + 2];
        const color = `${r},${g},${b}`;

        if (!colorCount[color]) colorCount[color] = 0;
        colorCount[color]++;
      }

      const dominantColor = Object.keys(colorCount).reduce((prev, current) =>
        colorCount[prev] > colorCount[current] ? prev : current
      );

      return dominantColor.split(',').map(Number);
    };

    const dominantColor = getDominantColor(imageData);
    const [r, g, b] = dominantColor;
    const threshold = 60; // Color similarity threshold to decide background removal

    const isBackgroundColor = (r1: number, g1: number, b1: number) => {
      return Math.abs(r1 - r) < threshold && Math.abs(g1 - g) < threshold && Math.abs(b1 - b) < threshold;
    };

    const visited = new Uint8Array(width * height);
    const queue: { x: number; y: number }[] = [];

    // Top and bottom rows
    for (let x = 0; x < width; x++) {
      if (isBackgroundColor(data[(0 * width + x) * 4], data[(0 * width + x) * 4 + 1], data[(0 * width + x) * 4 + 2])) {
        queue.push({ x, y: 0 });
        visited[0 * width + x] = 1;
      }
      if (isBackgroundColor(data[((height - 1) * width + x) * 4], data[((height - 1) * width + x) * 4 + 1], data[((height - 1) * width + x) * 4 + 2])) {
        queue.push({ x, y: height - 1 });
        visited[(height - 1) * width + x] = 1;
      }
    }

    // Left and right columns
    for (let y = 0; y < height; y++) {
      if (isBackgroundColor(data[(y * width + 0) * 4], data[(y * width + 0) * 4 + 1], data[(y * width + 0) * 4 + 2])) {
        queue.push({ x: 0, y });
        visited[y * width + 0] = 1;
      }
      if (isBackgroundColor(data[(y * width + (width - 1)) * 4], data[(y * width + (width - 1)) * 4 + 1], data[(y * width + (width - 1)) * 4 + 2])) {
        queue.push({ x: width - 1, y });
        visited[y * width + (width - 1)] = 1;
      }
    }

    // Flood fill (4-connected neighbors)
    const directions = [
      { dx: -1, dy: 0 },
      { dx: 1, dy: 0 },
      { dx: 0, dy: -1 },
      { dx: 0, dy: 1 },
    ];

    while (queue.length) {
      const { x, y } = queue.shift()!;
      for (const { dx, dy } of directions) {
        const nx = x + dx, ny = y + dy;
        if (nx >= 0 && nx < width && ny >= 0 && ny < height) {
          const idx = ny * width + nx;
          if (!visited[idx] && isBackgroundColor(data[(ny * width + nx) * 4], data[(ny * width + nx) * 4 + 1], data[(ny * width + nx) * 4 + 2])) {
            visited[idx] = 1;
            queue.push({ x: nx, y: ny });
          }
        }
      }
    }

    // Remove (make transparent) only the background pixels
    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        if (visited[y * width + x] === 1) {
          const i = (y * width + x) * 4;
          data[i + 3] = 0; // Set alpha to 0 (transparent)
        }
      }
    }

    ctx.putImageData(imageData, 0, 0);
    outputImage = canvas.toDataURL('image/webp');
  };

  const downloadImage = () => {
    if (!outputImage) return;
    const link = document.createElement('a');
    link.href = outputImage;
    link.download = `${imageName}.webp`;
    link.click();
  };
</script>

<div class="flex flex-col items-center gap-4 p-6 h-screen">
  <h1 class="text-2xl font-bold">Background Remover</h1>
  <Input
    name="imageName"
    label="Provide a name for the image"
    placeholder="Image Name"
    size="md"
    bind:value={imageName}
  />
  <input type="file" accept="image/*" onchange={handleImageUpload} />

  {#if inputImage}
    <div class="mt-6 flex gap-8">
      <div>
        <h2 class="text-lg font-semibold">Original Image</h2>
        <img src={inputImage} alt="Original" class="max-w-sm rounded-xl shadow-lg" />
      </div>

      {#if outputImage}
        <div>
          <h2 class="text-lg font-semibold">No Background</h2>
          <img src={outputImage} alt="No Background" class="max-w-sm rounded-xl shadow-lg" />
          <button
            onclick={downloadImage}
            class="mt-4 rounded-lg bg-blue-500 px-4 py-2 text-white hover:bg-blue-600"
          >
            Download Image
          </button>
        </div>
      {/if}
    </div>
  {/if}
</div>

<style>
  :root {
    --checkerboard-size: 20px;
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
  }

  img[alt='No Background'] {
    background-image:
      linear-gradient(45deg, #ccc 25%, transparent 25%),
      linear-gradient(-45deg, #ccc 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, #ccc 75%),
      linear-gradient(-45deg, transparent 75%, #ccc 75%);
    background-size: var(--checkerboard-size) var(--checkerboard-size);
    background-position:
      0 0,
      0 0,
      0 0,
      0 0;
    animation: checkerboard 2s infinite linear;
  }

  @layer utilities {
    .bg-space {
      background-color: #000000;
      background-image: 
        radial-gradient(white 1px, transparent 1px),
        radial-gradient(white 1px, transparent 1px);
      background-size: 50px 50px;
      background-position: 0 0, 25px 25px;
    }
  }
</style>
