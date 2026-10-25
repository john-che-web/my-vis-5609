<script lang="ts">
  import * as d3 from 'd3';
  import type { TMovie } from '../types';

  export type ChannelAttribute =
    | 'year'
    | 'runtime_minutes'
    | 'average_rating'
    | 'num_votes'
    | 'title_type'
    | 'primary_title'
    | 'original_title'
    | 'simple_title'
    | 'tconst';

  type Props = {
    movies: TMovie[];
    selectedMovieId?: string;
    xAttribute?: ChannelAttribute;
    yAttribute?: ChannelAttribute;
    sizeAttribute?: ChannelAttribute;
    onSelect?: (movie: TMovie) => void;
    width?: number;
    height?: number;
  };

  let {
    movies,
    selectedMovieId,
    xAttribute = 'year',
    yAttribute = 'simple_title',
    sizeAttribute = 'year',
    onSelect,
    width = 700,
    height = 420
  }: Props = $props();

  const numericAttributes: ChannelAttribute[] = ['year', 'runtime_minutes', 'average_rating', 'num_votes'];
  const margin = { top: 24, right: 24, bottom: 48, left: 58 };
  const chartWidth = width - margin.left - margin.right;
  const chartHeight = height - margin.top - margin.bottom;

  function channelValue(movie: TMovie, attribute: ChannelAttribute): number | string | null {
    if (attribute === 'year') return movie.year && !Number.isNaN(movie.year.getTime()) ? movie.year.getFullYear() : null;
    if (attribute === 'runtime_minutes') return movie.runtime_minutes;
    if (attribute === 'average_rating') return movie.average_rating;
    if (attribute === 'num_votes') return movie.num_votes;
    return movie[attribute];
  }

  function isNumeric(attribute: ChannelAttribute): boolean {
    return numericAttributes.includes(attribute);
  }

  function numericDomain(values: number[]): [number, number] {
    const extent = d3.extent(values);
    if (extent[0] === undefined || extent[1] === undefined) return [0, 1];
    return extent;
  }

  const xNumericValues = $derived(movies.map(movie => channelValue(movie, xAttribute)).filter((value): value is number => typeof value === 'number' && Number.isFinite(value)));
  const yNumericValues = $derived(movies.map(movie => channelValue(movie, yAttribute)).filter((value): value is number => typeof value === 'number' && Number.isFinite(value)));
  const sizeNumericValues = $derived(movies.map(movie => channelValue(movie, sizeAttribute)).filter((value): value is number => typeof value === 'number' && Number.isFinite(value)));
  const xNumericScale = $derived(d3.scaleLinear().domain(numericDomain(xNumericValues)).nice().range([0, chartWidth]));
  const yNumericScale = $derived(d3.scaleLinear().domain(yAttribute === 'average_rating' ? [0, 10] : numericDomain(yNumericValues)).nice().range([chartHeight, 0]));
  const sizeNumericScale = $derived(d3.scaleSqrt().domain(numericDomain(sizeNumericValues)).range([3, 14]));
  const xCategoricalScale = $derived(d3.scalePoint<string>().domain([...new Set(movies.map(movie => channelValue(movie, xAttribute)).filter(value => value !== null).map(String))]).range([0, chartWidth]).padding(0.5));
  const yCategoricalScale = $derived(d3.scalePoint<string>().domain([...new Set(movies.map(movie => channelValue(movie, yAttribute)).filter(value => value !== null).map(String))]).range([chartHeight, 0]).padding(0.5));
  const xTicks = $derived(xNumericScale.ticks(8));
  const yTicks = $derived(yNumericScale.ticks(5));
  const xCategories = $derived(xCategoricalScale.domain());
  const yCategories = $derived(yCategoricalScale.domain());

  function xPosition(movie: TMovie): number {
    const value = channelValue(movie, xAttribute);
    return isNumeric(xAttribute) ? xNumericScale(Number(value)) : xCategoricalScale(String(value)) ?? 0;
  }

  function yPosition(movie: TMovie): number {
    const value = channelValue(movie, yAttribute);
    return isNumeric(yAttribute) ? yNumericScale(Number(value)) : yCategoricalScale(String(value)) ?? 0;
  }

  function pointSize(movie: TMovie): number {
    const value = channelValue(movie, sizeAttribute);
    if (isNumeric(sizeAttribute) && typeof value === 'number') return sizeNumericScale(value);
    return 8;
  }

  function channelLabel(attribute: ChannelAttribute): string {
    return {
      year: 'Year',
      runtime_minutes: 'Runtime (minutes)',
      average_rating: 'Average rating',
      num_votes: 'Number of votes',
      title_type: 'Title type',
      primary_title: 'Primary title',
      original_title: 'Original title',
      simple_title: 'Simple title',
      tconst: 'Movie ID'
    }[attribute];
  }
</script>

<svg {width} {height} role="img" aria-label="Configurable movie scatterplot">
  <g transform={`translate(${margin.left},${margin.top})`}>
    {#if isNumeric(yAttribute)}
      {#each yTicks as tick}
        <line x1="0" x2={chartWidth} y1={yNumericScale(tick)} y2={yNumericScale(tick)} stroke="#e5e7eb" />
        <text x={-10} y={yNumericScale(tick) + 4} text-anchor="end" font-size="11">{tick}</text>
      {/each}
    {:else}
      {#each yCategories as category}
        <text x={-10} y={yCategoricalScale(category)} text-anchor="end" dominant-baseline="middle" font-size="9">{category}</text>
      {/each}
    {/if}
    {#if isNumeric(xAttribute)}
      {#each xTicks as tick}
        <text x={xNumericScale(tick)} y={chartHeight + 24} text-anchor="middle" font-size="11">{tick}</text>
      {/each}
    {:else}
      {#each xCategories as category}
        <text x={xCategoricalScale(category)} y={chartHeight + 24} text-anchor="middle" font-size="9" transform={`rotate(45,${xCategoricalScale(category)},${chartHeight + 24})`}>{category}</text>
      {/each}
    {/if}
    {#each movies as movie (movie.tconst)}
      {#if channelValue(movie, xAttribute) !== null && channelValue(movie, yAttribute) !== null}
        <circle
          cx={xPosition(movie)}
          cy={yPosition(movie)}
          r={selectedMovieId === movie.tconst ? Math.max(8, pointSize(movie) + 3) : pointSize(movie)}
          fill={selectedMovieId === movie.tconst ? '#4e79a7' : 'none'}
          fill-opacity={selectedMovieId === movie.tconst ? 0.9 : 0}
          stroke={selectedMovieId === movie.tconst ? '#1f2937' : '#76a7d3'}
          stroke-width="2"
          opacity="0.75"
          role="button"
          tabindex="0"
          aria-label={`Select ${movie.primary_title}`}
          onclick={() => onSelect?.(movie)}
          onkeydown={(event) => {
            if (event.key === 'Enter' || event.key === ' ') onSelect?.(movie);
          }}
        >
          <title>{movie.primary_title} ({movie.year ? movie.year.getFullYear() : 'Unknown year'})</title>
        </circle>
      {/if}
    {/each}
    <line x1="0" x2={chartWidth} y1={chartHeight} y2={chartHeight} stroke="#374151" />
    <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
    <text x={chartWidth / 2} y={chartHeight + 42} text-anchor="middle" font-size="12">{channelLabel(xAttribute)}</text>
    <text transform={`translate(${-44},${chartHeight / 2}) rotate(-90)`} text-anchor="middle" font-size="12">{channelLabel(yAttribute)}</text>
  </g>
</svg>