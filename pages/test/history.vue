<template>
  <div class="p-6">
    <h1 class="text-2xl font-bold mb-4">預約紀錄</h1>
    <div class="space-y-4">
      <!-- 控制面板 -->
      <div class="flex flex-wrap justify-between items-center mb-4 gap-4">
        <div class="flex flex-wrap items-center gap-4">
          <div class="flex items-center gap-2">
            <span class="text-sm">每頁顯示：</span>
            <Select v-model="itemsPerPage">
              <SelectTrigger class="w-[120px]">
                <SelectValue :placeholder="itemsPerPage" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem v-for="size in pageSizes" :key="size" :value="size">
                  {{ size }} 筆
                </SelectItem>
              </SelectContent>
            </Select>
          </div>
          <div class="flex items-center gap-2">
            <span class="text-sm">排序：</span>
            <Select v-model="sortConfig.field">
              <SelectTrigger class="w-[120px]">
                <SelectValue placeholder="選擇排序欄位" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="title">書名</SelectItem>
                <SelectItem value="author">作者</SelectItem>
                <SelectItem value="borrowDate">借閱日</SelectItem>
                <SelectItem value="dueDate">到期日</SelectItem>
              </SelectContent>
            </Select>
            <Button variant="outline" @click="toggleSortOrder">
              <span v-if="sortConfig.ascending">↑ 升冪</span>
              <span v-else>↓ 降冪</span>
            </Button>
          </div>
        </div>
        <div class="flex items-center gap-2">
          <Button variant="outline" @click="viewMode = 'table'" :class="{ 'bg-primary text-primary-foreground': viewMode === 'table' }">
            <ListIcon class="h-4 w-4" />
          </Button>
          <Button variant="outline" @click="viewMode = 'grid'" :class="{ 'bg-primary text-primary-foreground': viewMode === 'grid' }">
            <GridIcon class="h-4 w-4" />
          </Button>
        </div>
      </div>

      <!-- 表格視圖 -->
      <Table v-if="viewMode === 'table'">
        <TableHeader>
          <TableRow>
            <TableHead class="cursor-pointer" @click="updateSort('title')">
              書名 {{ getSortIcon('title') }}
            </TableHead>
            <TableHead class="cursor-pointer" @click="updateSort('author')">
              作者 {{ getSortIcon('author') }}
            </TableHead>
            <TableHead class="cursor-pointer" @click="updateSort('borrowDate')">
              借閱日 {{ getSortIcon('borrowDate') }}
            </TableHead>
            <TableHead class="cursor-pointer" @click="updateSort('dueDate')">
              到期日 {{ getSortIcon('dueDate') }}
            </TableHead>
            <TableHead>書本資訊</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          <TableRow v-for="(book, index) in paginatedBooks" :key="index">
            <TableCell>{{ book.title }}</TableCell>
            <TableCell>{{ book.author }}</TableCell>
            <TableCell>{{ book.borrowDate }}</TableCell>
            <TableCell>{{ book.dueDate }}</TableCell>
            <TableCell>
              <Button @click="viewBookDetail(book)">詳情</Button>
            </TableCell>
          </TableRow>
        </TableBody>
      </Table>

      <!-- 網格視圖 -->
      <div v-else class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        <div v-for="(book, index) in paginatedBooks" 
             :key="index"
             class="border rounded-lg overflow-hidden shadow-sm hover:shadow-md transition-shadow">
          <div class="aspect-[3/4] relative bg-gray-100">
            <img :src="book.coverUrl || getDefaultCoverUrl(index)" 
                 :alt="book.title"
                 class="object-cover w-full h-full" />
          </div>
          <div class="p-4 space-y-2">
            <h3 class="font-semibold truncate">{{ book.title }}</h3>
            <p class="text-sm text-gray-600 truncate">{{ book.author }}</p>
            <div class="text-xs text-gray-500">
              <p>借閱日：{{ book.borrowDate }}</p>
              <p>到期日：{{ book.dueDate }}</p>
            </div>
            <Button class="w-full" @click="viewBookDetail(book)">詳情</Button>
          </div>
        </div>
      </div>

      <!-- 分頁控制 -->
      <div class="flex flex-col items-center gap-4">
        <div class="flex items-center gap-1">
          <Button 
            variant="outline" 
            class="h-8 w-8 p-0"
            :disabled="currentPage === 1"
            @click="currentPage--"
          >
            <span class="sr-only">上一頁</span>
            <ChevronLeftIcon class="h-4 w-4" />
          </Button>
          <div class="flex items-center">
            <div class="flex items-center gap-1">
              <span>共{{ totalPages }}頁</span>
              <Input
                type="number"
                :value="currentPage"
                class="h-8 w-12 text-center"
                min="1"
                :max="totalPages"
                @change="e => goToPage(parseInt(e.target.value))"
              />
              <span>/{{ totalPages }}頁</span>
            </div>
          </div>
          <Button 
            variant="outline"
            class="h-8 w-8 p-0"
            :disabled="currentPage >= totalPages"
            @click="currentPage++"
          >
            <span class="sr-only">下一頁</span>
            <ChevronRightIcon class="h-4 w-4" />
          </Button>
        </div>
        <div class="text-sm text-gray-500 text-center">
          顯示第 {{ (currentPage - 1) * itemsPerPage + 1 }} 到 {{ Math.min(currentPage * itemsPerPage, sortedBooks.length) }} 筆，共 {{ sortedBooks.length }} 筆
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import Table from '@/components/ui/table/Table.vue'
import TableBody from '@/components/ui/table/TableBody.vue'
import TableCell from '@/components/ui/table/TableCell.vue'
import TableHead from '@/components/ui/table/TableHead.vue'
import TableHeader from '@/components/ui/table/TableHeader.vue'
import TableRow from '@/components/ui/table/TableRow.vue'
import Button from '@/components/ui/button/Button.vue'
import Select from '@/components/ui/select/Select.vue'
import SelectContent from '@/components/ui/select/SelectContent.vue'
import SelectItem from '@/components/ui/select/SelectItem.vue'
import SelectTrigger from '@/components/ui/select/SelectTrigger.vue'
import SelectValue from '@/components/ui/select/SelectValue.vue'
import { useToast } from '@/components/ui/toast/use-toast'
import Input from '@/components/ui/input/Input.vue'
import { ListIcon, GridIcon, ChevronLeftIcon, ChevronRightIcon } from 'lucide-vue-next'
import { useRouter } from 'vue-router'

