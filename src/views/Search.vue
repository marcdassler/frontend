<template>
  <section>
    <Container variant="default" style="padding-top: 20px">
      <v-text-field
        id="searchInput"
        v-model="store.globalSearchTerm"
        clearable
        prepend-inner-icon="mdi-magnify"
        :label="$t('search')"
        hide-details
        variant="outlined"
        @focus="searchHasFocus = true"
        @blur="searchHasFocus = false"
      >
        <template #label>{{ $t("type_to_search") }}</template>
      </v-text-field>

      <v-chip-group
        v-model="selectedSearchType"
        style="margin-top: 10px; margin-left: 10px"
        selected-class="text-primary"
        mandatory
      >
        <v-chip
          v-for="item in [
            SEARCH_TYPE_ALL,
            MediaType.TRACK,
            MediaType.ARTIST,
            MediaType.ALBUM,
            MediaType.PLAYLIST,
            MediaType.PODCAST,
            MediaType.AUDIOBOOK,
            MediaType.RADIO,
            MediaType.GENRE,
          ]"
          :key="item"
          :text="$t(item === SEARCH_TYPE_ALL ? 'searchtype_all' : item + 's')"
          :value="item"
          filter
        />
      </v-chip-group>

      <v-progress-linear
        v-if="loading"
        color="accent"
        height="4"
        indeterminate
        rounded
        style="margin-top: 15px"
      />

      <!-- compact all-media-types searchresult -->
      <div v-if="!store.globalSearchType" class="search-shelves">
        <EditorialShelf
          v-for="section in searchSections"
          :key="section.key"
          :title="section.title"
          :tiles-per-view="tilesPerView"
        >
          <EditorialMediaCard
            v-for="item in section.items"
            :key="item.uri"
            :item="item"
            :show-provider-on-cover="true"
            :is-available="itemIsAvailable(item)"
          />
        </EditorialShelf>
      </div>
      <!-- tracks-only searchresult: render progressively while the stream is
           still landing chunks; the progress bar above is the loading affordance. -->
      <div v-else>
        <ItemsListing
          :itemtype="`${store.globalSearchType}s`"
          :show-provider="true"
          :show-favorites-only-filter="false"
          :show-select-button="false"
          :show-refresh-button="false"
          :load-items="
            async (params) => {
              return filteredItems(store.globalSearchType!);
            }
          "
          :title="$t(`${store.globalSearchType}s`)"
          :allow-key-hooks="false"
          :show-search-button="false"
          :infinite-scroll="true"
          :sort-keys="[]"
          style="padding: 0"
        />
      </div>
    </Container>
  </section>
</template>

<script setup lang="ts">
/* eslint-disable @typescript-eslint/no-unused-vars,vue/no-setup-props-destructure */
import Container from "@/components/Container.vue";
import EditorialMediaCard from "@/components/discover/EditorialMediaCard.vue";
import EditorialShelf from "@/components/discover/EditorialShelf.vue";
import ItemsListing from "@/components/ItemsListing.vue";
import { useUserPreferences } from "@/composables/userPreferences";
import { panelViewItemResponsive } from "@/helpers/utils";
import { api } from "@/plugins/api";
import { itemIsAvailable } from "@/plugins/api/helpers";
import { Genre, MediaType, SearchResults } from "@/plugins/api/interfaces";
import { $t } from "@/plugins/i18n";
import { store } from "@/plugins/store";
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";

const SEARCH_TYPE_ALL = "all";

// computed to bridge between chip-group (needs a real value) and store (uses undefined for "all")
const selectedSearchType = computed({
  get: () => store.globalSearchType || SEARCH_TYPE_ALL,
  set: (val: string) => {
    store.globalSearchType =
      val === SEARCH_TYPE_ALL ? undefined : (val as MediaType);
  },
});

// local refs
const searchHasFocus = ref(false);
const searchResult = ref<SearchResults>();
const loading = ref(false);
const throttleId = ref();
// AbortController for the in-flight streaming search; abort it when a new
// search starts so we don't merge stale chunks into the new result.
const streamAbort = ref<AbortController>();
const { getPreference, setPreference } = useUserPreferences();

