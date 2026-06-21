<script setup>
import { computed, onMounted, ref, watch } from "vue";
import { useMovieStore } from "../stores/movieStore";

const store = useMovieStore();
const searchKeyword = ref("");
const sortOption = ref("default");
const currentPage = ref(1);
const itemsPerPage = 8;

const filteredMovies = computed(() => {
  const keyword = searchKeyword.value.trim().toLowerCase();

  if (!keyword) {
    return store.movies;
  }

  return store.movies.filter((movie) => {
    const title = (movie.title || "").toLowerCase();
    const overview = (movie.overview || "").toLowerCase();

    return title.includes(keyword) || overview.includes(keyword);
  });
});

const sortedMovies = computed(() => {
  const movies = [...filteredMovies.value];

  if (sortOption.value === "title") {
    return movies.sort((a, b) => {
      return (a.title || "").localeCompare(b.title || "", "ko-KR");
    });
  }

  if (sortOption.value === "releaseDate") {
    return movies.sort((a, b) => {
      return new Date(b.release_date || 0) - new Date(a.release_date || 0);
    });
  }

  if (sortOption.value === "rating") {
    return movies.sort((a, b) => {
      return (b.vote_average || 0) - (a.vote_average || 0);
    });
  }

  return movies;
});

const totalPages = computed(() => {
  return Math.ceil(sortedMovies.value.length / itemsPerPage);
});

const pageNumbers = computed(() => {
  return Array.from({ length: totalPages.value }, (_, index) => index + 1);
});

const pagedMovies = computed(() => {
  const startIndex = (currentPage.value - 1) * itemsPerPage;
  const endIndex = startIndex + itemsPerPage;

  return sortedMovies.value.slice(startIndex, endIndex);
});

watch([searchKeyword, sortOption], () => {
  currentPage.value = 1;
});

onMounted(() => {
  store.fetchMovies();
  document.title = "🍿 국내 극장 화제작 (인기순)";
});
</script>

<template>
  <main class="page">
    <div class="header-section">
      <h1>🍿 국내 극장 화제작 (인기순)</h1>
      <p class="sub-title">2025년 이후 국내 정식 개봉한 실시간 인기 상영작</p>
    </div>

    <section class="control-panel" aria-label="영화 검색 및 정렬">
      <div class="search-group">
        <label for="movie-search">검색</label>
        <input
          id="movie-search"
          v-model="searchKeyword"
          type="search"
          placeholder="제목 또는 줄거리로 검색"
          class="search-input"
        />
      </div>

      <div class="sort-group" aria-label="정렬 옵션">
        <span class="sort-label">정렬</span>
        <button
          type="button"
          class="sort-btn"
          :class="{ active: sortOption === 'default' }"
          @click="sortOption = 'default'"
        >
          기본순
        </button>
        <button
          type="button"
          class="sort-btn"
          :class="{ active: sortOption === 'title' }"
          @click="sortOption = 'title'"
        >
          제목순
        </button>
        <button
          type="button"
          class="sort-btn"
          :class="{ active: sortOption === 'releaseDate' }"
          @click="sortOption = 'releaseDate'"
        >
          개봉일순
        </button>
        <button
          type="button"
          class="sort-btn"
          :class="{ active: sortOption === 'rating' }"
          @click="sortOption = 'rating'"
        >
          평점순
        </button>
      </div>
    </section>

    <div v-if="store.isLoading" class="status-message loading">
      ⏳ 실시간 국내 개봉작 데이터를 싣고 오는 중입니다...
    </div>

    <div v-else-if="store.errorMessage" class="status-message error">
      🚨 {{ store.errorMessage }}
    </div>

    <template v-else>
      <div class="result-summary">
        총 {{ sortedMovies.length }}개의 영화가 표시됩니다.
      </div>

      <div v-if="sortedMovies.length === 0" class="status-message empty">
        검색 결과가 없습니다.
      </div>

      <div v-else class="movie-list">
        <div v-for="movie in pagedMovies" :key="movie.id" class="movie-card">
          <img
            v-if="movie.poster_path"
            :src="`https://image.tmdb.org/t/p/w500${movie.poster_path}`"
            :alt="movie.title"
            class="poster"
          />
          <div v-else class="poster-placeholder">이미지 준비 중</div>

          <div class="card-content">
            <h3 class="title">{{ movie.title }}</h3>
            <p v-if="movie.release_date" class="release-date">
              🗓️ 개봉일: {{ movie.release_date }}
            </p>
            <p class="rating">⭐ {{ movie.vote_average.toFixed(1) }} / 10</p>
            <p class="overview">
              {{
                movie.overview
                  ? movie.overview.substring(0, 60) + "..."
                  : "국내에 등록된 줄거리 요약 정보가 없습니다."
              }}
            </p>
            <button
              @click="store.toggleFavorite(movie.id)"
              :class="{ active: movie.isFavorite }"
              class="fav-btn"
            >
              {{ movie.isFavorite ? "❤️ 찜 해제" : "🤍 찜하기" }}
            </button>
          </div>
          <RouterLink
            :to="`/movies/${movie.id}`"
            class="stretched-link"
            :aria-label="`${movie.title} 상세 정보 보기`"
          ></RouterLink>
        </div>
      </div>

      <div v-if="totalPages > 1" class="pagination" aria-label="페이지 이동">
        <button
          v-for="pageNumber in pageNumbers"
          :key="pageNumber"
          type="button"
          class="page-btn"
          :class="{ active: currentPage === pageNumber }"
          @click="currentPage = pageNumber"
        >
          {{ pageNumber }}
        </button>
      </div>
    </template>
  </main>