const { toast } = useToast()
const router = useRouter()

// 視圖模式
const viewMode = ref('table') // 'table' 或 'grid'

// 分頁設定
const pageSizes = [10, 20, 30, 50, 100]
const itemsPerPage = ref(10)
const currentPage = ref(1)

// 排序設定
const sortConfig = ref({
  field: 'title',
  ascending: true
})

// 生成大量測試資料
const books = [
  '哈利波特：神秘的魔法石', '哈利波特：消失的密室', '哈利波特：阿茲卡班的逃犯',
  '魔戒首部曲：魔戒現身', '魔戒二部曲：雙城奇謀', '魔戒三部曲：王者再臨',
  '三體', '三體Ⅱ黑暗森林', '三體Ⅲ死神永生', '挪威的森林',
  '1984', '美麗新世界', '動物農莊', '華氏451度',
  '追風箏的孩子', '群山回唱', '燦爛千陽', '小王子',
  '百年孤寂', '霍爾的移動城堡', '神隱少女', '天空之城'
]

const authors = [
  'J.K. 羅琳', '托爾金', '劉慈欣', '村上春樹',
  '喬治·歐威爾', '赫胥黎', '卡勒德·胡賽尼', '安東尼·聖修伯里',
  '加西亞·馬奎斯', '宮崎駿'
]

// 預設封面圖片（這裡使用隨機顏色作為示例）
function getDefaultCoverUrl(index) {
  const colors = ['FF6B6B', 'FF8787', '4ECDC4', '45B7D1', '96CEB4', 'FFEEAD', 'D4A5A5', 'FFE3E3']
  const color = colors[index % colors.length]
  return `https://via.placeholder.com/300x400/${color}/FFFFFF?text=${encodeURIComponent('書籍封面')}`
}

const borrowedBooks = ref(Array.from({ length: 50 }, (_, i) => ({
  id: String(i + 1),  // 添加 id 字段
  title: books[i % books.length],
  author: authors[i % authors.length],
  borrowDate: new Date(2025, 4, i % 28 + 1).toISOString().split('T')[0],
  dueDate: new Date(2025, 5, i % 28 + 1).toISOString().split('T')[0],
  coverUrl: null // 這裡之後可以放資料庫的圖片URL
})))

// 排序功能
function toggleSortOrder() {
  sortConfig.value.ascending = !sortConfig.value.ascending
}

function updateSort(field) {
  if (sortConfig.value.field === field) {
    sortConfig.value.ascending = !sortConfig.value.ascending
  } else {
    sortConfig.value.field = field
    sortConfig.value.ascending = true
  }
}

function getSortIcon(field) {
  if (sortConfig.value.field !== field) return ''
  return sortConfig.value.ascending ? '↑' : '↓'
}

// 排序後的資料
const sortedBooks = computed(() => {
  return [...borrowedBooks.value].sort((a, b) => {
    const field = sortConfig.value.field
    const modifier = sortConfig.value.ascending ? 1 : -1
    
    if (a[field] < b[field]) return -1 * modifier
    if (a[field] > b[field]) return 1 * modifier
    return 0
  })
})

// 計算總頁數
const totalPages = computed(() => Math.ceil(sortedBooks.value.length / itemsPerPage.value))

// 監聽每頁顯示數量變更
watch(itemsPerPage, () => {
  currentPage.value = 1
})

// 計算當前頁面應該顯示的資料
const paginatedBooks = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value
  const end = start + itemsPerPage.value
  return sortedBooks.value.slice(start, end)
})

function viewBookDetail(book) {
  // 跳轉到書本詳情頁面
  router.push({
    path: `/book/${book.id}`,
    query: {
      title: book.title,
      author: book.author,
      coverUrl: book.coverUrl || getDefaultCoverUrl(book.id)
    }
  })
}

// 添加 goToPage 函數
function goToPage(page) {
  const pageNum = parseInt(page)
  if (pageNum && !isNaN(pageNum) && pageNum >= 1 && pageNum <= totalPages.value) {
    currentPage.value = pageNum
  } else {
    // 如果輸入無效，重置為當前頁碼
    e.target.value = currentPage.value
  }
}
</script>

<style scoped>
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
}

@media (min-width: 640px) {
  .grid {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  }
}

@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  }
}

@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  }
}

/* 隱藏數字輸入框的上下箭頭 */
input[type="number"]::-webkit-inner-spin-button,
input[type="number"]::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input[type="number"] {
  -moz-appearance: textfield;
}
</style>