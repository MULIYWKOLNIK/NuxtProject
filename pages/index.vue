<script setup lang="ts">
import { h, resolveComponent } from 'vue'
import { getPaginationRowModel } from '@tanstack/vue-table'
import type { TableColumn } from '@nuxt/ui'
import type { Column } from '@tanstack/vue-table'

useHead({
  title: 'Список продуктів'
})

const UButton = resolveComponent('UButton')
const UDropdownMenu = resolveComponent('UDropdownMenu')

const table = useTemplateRef('table')

interface Product {
  id: number
  title: string
  description: string
  price: number
  rating: number
  brand: string
  category: string
  thumbnail: string
}

interface ProductsResponse {
  products: Product[]
  total: number
  skip: number
  limit: number
}

const { data: productsData, status } = await useFetch<ProductsResponse>('https://dummyjson.com/products?limit=0', {
  key: 'products-table',
  lazy: true
})

const products = computed(() => productsData.value?.products || [])

function getHeaderWithoutSorting(label: string) {
  return h(UButton, {
    color: 'neutral',
    variant: 'ghost',
    label,
    class: '-mx-2.5',
  })
}

function getHeader(column: Column<any>, label: string) {
  const isSorted = column.getIsSorted()

  return h(
      UDropdownMenu,
      {
        content: {
          align: 'start'
        },
        'aria-label': 'Actions dropdown',
        items: [
          {
            label: 'Asc',
            type: 'checkbox',
            icon: 'i-lucide-arrow-up-narrow-wide',
            checked: isSorted === 'asc',
            onSelect: () => {
              if (isSorted === 'asc') {
                column.clearSorting()
              } else {
                column.toggleSorting(false)
              }
            }
          },
          {
            label: 'Desc',
            icon: 'i-lucide-arrow-down-wide-narrow',
            type: 'checkbox',
            checked: isSorted === 'desc',
            onSelect: () => {
              if (isSorted === 'desc') {
                column.clearSorting()
              } else {
                column.toggleSorting(true)
              }
            }
          }
        ]
      },
      () =>
          h(UButton, {
            color: 'neutral',
            variant: 'ghost',
            label,
            icon: isSorted
                ? isSorted === 'asc'
                    ? 'i-lucide-arrow-up-narrow-wide'
                    : 'i-lucide-arrow-down-wide-narrow'
                : 'i-lucide-arrow-up-down',
            class: '-mx-2.5 data-[state=open]:bg-(--ui-bg-elevated)',
            'aria-label': `Sort by ${isSorted === 'asc' ? 'descending' : 'ascending'}`
          })
  )
}

const isMobile = ref(false)
const isTablet = ref(false)

const columns = computed(() => {
  const baseColumns: TableColumn<Product>[] = [
    {
      accessorKey: 'title',
      header: ({ column }) => getHeader(column, 'Назва'),
    },
    {
      accessorKey: 'price',
      header: ({ column }) => getHeader(column, 'Ціна'),
      cell: ({ row }) => {
        const price = Number.parseFloat(row.getValue('price'))
        return h('div', { class: 'text-right font-medium pr-2 md:pr-4' }, `$${price}`)
      }
    },
    {
      accessorKey: 'thumbnail',
      header: () => getHeaderWithoutSorting('Фото'),
      cell: ({ row }) => {
        const thumbnail = row.getValue('thumbnail') as string
        return h('img', {
          src: thumbnail,
          alt: `${row.getValue('title')} thumbnail`,
          class: 'w-16 h-16 sm:w-20 sm:h-20 md:w-[100px] md:h-[100px] object-cover rounded-md'
        })
      }
    },
  ]

  if (!isMobile.value) {
    baseColumns.splice(1, 0, {
      accessorKey: 'description',
      header: ({ column }) => getHeader(column, 'Опис'),
      cell: ({ row }) => {
        const description = row.getValue('description') as string
        return description.length > (isTablet.value ? 30 : 50) ?
            `${description.substring(0, isTablet.value ? 30 : 50)}...` : description
      }
    })
  }

  if (!isMobile.value || !isTablet.value) {
    baseColumns.splice(isMobile.value ? 2 : 3, 0, {
      accessorKey: 'rating',
      header: ({ column }) => getHeader(column, 'Оцінка'),
      cell: ({ row }) => {
        const rating = Number.parseFloat(row.getValue('rating'))
        const color = rating >= 4.5 ? 'text-green-500' : 'text-red-500'
        return h('div', { class: `font-medium ${color} pl-2 md:pl-4` }, rating.toFixed(1))
      }
    })
  }

  if (!isTablet.value) {
    baseColumns.push(
        {
          accessorKey: 'brand',
          header: ({ column }) => getHeader(column, 'Бренд'),
        },
        {
          accessorKey: 'category',
          header: ({ column }) => getHeader(column, 'Категорія'),
        }
    )
  }

  return baseColumns
})

