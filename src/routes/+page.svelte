<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import { base } from "$app/paths";

  import Bar from "$lib/Bar.svelte";
  import Q1 from "$lib/Q1.svelte";
  import Q2 from "$lib/Q2.svelte";

  let movies: any[] = [];
  let loading = true;

  onMount(async () => {
    try {
      const data = await d3.csv(`${base}/summer_movies.csv`, d => ({
        ...d,
        year: +d.year!,
        runtime_minutes: +d.runtime_minutes!,
        average_rating: +d.average_rating!,
        num_votes: +d.num_votes!
      }));

      movies = data;
      loading = false;

    } catch (error) {
      console.error("CSV load error:", error);
    }
  });
</script>

<h1>A1: Summer Movies Dataset</h1>

{#if loading}
  <p>Loading data...</p>
{:else}
  <p>Loaded {movies.length} movies.</p>

  <Bar {movies} />
  <Q1 {movies} />
  <Q2 {movies} />
{/if}
