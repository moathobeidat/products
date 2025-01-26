<template>
  <div class="container mx-auto px-4 relative">
    <div v-if="error" class="text-red-500 text-center my-4">
      {{ error }}
    </div>
    <div v-else class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-4 gap-4">
      <template v-for="(product, index) in products" :key="product.productId">
        <ProductCard :product="product" :loading="false" />
      </template>
      <template v-if="loading" v-for="i in skeletonCount" :key="`skeleton-${i}`">
        <ProductCard :loading="true" />
      </template>
    </div>
    <div v-if="products.length === 0 && !loading && !error" class="text-center my-4">
      No products found.
    </div>
    <div ref="intersectionTarget" class="h-10 mt-4"></div>

    <Transition name="fade">
      <button
        v-show="showScrollTop"
        @click="scrollToTop"
        class="fixed bottom-4 right-4 bg-primary text-primary-foreground p-2 rounded-full shadow-lg hover:bg-primary/90 focus:outline-none focus:ring-2 focus:ring-primary focus:ring-offset-2 transition-all duration-300 ease-in-out"
        aria-label="Scroll to top"
      >
        <ChevronUp class="w-6 h-6" />
      </button>
    </Transition>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, onUnmounted, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import ProductCard from './ProductCard.vue'
import { ChevronUp } from 'lucide-vue-next'

const route = useRoute()
const router = useRouter()

const products = ref([])
const loading = ref(false)
const error = ref(null)
const page = ref(1)
const hasMore = ref(true)
const intersectionTarget = ref(null)
const observer = ref(null)
const skeletonCount = ref(8)
const showScrollTop = ref(false)

const fetchProductsForPage = async (pageNumber) => {
  const url = `https://stg.action.jo/api/v1/products?slug=accessories&page=${pageNumber}`
  console.log(`Fetching products for page ${pageNumber}:`, url)
  
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`)
  }
  const data = await response.json()
  console.log(`Received data for page ${pageNumber}:`, data)
  return data
}

const loadInitialProducts = async (targetPage) => {
  loading.value = true
  error.value = null
  
  try {
    console.log(`Loading initial products up to page ${targetPage}`)
    const pages = Array.from({ length: targetPage }, (_, i) => i + 1)
    const responses = await Promise.all(pages.map(pageNum => fetchProductsForPage(pageNum)))
    
    const allProducts = responses.flatMap(data => {
      if (data && data.items && Array.isArray(data.items)) {
        return data.items
      }
      console.warn('Invalid data structure received:', data)
      return []
    })

    const uniqueProducts = allProducts.filter((product, index, self) =>
      product && product.productId &&
      index === self.findIndex(p => p.productId === product.productId)
    )

    console.log(`Loaded ${uniqueProducts.length} unique products`)
    products.value = uniqueProducts
    
    const lastResponse = responses[responses.length - 1]
    hasMore.value = lastResponse.pagination && 
                    lastResponse.pagination.page < lastResponse.pagination.lastPage

    console.log('Has more products:', hasMore.value)
  } catch (err) {
    console.error('Error loading initial products:', err)
    error.value = `Failed to load products: ${err.message}`
  } finally {
    loading.value = false
  }
}

const fetchNextPage = async () => {
  if (!hasMore.value || loading.value) return

  loading.value = true
  error.value = null

  const nextPage = page.value + 1

  try {
    console.log(`Fetching next page: ${nextPage}`)
    const data = await fetchProductsForPage(nextPage)
    
    if (data && data.items && Array.isArray(data.items)) {
      const newProducts = data.items.filter(newProduct => 
        newProduct && newProduct.productId && 
        !products.value.some(existingProduct => existingProduct.productId === newProduct.productId)
      )
      
      console.log(`Adding ${newProducts.length} new products`)
      products.value = [...products.value, ...newProducts]
      hasMore.value = data.pagination && data.pagination.page < data.pagination.lastPage
      page.value = nextPage
      
      updateQueryParam()
    } else {
      console.warn('Invalid data structure received for next page:', data)
    }
  } catch (err) {
    console.error('Error fetching next page:', err)
    error.value = `Failed to load more products: ${err.message}`
  } finally {
    loading.value = false
  }
}

const handleIntersection = (entries) => {
  if (entries[0].isIntersecting && !loading.value && hasMore.value) {
    console.log('Intersection observed, fetching next page')
    fetchNextPage()
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
    console.log('Intersection observer set up')
  } else {
    console.warn('Intersection target not found')
  }
}

const handleScroll = () => {
  showScrollTop.value = window.pageYOffset > 300
}

const scrollToTop = () => {
  window.scrollTo({
    top: 0,
    behavior: 'smooth'
  })
}

onMounted(async () => {
  console.log('Component mounted')
  const urlPage = Number(route.query.page) || 1
  page.value = Math.max(1, urlPage)
  console.log(`Initial page: ${page.value}`)
  
  await loadInitialProducts(page.value)
  setupObserver()

  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  if (observer.value) {
    observer.value.disconnect()
    console.log('Intersection observer disconnected')
  }
  window.removeEventListener('scroll', handleScroll)
})

watch(() => route.query.page, async (newPage) => {
  if (newPage && Number(newPage) !== page.value) {
    const pageNum = Math.max(1, Number(newPage))
    if (pageNum !== page.value) {
      console.log(`Page changed to ${pageNum}`)
      page.value = pageNum
      await loadInitialProducts(pageNum)
    }
  }
})
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>

