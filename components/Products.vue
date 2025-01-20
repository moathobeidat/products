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

const fetchProducts = async () => {
  if (!hasMore.value || loading.value) return

  loading.value = true
  error.value = null

  try {
    const response = await fetch(`https://stg.action.jo/api/v1/products?slug=accessories&page=${page.value}`)
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    const data = await response.json()
    
    if (data && data.items && Array.isArray(data.items)) {
      const newProducts = data.items.filter(newProduct => 
        newProduct && newProduct.productId && 
        !products.value.some(existingProduct => existingProduct.productId === newProduct.productId)
      )
      
      products.value = [...products.value, ...newProducts]
      hasMore.value = data.pagination && data.pagination.page < data.pagination.lastPage
      page.value++
      
      updateQueryParam()
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
  if (entries[0].isIntersecting) {
    fetchProducts()
  }
}

const updateQueryParam = () => {
  router.push({ query: { ...route.query, page: page.value } }, { replace: true })
}

const handleScroll = () => {
  const scrollPosition = window.pageYOffset || document.documentElement.scrollTop
  const windowHeight = window.innerHeight
  const documentHeight = document.documentElement.scrollHeight

  if (scrollPosition + windowHeight < documentHeight - 200) {
    const newPage = Math.ceil(scrollPosition / (windowHeight * 0.5))
    if (newPage < page.value) {
      page.value = newPage
      updateQueryParam()
    }
  }
}

onMounted(() => {
  const observer = new IntersectionObserver(handleIntersection)
  observer.observe(intersectionTarget.value)

  const urlPage = Number(route.query.page) || 1
  page.value = urlPage
  fetchProducts()

  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})

watch(() => route.query.page, (newPage) => {
  if (newPage && Number(newPage) !== page.value) {
    page.value = Number(newPage)
    fetchProducts()
  }
})
</script>