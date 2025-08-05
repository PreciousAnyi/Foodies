<script setup lang="ts">
import { type RecipeResponse, type Recipe } from "../../types/types";

// SEO Meta
useSeoMeta({
  title: "Recipes - Discover Amazing Dishes",
  description:
    "Search and explore thousands of delicious recipes from around the world with our easy-to-use recipe finder.",
  ogTitle: "Recipes - Discover Amazing Dishes",
  ogDescription:
    "Search and explore thousands of delicious recipes from around the world.",
  ogImage: "/hero.png",
  ogUrl: "http://localhost:3000/recipes",
  twitterTitle: "Recipes - Discover Amazing Dishes",
  twitterDescription:
    "Search and explore thousands of delicious recipes from around the world.",
  twitterImage: "/hero.png",
  twitterCard: "summary_large_image",
});

// Reactive state
const recipes = ref<Recipe[]>([]);
const loading = ref(false);
const initialLoading = ref(true);
const hasMore = ref(true);
const skip = ref(0);
const searchTerm = ref("");
const activeSearchTerm = ref("");
const selectedCuisine = ref("All");
const selectedDifficulty = ref("All");
const totalRecipes = ref(0);
const error = ref<string | null>(null);

// Constants
const limit = 12;
const cuisines = [
  "All",
  "Italian",
  "Mexican",
  "Asian",
  "American",
  "Mediterranean",
  "Indian",
  "French",
];
const difficulties = ["All", "Easy", "Medium", "Hard"];

// Build API URL with filters
const buildApiUrl = (
  skipCount: number = 0,
  isSearch: boolean = false
): string => {
  let url = "https://dummyjson.com/recipes";

  if (isSearch && activeSearchTerm.value) {
    url += `/search?q=${encodeURIComponent(activeSearchTerm.value)}`;
  }

  url += `${
    isSearch && activeSearchTerm.value ? "&" : "?"
  }limit=${limit}&skip=${skipCount}`;

  return url;
};

// Filter recipes based on selected filters
const filterRecipes = (recipesToFilter: Recipe[]): Recipe[] => {
  return recipesToFilter.filter((recipe) => {
    const matchesCuisine =
      selectedCuisine.value === "All" ||
      recipe.cuisine === selectedCuisine.value;
    const matchesDifficulty =
      selectedDifficulty.value === "All" ||
      recipe.difficulty === selectedDifficulty.value;
    return matchesCuisine && matchesDifficulty;
  });
};

// Load initial recipes
const loadInitialRecipes = async () => {
  initialLoading.value = true;
  loading.value = true;
  error.value = null;

  try {
    const url = buildApiUrl(0, activeSearchTerm.value !== "");
    const response = await $fetch<RecipeResponse>(url);

    if (response) {
      const filteredRecipes = filterRecipes(response.recipes);
      recipes.value = filteredRecipes;
      totalRecipes.value = response.total;
      hasMore.value = response.recipes.length === limit;
      skip.value = limit;
    }
  } catch (err) {
    console.error("Error loading recipes:", err);
    error.value = "Failed to load recipes. Please try again.";
  } finally {
    loading.value = false;
    initialLoading.value = false;
  }
};

// Load more recipes for pagination
const loadMoreRecipes = async () => {
  if (loading.value) return;

  loading.value = true;

  try {
    const url = buildApiUrl(skip.value, activeSearchTerm.value !== "");
    const response = await $fetch<RecipeResponse>(url);

    if (response) {
      const filteredNewRecipes = filterRecipes(response.recipes);
      recipes.value = [...recipes.value, ...filteredNewRecipes];
      hasMore.value = response.recipes.length === limit;
      skip.value += limit;
    }
  } catch (err) {
    console.error("Error loading more recipes:", err);
    error.value = "Failed to load more recipes.";
  } finally {
    loading.value = false;
  }
};

// Handle search
const handleSearch = () => {
  activeSearchTerm.value = searchTerm.value;
  skip.value = 0;
  recipes.value = [];
  hasMore.value = true;
};

