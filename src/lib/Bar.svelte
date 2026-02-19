<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";

  export let movies: any[];

  let svgElement: SVGSVGElement;

  const width = 800;
  const height = 500;
  const margin = { top: 40, right: 20, bottom: 120, left: 60 };

  onMount(() => {
    const svg = d3
      .select(svgElement)
      .attr("width", width)
      .attr("height", height);

    const genres = movies.flatMap(m => m.genres.split(","));

    const genreCounts = d3.rollup(
      genres,
      v => v.length,
      d => d
    );

    const data = Array.from(genreCounts, ([genre, count]) => ({
      genre,
      count
    })).sort((a, b) => b.count - a.count);

    const x = d3.scaleBand()
      .domain(data.map(d => d.genre))
      .range([margin.left, width - margin.right])
      .padding(0.2);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count)!])
      .range([height - margin.bottom, margin.top]);

    svg.selectAll("rect")
      .data(data)
      .enter()
      .append("rect")
      .attr("x", d => x(d.genre)!)
      .attr("y", d => y(d.count))
      .attr("width", x.bandwidth())
      .attr("height", d => height - margin.bottom - y(d.count))
      .attr("fill", "steelblue")
      .on("mouseover", function () {
        d3.select(this).attr("fill", "orange");
      })
      .on("mouseout", function () {
        d3.select(this).attr("fill", "steelblue");
      });

    svg.append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x))
      .selectAll("text")
      .attr("transform", "rotate(-45)")
      .style("text-anchor", "end");

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y));
  });
</script>

<svg bind:this={svgElement}></svg>
