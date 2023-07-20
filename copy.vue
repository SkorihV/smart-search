<template>
  <div id="search-app" class="search" :class="renderMode">

    <form
        class="search__form"
        @submit.prevent
        action="{/literal}{get_seo_url uri_prefix=$shop2.uri mode='search'}{literal}"
        enctype="multipart/form-data"
    >
      <input
          class="search__input-text"
          @focus="canBeShownResultBlock = true"
          placeholder="Введите поисковый запрос..."
          v-model.trim="search_query"
          type="text"
          name="search_text"
      />
      <button class="search__submit-btn" @click="startSearching">Найти</button>
    </form>

    <div id="search__result-wrapper" v-if="isShowingResultBlock" v-cloak>
      <div class="search__product-wrapper" v-show="amountFoundsProducts">
        <h2 class="search__product-title">
          Товаров найдено:
          <strong class="search__product-amount">{{ amountFoundsProducts }}</strong>
        </h2>
        <div
            class="search__product-item"
            v-for="product in foundProductsAfterApplyingLimits"
        >
          <div class="search__product-image-wrapper" v-if="showImg">
            <img
                class="search__product-image"
                v-if="isSuggestMode"
                :src="'{/literal}{$IMAGES_DIR}{literal}' + product.image_filename"
                :alt="product.name"
                :width="imageWidth"
                :height="imageHeight"
            />
            <img
                class="search__product-image"
                v-else
                :src="product.image_url"
                :alt="product.name"
                :width="imageWidth"
                :height="imageHeight"
            />
          </div>
          <a class="search__product-name" :href="'/' + product.alias">{{ product.name }}</a>
          <div
              class="search__product-price-wrapper"
              v-if="showPrice && !isSuggestMode"
          >
            <span class="search__product-price">{{ product.price }}</span>
            <span class="search__product-currency">{{ currency }}</span>
          </div>
        </div>
        <hr />
      </div>

      <div class="search__result-folders-wrapper" v-show="amountFoundsFolders">
        <h2 class="search__folder-title">
          Категории найдено:
          <span class="search__folder-amount">{{ amountFoundsFolders }}</span>
        </h2>
        <div class="search__folder-item" v-for="folder in foundFoldersResult">
          <span
              class="search__folder-parent"
              v-for="folder_parent in folder.parents"
          >
            <a
                class="search__folder-parent-name"
                :href="'/' + folder_parent.alias"
            >{{ folder_parent.folder_name }}</a>
          </span>
          <a class="search__folder-name" :href="'/' + folder.alias">{{ folder.folder_name }}</a>
        </div>
      </div>

      <div class="search__result-vendors-wrapper" v-show="amountFoundsVendors">
        <h2 class="search__vendor-title">
          Производители <span>найдено: {{ amountFoundsVendors }}</span>
        </h2>
        <div class="search__vendors-item" v-for="vendor in foundVendorsResult">
          <a class="search__vendor-name" :href="vendor.alias">{{ vendor.name }}</a>
        </div>
      </div>

      <a
          class="search__more-results-btn"
          v-if="isButtonShowMoreResult || isSuggestMode"
          :href="'{/literal}{get_seo_url uri_prefix=$shop2.uri mode=search}{literal}?search_text=' + search_query"
      >Показать весь результаты</a>
    </div>
  </div>
