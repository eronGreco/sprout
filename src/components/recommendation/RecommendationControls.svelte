<script lang="ts" context="module">
  import { getDefaultNicheBoostFactor, ModelName, PopularityAttenuationFactor, ProfileSource } from './conf';

  const ALL_MODEL_OPTIONS: { id: ModelName; text: string }[] = [
    { id: ModelName.Model_2026_logq, text: 'Aug. 2026' },
    { id: ModelName.Model_2026_rc, text: 'Aug. 2026 v2 (Beta)' },
    { id: ModelName.Model_2025_jax, text: 'Dec. 2025' },
    { id: ModelName.Legacy_2023, text: 'Legacy (2023)' },
  ];

  const ALL_POPULARITY_ATTENUATION_FACTOR_OPTIONS: { id: PopularityAttenuationFactor; text: string }[] = [
    { id: PopularityAttenuationFactor.None, text: 'None' },
    { id: PopularityAttenuationFactor.VeryLow, text: 'Very Low' },
    { id: PopularityAttenuationFactor.Low, text: 'Low' },
    { id: PopularityAttenuationFactor.Medium, text: 'Medium' },
    { id: PopularityAttenuationFactor.High, text: 'High' },
    { id: PopularityAttenuationFactor.VeryHigh, text: 'Very High' },
  ];
</script>

<script lang="ts">
  import { Dropdown, InlineLoading, Tag, Toggle, Slider } from 'carbon-components-svelte';
  import SettingsAdjust from 'carbon-icons-svelte/lib/SettingsAdjust.svelte';
  import Renew from 'carbon-icons-svelte/lib/Renew.svelte';
  import type { Writable } from 'svelte/store';

  import type { AnimeDetails } from 'src/malAPI';
  import { browser } from '$app/environment';
  import { getSurface, submitAnalyticsEvent } from 'src/analytics';
  import type { RecommendationControlParams } from './utils';

  let innerWidth = browser ? window.innerWidth : 0;
  $: isMobile = innerWidth < 768;

  export let params: Writable<RecommendationControlParams>;
  export let animeMetadataDatabase: { [animeID: number]: AnimeDetails };
  export let isLoading: boolean;
  export let forceHideTopBar: boolean | undefined = false;
  /**
   * Hides the presence/rating weight slider entirely for non-rater profiles, where rating
   * predictions are too unreliable for the control to be meaningful; the server scores those
   * profiles presence-only regardless of the param.
   */
  export let hideLogitWeight = false;
  export let onForceProfileRefresh: (() => void) | undefined = undefined;
  export let isProfileRefreshing = false;
  export let profileRefreshError = '';

  // Local state for sliders to prevent updates while dragging
  let localLogitWeight = $params.logitWeight;
  let localNicheBoostFactor = $params.nicheBoostFactor;

  const submitFilterToggle = (filter: string, enabled: boolean) =>
    submitAnalyticsEvent({
      category: 'recommendations',
      subcategory: 'filter_toggle',
      payload: { filter, enabled, surface: getSurface() },
    });

  // Sync local state with params store when params change from outside
  // $: localLogitWeight = $params.logitWeight;
  // $: localNicheBoostFactor = $params.nicheBoostFactor;
</script>

<svelte:window bind:innerWidth />

