<script lang="ts">
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import { Line, Scatter } from '$lib';
  import type { TMovie } from '../../types';
  import type { NumericAttribute } from '$lib/Scatter.svelte';

  const numericAttributes: { value: NumericAttribute; label: string }[] = [
    { value: 'year', label: 'Year' },
    { value: 'runtime_minutes', label: 'Runtime' },
    { value: 'average_rating', label: 'Average rating' },
    { value: 'num_votes', label: 'Number of votes' }
  ];

  let movies: TMovie[] = $state([]);
  let selectedMovieId: string | undefined = $state(undefined);
  let selectedMovie = $derived(movies.find(movie => movie.tconst === selectedMovieId));
  let brushedYears: [number, number] | undefined = $state(undefined);
  let xAttribute: NumericAttribute = $state('runtime_minutes');
  let yAttribute: NumericAttribute = $state('average_rating');
  let sizeAttribute: NumericAttribute = $state('num_votes');
  let visibleMovies = $derived.by(() => {
    const yearRange = brushedYears;
    if (!yearRange) return movies;
    return movies.filter(movie => {
      const year = movie.year.getFullYear();
      return year >= yearRange[0] && year <= yearRange[1];
    });
  });

  onMount(async () => {
    const data = await d3.csv(`${base}/summer_movies.csv`, row => ({
      ...row,
      year: new Date(Number(row.year), 0, 1),
      runtime_minutes: Number(row.runtime_minutes),
      average_rating: Number(row.average_rating),
      num_votes: Number(row.num_votes),
      genres: (row.genres ?? '').split(',').map(genre => genre.trim()).filter(Boolean)
    }));
    movies = data as unknown as TMovie[];
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
  <p class="intro">Brush the timeline to filter movies, then compare numeric attributes in the scatterplot.</p>
  {#if movies.length === 0}
    <p>Loading movie data...</p>
  {:else}
    <section>
      <h2>Average rating over time</h2>
      <Line {movies} onBrushChange={updateBrush} />
      {#if brushedYears}
        <p class="filter-status">Showing movies from {brushedYears[0]} to {brushedYears[1]}.</p>
      {:else}
        <p class="filter-status">Showing all years. Drag across the line chart to filter.</p>
      {/if}
    </section>

    <section>
      <h2>Explore movie attributes</h2>
      <div class="controls">
        <label>X-axis
          <select bind:value={xAttribute}>
            {#each numericAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
        <label>Y-axis
          <select bind:value={yAttribute}>
            {#each numericAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
        <label>Point size
          <select bind:value={sizeAttribute}>
            {#each numericAttributes as attribute}
              <option value={attribute.value}>{attribute.label}</option>
            {/each}
          </select>
        </label>
      </div>
      <p class="filter-status">{visibleMovies.length} of {movies.length} movies shown.</p>
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
  .selection { display: flex; flex-direction: column; gap: 0.35rem; max-width: 700px; padding: 0.75rem 1rem; border-left: 4px solid #4e79a7; background: #eff6ff; color: #1f2937; }
</style>