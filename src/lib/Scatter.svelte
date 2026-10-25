<script lang="ts">
  import * as d3 from 'd3';
  import type { TMovie } from '../types';

  export type NumericAttribute = 'year' | 'runtime_minutes' | 'average_rating' | 'num_votes';
  type Props = {
    movies: TMovie[];
    selectedMovieId?: string;
    xAttribute?: NumericAttribute;
    yAttribute?: NumericAttribute;
    sizeAttribute?: NumericAttribute;
    onSelect?: (movie: TMovie) => void;
    width?: number;
    height?: number;
  };

  let {
    movies,
    selectedMovieId,
    xAttribute = 'runtime_minutes',
    yAttribute = 'average_rating',
    sizeAttribute = 'num_votes',
    onSelect,
    width = 700,
    height = 420
  }: Props = $props();

  const margin = { top: 24, right: 24, bottom: 48, left: 58 };
  const chartWidth = width - margin.left - margin.right;
  const chartHeight = height - margin.top - margin.bottom;

  function valueFor(movie: TMovie, attribute: NumericAttribute): number {
    if (attribute === 'year') return movie.year.getFullYear();
    return movie[attribute];
  }

  const xScale = $derived(
    d3.scaleLinear().domain(d3.extent(movies, movie => valueFor(movie, xAttribute)) as [number, number]).nice().range([0, chartWidth])
  );
  const yScale = $derived(
    d3.scaleLinear().domain(yAttribute === 'average_rating' ? [0, 10] : d3.extent(movies, movie => valueFor(movie, yAttribute)) as [number, number]).nice().range([chartHeight, 0])
  );
  const sizeScale = $derived(
    d3.scaleSqrt().domain(d3.extent(movies, movie => valueFor(movie, sizeAttribute)) as [number, number]).range([3, 14])
  );
  const xTicks = $derived(xScale.ticks(8));
  const yTicks = $derived(yScale.ticks(5));

  function channelLabel(attribute: NumericAttribute): string {
    return {
      year: 'Year',
      runtime_minutes: 'Runtime (minutes)',
      average_rating: 'Average rating',
      num_votes: 'Number of votes'
    }[attribute];
  }
</script>

<svg {width} {height} role="img" aria-label="Configurable movie scatterplot">
  <g transform={`translate(${margin.left},${margin.top})`}>
    {#each yTicks as tick}
      <line x1="0" x2={chartWidth} y1={yScale(tick)} y2={yScale(tick)} stroke="#e5e7eb" />
      <text x={-10} y={yScale(tick) + 4} text-anchor="end" font-size="11">{tick}</text>
    {/each}
    {#each xTicks as tick}
      <text x={xScale(tick)} y={chartHeight + 24} text-anchor="middle" font-size="11">{tick}</text>
    {/each}
    {#each movies as movie (movie.tconst)}
      <circle
        cx={xScale(valueFor(movie, xAttribute))}
        cy={yScale(valueFor(movie, yAttribute))}
        r={selectedMovieId === movie.tconst ? Math.max(8, sizeScale(valueFor(movie, sizeAttribute)) + 3) : sizeScale(valueFor(movie, sizeAttribute))}
        fill={selectedMovieId === movie.tconst ? '#4e79a7' : '#f28e2c'}
        fill-opacity={selectedMovieId === movie.tconst ? 0.9 : 0.5}
        stroke={selectedMovieId === movie.tconst ? '#1f2937' : 'none'}
        stroke-width="2"
        role="button"
        tabindex="0"
        aria-label={`Select ${movie.primary_title}`}
        onclick={() => onSelect?.(movie)}
        onkeydown={(event) => {
          if (event.key === 'Enter' || event.key === ' ') onSelect?.(movie);
        }}
      >
        <title>{movie.primary_title} ({movie.year.getFullYear()})</title>
      </circle>
    {/each}
    <line x1="0" x2={chartWidth} y1={chartHeight} y2={chartHeight} stroke="#374151" />
    <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
    <text x={chartWidth / 2} y={chartHeight + 42} text-anchor="middle" font-size="12">{channelLabel(xAttribute)}</text>
    <text transform={`translate(${-44},${chartHeight / 2}) rotate(-90)`} text-anchor="middle" font-size="12">{channelLabel(yAttribute)}</text>
  </g>
</svg>