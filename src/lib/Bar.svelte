<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";

  // Define the component props.
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };

  let {
    movies,
    progress = 100,
    width = 700,
    height = 550,
  }: Props = $props();

  // Store the genre currently being hovered over.
  let selectedGenre: string | undefined = $state(undefined);

  // Find the earliest and latest movie years.
  const yearRange = $derived(
    d3.extent(movies, (d) => d.year)
  );

  // Determine the latest year included in the chart.
  function getUpYear(
    range: [Date | undefined, Date | undefined]
  ): Date {
    if (!range[0] || !range[1]) {
      return new Date();
    }

    const timeScale = d3
      .scaleTime()
      .domain([range[0], range[1]])
      .range([0, 100]);

    return timeScale.invert(
      Math.max(0, Math.min(100, progress))
    );
  }

  const upYear = $derived(getUpYear(yearRange));

  // Count movies belonging to each genre up to the selected year.
  function getGenreNums(
    movies: TMovie[],
    upYear: Date
  ): Record<string, number> {
    const result: Record<string, number> = {};

    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        movie.genres.forEach((genre: string) => {
          result[genre] = (result[genre] || 0) + 1;
        });
      });

    return result;
  }

  const genreNums = $derived(
    getGenreNums(movies, upYear)
  );

  // Chart margins.
  const margin = {
    top: 35,
    bottom: 100,
    left: 40,
    right: 10,
  };

  // Calculate the available drawing area.
  const usableArea = $derived({
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  });

  // X-axis: one band for each genre.
  const xScale = $derived(
    d3
      .scaleBand<string>()
      .domain(Object.keys(genreNums))
      .range([
        usableArea.left,
        usableArea.right,
      ])
      .padding(0.12)
  );

  // Y-axis: movie counts.
  const yScale = $derived(
    d3
      .scaleLinear()
      .domain([
        0,
        Math.max(
          1,
          d3.max(Object.values(genreNums)) ?? 0
        ),
      ])
      .nice()
      .range([
        usableArea.bottom,
        usableArea.top,
      ])
  );

  // Width of each bar.
  const xBarwidth: number = $derived(
    xScale.bandwidth()
  );

  // DOM references for the axes.
  let xAxis: SVGGElement;
  let yAxis: SVGGElement;

  // Draw the axes using D3.
  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale))
      .selectAll("text")
      .attr("transform", "rotate(45)")
      .attr("dx", "0.5em")
      .attr("dy", "0.5em")
      .style("text-anchor", "start")
      .style("font-size", "12px");

    d3.select(yAxis).call(
      d3
        .axisLeft(yScale)
        .ticks(10)
        .tickFormat(d3.format("d"))
    );
  }

  // Redraw the axes when their scales change.
  $effect(() => {
    updateAxis();
  });
</script>

<h3>
  The Distribution of Genres
  {yearRange[0]?.getFullYear()}
  -
  {yearRange[1]?.getFullYear()}
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <!-- Draw one group for each genre. -->
    <g class="bars">
      {#each Object.entries(genreNums) as [genre, cnt] (genre)}
        <g class="genre-bar">
          <!-- Draw the bar. -->
          <rect
            class="bar"
            x={xScale(genre)!}
            y={yScale(cnt)}
            width={xBarwidth}
            height={usableArea.bottom - yScale(cnt)}
            fill="#9BCB7A"
            opacity={selectedGenre === genre ? 0.6 : 1}
            onmouseenter={() => {
              selectedGenre = genre;
            }}
            onmouseleave={() => {
              selectedGenre = undefined;
            }}
          />

          <!-- Display the count above the bar. -->
          <text
            class="bar-label"
            x={xScale(genre)! + xBarwidth / 2}
            y={yScale(cnt) - 5}
            font-size="12"
            text-anchor="middle"
          >
            {cnt}
          </text>
        </g>
      {/each}
    </g>

    <!-- X-axis. -->
    <g
      transform="translate(0, {usableArea.bottom})"
      bind:this={xAxis}
    />

    <!-- Y-axis. -->
    <g
      transform="translate({usableArea.left}, 0)"
      bind:this={yAxis}
    />
  </svg>
{/if}

<style>
  h3 {
    font-family: Arial, Helvetica, sans-serif;
    font-size: 1.5rem;
    margin-bottom: 25px;
  }

  svg {
    overflow: visible;
    font-family: Arial, Helvetica, sans-serif;
  }

  .bar {
    transition:
      opacity 0.15s ease,
      y 0.1s ease,
      height 0.1s ease;
  }

  .bar-label {
    fill: #111;
  }
</style>