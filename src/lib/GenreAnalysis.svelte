<script lang="ts">
  import * as d3 from 'd3';
  import type { TMovie } from '../types';

  type GenreStat = { genre: string; count: number; average: number };
  type Props = { movies: TMovie[]; width?: number; height?: number };
  let { movies, width = 920, height = 430 }: Props = $props();
  let selectedGenre = $state('Drama');

  const genreStats = $derived.by(() => {
    const groups = new Map<string, { count: number; ratings: number[] }>();
    movies.forEach(movie => {
      if (movie.average_rating === null) return;
      movie.genres.forEach(genre => {
        const group = groups.get(genre) ?? { count: 0, ratings: [] };
        group.count += 1;
        group.ratings.push(movie.average_rating as number);
        groups.set(genre, group);
      });
    });
    return [...groups.entries()]
      .map(([genre, group]) => ({ genre, count: group.count, average: d3.mean(group.ratings) ?? 0 }))
      .sort((a, b) => b.average - a.average);
  });

  const selectedStat = $derived(genreStats.find(stat => stat.genre === selectedGenre));
  const selectedMovies = $derived(
    movies.filter(movie => movie.genres.includes(selectedGenre) && movie.year !== null && movie.average_rating !== null)
  );
  const margin = { top: 30, right: 24, bottom: 54, left: 100 };
  const gap = 70;
  const viewWidth = (width - gap) / 2;
  const chartHeight = height - margin.top - margin.bottom;
  const barX = $derived(d3.scaleLinear().domain([0, 10]).range([0, viewWidth - margin.left - margin.right]));
  const barY = $derived(d3.scaleBand<string>().domain(genreStats.map(stat => stat.genre)).range([0, chartHeight]).padding(0.18));
  const selectedYearExtent = $derived(d3.extent(selectedMovies, movie => movie.year!.getFullYear()));
  const pointX = $derived(d3.scaleLinear().domain(selectedYearExtent[0] === undefined || selectedYearExtent[1] === undefined ? [0, 1] : selectedYearExtent as [number, number]).range([0, viewWidth - margin.left - margin.right]));
  const pointY = $derived(d3.scaleLinear().domain([0, 10]).range([chartHeight, 0]));
  const yearTicks = $derived(pointX.ticks(5));
  const ratingTicks = $derived(pointY.ticks(5));
</script>

<div class="analysis" style={`--chart-width: ${width}px; --title-width: ${2 * (viewWidth - margin.left - margin.right) + gap}px; --title-left: ${margin.left}px; --title-gap: ${gap}px`}>
  <div class="title-row">
    <div class="view-title">Average rating by genre</div>
    <div class="view-title">Selected genre over time</div>
  </div>
  <svg {width} {height} role="img" aria-label="Linked genre rating analysis">
    <g transform={`translate(${margin.left},${margin.top})`}>
      {#each genreStats as stat (stat.genre)}
        <rect
          x="0"
          y={barY(stat.genre)!}
          width={barX(stat.average)}
          height={barY.bandwidth()}
          fill={selectedGenre === stat.genre ? '#4e79a7' : '#a9c7e3'}
          role="button"
          tabindex="0"
          aria-label={`Select ${stat.genre}`}
          onclick={() => selectedGenre = stat.genre}
          onkeydown={(event) => {
            if (event.key === 'Enter' || event.key === ' ') selectedGenre = stat.genre;
          }}
        />
        <text x={-8} y={barY(stat.genre)! + barY.bandwidth() / 2} text-anchor="end" dominant-baseline="middle" font-size="11">{stat.genre}</text>
        <text x={barX(stat.average) + 5} y={barY(stat.genre)! + barY.bandwidth() / 2} dominant-baseline="middle" font-size="10">{stat.average.toFixed(2)}</text>
      {/each}
      <line x1="0" x2={viewWidth - margin.left - margin.right} y1={chartHeight} y2={chartHeight} stroke="#374151" />
      <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
      <text x={(viewWidth - margin.left - margin.right) / 2} y={chartHeight + 38} text-anchor="middle" font-size="11">Average rating</text>
    </g>

    <g transform={`translate(${viewWidth + gap + margin.left},${margin.top})`}>
      {#each ratingTicks as tick}
        <line x1="0" x2={viewWidth - margin.left - margin.right} y1={pointY(tick)} y2={pointY(tick)} stroke="#e5e7eb" />
        <text x={-8} y={pointY(tick) + 4} text-anchor="end" font-size="10">{tick}</text>
      {/each}
      {#each yearTicks as tick}
        <text x={pointX(tick)} y={chartHeight + 20} text-anchor="middle" font-size="10">{tick}</text>
      {/each}
      {#each selectedMovies as movie (movie.tconst)}
        <circle cx={pointX(movie.year!.getFullYear())} cy={pointY(movie.average_rating!)} r={movie.num_votes === null ? 4 : Math.max(3, Math.min(12, Math.sqrt(movie.num_votes) / 25))} fill="#4e79a7" fill-opacity="0.55">
          <title>{movie.primary_title}: {movie.average_rating}</title>
        </circle>
      {/each}
      <line x1="0" x2={viewWidth - margin.left - margin.right} y1={chartHeight} y2={chartHeight} stroke="#374151" />
      <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
      <text x={(viewWidth - margin.left - margin.right) / 2} y={chartHeight + 38} text-anchor="middle" font-size="11">Year</text>
      <text transform={`translate(${-42},${chartHeight / 2}) rotate(-90)`} text-anchor="middle" font-size="11">Average rating</text>
    </g>
  </svg>
  {#if selectedStat}
    <p class="selection-note"><strong>{selectedGenre}</strong>: {selectedStat.count} movies, average rating {selectedStat.average.toFixed(2)}. Click a genre bar to update the time view.</p>
  {/if}
</div>

<style>
  .analysis { max-width: var(--chart-width); overflow-x: auto; font-family: Arial, Helvetica, sans-serif; }
  .view-title { display: inline-block; width: calc(50% - 2rem); margin-left: 100px; font-weight: 700; }
  .right-title { margin-left: 0; }
  svg { display: block; overflow: visible; }
  .selection-note { margin: 0.25rem 0 0; color: #4b5563; font-size: 0.9rem; }
</style>