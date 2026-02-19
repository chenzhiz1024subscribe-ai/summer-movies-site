<script>
import * as d3 from "d3";
import { onMount } from "svelte";

let data = [];

onMount(async () => {
  data = await d3.csv("/summer_movies.csv", d => ({
    year: +d.year,
    runtime: +d.runtime_minutes,
    rating: +d.average_rating,
    votes: +d.num_votes,
    genre: d.genre
  }));

  drawMoviesPerYear();
  drawAverageRating();
  drawScatter();
  drawTopGenres();
  drawRatingHistogram();
  drawTopVoted();
});

/* ---------- 1. Movies Per Year ---------- */
function drawMoviesPerYear() {
  const svg = d3.select("#chart1");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 40;

  const counts = d3.rollups(
    data,
    v => v.length,
    d => d.year
  ).sort((a,b) => a[0] - b[0]);

  const x = d3.scaleBand()
    .domain(counts.map(d => d[0]))
    .range([margin, width - margin])
    .padding(0.1);

  const y = d3.scaleLinear()
    .domain([0, d3.max(counts, d => d[1])])
    .range([height - margin, margin]);

  svg.selectAll("rect")
    .data(counts)
    .enter()
    .append("rect")
    .attr("x", d => x(d[0]))
    .attr("y", d => y(d[1]))
    .attr("width", x.bandwidth())
    .attr("height", d => height - margin - y(d[1]))
    .attr("fill", "#4e79a7");

  svg.append("g")
    .attr("transform", `translate(0,${height - margin})`)
    .call(d3.axisBottom(x).tickValues(x.domain().filter((d,i)=>!(i%5))))
    .selectAll("text")
    .attr("transform", "rotate(-45)")
    .style("text-anchor", "end");

  svg.append("g")
    .attr("transform", `translate(${margin},0)`)
    .call(d3.axisLeft(y));
}

/* ---------- 2. Average Rating Over Time ---------- */
function drawAverageRating() {
  const svg = d3.select("#chart2");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 40;

  const yearly = d3.rollups(
    data,
    v => d3.mean(v, d => d.rating),
    d => d.year
  ).sort((a,b)=>a[0]-b[0]);

  const x = d3.scaleLinear()
    .domain(d3.extent(yearly, d=>d[0]))
    .range([margin, width-margin]);

  const y = d3.scaleLinear()
    .domain([0,10])
    .range([height-margin, margin]);

  const line = d3.line()
    .x(d=>x(d[0]))
    .y(d=>y(d[1]));

  svg.append("path")
    .datum(yearly)
    .attr("fill","none")
    .attr("stroke","#f28e2b")
    .attr("stroke-width",2)
    .attr("d",line);

  svg.append("g")
    .attr("transform",`translate(0,${height-margin})`)
    .call(d3.axisBottom(x));

  svg.append("g")
    .attr("transform",`translate(${margin},0)`)
    .call(d3.axisLeft(y));
}

/* ---------- 3. Runtime vs Rating ---------- */
function drawScatter() {
  const svg = d3.select("#chart3");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 40;

  const x = d3.scaleLinear()
    .domain(d3.extent(data,d=>d.runtime))
    .range([margin,width-margin]);

  const y = d3.scaleLinear()
    .domain([0,10])
    .range([height-margin,margin]);

  svg.selectAll("circle")
    .data(data)
    .enter()
    .append("circle")
    .attr("cx",d=>x(d.runtime))
    .attr("cy",d=>y(d.rating))
    .attr("r",3)
    .attr("fill","purple")
    .attr("opacity",0.6);

  svg.append("g")
    .attr("transform",`translate(0,${height-margin})`)
    .call(d3.axisBottom(x));

  svg.append("g")
    .attr("transform",`translate(${margin},0)`)
    .call(d3.axisLeft(y));
}

