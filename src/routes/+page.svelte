<script>
  import { onMount } from 'svelte';
  import { base } from '$app/paths';
  import * as d3 from 'd3';

  let data = [];

  onMount(async () => {
    const response = await fetch(`${base}/summer_movies.csv`);
    const text = await response.text();
    data = d3.csvParse(text, d => ({
      title: d.title,
      year: +d.year,
      rating: +d.rating,
      votes: +d.votes,
      runtime: +d.runtime,
      genre: d.genre
    }));

    drawCharts();
  });

  function drawCharts() {
    if (!data.length) return;

    // Clear old charts
    d3.selectAll("svg").remove();

    drawMoviesPerYear();
    drawAverageRating();
    drawRuntimeVsRating();
    drawTopGenres();
    drawRatingDistribution();
    drawTopMovies();
  }

  function createSVG(id) {
    return d3.select(id)
      .append("svg")
      .attr("width", 350)
      .attr("height", 250);
  }

  function drawMoviesPerYear() {
    const svg = createSVG("#chart1");

    const counts = d3.rollups(data, v => v.length, d => d.year)
      .sort((a,b) => a[0] - b[0]);

    const x = d3.scaleBand()
      .domain(counts.map(d => d[0]))
      .range([40, 330])
      .padding(0.2);

    const y = d3.scaleLinear()
      .domain([0, d3.max(counts, d => d[1])])
      .range([220, 20]);

    svg.selectAll("rect")
      .data(counts)
      .enter()
      .append("rect")
      .attr("x", d => x(d[0]))
      .attr("y", d => y(d[1]))
      .attr("width", x.bandwidth())
      .attr("height", d => 220 - y(d[1]))
      .attr("fill", "steelblue");
  }

  function drawAverageRating() {
    const svg = createSVG("#chart2");

    const avg = d3.rollups(data,
      v => d3.mean(v, d => d.rating),
      d => d.year
    ).sort((a,b) => a[0] - b[0]);

    const x = d3.scaleLinear()
      .domain(d3.extent(avg, d => d[0]))
      .range([40, 330]);

    const y = d3.scaleLinear()
      .domain([0, 10])
      .range([220, 20]);

    const line = d3.line()
      .x(d => x(d[0]))
      .y(d => y(d[1]));

    svg.append("path")
      .datum(avg)
      .attr("fill", "none")
      .attr("stroke", "green")
      .attr("stroke-width", 2)
      .attr("d", line);
  }

  function drawRuntimeVsRating() {
    const svg = createSVG("#chart3");

    const x = d3.scaleLinear()
      .domain(d3.extent(data, d => d.runtime))
      .range([40, 330]);

    const y = d3.scaleLinear()
      .domain([0, 10])
      .range([220, 20]);

    svg.selectAll("circle")
      .data(data)
      .enter()
      .append("circle")
      .attr("cx", d => x(d.runtime))
      .attr("cy", d => y(d.rating))
      .attr("r", 3)
      .attr("fill", "purple")
      .attr("opacity", 0.6);
  }

  function drawTopGenres() {
    const svg = createSVG("#chart4");

    const genres = d3.rollups(data,
      v => v.length,
      d => d.genre
    )
    .sort((a,b) => b[1] - a[1])
    .slice(0,5);

    const x = d3.scaleBand()
      .domain(genres.map(d => d[0]))
      .range([40, 330])
      .padding(0.2);

    const y = d3.scaleLinear()
      .domain([0, d3.max(genres, d => d[1])])
      .range([220, 20]);

    svg.selectAll("rect")
      .data(genres)
      .enter()
      .append("rect")
      .attr("x", d => x(d[0]))
      .attr("y", d => y(d[1]))
      .attr("width", x.bandwidth())
      .attr("height", d => 220 - y(d[1]))
      .attr("fill", "orange");
  }

  function drawRatingDistribution() {
    const svg = createSVG("#chart5");

    const bins = d3.bin()
      .thresholds(10)
      (data.map(d => d.rating));

    const x = d3.scaleLinear()
      .domain([0, 10])
      .range([40, 330]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(bins, d => d.length)])
      .range([220, 20]);

    svg.selectAll("rect")
      .data(bins)
      .enter()
      .append("rect")
      .attr("x", d => x(d.x0))
      .attr("y", d => y(d.length))
      .attr("width", x(d.x1) - x(d.x0) - 1)
      .attr("height", d => 220 - y(d.length))
      .attr("fill", "teal");
  }

  function drawTopMovies() {
    const svg = createSVG("#chart6");

    const top = [...data]
      .sort((a,b) => b.votes - a.votes)
      .slice(0,5);

    const x = d3.scaleBand()
      .domain(top.map(d => d.title))
      .range([40, 330])
      .padding(0.2);

    const y = d3.scaleLinear()
      .domain([0, d3.max(top, d => d.votes)])
      .range([220, 20]);

    svg.selectAll("rect")
      .data(top)
      .enter()
      .append("rect")
      .attr("x", d => x(d.title))
      .attr("y", d => y(d.votes))
      .attr("width", x.bandwidth())
      .attr("height", d => 220 - y(d.votes))
      .attr("fill", "crimson");
  }
</script>

<h1>A1: Summer Movies Dataset</h1>

<div class="grid">
  <div id="chart1"><h3>Movies per Year</h3></div>
  <div id="chart2"><h3>Average Rating Over Time</h3></div>
  <div id="chart3"><h3>Runtime vs Rating</h3></div>
  <div id="chart4"><h3>Top 5 Genres</h3></div>
  <div id="chart5"><h3>Rating Distribution</h3></div>
  <div id="chart6"><h3>Top 5 Most Voted Movies</h3></div>
</div>

<style>
  .grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 40px;
    margin-top: 40px;
  }

  h3 {
    text-align: center;
  }

  h1 {
    text-align: center;
  }
</style>