const pagination = ref({
  pageIndex: 0,
  pageSize: isMobile.value ? 5 : 10
})

const sorting = ref([{
  id: 'title',
  desc: false
}])

onMounted(() => {
  checkScreenSize()
  window.addEventListener('resize', checkScreenSize)
})

onUnmounted(() => {
  window.removeEventListener('resize', checkScreenSize)
})

function checkScreenSize() {
  isMobile.value = window.innerWidth < 640
  isTablet.value = window.innerWidth >= 640 && window.innerWidth < 1024

  pagination.value.pageSize = isMobile.value ? 5 : 10
}
</script>

<template>
  <div class="min-h-screen bg-[#2e2e2e] p-2 sm:p-4 md:p-8">
    <div class="mb-4 py-2 sm:py-3 w-full">
      <UInput
          :model-value="table?.tableApi?.getColumn('title')?.getFilterValue() as string"
          class="w-full md:max-w-md px-3 sm:px-4 py-1.5 sm:py-2 rounded-lg"
          placeholder="Пошук за назвою"
          @update:model-value="table?.tableApi?.getColumn('title')?.setFilterValue($event)"
      />
    </div>

    <div class="bg-gray-700 rounded-lg shadow-lg flex flex-col justify-center items-center">
      <UTable
          ref="table"
          v-model:pagination="pagination"
          v-model:sorting="sorting"
          :columns="columns"
          :data="products"
          :loading="status === 'pending'"
          search-placeholder="Пошук товарів..."
          :pagination-options="{
              getPaginationRowModel: getPaginationRowModel()
            }"
          sort-mode="all"
          class="rounded-lg overflow-hidden w-full"
          :ui="{
              wrapper: 'overflow-auto',
              td: {
                base: 'whitespace-normal break-words p-1 sm:p-2 md:p-3',
                padding: 'px-1 sm:px-2 md:px-4 py-1.5 sm:py-2 md:py-4'
              },
              th: {
                base: 'whitespace-nowrap p-1 sm:p-2 md:p-3',
                padding: 'px-1 sm:px-2 md:px-4 py-1.5 sm:py-2 md:py-4'
              },
              tbody: {
                base: 'divide-y divide-gray-600'
              },
              tr: {
                base: 'hover:bg-gray-600 transition-colors'
              }
            }"
      />
    </div>

    <div class="flex justify-center mt-4 sm:mt-6 space-x-1 sm:space-x-2 overflow-x-auto">
      <UPagination
          :default-page="(table?.tableApi?.getState().pagination.pageIndex || 0) + 1"
          :items-per-page="table?.tableApi?.getState().pagination.pageSize"
          :total="products.length"
          @update:page="(p: number) => table?.tableApi?.setPageIndex(p - 1)"
          :ui="{
              wrapper: 'flex flex-wrap justify-center gap-1 md:gap-2',
              item: {
                size: 'xs sm:sm md:md',
                padding: 'px-1.5 sm:px-2.5 md:px-3.5 py-0.5 sm:py-1 md:py-1.5'
              },
              rounded: 'rounded-md'
            }"
      />
    </div>
  </div>
</template>