<script setup>
import { ref, onMounted } from 'vue'
import { storage } from '@/firebase'
import { ref as storageRef, listAll, getDownloadURL, getMetadata } from 'firebase/storage'

// =============================================
// REACTIVE STATE
// =============================================

const images = ref([])
const loading = ref(true)
const expandedItems = ref({})
const showModal = ref(false)
const modalImageUrl = ref('')
const modalImageName = ref('')

// =============================================
// UTILITY FUNCTIONS
// =============================================

// Format file size
function formatFileSize(bytes) {
  if (bytes === 0) return '0 Bytes'
  const k = 1024
  const sizes = ['Bytes', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

// Format date for display
function formatDate(dateString) {
  const date = new Date(dateString)
  return `${date.toLocaleDateString()} ${date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}`
}

// Format full date for details
function formatFullDate(dateString) {
  const date = new Date(dateString)
  return `${date.toLocaleDateString()} ${date.toLocaleTimeString()}`
}

// =============================================
// IMAGE MANAGEMENT FUNCTIONS
// =============================================

// Load images from Firebase Storage
async function loadImages() {
  try {
    loading.value = true
    const imagesRef = storageRef(storage, 'Food_Images') //
    const result = await listAll(imagesRef)

    const imagePromises = result.items.map(async (itemRef) => {
      try {
        const [url, metadata] = await Promise.all([
          getDownloadURL(itemRef),
          getMetadata(itemRef)
        ])

        return {
          name: itemRef.name,
          fullPath: itemRef.fullPath,
          downloadUrl: url,
          previewUrl: url, // You might want to create thumbnails in production
          size: metadata.size,
          timeCreated: metadata.timeCreated,
          contentType: metadata.contentType
        }
      } catch (error) {
        console.error(`Error loading image ${itemRef.name}:`, error)
        return null
      }
    })

    const loadedImages = await Promise.all(imagePromises)
    images.value = loadedImages
      .filter(img => img !== null)
      .sort((a, b) => new Date(b.timeCreated) - new Date(a.timeCreated)) // Sort by newest first

  } catch (error) {
    console.error('Error loading images:', error)
    // You might want to show an error message to the user
  } finally {
    loading.value = false
  }
}

// Toggle expanded details for an image
function toggleDetails(imageName) {
  expandedItems.value[imageName] = !expandedItems.value[imageName]
}

// View image in modal
function viewImage(image) {
  viewFullSize(image)
}

// View full size image
function viewFullSize(image) {
  modalImageUrl.value = image.downloadUrl
  modalImageName.value = image.name
  showModal.value = true
}

// Close modal
function closeModal() {
  showModal.value = false
  modalImageUrl.value = ''
  modalImageName.value = ''
}

// Download image
function downloadImage(image) {
  try {
    console.log('Downloading image:', image.name) // Debug log

    // Create a temporary anchor element
    const link = document.createElement('a')

    // Set the href to the Firebase Storage download URL
    link.href = image.downloadUrl

    // Set the download attribute to specify the filename (ensure it's the original name)
    link.download = image.name || 'downloaded-image.jpg'

    // Set target to _blank to open in new tab (fallback for some browsers)
    link.target = '_blank'

    // Make the link invisible
    link.style.display = 'none'

    // Add to DOM, click, and remove
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)

    console.log('Download initiated for:', image.name) // Debug log
  } catch (error) {
    console.error('Error downloading image:', error)
    alert('Failed to download image. Please try again.')
  }
}

// =============================================
// LIFECYCLE HOOKS
// =============================================

onMounted(() => {
  loadImages()
})
</script>

<template>
  <main class="min-h-screen bg-gray-50 font-inter leading-relaxed tracking-wide">
    <div class="container mx-auto p-6 max-w-7xl">
      <!-- Header Section -->
      <div class="flex justify-between items-center mb-8">
        <div>
          <h1 class="text-4xl font-light text-gray-800 tracking-wide">Captured Images</h1>
          <p class="text-gray-600 mt-2">View and download captured food images from sensor</p>
        </div>

        <div class="flex gap-3">
          <!-- Return to Dashboard Button -->
          <router-link to="/" class="inline-flex items-center px-4 py-2 bg-gray-800 text-white rounded-lg hover:bg-gray-900 transition-colors font-medium">
            <svg class="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"></path>
            </svg>
            Return to Dashboard
          </router-link>
        </div>
      </div>

      <!-- Loading State -->
      <div v-if="loading" class="flex justify-center items-center py-12">
        <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-slate-600"></div>
      </div>

      <!-- No Images State -->
      <div v-else-if="images.length === 0" class="text-center py-12">
        <div class="text-gray-500 text-lg">No images found</div>
      </div>

      <!-- Images List -->
      <div v-else class="space-y-4">
        <div
          v-for="image in images"
          :key="image.name"
          class="bg-white shadow-sm rounded-lg border border-gray-200 overflow-hidden"
        >
          <div class="p-6">
            <div class="flex items-start justify-between">
              <!-- Image Info -->
              <div class="flex items-center space-x-4">
                <!-- File Type Badge -->
                <div class="bg-gray-100 rounded px-2 py-1 text-xs font-medium text-gray-600 uppercase">
                  JPEG
                </div>

                <!-- Image Details -->
                <div>
                  <h3 class="font-medium text-gray-900">{{ image.name }}</h3>
                  <div class="flex items-center space-x-4 text-sm text-gray-500 mt-1">
                    <span class="flex items-center">
                      <svg class="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"></path>
                      </svg>
                      {{ formatDate(image.timeCreated) }}
                    </span>
                    <span>{{ formatFileSize(image.size) }}</span>
                  </div>
                </div>
              </div>

              <!-- Action Buttons -->
              <div class="flex items-center space-x-2">
                <button
                  @click="viewImage(image)"
                  class="p-2 text-gray-500 hover:text-gray-700 hover:bg-gray-100 rounded-lg transition-colors"
                  title="View"
                >
                  <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
                  </svg>
                </button>

                <button
                  @click="downloadImage(image)"
                  class="p-2 text-gray-500 hover:text-gray-700 hover:bg-gray-100 rounded-lg transition-colors"
                  title="Download"
                >
                  <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"></path>
                  </svg>
                </button>

                <button
                  @click="toggleDetails(image.name)"
                  class="p-2 text-gray-500 hover:text-gray-700 hover:bg-gray-100 rounded-lg transition-colors"
                  :title="expandedItems[image.name] ? 'Collapse' : 'Expand'"
                >
                  <svg class="w-5 h-5 transform transition-transform" :class="{ 'rotate-180': expandedItems[image.name] }" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
                  </svg>
                </button>
              </div>
            </div>

            <!-- Expanded Details -->
            <div v-if="expandedItems[image.name]" class="mt-6 pt-6 border-t border-gray-200">
              <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <!-- Image Preview -->
                <div class="bg-gray-100 rounded-lg p-4 flex items-center justify-center min-h-48">
                  <img
                    v-if="image.previewUrl"
                    :src="image.previewUrl"
                    :alt="image.name"
                    class="max-w-full max-h-48 object-contain rounded"
                  />
                  <div v-else class="text-gray-400">
                    <svg class="w-16 h-16 mx-auto" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z"></path>
                    </svg>
                    <p class="text-sm mt-2">Image Preview</p>
                  </div>
                </div>

                <!-- Image Details -->
                <div class="space-y-4">
                  <h3 class="text-lg font-medium text-gray-800 mb-4">Image Details</h3>

                  <div class="space-y-3">
                    <div class="flex justify-between py-2">
                      <span class="text-gray-600">File Name:</span>
                      <span class="font-medium text-gray-900">{{ image.name }}</span>
                    </div>

                    <div class="flex justify-between py-2">
                      <span class="text-gray-600">Captured:</span>
                      <span class="font-medium text-gray-900">{{ formatFullDate(image.timeCreated) }}</span>
                    </div>

                    <div class="flex justify-between py-2">
                      <span class="text-gray-600">File Size:</span>
                      <span class="font-medium text-gray-900">{{ formatFileSize(image.size) }}</span>
                    </div>

                    <div class="flex justify-between py-2">
                      <span class="text-gray-600">Format:</span>
                      <span class="font-medium text-gray-900">image/jpeg</span>
                    </div>
                  </div>

                  <div class="flex space-x-3 pt-4">
                    <button
                      @click="viewFullSize(image)"
                      class="flex-1 flex items-center justify-center px-4 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 transition-colors"
                    >
                      <svg class="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
                      </svg>
                      View Full Size
                    </button>

                    <button
                      @click="downloadImage(image)"
                      class="flex-1 flex items-center justify-center px-4 py-2 bg-slate-600 text-white rounded-lg hover:bg-slate-700 transition-colors"
                    >
                      <svg class="w-4 h-4 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"></path>
                      </svg>
                      Download
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Full Size Image Modal -->
    <div v-if="showModal" class="fixed inset-0 bg-black bg-opacity-75 flex items-center justify-center z-50" @click="closeModal">
      <div class="max-w-4xl max-h-full p-4">
        <img
          :src="modalImageUrl"
          :alt="modalImageName"
          class="max-w-full max-h-full object-contain"
          @click.stop
        />
      </div>
      <button
        @click="closeModal"
        class="absolute top-4 right-4 text-white hover:text-gray-300 transition-colors"
      >
        <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
        </svg>
      </button>
    </div>
  </main>
</template>

<style scoped>
/* Font styling for modern look */
.font-inter {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
}

/* Custom focus styles for better accessibility */
button:focus {
  box-shadow: 0 0 0 2px rgba(156, 163, 175, 0.5);
}

/* Smooth transitions for interactive elements */
button {
  transition: all 0.2s ease-in-out;
}

/* Custom scrollbar for better visual consistency */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  background: #f1f5f9;
}

::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

/* Modal backdrop blur effect */
.backdrop-blur {
  backdrop-filter: blur(4px);
}
</style>
