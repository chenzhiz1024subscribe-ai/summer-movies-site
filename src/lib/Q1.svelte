<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";

  export let movies: any[];
  let svgElement: SVGSVGElement;

  const width = 900;
  const height = 500;
  const margin = { top: 40, right: 150, bottom: 60, left: 60 };

  onMount(() => {
    const svg = d3.select(svgElement)
      .attr("width", width)
      .attr("height", height);

    const yearly = movies.flatMap(m =>
      m.genres.split(",").map((g: string) => ({
        year: m.year,
        genre: g
      }))
    );

    const grouped = d3.rollups(
      yearly,
      v => v.length,
      d => d.year,
      d => d.genre
    );

    let processed: any[] = [];

    grouped.forEach(([year, genres]) => {
      const sorted = genres
        .map(([genre, count]) => ({ year, genre, count }))
        .sort((a, b) => b.count - a.count)
        .slice(0, 3);

      processed.push(...sorted);
    });

    const genreFrequency = d3.rollup(
      processed,
      v => v.length,
      d => d.genre
    );

    const topGenres = Array.from(genreFrequency.entries())
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(d => d[0]);

    const filtered = processed.filter(d => topGenres.includes(d.genre));

    const years = Array.from(new Set(filtered.map(d => d.year))).sort();

    const x = d3.scaleLinear()
      .domain(d3.extent(years) as [number, number])
      .range([margin.left, width - margin.right]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(filtered, d => d.count)!])
      .range([height - margin.bottom, margin.top]);

    const color = d3.scaleOrdinal()
      .domain(topGenres)
      .range(d3.schemeTableau10);

    const line = d3.line<any>()
      .x(d => x(d.year))
      .y(d => y(d.count));

    topGenres.forEach(genre => {
      const data = filtered.filter(d => d.genre === genre)
        .sort((a, b) => a.year - b.year);

      svg.append("path")
        .datum(data)
        .attr("fill", "none")
        .attr("stroke", color(genre) as string)
        .attr("stroke-width", 2.5)
        .attr("d", line);
    });

    svg.append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x).tickFormat(d3.format("d")));

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y));

    // 添加 legend
    const legend = svg.selectAll(".legend")
      .data(topGenres)
      .enter()
      .append("g")
      .attr("transform", (_, i) => `translate(${width - margin.right + 20},${margin.top + i * 25})`);

    legend.append("rect")
      .attr("width", 15)
      .attr("height", 15)
      .attr("fill", d => color(d) as string);

    legend.append("text")
      .attr("x", 20)
      .attr("y", 12)
      .text(d => d)
      .style("font-size", "12px");
  });
</script>

<h2>Q1: Annual Top 3 Genres Over Time</h2>
<svg bind:this={svgElement}></svg>
