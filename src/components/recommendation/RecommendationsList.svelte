<script lang="ts">
  import { fade } from 'svelte/transition';
  import { flip } from 'svelte/animate';

  import { getSurface, submitAnalyticsEvent } from 'src/analytics';
  import type { AnimeDetails } from 'src/malAPI';
  import type { Recommendation, UserRatingStats } from '../../routes/recommendation/recommendation/recommendation';
  import { getRatingTier, type RatingTier } from 'src/util/ratingTiers';
  import RecommendationListItem from './RecommendationListItem.svelte';
  import Filter from 'carbon-icons-svelte/lib/Filter.svelte';
  import Search from 'carbon-icons-svelte/lib/Search.svelte';
  import Calendar from 'carbon-icons-svelte/lib/Calendar.svelte';
  import ArrowsVertical from 'carbon-icons-svelte/lib/ArrowsVertical.svelte';
  import Video from 'carbon-icons-svelte/lib/Video.svelte';
  import Catalog from 'carbon-icons-svelte/lib/Catalog.svelte';
  import GenreIcon from './GenreIcon.svelte';

  export let recommendations: Recommendation[];
  export let animeMetadataDatabase: { [animeID: number]: AnimeDetails };
  export let excludeRanking: ((animeID: number) => void) | undefined = undefined;
  export let excludeGenre: ((genreID: number, genreName: string) => void) | undefined = undefined;
  export let includeGenre: ((genreID: number) => void) | undefined = undefined;
  export let excludedGenreIDs: number[] = [];
  export let genreNames: Map<number, string> = new Map();
  export let addRanking: ((animeID: number) => void) | undefined = undefined;
  export let contributorsLoading: boolean;
  export let userRatingStats: UserRatingStats | null = null;
  export let contributionBaseline: number | undefined = undefined;

  type SortMode =
    | 'model'
    | 'score-desc'
    | 'score-asc'
    | 'predicted-desc'
    | 'predicted-asc'
    | 'year-desc'
    | 'year-asc'
    | 'title-asc';

  const MEDIA_TYPE_NAMES: { [mediaType: string]: string } = {
    tv: 'TV',
    tv_special: 'TV Special',
    ova: 'OVA',
    ona: 'ONA',
    movie: 'Movie',
    special: 'Special',
    music: 'Music',
    cm: 'Commercial',
    pv: 'PV',
    unknown: 'Unknown',
  };

  let sortMode: SortMode = 'model';
  let genreFilter = '';
  let mediaTypeFilter = '';
  let yearFrom: number | undefined = undefined;
  let yearTo: number | undefined = undefined;
  let titleFilter = '';

  // Drop contributors dwarfed by the rec's top one or below the profile-wide
  // significance scale; always keep at least the top contributor.
  const REL_CUTOFF = 0.15;
  const SIG_MULT = 3;
  const visibleContributors = (contributors: Recommendation['topContributors']) => {
    if (!contributors?.length) {
      return contributors;
    }
    const sigFloor = contributionBaseline !== undefined ? SIG_MULT * contributionBaseline : 0;
    const cutoff = Math.max(REL_CUTOFF * contributors[0].strength, sigFloor);
    const kept = contributors.filter((c) => c.strength >= cutoff);
    return kept.length > 0 ? kept : contributors.slice(0, 1);
  };

  const getTitle = (datum: AnimeDetails): string => datum.alternative_titles?.en || datum.title || '';
  const getYear = (datum: AnimeDetails): number | null => {
    const year = Number(datum.start_date?.slice(0, 4));
    return Number.isFinite(year) && year > 0 ? year : null;
  };
  const parseYearFilter = (raw: number | undefined): number | null => {
    if (raw === undefined || raw === null) {
      return null;
    }
    return Number.isFinite(raw) ? raw : null;
  };

  $: hasPredictedRatings =
    !!userRatingStats &&
    !userRatingStats.isNonRater &&
    recommendations.some((reco) => typeof reco.predictedRating === 'number');
  $: if (!hasPredictedRatings && (sortMode === 'predicted-desc' || sortMode === 'predicted-asc')) {
    sortMode = 'model';
  }

  $: availableGenres = Array.from(
    new Set(recommendations.flatMap((reco) => animeMetadataDatabase[reco.id]?.genres?.map((genre) => genre.name) ?? []))
  ).sort((a, b) => a.localeCompare(b));

  $: exclusionGenres = (() => {
    const genres = new Map(genreNames);
    for (const anime of Object.values(animeMetadataDatabase)) {
      for (const genre of anime.genres ?? []) genres.set(genre.id, genre.name);
    }
    for (const id of excludedGenreIDs) {
      if (!genres.has(id)) genres.set(id, `Genre ${id}`);
    }
    return Array.from(genres).sort((a, b) => a[1].localeCompare(b[1]));
  })();

  $: availableMediaTypes = Array.from(
    new Set(
      recommendations
        .map((reco) => animeMetadataDatabase[reco.id]?.media_type)
        .filter((mediaType): mediaType is NonNullable<typeof mediaType> => !!mediaType)
    )
  ).sort((a, b) => (MEDIA_TYPE_NAMES[a] ?? a).localeCompare(MEDIA_TYPE_NAMES[b] ?? b));

  let displayRecommendations: (Recommendation & {
    shownPredictedRating: number | null;
    ratingTier: RatingTier | null;
    modelRank: number;
  })[] = [];

  $: {
    const minYear = parseYearFilter(yearFrom);
    const maxYear = parseYearFilter(yearTo);
    const titleNeedle = titleFilter.trim().toLocaleLowerCase();

    displayRecommendations = recommendations
      .map((reco, modelRank) => {
        const showRating = !!userRatingStats && !userRatingStats.isNonRater && typeof reco.predictedRating === 'number';
        return {
          ...reco,
          modelRank,
          shownPredictedRating: showRating ? reco.predictedRating! : null,
          ratingTier: showRating ? getRatingTier(reco.predictedRating!, userRatingStats!) : null,
        };
      })
      .filter((reco) => {
        const metadata = animeMetadataDatabase[reco.id];
        if (!metadata) {
          return false;
        }

        if (genreFilter && !metadata.genres?.some((genre) => genre.name === genreFilter)) {
          return false;
        }
        if (mediaTypeFilter && metadata.media_type !== mediaTypeFilter) {
          return false;
        }

        const year = getYear(metadata);
        if ((minYear !== null || maxYear !== null) && year === null) {
          return false;
        }
        if (minYear !== null && year !== null && year < minYear) {
          return false;
        }
        if (maxYear !== null && year !== null && year > maxYear) {
          return false;
        }

        if (titleNeedle) {
          const searchableTitles = [
            metadata.title,
            metadata.alternative_titles?.en,
            metadata.alternative_titles?.ja,
            ...(metadata.alternative_titles?.synonyms ?? []),
          ]
            .filter(Boolean)
            .join(' ')
            .toLocaleLowerCase();
          if (!searchableTitles.includes(titleNeedle)) {
            return false;
          }
        }

        return true;
      })
      .sort((a, b) => {
        const metadataA = animeMetadataDatabase[a.id];
        const metadataB = animeMetadataDatabase[b.id];

        switch (sortMode) {
          case 'score-desc':
            return b.score - a.score || a.modelRank - b.modelRank;
          case 'score-asc':
            return a.score - b.score || a.modelRank - b.modelRank;
          case 'predicted-desc':
            if (a.shownPredictedRating === null && b.shownPredictedRating === null) return a.modelRank - b.modelRank;
            if (a.shownPredictedRating === null) return 1;
            if (b.shownPredictedRating === null) return -1;
            return b.shownPredictedRating - a.shownPredictedRating || a.modelRank - b.modelRank;
          case 'predicted-asc':
            if (a.shownPredictedRating === null && b.shownPredictedRating === null) return a.modelRank - b.modelRank;
            if (a.shownPredictedRating === null) return 1;
            if (b.shownPredictedRating === null) return -1;
            return a.shownPredictedRating - b.shownPredictedRating || a.modelRank - b.modelRank;
          case 'year-desc': {
            const yearA = getYear(metadataA) ?? Number.NEGATIVE_INFINITY;
            const yearB = getYear(metadataB) ?? Number.NEGATIVE_INFINITY;
            return yearB - yearA || a.modelRank - b.modelRank;
          }
          case 'year-asc': {
            const yearA = getYear(metadataA) ?? Number.POSITIVE_INFINITY;
            const yearB = getYear(metadataB) ?? Number.POSITIVE_INFINITY;
            return yearA - yearB || a.modelRank - b.modelRank;
          }
          case 'title-asc':
            return getTitle(metadataA).localeCompare(getTitle(metadataB)) || a.modelRank - b.modelRank;
          case 'model':
          default:
            return a.modelRank - b.modelRank;
        }
      });
  }

  const clearDisplayFilters = () => {
    genreFilter = '';
    mediaTypeFilter = '';
    yearFrom = undefined;
    yearTo = undefined;
    titleFilter = '';
    [...excludedGenreIDs].forEach((id) => includeGenre?.(id));
  };

  $: displayFiltersActive =
    !!genreFilter ||
    !!mediaTypeFilter ||
    yearFrom !== undefined ||
    yearTo !== undefined ||
    !!titleFilter ||
    excludedGenreIDs.length > 0;

  let expandedAnimeID: number | null = null;
  $: if (expandedAnimeID !== null && !displayRecommendations.some((reco) => reco.id === expandedAnimeID)) {
    expandedAnimeID = null;
  }
