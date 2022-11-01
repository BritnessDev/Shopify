<template>
  <div>
    <SfHeader
      class="sf-header--has-mobile-search"
      :class="{
        'header-on-top': isSearchOpen,
      }"
    >
      <!-- TODO: add mobile view buttons after SFUI team PR -->
      <template #logo>
        <nuxt-link :to="localePath('/')" class="sf-header__logo">
          <SfImage
            :width="35"
            :height="35"
            src="/icons/logo.svg"
            alt="Vue Storefront Next"
            class="sf-header__logo-image"
          />
        </nuxt-link>
      </template>
      <template #navigation>
        <SfHeaderNavigationItem
          v-for="(category, index) in filteredTopCategories"
          :key="index"
          data-cy="app-header-top-categories"
          class="nav-item"
          :label="category.name"
          :link="localePath(category.slug)"
        />
      </template>
      <template #aside>
        <LocaleSelector class="smartphone-only" />
      </template>
      <template #header-icons>
        <div class="sf-header__icons">
          <SfButton
            class="sf-button--pure sf-header__action"
            @click="handleAccountClick"
          >
            <SfIcon :icon="accountIcon" size="1.25rem" />
          </SfButton>
          <SfButton
            class="sf-button--pure sf-header__action"
            @click="toggleWishlistSidebar"
          >
            <SfIcon
              class="sf-header__icon"
              :icon="wishlistHasItens ? 'heart_fill' : 'heart'"
              size="1.25rem"
            />
          </SfButton>
          <SfButton
            class="sf-button--pure sf-header__action"
            @click="toggleCartSidebar"
          >
            <SfIcon class="sf-header__icon" icon="empty_cart" size="1.25rem" />

            <SfBadge
              v-if="cartTotalItems"
              class="sf-badge--number cart-badge"
              >{{ cartTotalItems }}</SfBadge
            >
          </SfButton>
        </div>
      </template>
      <template #search>
        <SfSearchBar
          ref="searchBarRef"
          :placeholder="$t('Search for items')"
          aria-label="Search"
          class="sf-header__search"
          :value="term"
          @input="handleSearch"
          @keydown.enter="handleSearch($event)"
          @focus="isSearchOpen = true"
          @keydown.esc="closeSearch"
          v-click-outside="closeSearch"
        >
          <template #icon>
            <SfButton
              v-if="!!term"
              class="sf-search-bar__button sf-button--pure"
              @click="closeOrFocusSearchBar"
            >
              <span class="sf-search-bar__icon">
                <SfIcon color="var(--c-text)" size="18px" icon="cross" />
              </span>
            </SfButton>
            <SfButton
              v-else
              class="sf-search-bar__button sf-button--pure"
              @click="
                isSearchOpen ? (isSearchOpen = false) : (isSearchOpen = true)
              "
            >
              <span class="sf-search-bar__icon">
                <SfIcon color="var(--c-text)" size="20px" icon="search" />
              </span>
            </SfButton>
          </template>
        </SfSearchBar>
      </template>
    </SfHeader>
    <SearchResults
      :visible="isSearchOpen"
      :result="formatedResult"
      @close="closeSearch"
      @removeSearchResults="removeSearchResults"
    />
    <SfOverlay :visible="isSearchOpen" />
  </div>
</template>

<script>
import {
  SfImage,
  SfSearchBar,
  SfIcon,
  SfButton,
  SfOverlay,
  SfBadge,
  SfHeader
} from '@storefront-ui/vue';
import { useUiState } from '~/composables';
import {
  useCart,
  useWishlist,
  useUser,
  cartGetters,
  categoryGetters,
  useCategories,
  useFacet
} from '@vue-storefront/odoo';
import { clickOutside } from '@storefront-ui/vue/src/utilities/directives/click-outside/click-outside-directive.js';
import { computed, ref, watch } from '@nuxtjs/composition-api';
import { onSSR } from '@vue-storefront/core';
import { useUiHelpers } from '~/composables';
import LocaleSelector from './LocaleSelector';
import SearchResults from '~/components/SearchResults';

