<script setup lang="ts">
import { h, resolveComponent } from 'vue'
import { getPaginationRowModel } from '@tanstack/vue-table'
import type { TableColumn } from '@nuxt/ui'
import type { Column } from '@tanstack/vue-table'

useHead({
  title: 'Список продуктів'
})

const UBadge = resolveComponent('UBadge')
const UButton = resolveComponent('UButton')
const UDropdownMenu = resolveComponent('UDropdownMenu')
const UAvatar = resolveComponent('UAvatar')

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

const columns: TableColumn<Product>[] = [
  {
    accessorKey: 'title',
    header: ({ column }) => getHeader(column, 'Назва'),
  },
  {
    accessorKey: 'description',
    header: ({ column }) => getHeader(column, 'Опис'),
    cell: ({ row }) => {
      const description = row.getValue('description') as string
      return description.length > 50 ? `${description.substring(0, 50)}...` : description
    }
  },
  {
    accessorKey: 'price',
    header: ({ column }) => getHeader(column, 'Ціна'),
    cell: ({ row }) => {
      const price = Number.parseFloat(row.getValue('price'))
      return h('div', { class: 'text-right font-medium pr-4' }, `$${price}`)
    }
  },
  {
    accessorKey: 'rating',
    header: ({ column }) => getHeader(column, 'Оцінка'),
    cell: ({ row }) => {
      const rating = Number.parseFloat(row.getValue('rating'))
      const color = rating >= 4.5 ? 'text-green-500' : 'text-red-500'
      return h('div', { class: `font-medium ${color} pl-4` }, rating.toFixed(1))
    }
  },
  {
    accessorKey: 'brand',
    header: ({ column }) => getHeader(column, 'Бренд'),
  },
  {
    accessorKey: 'category',
    header: ({ column }) => getHeader(column, 'Категорія'),
  },
  {
    accessorKey: 'thumbnail',
    header: () => getHeaderWithoutSorting('Фото'),
    cell: ({ row }) => {
      const thumbnail = row.getValue('thumbnail') as string
      return h('img', {
        src: thumbnail,
        alt: `${row.getValue('title')} thumbnail`,
        class: 'w-[100px] h-[100px] object-cover rounded-md' // Розмір 100x100 та обтинання зображення
      })
    }
  },
]

const pagination = ref({
  pageIndex: 0,
  pageSize: 10
})

const sorting = ref([{
  id: 'title',
  desc: false
}])
</script>

<template>
  <div class="min-h-screen bg-[#2e2e2e] p-8">
    <div class="mx-auto max-w-7xl">
      <div class="bg-gray-700 rounded-lg shadow-lg">
        <div class="mb-4 py-3 flex justify-end">
          <UInput
              :model-value="table?.tableApi?.getColumn('title')?.getFilterValue() as string"
              class="px-4 py-2 min-w-80 me-4 rounded-lg"
              placeholder="Пошук за назвою"
              @update:model-value="table?.tableApi?.getColumn('title')?.setFilterValue($event)"
          />
        </div>

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
            class="rounded-lg overflow-hidden"
        />
      </div>

      <!-- Покращена пагінація -->
      <div class="flex justify-center mt-6 space-x-2">
        <UPagination
            :default-page="(table?.tableApi?.getState().pagination.pageIndex || 0) + 1"
            :items-per-page="table?.tableApi?.getState().pagination.pageSize"
            :total="products.length"
            @update:page="(p) => table?.tableApi?.setPageIndex(p - 1)"
            class="bg-gray-600 text-white p-3 rounded-lg"
        />
      </div>
    </div>
  </div>
</template>
