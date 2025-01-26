<template>
  <div class="bg-white shadow-md rounded-lg overflow-hidden">
    <div v-if="loading" class="animate-pulse">
      <div class="h-48 bg-gray-300"></div>
      <div class="p-4">
        <div class="h-4 bg-gray-300 rounded w-3/4 mb-2"></div>
        <div class="h-4 bg-gray-300 rounded w-1/2"></div>
      </div>
    </div>
    <template v-else-if="product && product.productId">
      <img 
        :src="productImage" 
        :alt="product.name" 
        class="w-full h-48 object-cover"
        @error="handleImageError"
      />
      <div class="p-4">
        <h2 class="font-bold text-xl mb-2 text-gray-800">{{ product.name }}</h2>
        <p class="text-gray-600">{{ formatPrice(product.minPrice.value) }}</p>
        <p v-if="product.hasOffer" class="text-sm text-red-500">
          <s>{{ formatPrice(product.minPriceBeforeOffer.value) }}</s>
        </p>
        <p class="text-sm text-gray-500">{{ product.brand }}</p>
      </div>
    </template>
    <div v-else class="p-4 text-red-500">
      Error: Invalid product data
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  product: {
    type: Object,
    default: () => ({})
  },
  loading: {
    type: Boolean,
    default: false
  }
})

const productImage = computed(() => {
  if (props.product && props.product.media && props.product.media.cover && props.product.media.cover.length > 0) {
    return props.product.media.cover[0].src
  }
  return '/placeholder.jpg'
})

const handleImageError = (e) => {
  e.target.src = '/placeholder.jpg'
}

const formatPrice = (price) => {
  if (price === null || price === undefined) {
    return 'Price not available'
  }
  return `${price} JOD`
}
</script>

