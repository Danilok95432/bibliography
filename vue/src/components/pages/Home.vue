<template>
  <PageLayout>
    <section>
      <div class="navigation">
        <h2 class="flex-1">
          Список источников
        </h2>
        <ElSelect
          v-model="typeOfList"
          size="small"
          placeholder="Выберите тип списка"
        >
          <ElOption
            v-for="item in options"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </ElSelect>
        <ElSwitch
          v-model="editMode"
          active-text="Режим редактирования"
        />
        <ElButton
          type="primary"
          size="small"
          @click="() => openNewBiblioModal()"
        >Добавить источник</ElButton>
      </div>
      <div class="order">
        <div class="order--sort">
          <ElSelect
            v-model="sortBy"
            clearable
            size="small"
            placeholder="Сортировать по"
            @change="() => getSortedBooks(sortBy)"
          >
            <ElOption label="По типу" value="type" />
            <ElOption label="По автору" value="author" />
            <ElOption label="По названию" value="title" />
            <ElOption label="По году" value="year" />
          </ElSelect>
        </div>
        <div class="order--filter">
          <ElSelect
            v-model="selectedTypes"
            multiple
            clearable
            size="small"
            placeholder="Выберите типы источников"
            @change="() => filteredAndSortedBooks(selectedTypes)"
            collapse-tags
          >
            <ElOption
              v-for="(value, key) in LABEL_SOURCE_TYPE"
              :key="key"
              :label="`${value}`"
              :value="key"
            />
          </ElSelect>
        </div>
        <div class="order--group">
          <ElSelect
            v-model="groupField"
            size="small"
            clearable
            placeholder="Группировать по"
            @change="() => groupBooks()"
          >
            <ElOption label="Автор" value="author" />
            <ElOption label="Год" value="year" />
            <ElOption label="Отметки" value="tags" />
            <ElOption label="Город" value="city" />
          </ElSelect>
        </div>
      </div>
      <div v-if="groupedBooks.length">
        <div v-for="(group, index) in groupedBooks" :key="index">
          <h3 class="text-center">{{ group.label }}</h3>
          <ListContainer :books="group.books" :type-list="editMode ? 'div' : typeOfList" :is-edit="editMode" />
        </div>
      </div>
    </section>
  </PageLayout>
</template>

<script>
import { mapGetters } from "vuex";
import { biblioModal } from "@/mixins/modals";
import ListContainer from "@/components/parts/biblio/List.vue";
import { RouteNames } from "@/router/routes";
import PageLayout from "@/components/parts/PageLayout.vue";
import { LABEL_SOURCE_TYPE } from "@/components/forms/BookForm.vue";

export default {
  name: 'HomePage',
  mixins: [
    biblioModal
  ],
  components: {
    PageLayout,
    ListContainer
  },
  data () {
    return {
      editMode: false,
      typeOfList: 'ol',
      selectedTypes: [],
      sortBy: null,
      sortFunc: null,
      groupField: null
    }
  },
  created() {
    this.checkUrlFilters()
  },
  watch: {
    selectedTypes: {
      handler () {
        localStorage.setItem('filters', JSON.stringify(this.selectedTypes))
        this.pushQueryParams()
      },
      deep: true
    },
    sortBy () {
      localStorage.setItem('sorting', this.sortBy)
      this.pushQueryParams()
    }
  },
  computed: {
    ...mapGetters('books', [
      'filteredAndSortedBooks'
    ]),
    RouteNames () {
      return RouteNames
    },
    books () {
      return this.filteredAndSortedBooks(this.selectedTypes, this.sortFunc)
    },
    options () {
      return [{
        value: "ol",
        label: "Нумерованный список"
      }, {
        value: "ul",
        label: "Простой список"
      }, {
        value: "div",
        label: "Блочный список"
      }]
    },
    LABEL_SOURCE_TYPE () {
      return LABEL_SOURCE_TYPE
    },
    sortFunctions () {
      return {
        type: (a, b) => a.type.localeCompare(b.type),
        author: (a, b) => {
          const authorA = (a.authors && a.authors.length > 0) ? a.authors[0].surname : ''
          const authorB = (b.authors && b.authors.length > 0) ? b.authors[0].surname : ''
          return authorA.localeCompare(authorB)
        },
        title: (a, b) => a.title.localeCompare(b.title),
        year: (a, b) => a.year - b.year
      }
    },
    groupedBooks () {
      if (!this.groupField) {
        return [{ label: "Все источники", books: this.books }]
      }
      const groups = {}
      this.books.forEach(book => {
        const groupValue = this.getGroupValue(book, this.groupField)
        if (!groups[groupValue]) {
          groups[groupValue] = []
        }
        groups[groupValue].push(book)
      })
      for (const group in groups) {
        if (this.groupField === "year") {
          groups[group] = groups[group].slice().sort((a, b) => b.year - a.year)
        } else {
          groups[group] = groups[group].slice().sort(this.sortFunctions[this.sortBy])
        }
      }
      const sortedGroups = Object.keys(groups).sort((a, b) => {
        if (this.groupField === "year") {
          return parseInt(b) - parseInt(a)
        } else {
          return a.localeCompare(b)
        }
      })
      return sortedGroups.map(key => ({
        label: key,
        books: groups[key]
      }))

    }
  },
  methods: {
    getSortedBooks (type) {
      if (this.sortFunctions[type]) {
        this.sortFunc = this.sortFunctions[type]
      }
      else {
        this.sortFunc = null
      }
    },
    groupBooks() {
      this.sortBy = null
    },
    getGroupValue(book, field) {
      switch (field) {
        case "author":
          return book.authors[0].surname
        case "year":
          return book.year
        case "tags":
          return book.tags
        case "city":
          return book.city
        default:
          return "Unknown"
      }
    },
    pushQueryParams () {
      this.$router.push({ 
        query: { 
          sorting: this.sortBy,
          filters: JSON.stringify(this.selectedTypes)
        } 
      })
    },
    checkUrlFilters () {
      const urlParams = new URLSearchParams(window.location.search)
      if (urlParams.has('filters') || urlParams.has('sorting')) {
        if (urlParams.has('filters')) {
          this.selectedTypes = JSON.parse(urlParams.get('filters'))
        }
        if (urlParams.has('sorting')) {
          this.sortBy = urlParams.get('sorting')
        }
      }
      else {
        const localFilters = localStorage.getItem('filters')
        const localSorting = localStorage.getItem('sorting')
        if (localFilters) {
          this.selectedTypes = JSON.parse(localFilters)
        }
        if (localSorting) {
          this.sortBy = localSorting
        }
      }
    }
  }
}
</script>

<style lang="less" scoped>
.order{
  display: flex;
  &--sort{
    padding: 10px 10px 10px 20px;
  }
  &--filter{
    padding: 10px;
  }
  &--group{
    padding: 10px;
  }
}
.caption{
  text-align: center;
}
</style>
