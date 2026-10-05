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

  // ---------------------------------------------------------------
  // Q1: Stacked bar chart of the top three genres per year.
  // ---------------------------------------------------------------

  // Store the genre currently being hovered over in the stacked chart.
  let selectedStackGenre: string | undefined =
    $state(undefined);

  type StackSegment = {
    genre: string;
    count: number;
    y0: number;
    y1: number;
  };

  type YearStack = {
    year: number;
    segments: StackSegment[];
    total: number;
  };

  // For every year, find the top three genres by movie count
  // (ties broken alphabetically) and stack them from the bottom up.
  const yearStacks: YearStack[] = $derived.by(() => {
    const byYear = new Map<number, Record<string, number>>();

    movies.forEach((movie) => {
      const y = movie.year.getFullYear();
      const counts = byYear.get(y) ?? {};
      movie.genres.forEach((genre: string) => {
        counts[genre] = (counts[genre] || 0) + 1;
      });
      byYear.set(y, counts);
    });

    return Array.from(byYear.entries())
      .sort((a, b) => a[0] - b[0])
      .map(([year, counts]) => {
        const top = Object.entries(counts)
          .sort(
            (a, b) =>
              b[1] - a[1] || a[0].localeCompare(b[0])
          )
          .slice(0, 3);

        let acc = 0;
        const segments = top.map(([genre, count]) => {
          const segment = {
            genre,
            count,
            y0: acc,
            y1: acc + count,
          };
          acc += count;
          return segment;
        });

        return { year, segments, total: acc };
      });
  });

  // Only draw bars for years up to the progress cutoff.
  const visibleStacks = $derived(
    yearStacks.filter(
      (d) => new Date(d.year, 0, 1) <= upYear
    )
  );

  // Every genre that appears in any year's top three.
  const stackGenres = $derived(
    Array.from(
      new Set(
        yearStacks.flatMap((d) =>
          d.segments.map((s) => s.genre)
        )
      )
    ).sort((a, b) => a.localeCompare(b))
  );

  // One color per genre, fixed across all years.
  const colorScale = $derived(
    d3
      .scaleOrdinal<string, string>()
      .domain(stackGenres)
      .range(d3.schemeTableau10)
  );

  // Stacked chart margins (the plot keeps the same size as the first chart).
  const stackMargin = {
    top: 35,
    bottom: 100,
    left: 40,
    right: 10,
  };

  const stackArea = $derived({
    top: stackMargin.top,
    right: width - stackMargin.right,
    bottom: height - stackMargin.bottom,
    left: stackMargin.left,
  });

  // X-axis: one band for each year (all years are kept).
  const stackXScale = $derived(
    d3
      .scaleBand<string>()
      .domain(yearStacks.map((d) => String(d.year)))
      .range([stackArea.left, stackArea.right])
      .padding(0.12)
  );

  // Y-axis: combined movie count, based on all years so it stays fixed.
  const stackYScale = $derived(
    d3
      .scaleLinear()
      .domain([
        0,
        Math.max(1, d3.max(yearStacks, (d) => d.total) ?? 0),
      ])
      .nice()
      .range([stackArea.bottom, stackArea.top])
  );

  // Legend layout: placed below the chart and wrapped into rows.
  const legendItemWidth = 140;
  const legendRowHeight = 22;
  const legendCols = $derived(
    Math.max(
      1,
      Math.floor(
        (width - stackMargin.left - stackMargin.right) /
          legendItemWidth
      )
    )
  );
  const legendTop = $derived(stackArea.bottom + 70);
  const stackSvgHeight = $derived(
    legendTop +
      Math.ceil(stackGenres.length / legendCols) *
        legendRowHeight +
      10
  );

  // DOM references for the stacked chart axes.
  let stackXAxis: SVGGElement;
  let stackYAxis: SVGGElement;

  function updateStackAxis() {
    // Keep every bar, but only label every 5th year.
    d3.select(stackXAxis)
      .call(
        d3
          .axisBottom(stackXScale)
          .tickValues(
            stackXScale
              .domain()
              .filter((year) => Number(year) % 5 === 0)
          )
      )
      .selectAll("text")
      .style("font-size", "12px");

    d3.select(stackYAxis).call(
      d3
        .axisLeft(stackYScale)
        .ticks(10)
        .tickFormat(d3.format("d"))
    );
  }

  // Redraw the stacked chart axes when their scales change.
  $effect(() => {
    updateStackAxis();
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

<h3 class="q1-title">
  Q1: How do the top three movie genres (by number of movies)
  change over time?
</h3>

{#if movies.length > 0}
  <svg {width} height={stackSvgHeight}>
    <!-- One stacked bar per year. -->
    <g class="stacked-bars">
      {#each visibleStacks as d (d.year)}
        <g class="year-stack">
          {#each d.segments as segment (segment.genre)}
            <rect
              class="bar"
              x={stackXScale(String(d.year))!}
              y={stackYScale(segment.y1)}
              width={stackXScale.bandwidth()}
              height={stackYScale(segment.y0) -
                stackYScale(segment.y1)}
              fill={colorScale(segment.genre)}
              opacity={selectedStackGenre === segment.genre
                ? 0.6
                : 1}
              onmouseenter={() => {
                selectedStackGenre = segment.genre;
              }}
              onmouseleave={() => {
                selectedStackGenre = undefined;
              }}
            />
          {/each}
        </g>
      {/each}
    </g>

    <!-- X-axis. -->
    <g
      transform="translate(0, {stackArea.bottom})"
      bind:this={stackXAxis}
    />

    <!-- Y-axis. -->
    <g
      transform="translate({stackArea.left}, 0)"
      bind:this={stackYAxis}
    />

    <!-- Legend below the chart. -->
    <g class="legend">
      {#each stackGenres as genre, i (genre)}
        <g
          transform="translate({stackMargin.left +
            (i % legendCols) * legendItemWidth}, {legendTop +
            Math.floor(i / legendCols) * legendRowHeight})"
        >
          <rect
            width="14"
            height="14"
            fill={colorScale(genre)}
            opacity={selectedStackGenre === genre ? 0.6 : 1}
          />
          <text
            x="20"
            y="12"
            font-size="12"
            font-weight={selectedStackGenre === genre
              ? "bold"
              : "normal"}
          >
            {genre}
          </text>
        </g>
      {/each}
    </g>
  </svg>
{/if}

<style>
  h3 {
    font-family: Arial, Helvetica, sans-serif;
    font-size: 1.5rem;
    margin-bottom: 25px;
  }

  .q1-title {
    margin-top: 40px;
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