// Clear all filters
const clearFilters = () => {
  searchTerm.value = "";
  activeSearchTerm.value = "";
  selectedCuisine.value = "All";
  selectedDifficulty.value = "All";
  skip.value = 0;
  recipes.value = [];
  hasMore.value = true;
};

// Handle filter changes
const handleFilterChange = () => {
  skip.value = 0;
  recipes.value = [];
  hasMore.value = true;
};

// Watch for filter changes
watch([activeSearchTerm, selectedCuisine, selectedDifficulty], () => {
  loadInitialRecipes();
});

// Watch for cuisine and difficulty filter changes
watch([selectedCuisine, selectedDifficulty], () => {
  handleFilterChange();
});

// Intersection Observer for infinite scroll
const lastRecipeElement = ref<HTMLElement>();

onMounted(() => {
  loadInitialRecipes();

  // Set up intersection observer for infinite scroll
  if (process.client) {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0]?.isIntersecting && hasMore.value && !loading.value) {
          loadMoreRecipes();
        }
      },
      { threshold: 0.1 }
    );

    watch(lastRecipeElement, (newElement, oldElement) => {
      if (oldElement) observer.unobserve(oldElement);
      if (newElement) observer.observe(newElement);
    });

    onUnmounted(() => {
      observer.disconnect();
    });
  }
});

// Handle Enter key for search
const handleSearchKeyPress = (event: KeyboardEvent) => {
  if (event.key === "Enter") {
    handleSearch();
  }
};
</script>