// Helper: dedupe by `uri` (every MediaItem carries one) while preserving order.
// Used to merge progressive chunks without duplicating items the library and
// providers both surface.
function dedupByUri<T extends { uri?: string }>(items: T[]): T[] {
  const seen = new Set<string>();
  const out: T[] = [];
  for (const item of items) {
    const key = item.uri ?? `${out.length}`;
    if (seen.has(key)) continue;
    seen.add(key);
    out.push(item);
  }
  return out;
}

// Merge a streamed chunk into the accumulator. Re-emits a NEW SearchResults
// object so Vue's reactivity picks up the change cheaply (one ref assignment).
function mergeChunk(
  acc: SearchResults,
  chunk: SearchResults,
  limit: number,
): SearchResults {
  return {
    artists: dedupByUri([
      ...(acc.artists ?? []),
      ...(chunk.artists ?? []),
    ]).slice(0, limit),
    albums: dedupByUri([...(acc.albums ?? []), ...(chunk.albums ?? [])]).slice(
      0,
      limit,
    ),
    tracks: dedupByUri([...(acc.tracks ?? []), ...(chunk.tracks ?? [])]).slice(
      0,
      limit,
    ),
    playlists: dedupByUri([
      ...(acc.playlists ?? []),
      ...(chunk.playlists ?? []),
    ]).slice(0, limit),
    radio: dedupByUri([...(acc.radio ?? []), ...(chunk.radio ?? [])]).slice(
      0,
      limit,
    ),
    podcasts: dedupByUri([
      ...(acc.podcasts ?? []),
      ...(chunk.podcasts ?? []),
    ]).slice(0, limit),
    audiobooks: dedupByUri([
      ...(acc.audiobooks ?? []),
      ...(chunk.audiobooks ?? []),
    ]).slice(0, limit),
    genres: acc.genres ?? chunk.genres ?? [],
  };
}

// Responsive tile sizing, shared curve with the rest of the app.
const tilesPerView = computed(() => panelViewItemResponsive(0) + 0.5);

// Compact "all" results as horizontal shelves; empty categories are hidden.
// NOTE: render-while-loading is intentional - the progress bar above provides
// the loading affordance while shelves fill in progressively from the stream.
const searchSections = computed(() => {
  const r = searchResult.value;
  if (!r) return [];
  return [
    { key: "tracks", title: $t("tracks"), items: r.tracks },
    { key: "artists", title: $t("artists"), items: r.artists },
    { key: "albums", title: $t("albums"), items: r.albums },
    { key: "playlists", title: $t("playlists"), items: r.playlists },
    { key: "podcasts", title: $t("podcasts"), items: r.podcasts },
    { key: "audiobooks", title: $t("audiobooks"), items: r.audiobooks },
    { key: "radios", title: $t("radios"), items: r.radio },
    { key: "genres", title: $t("genres"), items: r.genres },
  ].filter((s) => s.items?.length);
});

// watchers
watch(
  () => store.globalSearchTerm,
  () => {
    clearTimeout(throttleId.value);
    throttleId.value = setTimeout(() => {
      loadSearchResults(store.globalSearchTerm, store.globalSearchType);
    }, 1000);
  },
  { immediate: true },
);
watch(
  () => store.globalSearchType,
  () => {
    setPreference(
      "globalSearchType",
      store.globalSearchType || SEARCH_TYPE_ALL,
    );
    loadSearchResults(store.globalSearchTerm, store.globalSearchType);
  },
);

