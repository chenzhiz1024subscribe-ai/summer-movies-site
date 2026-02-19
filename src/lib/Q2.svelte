<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";

  export let movies: any[];
  let svgElement: SVGSVGElement;

  const width = 900;
  const height = 850;
  const margin = { top: 150, right: 50, bottom: 100, left: 150 };

  onMount(() => {
    const svg = d3.select(svgElement)
      .attr("width", width)
      .attr("height", height);

    const genreSet = new Set<string>();
    movies.forEach(m => {
      m.genres.split(",").forEach((g: string) => genreSet.add(g));
    });

    const genres = Array.from(genreSet).sort();

    const matrix: any[] = [];

    genres.forEach((g1, i) => {
      genres.forEach((g2, j) => {
        if (i < j) {  
          matrix.push({
            g1,
            g2,
            value: 0
          });
        }
      });
    });

    movies.forEach(m => {
      const gs = m.genres.split(",");
      gs.forEach((g1: string, i: number) => {
        gs.forEach((g2: string, j: number) => {
          if (i < j) {
            const cell = matrix.find(d => d.g1 === g1 && d.g2 === g2);
            if (cell) cell.value += 1;
          }
        });
      });
    });

    const maxValue = d3.max(matrix, d => d.value)!;

    const x = d3.scaleBand()
      .domain(genres)
      .range([margin.left, width - margin.right]);

    const y = d3.scaleBand()
      .domain(genres)
      .range([margin.top, height - margin.bottom]);

    const color = d3.scaleSequential(d3.interpolateOrRd)
      .domain([0, maxValue]);

    svg.selectAll("rect.cell")
      .data(matrix)
      .enter()
      .append("rect")
      .attr("class", "cell")
      .attr("x", d => x(d.g1)!)
      .attr("y", d => y(d.g2)!)
      .attr("width", x.bandwidth())
      .attr("height", y.bandwidth())
      .attr("fill", d => color(d.value))
      .on("mouseover", function (event, d) {
        d3.select(this).attr("stroke", "black").attr("stroke-width", 2);
      })
      .on("mouseout", function () {
        d3.select(this).attr("stroke", "none");
      })
      .append("title")
      .text(d => `${d.g1} & ${d.g2}: ${d.value}`);

    svg.append("g")
      .attr("transform", `translate(0,${margin.top})`)
      .call(d3.axisTop(x))
      .selectAll("text")
      .attr("transform", "rotate(-45)")
      .style("text-anchor", "start");

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y));

    const legendHeight = 200;
    const legendWidth = 20;

    const legendScale = d3.scaleLinear()
      .domain([0, maxValue])
      .range([legendHeight, 0]);

    const legendAxis = d3.axisRight(legendScale);

    const legendGroup = svg.append("g")
      .attr("transform", `translate(${width - 60}, ${margin.top})`);

    const legendGradient = svg.append("defs")
      .append("linearGradient")
      .attr("id", "legend-gradient")
      .attr("x1", "0%")
      .attr("x2", "0%")
      .attr("y1", "100%")
      .attr("y2", "0%");

    legendGradient.selectAll("stop")
      .data(d3.range(0, 1.01, 0.1))
      .enter()
      .append("stop")
      .attr("offset", d => `${d * 100}%`)
      .attr("stop-color", d => color(d * maxValue));

    legendGroup.append("rect")
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .style("fill", "url(#legend-gradient)");

    legendGroup.append("g")
      .attr("transform", `translate(${legendWidth}, 0)`)
      .call(legendAxis);
  });
</script>

<h2>Q2: Genre Correlation Heatmap</h2>
<svg bind:this={svgElement}></svg>