<template>
  <main class="min-h-screen">
    <!-- Search and Filter Section -->
    <section class="container mx-auto px-4 py-8">
      <!-- Search Bar -->
      <div class="relative max-w-2xl mx-auto mb-8">
        <div class="flex gap-2">
          <div class="relative flex-1">
            <input
              v-model="searchTerm"
              type="text"
              placeholder="Search recipes by name, cuisine, or ingredients..."
              class="w-full px-4 py-3 pr-12 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
              @keypress="handleSearchKeyPress"
            />
            <div class="absolute inset-y-0 right-0 flex items-center pr-3">
              <svg
                class="w-5 h-5 text-gray-400"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
                />
              </svg>
            </div>
          </div>
          <BaseBtn label="Search" @click="handleSearch" class="self-center" />
        </div>
      </div>

      <!-- Filter Bar -->
      <div class="flex flex-wrap gap-4 mb-8 p-4 rounded-lg">
        <div class="flex flex-col gap-2">
          <label class="text-sm font-medium text-gray-700">Cuisine:</label>
          <select
            v-model="selectedCuisine"
            class="px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            <option v-for="cuisine in cuisines" :key="cuisine" :value="cuisine">
              {{ cuisine }}
            </option>
          </select>
        </div>

        <div class="flex flex-col gap-2">
          <label class="text-sm font-medium text-gray-700">Difficulty:</label>
          <select
            v-model="selectedDifficulty"
            class="px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            <option
              v-for="difficulty in difficulties"
              :key="difficulty"
              :value="difficulty"
            >
              {{ difficulty }}
            </option>
          </select>
        </div>

        <div class="flex items-end">
          <BaseBtn
            label="Clear Filters"
            @click="clearFilters"
            class="self-end"
          />
        </div>
      </div>

      <!-- Results Count -->
      <div class="mb-6">
        <p class="text-gray-600">
          <span v-if="initialLoading">Loading...</span>
          <span v-else>Showing {{ recipes.length }} recipes</span>
          <span v-if="activeSearchTerm"> for "{{ activeSearchTerm }}"</span>
        </p>
      </div>

      <!-- Error Message -->
      <div
        v-if="error"
        class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-6"
      >
        {{ error }}
      </div>

      <!-- Recipes Grid -->
      <div
        class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6"
      >
        <!-- Recipe Cards -->
        <div
          v-for="(recipe, index) in recipes"
          :key="`${recipe.id}-${index}`"
          :ref="index === recipes.length - 1 ? 'lastRecipeElement' : undefined"
          class="bg-white rounded-lg shadow-md overflow-hidden hover:shadow-lg transition-shadow duration-300"
        >
          <div class="relative h-48">
            <NuxtImg
              :src="recipe.image"
              :alt="recipe.name"
              class="w-full h-full object-cover"
              loading="lazy"
              format="webp"
              sizes="xs:100vw sm:300px md:250px lg:280px xl:300px"
            />
            <div
              class="absolute top-2 right-2 bg-white rounded-full px-2 py-1 text-sm font-semibold"
            >
              ⭐ {{ recipe.rating }}
            </div>
          </div>

          <div class="p-4">
            <h3 class="font-bold text-lg mb-2 line-clamp-2">
              {{ recipe.name }}
            </h3>

            <div class="flex flex-wrap gap-2 mb-3">
              <span class="bg-blue-100 text-blue-800 text-xs px-2 py-1 rounded">
                {{ recipe.cuisine }}
              </span>
              <span
                class="bg-green-100 text-green-800 text-xs px-2 py-1 rounded"
              >
                {{ recipe.difficulty }}
              </span>
            </div>

            <div class="flex justify-between text-sm text-gray-600 mb-3">
              <span
                >🕒
                {{ recipe.prepTimeMinutes + recipe.cookTimeMinutes }} min</span
              >
              <span>👥 {{ recipe.servings }} servings</span>
              <span>🔥 {{ recipe.caloriesPerServing }} cal</span>
            </div>

            <div class="flex flex-wrap gap-1 mb-3">
              <span
                v-for="(tag, tagIndex) in recipe.tags.slice(0, 3)"
                :key="tagIndex"
                class="bg-gray-100 text-gray-700 text-xs px-2 py-1 rounded"
              >
                #{{ tag }}
              </span>
              <span v-if="recipe.tags.length > 3" class="text-xs text-gray-500">
                +{{ recipe.tags.length - 3 }} more
              </span>
            </div>

            <BaseBtn :to="`/recipe/${recipe.id}`" label="View Recipe" />
          </div>
        </div>

        <!-- Loading Skeleton Cards -->
        <div
          v-if="loading"
          v-for="n in 4"
          :key="`loading-${n}`"
          class="bg-white rounded-lg shadow-md overflow-hidden animate-pulse"
        >
          <div class="h-48 bg-gray-300"></div>
          <div class="p-4">
            <div class="h-4 bg-gray-300 rounded mb-2"></div>
            <div class="h-4 bg-gray-300 rounded w-3/4 mb-3"></div>
            <div class="flex gap-2 mb-3">
              <div class="h-6 bg-gray-300 rounded w-16"></div>
              <div class="h-6 bg-gray-300 rounded w-16"></div>
            </div>
            <div class="h-8 bg-gray-300 rounded"></div>
          </div>
        </div>
      </div>

      <!-- No Results -->
      <div
        v-if="!initialLoading && recipes.length === 0 && !error"
        class="text-center py-12"
      >
        <div class="text-6xl mb-4">🍽️</div>
        <h3 class="text-2xl font-semibold mb-2">No recipes found</h3>
        <p class="text-gray-600 mb-4">
          Try adjusting your search terms or filters
        </p>
        <BaseBtn @click="clearFilters" label="Clear All Filters" />
      </div>

      <!-- Load More Button (fallback) -->
      <div
        v-if="!loading && hasMore && recipes.length > 0"
        class="text-center mt-8"
      >
        <BaseBtn label="Load More Recipes" @click="loadMoreRecipes" />
      </div>

      <!-- End of Results -->
      <div v-if="!hasMore && recipes.length > 0" class="text-center mt-8 py-4">
        <p class="text-gray-600">🎉 You've seen all available recipes!</p>
      </div>
    </section>
  </main>
</template>

<style scoped>
.line-clamp-2 {
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>