const loadSearchResults = async function (
  searchTerm?: string,
  filter?: MediaType,
) {
  // Cancel any in-flight stream from a previous query so its late chunks do
  // not leak into the new searchResult.
  streamAbort.value?.abort();

  setPreference("globalSearch", searchTerm || "");
  const limit = store.globalSearchType ? 50 : 8;
  const mediaTypes = filter ? [filter] : undefined;

  if (!searchTerm) {
    searchResult.value = undefined;
    loading.value = false;
    return;
  }

  loading.value = true;

  // Genre-only search: pure library lookup, no streaming needed.
  if (filter === MediaType.GENRE) {
    const genres = await api.getLibraryGenres({
      search: searchTerm,
      limit,
      offset: 0,
      order_by: "name",
    });
    searchResult.value = {
      artists: [],
      albums: [],
      tracks: [],
      playlists: [],
      radio: [],
      podcasts: [],
      audiobooks: [],
      genres,
    };
    loading.value = false;
    return;
  }

  // Set up the streaming search and the parallel genre supplement.
  const ac = new AbortController();
  streamAbort.value = ac;

  // Start with an empty accumulator so the EditorialShelf rows render the
  // moment the first chunk lands - no blank screen while we wait.
  let acc: SearchResults = {
    artists: [],
    albums: [],
    tracks: [],
    playlists: [],
    radio: [],
    podcasts: [],
    audiobooks: [],
    genres: [],
  };
  searchResult.value = acc;

  const genresPromise: Promise<Genre[]> = !filter
    ? api.getLibraryGenres({
        search: searchTerm,
        limit,
        offset: 0,
        order_by: "name",
      })
    : Promise.resolve([]);

  try {
    for await (const chunk of api.searchStream(
      searchTerm,
      mediaTypes,
      limit,
      ac.signal,
    )) {
      if (ac.signal.aborted) return;
      acc = mergeChunk(acc, chunk, limit);
      // Reassign the ref so Vue picks up the merged result on every chunk.
      searchResult.value = acc;
    }
  } catch (err) {
    if (err instanceof DOMException && err.name === "AbortError") {
      // a newer search took over; nothing to do
      return;
    }
    // unexpected stream error: log + leave whatever we have rendered
    console.error("[Search] streaming search failed:", err);
  }

  // The streamed result usually carries genres in the library chunk; only
  // supplement from getLibraryGenres if the stream did not surface any.
  if (!ac.signal.aborted) {
    const genres = await genresPromise;
    if (!ac.signal.aborted && !acc.genres?.length) {
      acc = { ...acc, genres };
      searchResult.value = acc;
    }
  }

  if (streamAbort.value === ac) {
    loading.value = false;
    streamAbort.value = undefined;
  }
};

onMounted(() => {
  if (!store.globalSearchTerm) {
    const savedSearch = getPreference<string>("globalSearch").value;
    if (savedSearch && savedSearch !== "null") {
      store.globalSearchTerm = savedSearch;
    }
  }
  const savedSearchType = getPreference<string>("globalSearchType").value;
  if (
    savedSearchType &&
    savedSearchType !== "null" &&
    savedSearchType !== SEARCH_TYPE_ALL
  ) {
    store.globalSearchType = savedSearchType as MediaType;
  }
});

// lifecycle hooks
const keyListener = function (e: KeyboardEvent) {
  // Ignore keyboard events with modifier keys
  if (e.ctrlKey || e.altKey || e.metaKey) {
    return;
  }

  if (store.showPlayersMenu) {
    return;
  }

  if (!searchHasFocus.value && e.key == "Backspace" && store.globalSearchTerm) {
    store.globalSearchTerm = store.globalSearchTerm.slice(0, -1);
  } else if (!searchHasFocus.value && e.key.length == 1) {
    store.globalSearchTerm += e.key;
  }
};
document.addEventListener("keyup", keyListener);

onBeforeUnmount(() => {
  document.removeEventListener("keyup", keyListener);
});

const filteredItems = function (mediaType: MediaType) {
  if (!searchResult.value) return [];
  if (mediaType == MediaType.TRACK) return searchResult.value.tracks;
  if (mediaType == MediaType.ARTIST) return searchResult.value.artists;
  if (mediaType == MediaType.ALBUM) return searchResult.value.albums;
  if (mediaType == MediaType.PLAYLIST) return searchResult.value.playlists;
  if (mediaType == MediaType.PODCAST) return searchResult.value.podcasts;
  if (mediaType == MediaType.AUDIOBOOK) return searchResult.value.audiobooks;
  if (mediaType == MediaType.RADIO) return searchResult.value.radio;
  if (mediaType == MediaType.GENRE) return searchResult.value.genres;
  return [];
};
</script>