</template>
<script>
export default {
  el: "#search-app",
  data() {
    return {
      search_query:                    "", // Строка поиска
      foundProductsResult:             [], // Найденные продукты

      foldersList:                     outerSettingsSmartSearch?.folders || [], // Список категорий
      vendorsList:                     outerSettingsSmartSearch?.vendors || [], // Список производителей

      imageWidth:                      outerSettingsSmartSearch?.img_width || 50, // Ширина картинок
      imageHeight:                     outerSettingsSmartSearch?.img_height || 50, // Высота картинок

      limitFolders:                    outerSettingsSmartSearch?.limit_folders || 30, // Лимит на вывод категорий
      limitVendors:                    outerSettingsSmartSearch?.limit_vendors || 30, // Лимит на вывод категорий
      limitProducts:                   outerSettingsSmartSearch?.limit_products || 30, // Лимит на вывод товаров

      showImg:                         outerSettingsSmartSearch?.show_img, //отображать ли картинку
      showPrice:                       outerSettingsSmartSearch?.show_price, // отображать ли цену

      typeRequest:                     outerSettingsSmartSearch?.type_request || "getSuggest", // может быть getSuggest или getProductsBySearchMatches
      extraParams:                     outerSettingsSmartSearch?.extra_params || "",
      currency:                        outerSettingsSmartSearch?.currency || " руб.",
      minimalAmountCharactersInSearch: outerSettingsSmartSearch?.minimal_amount_characters_in_search || 3,
      sortPrice:                       outerSettingsSmartSearch?.sort_price || "desc",

      windowWidth: window.innerWidth,
      search_timeout: null,
      canBeShownResultBlock: false,
      resultSearchingProductsOnQuery: "",
    };
  },

  mounted() {
    window.addEventListener(
        "resize",
        (e) => (this.windowWidth = e.currentTarget.innerWidth)
    );
    window.addEventListener("click", (e) =>
        !this.$el.contains(e.target)
            ? this.showResultBlockAfterRequest(false)
            : this.showResultBlockAfterRequest(true)
    );
    if (this.minimalAmountCharactersInSearch < 3) {
      this.minimalAmountCharactersInSearch = 3;
    }
  },

  created() {
    this.foldersList.shift();
  },

  methods: {
    startSearching() {
      if (this.isSearching) {
        this.typeRequest === "getSuggest" ?  this.searchingProductsOnQuerySuggest()
            : this.typeRequest === "getProductsBySearchMatches" ? this.searchingProductsOnQueryMatches()
                : false;
      } else {
        this.resetData();
      }
    },

    searchingProductsOnQuerySuggest() {
      const urlRequest = "/my/s3/xapi/public/?method=shop2/getSuggest";
      const bodyData = {
        text: this.search_query,
        limit: this.limitProducts,
        params: this.suggestXapiParams,
      };

      fetch(urlRequest, {
        method: "POST",
        headers: {
          "Accept": "application/json",
          "Content-Type": "application/json",
        },
        body: JSON.stringify(bodyData),
      })
          .then((response) => {
            if (response.ok) {
              this.showResultBlockAfterRequest(true);
              return response.json();
            }
            this.foundProductsResult = [];
            throw new Error("Network response was not ok");
          })
          .then((json) => {
            this.foundProductsResult = json?.result?.data || [];
          })
          .catch((error) => {
            this.foundProductsResult = [];
            console.log("Ошибка при запросе -", error);
          });
    },

    searchingProductsOnQueryMatches() {

      const urlRequest =
          "/my/s3/xapi/public/?method=shop2/getProductsBySearchMatches&param[s][search_text]=" +
          this.search_query +
          "&param[template]=db:shop2.search-smart-list-product.tpl" +
          this.imgParamsForQueryOnShop +
          "&param[limit]=" +
          this.limitForRequest;


      fetch(urlRequest)
          .then((response) => {
            if (response.ok) {
              return response.json();
            }
            this.foundProductsResult = [];
            throw new Error("Network response was not ok");
          })
          .then((json) => {
            this.foundProductsResult = json.result
                ? this.convertHTMLDataFromProductList(json.result)
                : [];
            this.resultSearchingProductsOnQuery = json.result;
            this.showResultBlockAfterRequest(true);
          })
          .catch((error) => {
            this.foundProductsResult = [];
            console.log("Ошибка при запросе -", error);
          });
    },

    resetData() {
      this.showResultBlockAfterRequest(false);
      this.foundProductsResult = [];
    },

    showResultBlockAfterRequest(flag) {
      this.$nextTick(() => {
        this.canBeShownResultBlock = flag;
      });
    },

    convertHTMLDataFromProductList(dataHtml) {
      let products = [];
      let data = {};
      if (dataHtml?.html) {
        data = JSON.parse(dataHtml.html);
        for (let item in data) {
          products.push(data[item]);
        }
      }
      return products;
    },
  },

  watch: {

    search_query() {
      if (this.search_timeout) clearTimeout(this.search_timeout);
      this.search_timeout = setTimeout(() => {
        this.startSearching();
      }, 300);
    },

    foundProductsResult() {
      if (this.foundProductsResult.length) {
        this.showResultBlockAfterRequest(true);
      }
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

    isAsc() {
      return this.sortPrice === 'asc';
    },

    limitForRequest() {
      return this.isAsc ? '100' : this.limitProducts;
    },

    isSuggestMode() {
      return this.typeRequest === "getSuggest";
    },

    suggestXapiParams() {
      const params = this.extraParams.split(",").map((param) => param.trim());
      return this.extraParams.length ? params : [];
    },

    isShowingResultBlock() {
      return Boolean(
          this.canBeShownResultBlock &&
          (this.amountFoundsProducts ||
              this.amountFoundsFolders ||
              this.amountFoundsVendors)
      );
    },

    isSearching() {
      return this.search_query.length >= this.minimalAmountCharactersInSearch;
    },

    imgParamsForQueryOnShop() {
      return this.imageWidth && this.imageHeight
          ? `&img_width=${this.imageWidth}&img_height=${this.imageHeight}`
          : "";
    },

    foundProductsAfterSortPrice() {
      return this.isAsc
          ? this.foundProductsResult.sort((productFirst, productSecond) => Number(productFirst.price) - Number(productSecond.price))
          : this.foundProductsResult;
    },

    foundProductsAfterApplyingLimits() {
      return this.foundProductsAfterSortPrice.slice(0, this.limitProducts);
    },

    amountFoundsProducts() {
      return this.resultSearchingProductsOnQuery?.found
          ? this.resultSearchingProductsOnQuery.found
          : this.foundProductsResult.length;
    },

    isButtonShowMoreResult() {
      return this.amountFoundsProducts > this.foundProductsAfterApplyingLimits.length;
    },

    foundFolderByKeyword() {
      let foundFolders = [];
      if (!this.isSearching) {
        return [];
      }
      let search_query = new RegExp(this.search_query, "i");

      for (let item in this.foldersList) {
        const is_match_name =
            this.foldersList[item].folder_name.match(search_query);
        const is_match_alias = this.foldersList[item].alias.match(search_query);

        if (is_match_name || is_match_alias) {
          foundFolders.push(this.foldersList[item]);
        }
      }
      return foundFolders;
    },

    foundParentsToFolders() {
      let parentsFolder = [];

      for (let item in this.foundFolderByKeyword) {
        if (this.foundFolderByKeyword[item]._level < 2) {
          return [];
        }
        this.foundFolderByKeyword[item].parents = [];

        for (let item_a in this.foldersList) {
          this.foldersList[item_a]._left = parseInt(
              this.foldersList[item_a]._left
          );
          this.foldersList[item_a]._right = parseInt(
              this.foldersList[item_a]._right
          );

          if (
              this.foldersList[item_a]._left <
              this.foundFolderByKeyword[item]._left &&
              this.foldersList[item_a]._right > this.foldersList[item]._right
          ) {
            this.foundFolderByKeyword[item].parents.push(
                this.foldersList[item_a]
            );
          }
        }
        parentsFolder.push(this.foundFolderByKeyword[item]);
      }
      return parentsFolder;
    },

    foundFolders() {
      return this.foundParentsToFolders.length
          ? this.foundParentsToFolders
          : this.foundFolderByKeyword;
    },

    foundFoldersResult() {
      return this.foundFolders.slice(0, this.limitFolders);
    },

    amountFoundsFolders() {
      return this.foundFolders.length;
    },

    foundVendors() {
      let foundVendors = [];
      if (!this.isSearching) {
        return [];
      }
      let search_query = new RegExp(this.search_query, "i");

      for (let item in this.vendorsList) {
        const is_match_name = this.vendorsList[item].name.match(search_query);
        const is_match_alias = this.vendorsList[item].alias.match(search_query);

        if (is_match_name || is_match_alias) {
          foundVendors.push(this.vendorsList[item]);
        }
      }
      return foundVendors;
    },

    foundVendorsResult() {
      return this.foundVendors.slice(0, this.limitVendors);
    },

    amountFoundsVendors() {
      return this.foundVendors.length;
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
