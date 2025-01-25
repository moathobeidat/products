<template>
  <div class="container mx-auto px-4">
    <div v-if="error" class="text-red-500 text-center my-4">
      {{ error }}
    </div>
    <div v-else class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-4">
      <template v-for="(product, index) in products" :key="product.productId">
        <ProductCard :product="product" />
        <LoaderSvg v-if="index === products.length - 1 && loading" />
      </template>
    </div>
    <div v-if="products.length === 0 && loading" class="text-center my-4">
      <LoaderSvg />
    </div>
    <div v-if="products.length === 0 && !loading && !error" class="text-center my-4">
      No products found.
    </div>
    <div ref="intersectionTarget" class="h-10 mt-4"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, onUnmounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import ProductCard from './ProductCard.vue'
import LoaderSvg from './LoaderSvg.vue'

const route = useRoute()
const router = useRouter()

const products = ref([])
const loading = ref(false)
const error = ref(null)
const page = ref(1)
const hasMore = ref(true)
const intersectionTarget = ref(null)
const observer = ref(null)

const fetchProducts = async (resetProducts = false) => {
  if (!hasMore.value || loading.value) return

  loading.value = true
  error.value = null

  const currentPage = resetProducts ? page.value : page.value + 1

  try {
    const response = await fetch(`https://stg.action.jo/api/v1/products?slug=accessories&page=${currentPage}`)
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    const data = await response.json()
    
    if (data && data.items && Array.isArray(data.items)) {
      const newProducts = data.items.filter(newProduct => 
        newProduct && newProduct.productId && 
        !products.value.some(existingProduct => existingProduct.productId === newProduct.productId)
      )
      
      if (resetProducts) {
        products.value = newProducts
      } else {
        products.value = [...products.value, ...newProducts]
        page.value = currentPage
      }
      
      hasMore.value = data.pagination && data.pagination.page < data.pagination.lastPage
      
      if (!resetProducts) {
        updateQueryParam()
      }
    } else {
      console.error('Invalid data structure:', data)
      throw new Error('Invalid data structure received from API')
    }
  } catch (err) {
    console.error('Error fetching products:', err)
    error.value = `Failed to load products: ${err.message}`
  } finally {
    loading.value = false
  }
}

const handleIntersection = (entries) => {
  if (entries[0].isIntersecting && !loading.value && hasMore.value) {
    fetchProducts(false)
  }
}

const updateQueryParam = () => {
  router.push({ query: { ...route.query, page: page.value } }, { replace: true })
}

const setupObserver = () => {
  if (observer.value) {
    observer.value.disconnect()
  }
  
  observer.value = new IntersectionObserver(handleIntersection, {
    root: null,
    rootMargin: '100px',
    threshold: 0.1
  })
  
  if (intersectionTarget.value) {
    observer.value.observe(intersectionTarget.value)
  }
}

onMounted(() => {
  const urlPage = Number(route.query.page) || 1
  page.value = Math.max(1, urlPage)
  setupObserver()
  fetchProducts(true)
})

onUnmounted(() => {
  if (observer.value) {
    observer.value.disconnect()
  }
})

watch(() => route.query.page, (newPage) => {
  if (newPage && Number(newPage) !== page.value) {
    const pageNum = Math.max(1, Number(newPage))
    if (pageNum !== page.value) {
      page.value = pageNum
      fetchProducts(true)
    }
  }
})
</script>

