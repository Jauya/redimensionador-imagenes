<template>
  <div class="flex flex-col justify-center gap-5 max-w-6xl mx-auto">
    <div class="flex flex-col gap-4">
      <input class="block w-full border border-gray-200 shadow-sm rounded-lg text-sm focus:z-10 focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400
      file:bg-gray-50 file:border-0
      file:me-4
      file:py-3 file:px-4
      dark:file:bg-neutral-700 dark:file:text-neutral-400" type="file" multiple accept="image/*"
        @change="handleImageSelect">
      <div class="flex gap-5 w-full">

        <label class="w-full flex items-center gap-2" for="width">Ancho
          <input
            class="py-3 px-4 block w-full border-gray-200 rounded-lg text-sm focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400 dark:placeholder-neutral-500 dark:focus:ring-neutral-600"
            type="number" v-model="width" placeholder="Ej: 1500">
        </label>

        <label class="w-full flex items-center gap-2" for="height">Alto
          <input
            class="py-3 px-4 block w-full border-gray-200 rounded-lg text-sm focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400 dark:placeholder-neutral-500 dark:focus:ring-neutral-600"
            type="number" v-model="height" placeholder="Opcional">
        </label>
      </div>

      <button
        class="cursor-pointer py-3 px-4 inline-flex items-center justify-center gap-x-2 text-sm font-medium rounded-lg border border-transparent bg-blue-600 text-white hover:bg-blue-700 focus:outline-none focus:bg-blue-700 disabled:opacity-50 disabled:pointer-events-none"
        @click="resizeImages">Redimensionar y Descargar ZIP</button>

    </div>
    <div class="border columns-3 gap-5 p-2 border-zinc-700 rounded-lg bg-zinc-900" id="imagePreview">
      <img class="rounded-lg object-cover aspect-square" v-for="image in previewImages" :key="image.name"
        :src="image.dataUrl" :alt="image.name">
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { ZipWriter, BlobWriter, BlobReader } from '@zip.js/zip.js';

const width = ref<number | null>(1500); // Se puede establecer o dejar en null
const height = ref<number | null>(null); // Si es null, se calcula automáticamente
const selectedImages = ref<File[]>([]);
const previewImages = ref<{ name: string; dataUrl: string }[]>([]); // Ahora muestra las imágenes originales

const handleImageSelect = (event: Event) => {
  const files = (event.target as HTMLInputElement).files;
  if (files) {
    selectedImages.value = Array.from(files);

    previewImages.value = [];
    for (const file of selectedImages.value) {
      const reader = new FileReader();
      reader.onload = () => {
        previewImages.value.push({ name: file.name, dataUrl: reader.result as string });
      };
      reader.readAsDataURL(file);
    }
  }
};

const resizeImage = (file: File): Promise<{ name: string; dataUrl: string }> => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.src = reader.result as string;

      img.onload = () => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        if (!ctx) return;

        let newWidth = img.width;
        let newHeight = img.height;

        if (width.value && !height.value) {
          newWidth = width.value;
          newHeight = (img.height / img.width) * width.value;
        } else if (!width.value && height.value) {
          newHeight = height.value;
          newWidth = (img.width / img.height) * height.value;
        } else if (width.value && height.value) {
          newWidth = width.value;
          newHeight = height.value;
        }

        canvas.width = newWidth;
        canvas.height = newHeight;
        ctx.drawImage(img, 0, 0, newWidth, newHeight);

        const resizedImage = canvas.toDataURL('image/jpeg');
        resolve({ name: file.name, dataUrl: resizedImage });
      };
    };
    reader.readAsDataURL(file);
  });
};

const resizeImages = async () => {
  if (!selectedImages.value.length) return;

  const zipWriter = new ZipWriter(new BlobWriter("application/zip"));

  await Promise.all(
    selectedImages.value.map(async (file) => {
      const resized = await resizeImage(file);
      const blob = await fetch(resized.dataUrl).then(res => res.blob());
      await zipWriter.add(file.name, new BlobReader(blob));
    })
  );

  const zipBlob = await zipWriter.close();
  const link = document.createElement('a');
  link.href = URL.createObjectURL(zipBlob);
  link.download = 'images.zip';
  link.click();
};
</script>

<style scoped>
#imagePreview img {
  max-width: 200px;
  margin: 5px;
}
</style>