import debounce from 'lodash.debounce';
export default {
  components: {
    SfHeader,
    SfImage,
    SfIcon,
    SfButton,
    SfSearchBar,
    LocaleSelector,
    SearchResults,
    SfOverlay,
    SfBadge
  },
  directives: { clickOutside },
  setup(props, { root }) {
    const searchBarRef = ref(null);
    const term = ref(null);
    const formatedResult = ref(null);
    const isSearchOpen = ref(false);

    const { changeSearchTerm } = useUiHelpers();
    const { toggleCartSidebar, toggleWishlistSidebar, toggleLoginModal } =
      useUiState();

    const { load: loadUser, isAuthenticated } = useUser();
    const { load: loadCart, cart } = useCart();
    const { load: loadWishlist, wishlist } = useWishlist();
    const { search: searchProductApi, result } = useFacet('AppHeader:Search');
    const { categories: topCategories, search: searchTopCategoryApi } =
      useCategories('AppHeader:TopCategories');

    const cartTotalItems = computed(() => {
      const count = cartGetters.getTotalItems(cart.value);
      return count ? count.toString() : null;
    });
    const accountIcon = computed(() =>
      isAuthenticated.value ? 'profile_fill' : 'profile'
    );

    const removeSearchResults = () => {
      formatedResult.value = null;
    };

    const closeSearch = () => {
      if (!isSearchOpen.value) return;
      term.value = '';
      isSearchOpen.value = false;
    };

    const handleSearch = debounce(async (paramValue) => {
      if (!paramValue.target) {
        term.value = paramValue;
      } else {
        term.value = paramValue.target.value;
      }
      if (term.value.length < 2) return;

      await searchProductApi({
        fetchCategories: true,
        productParams: {
          search: term.value,
          pageSize: 12
        },
        categoryParams: {
          search: term.value
        }
      });

      formatedResult.value = {
        products: result?.value?.data?.products,
        categories:
          result?.value?.data?.categories
            ?.filter((category) => category.childs === null)
            ?.map((category) => categoryGetters.getTree(category)) || []
      };
    }, 100);
    const closeOrFocusSearchBar = () => {
      term.value = '';
      return searchBarRef.value.$el.children[0].focus();
    };
    // TODO: https://github.com/DivanteLtd/vue-storefront/issues/4927
    const handleAccountClick = async () => {
      if (isAuthenticated.value) {
        return root.$router.push(root.localePath('/my-account'));
      }

      toggleLoginModal();
    };

    const filteredTopCategories = computed(() => {
      let product = {
        "id": 75,
        "name": "SHOP",
        "slug": "/category/75",
          "childs": {
            "id": 13,
            "name": "WOMEN",
            "slug": "/category/13",
            "childs":[{
              "id": 15,
              "name": "Clothing",
              "slug": "/category/15",
              "childs": [{
                "id": 19,
                "name": "All",
                "slug": "/category/19",
                "__typename": "Category"
              }, {
                "id": 20,
                "name": "Jackets",
                "slug": "/category/20",
                "__typename": "Category"
              }, {
                "id": 21,
                "name": "Blazer",
                "slug": "/category/21",
                "__typename": "Category"
              }, {
                "id": 22,
                "name": "Tops",
                "slug": "/category/22",
                "__typename": "Category"
              }, {
                "id": 23,
                "name": "Shirts",
                "slug": "/category/23",
                "__typename": "Category"
              }, {
                "id": 24,
                "name": "T-shirts",
                "slug": "/category/24",
                "__typename": "Category"
              }, {
                "id": 25,
                "name": "Jeans",
                "slug": "/category/25",
                "__typename": "Category"
              }, {
                "id": 26,
                "name": "Trouser",
                "slug": "/category/26",
                "__typename": "Category"
              }, {
                "id": 27,
                "name": "Skirts",
                "slug": "/category/27",
                "__typename": "Category"
              }, {
                "id": 28,
                "name": "Dresses",
                "slug": "/category/28",
                "__typename": "Category"
              }, {
                "id": 29,
                "name": "Swimwear",
                "slug": "/category/29",
                "__typename": "Category"
              }],
              "__typename": "Category"
            }, {
              "id": 16,
              "name": "Shoes",
              "slug": "/category/16",
              "childs": [{
                "id": 30,
                "name": "All",
                "slug": "/category/30",
                "__typename": "Category"
              }, {
                "id": 31,
                "name": "Sneakers",
                "slug": "/category/31",
                "__typename": "Category"
              }, {
                "id": 32,
                "name": "Boots",
                "slug": "/category/32",
                "__typename": "Category"
              }, {
                "id": 33,
                "name": "Ankle boots",
                "slug": "/category/33",
                "__typename": "Category"
              }, {
                "id": 34,
                "name": "Pumps",
                "slug": "/category/34",
                "__typename": "Category"
              }, {
                "id": 35,
                "name": "Ballerinas",
                "slug": "/category/35",
                "__typename": "Category"
              }, {
                "id": 36,
                "name": "Lace-up shoes",
                "slug": "/category/36",
                "__typename": "Category"
              }, {
                "id": 37,
                "name": "Loafers",
                "slug": "/category/37",
                "__typename": "Category"
              }, {
                "id": 38,
                "name": "Sandals",
                "slug": "/category/38",
                "__typename": "Category"
              }, {
                "id": 39,
                "name": "Winterboots",
                "slug": "/category/39",
                "__typename": "Category"
              }],
              "__typename": "Category"
            }, {
              "id": 17,
              "name": "bags",
              "slug": "/category/17",
              "childs": [{
                "id": 40,
                "name": "All",
                "slug": "/category/40",
                "__typename": "Category"
              }, {
                "id": 41,
                "name": "Clutches",
                "slug": "/category/41",
                "__typename": "Category"
              }, {
                "id": 42,
                "name": "Shoulder Bags",
                "slug": "/category/42",
                "__typename": "Category"
              }, {
                "id": 43,
                "name": "Shopper",
                "slug": "/category/43",
                "__typename": "Category"
              }, {
                "id": 44,
                "name": "Handbag",
                "slug": "/category/44",
                "__typename": "Category"
              }, {
                "id": 45,
                "name": "Wallets",
                "slug": "/category/45",
                "__typename": "Category"
              }, {
                "id": 46,
                "name": "Bucketbag/packbag",
                "slug": "/category/46",
                "__typename": "Category"
              }],
              "__typename": "Category"
            }, {
              "id": 18,
              "name": "Looks",
              "slug": "/category/18",
              "childs": [{
                "id": 47,
                "name": "All",
                "slug": "/category/47",
                "__typename": "Category"
              }],
              "__typename": "Category"
            }]},
          "__typename": "Category"
        }
      topCategories.value.push(product);
      // console.log(JSON.stringify(topCategories.value))
      return topCategories.value?.filter(
        (cat) => cat.name === 'WOMEN' || cat.name === 'MEN' || cat.name === 'SHOP'
      )
    }
    );

    watch(
      () => term.value,
      (newVal, oldVal) => {
        const shouldSearchBeOpened =
          term.value.length > 0 &&
          ((!oldVal && newVal) ||
            (newVal.length !== oldVal.length && isSearchOpen.value === false));
        if (shouldSearchBeOpened) {
          isSearchOpen.value = true;
        }
      }
    );

    onSSR(async () => {
      await Promise.all([
        searchTopCategoryApi({
          filter: { parent: true }
        }),
        loadUser(),
        loadWishlist(),
        loadCart()
      ]);
    });

    return {
      wishlistHasItens: computed(
        () => wishlist.value?.wishlistItems.length > 0
      ),
      filteredTopCategories,
      accountIcon,
      closeOrFocusSearchBar,
      cartTotalItems,
      removeSearchResults,
      isSearchOpen,
      searchBarRef,
      handleAccountClick,
      toggleCartSidebar,
      toggleWishlistSidebar,
      changeSearchTerm,
      formatedResult,
      term,
      handleSearch,
      closeSearch
    };
  }
};
</script>

<style lang="scss" scoped>
.sf-header {
  --header-padding: var(--spacer-sm);
  @include for-desktop {
    --header-padding: 0;
  }
  &__logo-image {
    height: 100%;
  }
}

.header-on-top {
  z-index: 2;
}
.nav-item {
  --header-navigation-item-margin: 0 var(--spacer-base);
  .sf-header-navigation-item__item--mobile {
    display: none;
  }
}
.cart-badge {
  position: absolute;
  bottom: 40%;
  left: 40%;
}
</style>
