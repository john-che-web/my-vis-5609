<script lang="ts">
  import * as d3 from 'd3';
  import type { TMovie } from '../types';

  type Props = {
    movies: TMovie[];
    onBrushChange?: (years: [number, number] | undefined) => void;
    width?: number;
    height?: number;
  };

  let { movies, onBrushChange, width = 700, height = 180 }: Props = $props();
  let brushLayer: SVGGElement;

  const margin = { top: 12, right: 24, bottom: 34, left: 52 };
  const chartWidth = width - margin.left - margin.right;
  const chartHeight = height - margin.top - margin.bottom;
  const yearlyCounts = $derived.by(() => {
    const grouped = d3.rollups(
      movies.filter(movie => movie.year !== null && !Number.isNaN(movie.year.getTime())),
      values => values.length,
      movie => movie.year!.getFullYear()
    );
    return grouped.map(([year, count]) => ({ year, count })).sort((a, b) => a.year - b.year);
  });
  const yearExtent = $derived(d3.extent(yearlyCounts, d => d.year));
  const maxCount = $derived(d3.max(yearlyCounts, d => d.count) ?? 1);
  const xScale = $derived(
    d3.scaleLinear().domain(yearExtent[0] === undefined || yearExtent[1] === undefined ? [0, 1] : yearExtent as [number, number]).range([0, chartWidth])
  );
  const yScale = $derived(d3.scaleLinear().domain([0, maxCount]).nice().range([chartHeight, 0]));
  const xTicks = $derived(xScale.ticks(8));
  const yTicks = $derived(yScale.ticks(4));

  $effect(() => {
    if (!brushLayer) return;
    const brush = d3
      .brushX()
      .extent([[0, 0], [chartWidth, chartHeight]])
      .on('end', event => {
        if (!event.selection) {
          onBrushChange?.(undefined);
          return;
        }
        const selection = event.selection as [number, number];
        const startYear = Math.round(xScale.invert(selection[0]));
        const endYear = Math.round(xScale.invert(selection[1]));
        onBrushChange?.([Math.min(startYear, endYear), Math.max(startYear, endYear)]);
      });

    d3.select(brushLayer).call(brush);
  });
</script>

<svg {width} {height} role="img" aria-label="Number of movies by year with a brush selection">
  <g transform={`translate(${margin.left},${margin.top})`}>
    {#each yTicks as tick}
      <line x1="0" x2={chartWidth} y1={yScale(tick)} y2={yScale(tick)} stroke="#e5e7eb" />
      <text x={-10} y={yScale(tick) + 4} text-anchor="end" font-size="10">{tick}</text>
    {/each}
    {#each xTicks as tick}
      <text x={xScale(tick)} y={chartHeight + 18} text-anchor="middle" font-size="10">{tick}</text>
    {/each}
    {#each yearlyCounts as point (point.year)}
      <circle cx={xScale(point.year)} cy={yScale(point.count)} r="3" fill="#222">
        <title>{point.year}: {point.count} movies</title>
      </circle>
    {/each}
    <line x1="0" x2={chartWidth} y1={chartHeight} y2={chartHeight} stroke="#374151" />
    <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
    <g bind:this={brushLayer} aria-label="Year brush selection"></g>
    <text x={chartWidth / 2} y={chartHeight + 32} text-anchor="middle" font-size="11">Number of Movies by Year</text>
  </g>
</svg>