</template>

<style scoped>
.page {
  padding: 40px;
  background-color: #f8f9fa;
  min-height: 100vh;
}
.header-section {
  margin-bottom: 30px;
  text-align: center;
  color: #2c3e50;
}
.sub-title {
  font-size: 14px;
  color: #7f8c8d;
  margin-top: 5px;
}
.control-panel {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  gap: 20px;
  max-width: 1200px;
  margin: 0 auto 24px;
  padding: 20px;
  background: #ffffff;
  border: 1px solid #e9ecef;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.04);
}
.search-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
  flex: 1;
  min-width: 220px;
}
.search-group label,
.sort-label {
  font-size: 13px;
  font-weight: 800;
  color: #2c3e50;
}
.search-input {
  width: 100%;
  min-height: 44px;
  padding: 0 14px;
  border: 1px solid #dfe4ea;
  border-radius: 8px;
  color: #2c3e50;
  background: #ffffff;
  outline: none;
}
.search-input:focus {
  border-color: #ff4757;
  box-shadow: 0 0 0 3px rgba(255, 71, 87, 0.12);
}
.sort-group {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 10px;
}
.sort-btn,
.page-btn {
  min-height: 40px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 800;
  transition: 0.2s;
}
.sort-btn {
  padding: 0 14px;
  background: #ecf0f1;
  color: #2c3e50;
}
.sort-btn:hover,
.page-btn:hover {
  transform: translateY(-1px);
}
.sort-btn.active,
.page-btn.active {
  background: #ff4757;
  color: #ffffff;
}
.result-summary {
  max-width: 1200px;
  margin: 0 auto 20px;
  color: #57606f;
  font-size: 14px;
  font-weight: 700;
}
.status-message {
  text-align: center;
  font-size: 20px;
  font-weight: bold;
  padding: 50px;
  border-radius: 12px;
}
.loading {
  color: #3498db;
  background-color: #e3f2fd;
}
.error {
  color: #e74c3c;
  background-color: #fdeaea;
}
.empty {
  color: #57606f;
  background-color: #ffffff;
  border: 1px solid #e9ecef;
}
.movie-list {
  max-width: 1200px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 30px;
}
.movie-card {
  position: relative;
  border-radius: 12px;
  overflow: hidden;
  background: white;
  text-align: left;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
  transition: transform 0.2s ease;
  display: flex;
  flex-direction: column;
}
.movie-card:hover {
  transform: translateY(-5px);
}
.poster {
  width: 100%;
  height: 380px;
  object-fit: cover;
}
.poster-placeholder {
  width: 100%;
  height: 380px;
  background-color: #ddd;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #7f8c8d;
  font-weight: bold;
}
.card-content {
  padding: 20px;
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}
.title {
  font-size: 18px;
  color: #333;
  margin: 0 0 6px 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-weight: bold;
}
.release-date {
  font-size: 13px;
  color: #7f8c8d;
  margin-bottom: 10px;
  font-weight: 500;
}
.rating {
  font-weight: bold;
  color: #f39c12;
  margin-bottom: 10px;
  font-size: 16px;
}
.overview {
  font-size: 13px;
  color: #555;
  line-height: 1.4;
  margin-bottom: 20px;
  flex-grow: 1;
}
.fav-btn {
  position: relative;
  z-index: 2;
  width: 100%;
  padding: 12px;
  cursor: pointer;
  border: none;
  background: #ecf0f1;
  color: #333;
  border-radius: 8px;
  font-weight: bold;
  font-size: 14px;
  transition: 0.3s;
  margin-top: auto;
}
.fav-btn.active {
  background: #ff4757;
  color: white;
}
.stretched-link {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1;
}
.pagination {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 34px;
}
.page-btn {
  min-width: 42px;
  padding: 0 12px;
  background: #ffffff;
  color: #2c3e50;
  border: 1px solid #dfe4ea;
}
@media (max-width: 760px) {
  .page {
    padding: 24px 16px;
  }

  .control-panel {
    align-items: stretch;
    flex-direction: column;
  }

  .sort-group {
    align-items: stretch;
  }

  .sort-label {
    width: 100%;
  }

  .sort-btn {
    flex: 1 1 calc(50% - 8px);
  }
}
</style>
