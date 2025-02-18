<template>
  <div class="flex flex-col justify-center gap-5 max-w-3xl mx-auto min-w-3xl">
    <div class="flex flex-col gap-4">
      <input
        class="transition-all duration-300 block w-full border border-gray-200 shadow-sm rounded-lg text-sm focus:z-10 focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400 file:bg-gray-50 file:border-0 file:me-4 file:py-3 file:px-4 dark:file:bg-neutral-700 dark:file:text-neutral-400"
        type="file" multiple accept="image/*" @change="handleImageSelect" :disabled="loading">
      <div class="flex gap-5 w-full">

        <label class="w-full flex items-center gap-2" for="width">Ancho
          <input
            class="py-3 px-4 block w-full border-gray-200 rounded-lg text-sm focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400 dark:placeholder-neutral-500 dark:focus:ring-neutral-600"
            type="number" v-model="width" placeholder="Ej: 1500" :disabled="loading">
        </label>

        <label class="w-full flex items-center gap-2" for="height">Alto
          <input
            class="py-3 px-4 block w-full border-gray-200 rounded-lg text-sm focus:border-blue-500 focus:ring-blue-500 disabled:opacity-50 disabled:pointer-events-none dark:bg-neutral-900 dark:border-neutral-700 dark:text-neutral-400 dark:placeholder-neutral-500 dark:focus:ring-neutral-600"
            type="number" v-model="height" placeholder="Opcional" :disabled="loading">
        </label>
      </div>

      <button
        :class="['transition-all duration-300 cursor-pointer py-3 px-4 inline-flex items-center justify-center gap-x-2 text-sm font-medium rounded-lg border border-transparent bg-blue-600 text-white hover:bg-blue-700 focus:outline-none focus:bg-blue-700 disabled:opacity-50 disabled:pointer-events-none', loading && 'animate-pulse']"
        @click="resizeImages" :disabled="loading || !previewImages.length">Redimensionar y Descargar ZIP</button>

    </div>
    <div :class="['border columns-3 gap-5 p-2 border-zinc-700 rounded-lg bg-zinc-900', loading && 'animate-pulse']"
      id="imagePreview">
      <img class="rounded-lg object-cover mb-5 w-full" v-for="image in previewImages" :key="image.name"
        :src="image.previewDataUrl" :alt="image.name" loading="lazy">
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { ZipWriter, BlobWriter, BlobReader } from '@zip.js/zip.js';

interface ImageData {
  name: string;
  previewDataUrl: string; // Thumbnail optimizado (ej. 400px de ancho)
  file: File;             // Archivo original para la redimensión final
}

// Valores para la redimensión final (ZIP)
const width = ref<number | null>(1500);
const height = ref<number | null>(null);

// Estado de carga global
const loading = ref(false);

// Array que contendrá la información de cada imagen
const previewImages = ref<ImageData[]>([]);

/**
 * Lee un archivo y retorna una promesa con su dataURL (full resolution).
 */
function readFileAsDataUrl(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

/**
 * Genera un thumbnail de una imagen a partir de su dataURL.
 * Se redimensiona al targetWidth manteniendo la relación de aspecto.
 */
function generateThumbnail(dataUrl: string, targetWidth: number): Promise<string> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => {
      const aspectRatio = img.height / img.width;
      const targetHeight = targetWidth * aspectRatio;
      const canvas = document.createElement('canvas');
      canvas.width = targetWidth;
      canvas.height = targetHeight;
      const ctx = canvas.getContext('2d');
      if (!ctx) {
        reject(new Error('No se pudo obtener el contexto 2D del canvas'));
        return;
      }
      ctx.drawImage(img, 0, 0, targetWidth, targetHeight);
      resolve(canvas.toDataURL('image/jpeg'));
    };
    img.onerror = reject;
    img.src = dataUrl;
  });
}

/**
 * Maneja la selección de archivos:
 * - Lee cada archivo
 * - Genera un thumbnail de 400px de ancho para la vista previa
 * - Guarda la información en previewImages
 */
const handleImageSelect = async (event: Event) => {
  loading.value = true;
  const input = event.target as HTMLInputElement;
  if (!input.files) {
    loading.value = false;
    return;
  }
  const files = Array.from(input.files);
  previewImages.value = await Promise.all(
    files.map(async (file) => {
      const fullDataUrl = await readFileAsDataUrl(file);
      const previewDataUrl = await generateThumbnail(fullDataUrl, 400);
      return { name: file.name, previewDataUrl, file };
    })
  );
  loading.value = false;
};

/**
 * Redimensiona una imagen a partir de su dataURL y devuelve una nueva dataURL.
 * Se mantiene la relación de aspecto si se define solo un valor (width o height).
 */
const resizeImage = (
  dataUrl: string,
  name: string
): Promise<{ name: string; dataUrl: string }> => {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => {
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

      const canvas = document.createElement('canvas');
      canvas.width = newWidth;
      canvas.height = newHeight;
      const ctx = canvas.getContext('2d');
      if (!ctx) {
        reject(new Error('No se pudo obtener el contexto 2D'));
        return;
      }
      ctx.drawImage(img, 0, 0, newWidth, newHeight);
      const resizedDataUrl = canvas.toDataURL('image/jpeg');
      resolve({ name, dataUrl: resizedDataUrl });
    };
    img.onerror = reject;
    img.src = dataUrl;
  });
};

/**
 * Redimensiona todas las imágenes (usando el archivo original) y genera un ZIP para descargar.
 */
const resizeImages = async () => {
  if (!previewImages.value.length) return;
  loading.value = true;
  const zipWriter = new ZipWriter(new BlobWriter('application/zip'));

  await Promise.all(
    previewImages.value.map(async (image) => {
      // Usar el archivo original para obtener la máxima calidad en el redimensionado
      const originalDataUrl = await readFileAsDataUrl(image.file);
      const resized = await resizeImage(originalDataUrl, image.name);
      const blob = await fetch(resized.dataUrl).then((res) => res.blob());
      await zipWriter.add(resized.name, new BlobReader(blob));
    })
  );

  const zipBlob = await zipWriter.close();
  const link = document.createElement('a');
  link.href = URL.createObjectURL(zipBlob);
  link.download = 'images.zip';
  link.click();
  loading.value = false;
};

</script>
