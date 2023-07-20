<template></template>

<script>
export default {
  el: "#search-app",
  data() {
    return {
      search_query: "", // Строка поиска
      foldersList: outerSettingsSmartSearch.folders, // Список категорий
      vendorsList: outerSettingsSmartSearch.vendors, // Список производителей
      img_w: outerSettingsSmartSearch.img_width, // Ширина картинок
      img_h: outerSettingsSmartSearch.img_height, // Высота картинок
      amountFoundsProducts: 0, // Количество найденных продуктов
      amountFoundsFolders: 0, // Количество найденных категорий
      amountFoundsVendors: 0, // Количество найденных производителей
      limitProducts: outerSettingsSmartSearch.limit_products, // Лимит на вывод товаров
      foundProducts: [], // Найденные продукты
      foundFolders: [], // Найденные категории
      foundVendors: [], // Найденные производители
      windowWidth: window.innerWidth,
      search_timeout: null,
    };
  },
  mounted() {
    window.addEventListener(
      "resize",
      (e) => (this.windowWidth = e.currentTarget.innerWidth)
    );
    window.addEventListener("click", (e) =>
      !this.$el.contains(e.target) ? this.resetData() : false
    );
  },
  created() {
    this.foldersList.shift();
  },
  methods: {
    resetData() {
      // Cброс данных
      this.resetDataProducts();
      this.resetDataFolders();
      this.resetDataVendors();
    },
    resetDataProducts() {
      this.foundProducts = [];
      this.amountFoundsProducts = 0;
    },
    resetDataFolders() {
      this.foundFolders = [];
      this.amountFoundsFolders = 0;
    },
    resetDataVendors() {
      this.foundVendors = [];
      this.amountFoundsVendors = 0;
    },
    searchStart() {
      // Стартовый запрос поиска
      this.searchProductsQuery();
      this.searchFoldersQuery();
      this.searchVendorsQuery();
    },
    searchProductsQuery() {
      // Запрос на поиск продуктов

      let imgQuery = "";

      if (this.img_w || this.img_h) {
        imgQuery = "&img_width=" + this.img_w + "&img_height=" + this.img_h;
      }

      fetch(
        "/my/s3/xapi/public/?method=shop2/getProductsBySearchMatches&param[s][search_text]=" +
          this.search_query +
          "&param[template]=db:shop2.search-smart-list-product.tpl" +
          imgQuery +
          "&param[limit]=" +
          this.limitProducts
      )
        .then((response) => {
          if (response.ok) {
            console.log(response);
            return response.json();
          }
          throw new Error("Network response was not ok");
        })
        .then((json) => {
          this.foundProducts = [];
          console.log(json.result);
          this.$nextTick()
          if (json.result.found == 0) {
            this.amountFoundsProducts = 0;
            return;
          }
          let data = JSON.parse(json.result.html);

          console.log(json);

          this.amountFoundsProducts = json.result.found;

          for (let item in data) {
            this.foundProducts.push(data[item]);
          }
        })
        .catch((error) => {
          console.log("Ошибка при запросе -", error);
        });
      console.log(this.foundProducts);
    },
    searchFoldersQuery() {
      // Запрос на поиск категорий
      let result = this.searchFolders(this.foldersList, this.search_query);
      if (this.addParentsToFolders(result, this.foldersList) !== undefined) {
        this.foundFolders = this.addParentsToFolders(result, this.foldersList);
      } else {
        this.foundFolders = result;
      }

      this.amountFoundsFolders = this.foundFolders.length;
    },
    searchFolders(folders, text) {
      // Поиск категорий по ключевому слову
      let matches = [];
      let search_query = new RegExp(text, "i");

      for (let item in folders) {
        const is_match_name = folders[item].folder_name.match(search_query);
        const is_match_alias = folders[item].alias.match(search_query);

        if (is_match_name || is_match_alias) {
          matches.push(folders[item]);
        }
      }
      return matches;
    },
    addParentsToFolders(folders, all_folders) {
      // Поиск родительских категорий
      let result = [];

      for (let item in folders) {
        if (folders[item]._level < 2) {
          return;
        }
        folders[item].parents = [];

        for (let item_a in all_folders) {
          all_folders[item_a]._left = parseInt(all_folders[item_a]._left);
          all_folders[item_a]._right = parseInt(all_folders[item_a]._right);
          if (
            all_folders[item_a]._left < folders[item]._left &&
            all_folders[item_a]._right > folders[item]._right
          ) {
            folders[item].parents.push(all_folders[item_a]);
          }
        }
        result.push(folders[item]);
      }
      return result;
    },
    searchVendorsQuery() {
      // Запрос на поиск производителей
      this.foundVendors = this.searchVendors(
        this.vendorsList,
        this.search_query
      );
      this.amountFoundsVendors = this.foundVendors.length;
    },
    searchVendors(vendors, text) {
      // Поиск производителей по ключевому слову
      let matches = [];
      let search_query = new RegExp(text, "i");

      for (let item in vendors) {
        const is_match_name = vendors[item].name.match(search_query);
        const is_match_alias = vendors[item].alias.match(search_query);

        if (is_match_name || is_match_alias) {
          matches.push(vendors[item]);
        }
      }
      return matches;
    },
  },
  watch: {
    search_query() {
      if (this.search_query === "" || this.search_query.length < 3) {
        this.resetData();
        return;
      }

      if (this.search_timeout) clearTimeout(this.search_timeout);

      this.search_timeout = setTimeout(() => {
        this.searchStart();
      }, 300);
    },
  },
  computed: {
    renderMode() {
      if (this.windowWidth > 960) {
        return "desktop";
      }
      if (this.windowWidth <= 960) {
        return "tab";
      }
      if (this.windowWidth <= 767) {
        return "mobile";
      }
    },
  },
};
</script>

<style scoped></style>