/* ---------- 4. Top Genres ---------- */
function drawTopGenres() {
  const svg = d3.select("#chart4");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 60;

  const counts = d3.rollups(
    data,
    v=>v.length,
    d=>d.genre
  ).sort((a,b)=>b[1]-a[1]).slice(0,5);

  const y = d3.scaleBand()
    .domain(counts.map(d=>d[0]))
    .range([margin,height-margin])
    .padding(0.2);

  const x = d3.scaleLinear()
    .domain([0,d3.max(counts,d=>d[1])])
    .range([margin,width-margin]);

  svg.selectAll("rect")
    .data(counts)
    .enter()
    .append("rect")
    .attr("y",d=>y(d[0]))
    .attr("x",margin)
    .attr("height",y.bandwidth())
    .attr("width",d=>x(d[1])-margin)
    .attr("fill","green");

  svg.append("g")
    .attr("transform",`translate(0,${height-margin})`)
    .call(d3.axisBottom(x));

  svg.append("g")
    .attr("transform",`translate(${margin},0)`)
    .call(d3.axisLeft(y));
}

/* ---------- 5. Rating Distribution ---------- */
function drawRatingHistogram() {
  const svg = d3.select("#chart5");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 40;

  const x = d3.scaleLinear()
    .domain([0,10])
    .range([margin,width-margin]);

  const bins = d3.bin()
    .domain(x.domain())
    .thresholds(8)
    (data.map(d=>d.rating));

  const y = d3.scaleLinear()
    .domain([0,d3.max(bins,d=>d.length)])
    .range([height-margin,margin]);

  svg.selectAll("rect")
    .data(bins)
    .enter()
    .append("rect")
    .attr("x",d=>x(d.x0))
    .attr("y",d=>y(d.length))
    .attr("width",d=>x(d.x1)-x(d.x0)-2)
    .attr("height",d=>height-margin-y(d.length))
    .attr("fill","teal");

  svg.append("g")
    .attr("transform",`translate(0,${height-margin})`)
    .call(d3.axisBottom(x));

  svg.append("g")
    .attr("transform",`translate(${margin},0)`)
    .call(d3.axisLeft(y));
}

/* ---------- 6. Top Voted Movies ---------- */
function drawTopVoted() {
  const svg = d3.select("#chart6");
  svg.selectAll("*").remove();

  const width = 400;
  const height = 250;
  const margin = 60;

  const top = [...data]
    .sort((a,b)=>b.votes-a.votes)
    .slice(0,5);

  const y = d3.scaleBand()
    .domain(top.map((d,i)=>`#${i+1}`))
    .range([margin,height-margin])
    .padding(0.2);

  const x = d3.scaleLinear()
    .domain([0,d3.max(top,d=>d.votes)])
    .range([margin,width-margin]);

  svg.selectAll("rect")
    .data(top)
    .enter()
    .append("rect")
    .attr("y",(d,i)=>y(`#${i+1}`))
    .attr("x",margin)
    .attr("height",y.bandwidth())
    .attr("width",d=>x(d.votes)-margin)
    .attr("fill","red");

  svg.append("g")
    .attr("transform",`translate(0,${height-margin})`)
    .call(d3.axisBottom(x));

  svg.append("g")
    .attr("transform",`translate(${margin},0)`)
    .call(d3.axisLeft(y));
}
</script>

<h1>A1: Summer Movies Dataset</h1>

<div class="grid">
  <div><h3>Movies per Year</h3><svg id="chart1" width="400" height="250"></svg></div>
  <div><h3>Average Rating Over Time</h3><svg id="chart2" width="400" height="250"></svg></div>
  <div><h3>Runtime vs Rating</h3><svg id="chart3" width="400" height="250"></svg></div>
  <div><h3>Top 5 Genres</h3><svg id="chart4" width="400" height="250"></svg></div>
  <div><h3>Rating Distribution</h3><svg id="chart5" width="400" height="250"></svg></div>
  <div><h3>Top 5 Most Voted Movies</h3><svg id="chart6" width="400" height="250"></svg></div>
</div>

<style>
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 30px;
}
h3 {
  text-align: center;
}
</style>
