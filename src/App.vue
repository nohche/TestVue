<template>
  <div class="autocomplete" ref="autocomplete">
    <div class="input-wrapper">
      <input v-model="query" @input="onInput" @keydown.down.prevent="onArrowDown" @keydown.up.prevent="onArrowUp"
        @keydown.enter.prevent="selectItem(activeItemIndex)" type="text" placeholder="Введите текст..."
        class="autocomplete-input" />
      <button v-if="query" class="clear-btn" @click="clearInput">✕</button>
    </div>

    <ul v-if="filteredSuggestions.length" class="suggestions" ref="suggestionsList" @scroll="onScroll">
      <li v-for="(item, index) in filteredSuggestions" :key="item.id" :class="{ active: index === activeItemIndex }"
        @click="selectItem(index)">
        {{ item.value }}
      </li>
      <li v-if="loading" class="loading-item">Загрузка...</li>
    </ul>
  </div>
</template>

<script>
import axios from 'axios';
import debounce from 'lodash.debounce';

export default {
  name: 'TestComplete',
  data() {
    return {
      query: '',
      allSuggestions: [],
      filteredSuggestions: [],
      activeItemIndex: -1,
      minQueryLength: 2,
      loading: false,
      dataLimit: 100, // Лимит записей на запрос
      allDataLoaded: false, // Флаг завершения загрузки всех данных
    };
  },
  methods: {
    onInput: debounce(function () {
      if (this.query.length >= this.minQueryLength) {
        this.filteredSuggestions = this.allSuggestions.filter(item =>
          item.value.toLowerCase().includes(this.query.toLowerCase())
        );
        if (this.filteredSuggestions.length === 0 && !this.allDataLoaded) {
          this.fetchSuggestions(); // Запрос данных, если нет совпадений и данные еще не все загружены
        }
      } else {
        this.filteredSuggestions = [];
      }
    }, 300),

    async fetchSuggestions() {
      if (this.loading || this.allDataLoaded) return; // Предотвращение лишних запросов

      this.loading = true;
      try {
        const currentPage = Math.ceil(this.allSuggestions.length / this.dataLimit) + 1;
        const response = await axios.get(
          `https://jsonplaceholder.typicode.com/todos?_limit=${this.dataLimit}&_page=${currentPage}`
        );

        const newSuggestions = response.data.map(todo => ({
          id: todo.id,
          value: todo.title,
        }));

        if (newSuggestions.length < this.dataLimit) {
          this.allDataLoaded = true; // Все данные загружены
        }

        this.allSuggestions.push(...newSuggestions);
        this.filteredSuggestions = this.allSuggestions.filter(item =>
          item.value.toLowerCase().includes(this.query.toLowerCase())
        );
      } catch (error) {
        console.error('Ошибка с загрузкой:', error);
      } finally {
        this.loading = false;
      }
    },

    selectItem(index) {
      this.query = this.filteredSuggestions[index].value;
      this.filteredSuggestions = [];
    },

    clearInput() {
      this.query = '';
      this.filteredSuggestions = [];
    },

    onArrowDown() {
      if (this.activeItemIndex < this.filteredSuggestions.length - 1) {
        this.activeItemIndex++;
        this.scrollToActive();
      }
    },

    onArrowUp() {
      if (this.activeItemIndex > 0) {
        this.activeItemIndex--;
        this.scrollToActive();
      }
    },

    scrollToActive() {
      const suggestionsList = this.$refs.suggestionsList;
      const activeItem = suggestionsList.children[this.activeItemIndex];

      if (activeItem) {
        const itemOffsetTop = activeItem.offsetTop;
        const listOffsetTop = suggestionsList.scrollTop;
        const listHeight = suggestionsList.clientHeight;

        if (itemOffsetTop + activeItem.clientHeight > listOffsetTop + listHeight) {
          suggestionsList.scrollTop = itemOffsetTop + activeItem.clientHeight - listHeight;
        }

        if (itemOffsetTop < listOffsetTop) {
          suggestionsList.scrollTop = itemOffsetTop;
        }
      }
    },

    onScroll() {
      const suggestionsList = this.$refs.suggestionsList;

      if (
        suggestionsList.scrollTop + suggestionsList.clientHeight >=
        suggestionsList.scrollHeight - 10
      ) {
        this.fetchSuggestions(); // Загрузка данных при скролле
      }
    },

    handleClickOutside(event) {
      const autocomplete = this.$refs.autocomplete;
      if (autocomplete && !autocomplete.contains(event.target)) {
        this.filteredSuggestions = [];
      }
    },
  },

  mounted() {
    document.addEventListener('click', this.handleClickOutside);
  },

  beforeUnmount() {
    document.removeEventListener('click', this.handleClickOutside);
  },
};
</script>

<style scoped>
.autocomplete {
  position: relative;
  width: 320px;
  font-family: Arial, sans-serif;
}

.input-wrapper {
  position: relative;
  width: 100%;
}

.autocomplete-input {
  width: 100%;
  padding: 12px 40px 12px 12px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.1);
  transition: border-color 0.2s ease-in-out;
}

.autocomplete-input:focus {
  border-color: #007bff;
  outline: none;
  box-shadow: 0px 4px 6px rgba(0, 123, 255, 0.3);
}

.clear-btn {
  position: absolute;
  right: -40px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  font-size: 20px;
  color: #888;
  cursor: pointer;
}

.clear-btn:hover {
  color: #000;
}

.suggestions {
  position: absolute;
  top: 100%;
  width: 100%;
  background-color: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0px 2px 6px rgba(0, 0, 0, 0.15);
  margin-top: 5px;
  z-index: 1000;
  max-height: 200px;
  overflow-y: auto;
  padding: 0;
  padding-right: 50px;
  text-align: start;
  list-style-type: none;
}

.suggestions li {
  padding: 12px 40px 12px 12px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.1s ease-in-out;
}

.suggestions li:hover {
  background-color: #f7f7f7;
}

.suggestions li.active {
  background-color: #007bff;
  color: white;
  width: calc(100% + 48px);
  margin-left: -12px;
  margin-right: -40px;
}
</style>
