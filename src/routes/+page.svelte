<script lang="ts">
  import { onMount } from 'svelte';

  // The original and processed images are stored as Base64 strings or null.
  let inputImage: string | null = null;
  let outputImage: string | null = null;

  // Handle file upload. We cast event.target as HTMLInputElement to access its files.
  const handleImageUpload = async (event: Event) => {
    const target = event.target as HTMLInputElement;
    const file = target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = async (e: ProgressEvent<FileReader>) => {
      // The result is expected to be a string (Base64 URL)
      inputImage = e.target?.result as string;
      await removeBackground(inputImage);
    };
    reader.readAsDataURL(file);
  };

  // Remove white background by setting pixels near white to transparent
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

    // Get pixel data from the canvas
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imageData.data;

    // Loop through each pixel (RGBA order)
    for (let i = 0; i < data.length; i += 4) {
      const r = data[i];
      const g = data[i + 1];
      const b = data[i + 2];

      // Remove white background (simple threshold)
      if (r > 240 && g > 240 && b > 240) {
        data[i + 3] = 0; // Set alpha to 0 for transparency
      }
    }

    ctx.putImageData(imageData, 0, 0);
    // Convert the canvas to a Base64 WebP image
    outputImage = canvas.toDataURL('image/webp');
  };

  // Download the processed image if available
  const downloadImage = () => {
    if (!outputImage) return;
    const link = document.createElement('a');
    link.href = outputImage;
    link.download = 'background_removed.webp';
    link.click();
  };
</script>

<div class="flex flex-col items-center gap-4 p-6">
  <h1 class="text-2xl font-bold">Background Remover</h1>
  <input type="file" accept="image/*" on:change={handleImageUpload} />

  {#if inputImage}
    <div class="flex gap-8 mt-6">
      <div>
        <h2 class="text-lg font-semibold">Original Image</h2>
        <img src={inputImage} alt="Original" class="max-w-sm rounded-xl shadow-lg" />
      </div>

      {#if outputImage}
        <div>
          <h2 class="text-lg font-semibold">No Background</h2>
          <img src={outputImage} alt="No Background" class="max-w-sm rounded-xl shadow-lg" />
          <button on:click={downloadImage} class="mt-4 px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600">
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
    0% { background-position: 0 0, var(--checkerboard-size) var(--checkerboard-size); }
    100% { background-position: var(--checkerboard-size) var(--checkerboard-size), 0 0; }
  }
  img {
    background: checkerboard;
  }
  img[alt="No Background"] {
    background-image: linear-gradient(45deg, #ccc 25%, transparent 25%),
      linear-gradient(-45deg, #ccc 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, #ccc 75%),
      linear-gradient(-45deg, transparent 75%, #ccc 75%);
    background-size: var(--checkerboard-size) var(--checkerboard-size);
    background-position: 0 0, 0 0, 0 0, 0 0;
    animation: checkerboard 2s infinite linear;
  }
</style>