</script>

<div class="browse-heading"><Filter size={20} aria-hidden="true" /><span>Browse recommendations</span></div>
<div class="browse-controls">
  <div class="control">
    <label for="recommendation-sort"><ArrowsVertical size={16} aria-hidden="true" />Sort</label>
    <select id="recommendation-sort" bind:value={sortMode}>
      <option value="model">Model ranking (default)</option>
      <option value="score-desc">Model score: high to low</option>
      <option value="score-asc">Model score: low to high</option>
      <option value="predicted-desc" disabled={!hasPredictedRatings}>Predicted rating: high to low</option>
      <option value="predicted-asc" disabled={!hasPredictedRatings}>Predicted rating: low to high</option>
      <option value="year-desc">Year: newest first</option>
      <option value="year-asc">Year: oldest first</option>
      <option value="title-asc">Title: A–Z</option>
    </select>
  </div>

  <div class="control">
    <label for="genre-filter"
      >{#if genreFilter}<GenreIcon name={genreFilter} />{:else}<Catalog size={16} aria-hidden="true" />{/if}Genre</label
    >
    <select id="genre-filter" bind:value={genreFilter}>
      <option value="">All genres</option>
      {#if genreFilter && !availableGenres.includes(genreFilter)}
        <option value={genreFilter}>{genreFilter} (not in current results)</option>
      {/if}
      {#each availableGenres as genre}
        <option value={genre}>{genre}</option>
      {/each}
    </select>
  </div>

  <div class="control">
    <label for="format-filter"><Video size={16} aria-hidden="true" />Format</label>
    <select id="format-filter" bind:value={mediaTypeFilter}>
      <option value="">All enabled formats</option>
      {#if mediaTypeFilter && !availableMediaTypes.some((mediaType) => mediaType === mediaTypeFilter)}
        <option value={mediaTypeFilter}
          >{MEDIA_TYPE_NAMES[mediaTypeFilter] ?? mediaTypeFilter} (not in current results)</option
        >
      {/if}
      {#each availableMediaTypes as mediaType}
        <option value={mediaType}>{MEDIA_TYPE_NAMES[mediaType] ?? mediaType}</option>
      {/each}
    </select>
  </div>

  <div class="control year-control">
    <label for="year-from"><Calendar size={16} aria-hidden="true" />Year</label>
    <div>
      <input id="year-from" type="number" min="1900" max="2200" placeholder="From" bind:value={yearFrom} />
      <span>–</span>
      <input type="number" min="1900" max="2200" placeholder="To" aria-label="Year to" bind:value={yearTo} />
    </div>
  </div>

  <div class="control search-control">
    <label for="title-filter"><Search size={16} aria-hidden="true" />Title</label>
    <input id="title-filter" type="search" placeholder="Filter titles" bind:value={titleFilter} />
  </div>

  <div class="control-actions">
    <span role="status">{displayRecommendations.length} of {recommendations.length}</span>
    <button type="button" on:click={clearDisplayFilters} disabled={!displayFiltersActive}>Clear filters</button>
  </div>
  {#if excludeGenre && includeGenre}
    <details class="genre-exclusions">
      <summary
        >Exclude genres{#if excludedGenreIDs.length}
          · {excludedGenreIDs.length} active{/if}</summary
      >
      <p>
        Checked genres are excluded from all recommendations and trigger a new model request. Uncheck a genre to restore
        it, or use Clear filters.
      </p>
      <div class="genre-options">
        {#each exclusionGenres as [id, name] (id)}
          <label class:excluded={excludedGenreIDs.includes(id)}>
            <input
              type="checkbox"
              checked={excludedGenreIDs.includes(id)}
              on:change={(evt) => (evt.currentTarget.checked ? excludeGenre?.(id, name) : includeGenre?.(id))}
            />
            <GenreIcon {name} /><span>{name}</span>
          </label>
        {/each}
      </div>
    </details>
  {/if}
</div>

<div class="browse-helper">
  <b>Model ranking</b> is Sprout's original recommendation order, based on the model's combined recommendation score. Predicted
  rating is the personalized 1–10 estimate shown beside each title. Sort, Genre, Format, Year, and Title organize the
  recommendations already generated. Exclude genres requests new recommendations without the checked genres. The current
  metadata exposes MAL genres, format, year, and titles; it does not expose a separate free-form tag taxonomy.
</div>

<div class="recommendations">
  {#each displayRecommendations as { id, topContributors, planToWatch, shownPredictedRating, ratingTier, modelRank } (id)}
    {@const animeMetadata = animeMetadataDatabase[id]}
    <div in:fade animate:flip={{ duration: (d) => 39 * Math.sqrt(d) }}>
      <RecommendationListItem
        {animeMetadata}
        rank={modelRank}
        expanded={expandedAnimeID === animeMetadata.id}
        toggleExpanded={() => {
          const expanded = expandedAnimeID !== animeMetadata.id;
          submitAnalyticsEvent({
            category: 'recommendations',
            subcategory: 'item_expand',
            payload: { anime_id: animeMetadata.id, rank: modelRank, expanded, surface: getSurface() },
          });
          expandedAnimeID = expanded ? animeMetadata.id : null;
        }}
        topContributors={visibleContributors(topContributors)?.map((c) => ({
          ...c,
          datum: animeMetadataDatabase[c.animeId],
        }))}
        {contributionBaseline}
        planToWatch={planToWatch ?? false}
        predictedRating={shownPredictedRating}
        {ratingTier}
        {excludeRanking}
        {addRanking}
        {contributorsLoading}
      />
    </div>
  {/each}
  {#if displayRecommendations.length === 0}
    <p class="empty-results">
      No recommendations match the current controls. Clear the list filters or enable more formats.
    </p>
  {/if}
</div>

<style lang="css">
  .genre-exclusions {
    grid-column: 1 / -1;
    border-top: 1px solid #ffffff16;
    padding-top: 10px;
  }

  .genre-exclusions summary {
    cursor: pointer;
    color: #d7e4da;
    font-size: 13px;
  }

  .genre-exclusions p {
    font-size: 12px;
    line-height: 1.4;
    color: #a8b0b8;
    margin: 8px 0;
  }

  .genre-options {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  .genre-options label {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 9px;
    border: 1px solid #ffffff20;
    border-radius: 6px;
    font-size: 12px;
    cursor: pointer;
  }

  .genre-options label.excluded {
    border-color: #bc8b61;
    background: #372b23;
  }

  .genre-options input {
    accent-color: #55d95f;
  }

  .browse-heading {
    display: flex;
    align-items: center;
    gap: 7px;
    color: #d7e4da;
    font-size: 14px;
    font-weight: 500;
    padding-top: 16px;
  }

  .browse-controls {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(min(100%, 190px), 1fr));
    gap: 8px;
    align-items: end;
    padding: 12px;
    margin-top: 10px;
    border: 1px solid #ffffff16;
    border-radius: 8px;
    background: #191d1b;
    border-bottom: 1px solid #cccccc22;
  }

  .control {
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-width: 0;
  }

  .control label {
    font-size: 0.75rem;
    color: #c6c6c6;
    display: flex;
    align-items: center;
    gap: 5px;
  }

  .control select,
  .control input,
  .control-actions button {
    min-height: 38px;
    border: 1px solid #525252;
    background: #262626;
    color: #f4f4f4;
    padding: 5px 8px;
    box-sizing: border-box;
    border-radius: 5px;
  }

  .control select,
  .control input {
    width: 100%;
  }

  .control select:focus-visible,
  .control input:focus-visible,
  .control-actions button:focus-visible {
    outline: 2px solid #55d95f;
    outline-offset: 2px;
  }

  .year-control > div {
    display: flex;
    align-items: center;
    gap: 5px;
  }

  .year-control input {
    min-width: 0;
  }

  .control-actions {
    display: flex;
    align-items: center;
    gap: 8px;
    white-space: nowrap;
  }

  .control-actions span {
    font-size: 0.75rem;
    color: #a8a8a8;
  }

  .control-actions button {
    cursor: pointer;
  }

  .control-actions button:hover:not(:disabled) {
    background: #353535;
  }

  .control-actions button:disabled {
    cursor: default;
    opacity: 0.45;
  }

  .browse-helper {
    padding: 10px 2px 14px;
    font-size: 0.75rem;
    line-height: 1.35;
    color: #9ba3ab;
    border-bottom: 1px solid #cccccc22;
  }

  .recommendations {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .empty-results {
    padding: 20px 8px;
    color: #c6c6c6;
  }

  @media (max-width: 1100px) {
    .browse-controls {
      grid-template-columns: repeat(3, minmax(0, 1fr));
    }
  }

  @media (max-width: 768px) {
    .browse-controls {
      grid-template-columns: repeat(2, minmax(0, 1fr));
    }

    .control-actions {
      align-self: stretch;
      justify-content: space-between;
    }
  }

  @media (max-width: 480px) {
    .browse-controls {
      grid-template-columns: minmax(0, 1fr);
    }
  }
</style>
