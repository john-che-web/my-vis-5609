<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte";

  // Store the movies loaded from the CSV file.
  let movies: TMovie[] = $state([]);

  // Load and convert the CSV data.
  async function loadCsv() {
    try {
      const csvUrl = "/summer_movies.csv";

      const data = await d3.csv(csvUrl, (row) => {
        return {
          ...row,

          // Convert the year into a Date object.
          year: new Date(Number(row.year), 0, 1),

          // Convert the genres into an array.
          genres: (row.genres ?? "")
            .split(",")
            .map((genre) => genre.trim())
            .filter((genre) => genre.length > 0),
        };
      });

      // Assign the converted data to the reactive variable.
      movies = data as unknown as TMovie[];

      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }

  // Load the data when the page mounts.
  onMount(() => {
    loadCsv();
  });
</script>

<svelte:head>
  <title>Summer Movies</title>
</svelte:head>

<main>
  <h1>Summer Movies</h1>

  <p>
    Here are {movies.length === 0 ? "..." : movies.length} movies
  </p>

  {#if movies.length > 0}
    <Bar {movies} width={700} height={550} />
  {/if}
</main>

<style>
  main {
    margin: 0 auto;
    max-width: 900px;
    padding: 0 20px;
    font-family: Arial, Helvetica, sans-serif;
  }

  h1 {
    font-size: 2.5rem;
    margin-bottom: 25px;
  }

  p {
    font-size: 1.25rem;
    margin-bottom: 25px;
  }
</style>