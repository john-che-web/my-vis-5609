<script lang="ts">
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import { Line, Scatter } from '$lib';
  import type { TMovie } from '../../types';
  import type { ChannelAttribute } from '$lib/Scatter.svelte';

  const channelAttributes: { value: ChannelAttribute; label: string }[] = [
    { value: 'year', label: 'Year' },
    { value: 'runtime_minutes', label: 'Runtime' },
    { value: 'average_rating', label: 'Average rating' },
    { value: 'num_votes', label: 'Number of votes' },
    { value: 'title_type', label: 'Title type' },
    { value: 'primary_title', label: 'Primary title' },
    { value: 'original_title', label: 'Original title' },
    { value: 'simple_title', label: 'Simple title' },
    { value: 'tconst', label: 'Movie ID' }
  ];

  let movies: TMovie[] = $state([]);
  let totalMovies = $state(0);
  let loadError: string | undefined = $state(undefined);
  let selectedMovieId: string | undefined = $state(undefined);
  let selectedMovie = $derived(movies.find(movie => movie.tconst === selectedMovieId));
  let brushedYears: [number, number] | undefined = $state(undefined);
  let xAttribute: ChannelAttribute = $state('year');
  let yAttribute: ChannelAttribute = $state('simple_title');
  let sizeAttribute: ChannelAttribute = $state('year');  let visibleMovies = $derived.by(() => {
    const yearRange = brushedYears;
    if (!yearRange) return movies;
    return movies.filter(movie => {
      const year = movie.year.getFullYear();
      return year >= yearRange[0] && year <= yearRange[1];
    });
  });

  onMount(async () => {
    try {
      const data = await d3.csv(`${base}/summer_movies.csv`, row => ({
        ...row,
        year: new Date(Number(row.year), 0, 1),
        runtime_minutes: Number(row.runtime_minutes),
        average_rating: Number(row.average_rating),
        num_votes: Number(row.num_votes),
        genres: (row.genres ?? '').split(',').map(genre => genre.trim()).filter(Boolean)
      }));

      totalMovies = data.length;
      movies = (data as unknown as TMovie[]).filter(movie =>
        !Number.isNaN(movie.year.getTime()) &&
        Number.isFinite(movie.runtime_minutes) &&
        Number.isFinite(movie.average_rating) &&
        Number.isFinite(movie.num_votes)
      );
    } catch (error) {
      loadError = error instanceof Error ? error.message : 'Unable to load movie data.';
    }
  });

  function selectMovie(movie: TMovie) {
    selectedMovieId = movie.tconst;
  }

  function updateBrush(years: [number, number] | undefined) {
    brushedYears = years;
    if (years && selectedMovie) {
      const selectedYear = selectedMovie.year.getFullYear();
      if (selectedYear < years[0] || selectedYear > years[1]) selectedMovieId = undefined;
    }
  }
</script>

<svelte:head>
  <title>Summer Movies: Analysis</title>
</svelte:head>

<main>
  <h1>Summer Movies: Analysis</h1>
  <p class="intro">Here are {totalMovies} movies. Brush the timeline to filter the scatterplot.</p>
  {#if loadError}
    <p class="error">Unable to load movie data: {loadError}</p>
  {:else if movies.length === 0}
    <p>Loading movie data...</p>
  {:else}
    <section>
      <h2>Explore movie attributes</h2>
      <div class="controls">
        <label>X-axis
          <select bind:value={xAttribute}>
            {#each channelAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
        <label>Y-axis
          <select bind:value={yAttribute}>
            {#each channelAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
        <label>Point size
          <select bind:value={sizeAttribute}>
            {#each channelAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
      </div>
      <p class="filter-status">{visibleMovies.length} of {totalMovies} movies shown.</p>
      <div class="chart-with-details">
        <Scatter
          movies={visibleMovies}
          {selectedMovieId}
          {xAttribute}
          {yAttribute}
          {sizeAttribute}
          onSelect={selectMovie}
        />
        {#if selectedMovie}
          <aside class="selection">
            <strong>{selectedMovie.primary_title}</strong>
            <span>{selectedMovie.year.getFullYear()} · Rating {selectedMovie.average_rating.toFixed(1)} · {selectedMovie.runtime_minutes} minutes · {selectedMovie.num_votes.toLocaleString()} votes</span>
          </aside>
        {:else}
          <aside class="selection empty">Click on a point to see details</aside>
        {/if}
      </div>
    </section>

    <section class="timeline">
      <h2>Movie count over time</h2>
      <Line {movies} onBrushChange={updateBrush} />
      {#if brushedYears}
        <p class="filter-status">Showing movies from {brushedYears[0]} to {brushedYears[1]}.</p>
      {:else}
        <p class="filter-status">Showing all years. Drag across the timeline to filter.</p>
      {/if}
    </section>
  {/if}
</main>

<style>
  main { max-width: 820px; margin: 0 auto; padding: 2rem 1.5rem 4rem; font-family: Arial, Helvetica, sans-serif; }
  h1 { margin-bottom: 0.5rem; }
  h2 { margin: 2rem 0 0.5rem; font-size: 1.25rem; }
  .intro { color: #4b5563; }
  section { overflow-x: auto; }
  .controls { display: flex; flex-wrap: wrap; gap: 1rem; margin: 0.75rem 0; }
  label { display: flex; flex-direction: column; gap: 0.25rem; font-size: 0.9rem; font-weight: 600; }
  select { min-width: 9rem; padding: 0.35rem; font: inherit; font-weight: 400; }
  .filter-status { color: #4b5563; font-size: 0.9rem; }
  .error { color: #b91c1c; }
  .chart-with-details { display: flex; align-items: flex-start; gap: 1rem; }
  .timeline { margin-top: 0.5rem; }
  .selection { display: flex; flex-direction: column; gap: 0.35rem; width: 220px; padding: 0.75rem 1rem; border-left: 4px solid #4e79a7; background: #eff6ff; color: #1f2937; }
</style>