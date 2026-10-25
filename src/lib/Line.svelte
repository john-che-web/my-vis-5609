<script lang="ts">
  import * as d3 from 'd3';
  import type { TMovie } from '../types';

  type Props = {
    movies: TMovie[];
    onBrushChange?: (years: [number, number] | undefined) => void;
    width?: number;
    height?: number;
  };

  let { movies, onBrushChange, width = 700, height = 320 }: Props = $props();
  let brushLayer: SVGGElement;

  const margin = { top: 24, right: 24, bottom: 42, left: 52 };
  const chartWidth = width - margin.left - margin.right;
  const chartHeight = height - margin.top - margin.bottom;
  const yearlyRatings = $derived.by(() => {
    const grouped = d3.rollups(
      movies,
      values => d3.mean(values, movie => movie.average_rating) ?? 0,
      movie => movie.year.getFullYear()
    );
    return grouped.map(([year, rating]) => ({ year, rating })).sort((a, b) => a.year - b.year);
  });
  const yearExtent = $derived(d3.extent(yearlyRatings, d => d.year));
  const xScale = $derived(
    d3.scaleLinear().domain(yearExtent[0] === undefined || yearExtent[1] === undefined ? [0, 1] : yearExtent as [number, number]).range([0, chartWidth])
  );
  const yScale = $derived(d3.scaleLinear().domain([0, 10]).range([chartHeight, 0]));
  const linePath = $derived(
    d3.line<{ year: number; rating: number }>().x(d => xScale(d.year)).y(d => yScale(d.rating))(yearlyRatings) ?? ''
  );
  const xTicks = $derived(xScale.ticks(8));
  const yTicks = $derived(yScale.ticks(5));

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

<svg {width} {height} role="img" aria-label="Average movie rating by year with a brush selection">
  <g transform={`translate(${margin.left},${margin.top})`}>
    {#each yTicks as tick}
      <line x1="0" x2={chartWidth} y1={yScale(tick)} y2={yScale(tick)} stroke="#e5e7eb" />
      <text x={-10} y={yScale(tick) + 4} text-anchor="end" font-size="11">{tick}</text>
    {/each}
    {#each xTicks as tick}
      <text x={xScale(tick)} y={chartHeight + 24} text-anchor="middle" font-size="11">{tick}</text>
    {/each}
    <path d={linePath} fill="none" stroke="#4e79a7" stroke-width="2.5" />
    {#each yearlyRatings as point (point.year)}
      <circle cx={xScale(point.year)} cy={yScale(point.rating)} r="3" fill="#4e79a7">
        <title>{point.year}: {point.rating.toFixed(2)}</title>
      </circle>
    {/each}
    <line x1="0" x2={chartWidth} y1={chartHeight} y2={chartHeight} stroke="#374151" />
    <line x1="0" x2="0" y1="0" y2={chartHeight} stroke="#374151" />
    <g bind:this={brushLayer} aria-label="Year brush selection"></g>
    <text x={chartWidth / 2} y={chartHeight + 40} text-anchor="middle" font-size="12">Year</text>
    <text transform={`translate(${-40},${chartHeight / 2}) rotate(-90)`} text-anchor="middle" font-size="12">Average rating</text>
  </g>
</svg>