<div class="root">
  <div class="section-title"><SettingsAdjust size={20} aria-hidden="true" /><span>Recommendation settings</span></div>
  <div class="toggles">
    <div>
      <Toggle
        labelText="Extra Seasons"
        bind:toggled={$params.includeExtraSeasons}
        on:toggle={(evt) => submitFilterToggle('extra_seasons', evt.detail.toggled)}
      />
      <span class="toggle-helper">
        Off hides TV/unknown recommendations connected through sequel, prequel, parent-story, or side-story
        relationships to anime you already watched. Other formats are controlled by their own toggles. On allows these
        extra seasons/stories.
      </span>
    </div>
    <div>
      <Toggle
        labelText="Movies"
        bind:toggled={$params.includeMovies}
        on:toggle={(evt) => submitFilterToggle('movies', evt.detail.toggled)}
      />
      <span class="toggle-helper">Off excludes MAL entries whose media type is Movie. On allows movies.</span>
    </div>
    <div>
      <Toggle
        labelText="ONAs / OVAs / Specials"
        bind:toggled={$params.includeONAsOVAsSpecials}
        on:toggle={(evt) => submitFilterToggle('onas_ovas_specials', evt.detail.toggled)}
      />
      <span class="toggle-helper">Off excludes ONA, OVA, Special, and TV Special entries. On allows them.</span>
    </div>
    <div style="position: relative">
      <Toggle
        labelText="Music"
        bind:toggled={$params.includeMusic}
        on:toggle={(evt) => submitFilterToggle('music', evt.detail.toggled)}
      />
      <span class="toggle-helper"
        >Off excludes Music, commercials (CM), and promotional videos (PV). On allows them.</span
      >
      {#if isLoading && isMobile}
        <div style="position: absolute; right: -4px; bottom: -4px; flex: 0;">
          <InlineLoading />
        </div>
      {/if}
    </div>
    {#if !isMobile && isLoading}
      <InlineLoading style="flex: 0; margin-left: 10px;" />
    {/if}
  </div>
  {#if $params.profileSource === ProfileSource.MyAnimeList && onForceProfileRefresh}
    <div class="profile-refresh">
      <button
        type="button"
        disabled={isProfileRefreshing || isLoading || $params.modelName === ModelName.Legacy_2023}
        on:click={() => onForceProfileRefresh?.()}
      >
        <Renew size={16} aria-hidden="true" />
        {isProfileRefreshing ? 'Refreshing MyAnimeList…' : 'Refresh MyAnimeList data'}
      </button>
      <span class="helper-text">
        {#if $params.modelName === ModelName.Legacy_2023}
          Force refresh is unavailable for Legacy (2023), which is served by a separate legacy deployment.
        {:else}
          Sprout caches MyAnimeList lists for 60 seconds after fetching them; an open page does not update
          automatically. This button bypasses Sprout's profile cache, requests the list currently returned by MAL, and
          updates the recommendations. Updates still depend on MAL making your changes available through its API.
        {/if}
      </span>
      {#if profileRefreshError}
        <span class="refresh-error">{profileRefreshError}</span>
      {/if}
    </div>
  {/if}
  {#if !forceHideTopBar}
    <details
      class="advanced-options"
      on:toggle={() =>
        submitAnalyticsEvent({
          category: 'recommendations',
          subcategory: 'advanced_options_toggle',
          payload: { surface: getSurface() },
        })}
    >
      <summary>Advanced Options</summary>
      <div class="top">
        <div class="top-row">
          <div>
            <Dropdown
              style="width: 100%;"
              titleText="Model"
              selectedId={$params.modelName}
              on:select={(selected) => {
                const model = selected.detail.selectedItem.id;
                if (model !== $params.modelName) {
                  submitAnalyticsEvent({
                    category: 'recommendations',
                    subcategory: 'model_select',
                    payload: { model, surface: getSurface() },
                  });
                  if ($params.nicheBoostFactor === getDefaultNicheBoostFactor($params.modelName)) {
                    $params.nicheBoostFactor = getDefaultNicheBoostFactor(model);
                    localNicheBoostFactor = getDefaultNicheBoostFactor(model);
                  }
                }
                $params.modelName = model;
              }}
              items={ALL_MODEL_OPTIONS}
              helperText="Chooses the trained recommendation version. Aug. 2026 is the current default, v2 is experimental, Dec. 2025 is the previous model, and Legacy uses the older 2023 serving path. Changing model can change scores and ranking order."
            />
          </div>
          <div>
            <Toggle
              labelText="Filter Plan to Watch"
              bind:toggled={$params.filterPlanToWatch}
              on:toggle={(evt) => submitFilterToggle('plan_to_watch', evt.detail.toggled)}
            />
            <span class="helper-text"
              >On removes anime already marked Plan to Watch in your profile. Off keeps them eligible and labels them in
              the results.</span
            >
          </div>
        </div>
        <div class="bottom-row">
          {#if $params.modelName === ModelName.Legacy_2023}
            <div>
              <Dropdown
                style="width: 100%;"
                titleText="Popularity Attenuation Factor"
                selectedId={$params.popularityAttenuationFactor}
                on:select={(selected) => {
                  const value = selected.detail.selectedItem.id;
                  if (value !== $params.popularityAttenuationFactor) {
                    submitAnalyticsEvent({
                      category: 'recommendations',
                      subcategory: 'attenuation_change',
                      payload: { value, surface: getSurface() },
                    });
                  }
                  $params.popularityAttenuationFactor = value;
                }}
                items={ALL_POPULARITY_ATTENUATION_FACTOR_OPTIONS}
                helperText="Legacy model only. Higher values increasingly favor less-popular anime; None preserves the legacy model ranking without this adjustment."
              />
            </div>
          {:else}
            {#if !hideLogitWeight}
              <div>
                <Slider
                  labelText="Presence/Rating Weight"
                  min={0}
                  max={1}
                  step={0.1}
                  bind:value={localLogitWeight}
                  on:change={(evt) => {
                    if ($params.logitWeight !== evt.detail) {
                      submitAnalyticsEvent({
                        category: 'recommendations',
                        subcategory: 'logit_weight_change',
                        payload: { value: evt.detail, surface: getSurface() },
                      });
                    }
                    $params.logitWeight = evt.detail;
                  }}
                />
                <span class="helper-text">
                  0 prioritizes your predicted 1–10 rating; 1 prioritizes how strongly the model expects the anime to
                  belong in your profile. Intermediate values blend both signals.
                </span>
              </div>
            {/if}
            <div>
              <Slider
                labelText="Niche Boost Factor"
                min={0}
                max={1}
                step={0.1}
                bind:value={localNicheBoostFactor}
                on:change={(evt) => {
                  if ($params.nicheBoostFactor !== evt.detail) {
                    submitAnalyticsEvent({
                      category: 'recommendations',
                      subcategory: 'niche_boost_change',
                      payload: { value: evt.detail, surface: getSurface() },
                    });
                  }
                  $params.nicheBoostFactor = evt.detail;
                }}
              />
              <span class="helper-text">
                0 adds no niche boost. Higher values increasingly favor anime the model predicts for you more strongly
                than their overall popularity would suggest.
              </span>
            </div>
          {/if}
        </div>
      </div>
    </details>
  {/if}
  {#if $params.excludedRankingAnimeIDs.length > 0}
    <div>
      <label for="excluded-rankings" class="bx--label">Excluded Rankings</label>
      <div class="tags-container" id="excluded-rankings">
        {#each [...new Set($params.excludedRankingAnimeIDs)] as animeID (animeID)}
          {@const datum = animeMetadataDatabase[animeID]}
          {@const title = datum?.alternative_titles.en || datum?.title || ''}
          <Tag
            filter
            on:close={() => {
              submitAnalyticsEvent({
                category: 'recommendations',
                subcategory: 'exclude_ranking_remove',
                payload: { anime_id: animeID, surface: getSurface() },
              });
              params.update((state) => {
                state.excludedRankingAnimeIDs = state.excludedRankingAnimeIDs.filter(
                  (oAnimeID) => oAnimeID !== animeID
                );
                return state;
              });
            }}
          >
            {title}
          </Tag>
        {/each}
      </div>
    </div>
  {/if}
</div>

<style lang="css">
  .root {
    display: flex;
    flex-direction: column;
    gap: 14px;
    padding: 5px 0px 10px 0px;
    border-bottom: 1px solid #cccccc22;
  }

  .section-title {
    display: flex;
    align-items: center;
    gap: 7px;
  }

  .section-title {
    color: #d7e4da;
    font-size: 14px;
    font-weight: 500;
  }

  .top {
    display: flex;
    flex-direction: row;
    min-width: 100%;
    gap: 16px;
    padding: 4px;
  }

  .top-row,
  .bottom-row {
    display: flex;
    flex-direction: row;
    gap: 16px;
  }

  .top-row > div,
  .bottom-row > div {
    display: flex;
    flex: 1;
    flex-direction: column;
    min-width: 0;
  }

  .top > div {
    display: flex;
    flex: 1;
    flex-direction: column;
  }

  .top :global(.bx--slider-container) {
    min-width: 0;
    width: 100%;
  }

  .top :global(.bx--slider) {
    min-width: 0;
    flex: 1;
  }

  .top :global(.bx--slider-text-input) {
    min-width: 48px;
    width: 48px;
  }

  .toggles {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    flex: 1;
    gap: 8px;
  }

  .toggles > div {
    display: flex;
    flex-direction: column;
    gap: 5px;
    padding: 12px;
    box-sizing: border-box;
    border: 1px solid #ffffff16;
    border-radius: 8px;
    background: #191d1b;
    flex: 1 1 190px;
    min-width: 0;
  }

  .toggle-helper {
    font-size: 0.72rem;
    line-height: 1.25;
    color: #9ba3ab;
  }

  .profile-refresh {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .profile-refresh button {
    border: 1px solid #6f6f6f;
    background: #2b2b2b;
    color: #f4f4f4;
    padding: 7px 12px;
    cursor: pointer;
    white-space: nowrap;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    border-radius: 6px;
  }

  .profile-refresh button:hover:not(:disabled) {
    background: #353535;
  }

  .profile-refresh button:disabled {
    cursor: not-allowed;
    opacity: 0.6;
  }

  .refresh-error {
    color: #ff8389;
    font-size: 0.75rem;
  }

  @media (max-width: 768px) {
    .toggles > div {
      flex: 1 1 calc(50% - 8px);
      min-width: 0;
    }

    .profile-refresh {
      align-items: flex-start;
      flex-direction: column;
      gap: 5px;
    }
  }

  .toggles > div:first-child {
    border-left: 1px solid #cccccc22;
  }

  .tags-container {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
  }

  .helper-text {
    font-size: 0.75rem;
    color: #a8a8a8;
    margin-top: 4px;
  }

  .advanced-options {
    background: #1b201d;
    padding: 12px;
    border: 1px solid #ffffff16;
    border-radius: 8px;
  }

  summary {
    cursor: pointer;
    padding: 4px 0;
  }

  summary:focus-visible {
    outline: 2px solid #78a9ff;
    outline-offset: 4px;
  }

  @media (max-width: 768px) {
    .top {
      flex-direction: column;
    }
  }
</style>
