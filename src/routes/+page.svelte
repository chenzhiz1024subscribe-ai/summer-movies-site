<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { base } from '$app/paths';

  // ---------- helpers: robust column detection ----------
  function pickKey(columns, candidates) {
    const lower = new Map(columns.map((c) => [String(c).toLowerCase(), c]));
    for (const cand of candidates) {
      const hit = lower.get(cand.toLowerCase());
      if (hit) return hit;
    }
    // fallback: partial match
    for (const cand of candidates) {
      const cLower = cand.toLowerCase();
      for (const [kLower, orig] of lower.entries()) {
        if (kLower.includes(cLower)) return orig;
      }
    }
    return null;
  }

  function toNumber(v) {
    if (v === null || v === undefined) return NaN;
    const s = String(v).trim();
    if (!s) return NaN;
    // remove commas
    const cleaned = s.replace(/,/g, '');
    const n = Number(cleaned);
    return Number.isFinite(n) ? n : NaN;
  }

  function toYear(v) {
    const n = toNumber(v);
    if (Number.isFinite(n) && n > 1800 && n < 2100) return Math.round(n);
    // try parse from date string
    const s = String(v ?? '').trim();
    const m = s.match(/(19|20)\d{2}/);
    if (m) return Number(m[0]);
    return NaN;
  }

  // ---------- chart drawing utilities ----------
  function clearSvg(svg) {
    svg.selectAll('*').remove();
  }

  function makeChartFrame(svgSel, width, height, margin) {
    const svg = d3.select(svgSel);
    clearSvg(svg);

    svg.attr('viewBox', `0 0 ${width} ${height}`)
      .attr('width', width)
      .attr('height', height);

    const g = svg.append('g').attr('transform', `translate(${margin.left},${margin.top})`);
    const innerW = width - margin.left - margin.right;
    const innerH = height - margin.top - margin.bottom;

    // border
    svg.append('rect')
      .attr('x', 0)
      .attr('y', 0)
      .attr('width', width)
      .attr('height', height)
      .attr('fill', 'none')
      .attr('stroke', '#e5e7eb');

    return { svg, g, innerW, innerH };
  }

  function axisLabels(svg, width, height, margin, xlabel, ylabel) {
    // x label
    svg.append('text')
      .attr('x', margin.left + (width - margin.left - margin.right) / 2)
      .attr('y', height - 6)
      .attr('text-anchor', 'middle')
      .attr('font-size', 12)
      .text(xlabel);

    // y label
    svg.append('text')
      .attr('transform', `translate(14, ${margin.top + (height - margin.top - margin.bottom) / 2}) rotate(-90)`)
      .attr('text-anchor', 'middle')
      .attr('font-size', 12)
      .text(ylabel);
  }

  // ---------- main render ----------
  onMount(async () => {
    // 1) load CSV from static with correct base path
    const url = `${base}/summer_movies.csv`;
    const raw = await d3.csv(url);

    if (!raw || raw.length === 0) {
      console.error('CSV loaded but empty:', url);
      return;
    }

    const columns = raw.columns ?? Object.keys(raw[0] ?? {});
    // 2) auto-detect likely columns
    const kYear = pickKey(columns, ['year', 'release_year', 'release year']);
    const kRating = pickKey(columns, ['rating', 'imdb_rating', 'imdb rating', 'score', 'avg_rating']);
    const kRuntime = pickKey(columns, ['runtime', 'duration', 'minutes', 'runtime_minutes']);
    const kVotes = pickKey(columns, ['votes', 'vote_count', 'vote count', 'num_votes', 'votecount']);
    const kGenre = pickKey(columns, ['genre', 'genres', 'primary_genre', 'type']);
    const kTitle = pickKey(columns, ['title', 'movie', 'name']);

    // 3) normalize rows with strict numeric parsing to avoid NaN
    const data = raw.map((d) => {
      const year = kYear ? toYear(d[kYear]) : NaN;
      const rating = kRating ? toNumber(d[kRating]) : NaN;
      const runtime = kRuntime ? toNumber(d[kRuntime]) : NaN;
      const votes = kVotes ? toNumber(d[kVotes]) : NaN;
      const genreRaw = kGenre ? String(d[kGenre] ?? '').trim() : '';
      const title = kTitle ? String(d[kTitle] ?? '').trim() : '';

      // split genres if like "Action, Drama"
      const genres = genreRaw
        ? genreRaw.split(/[,|/]/).map((s) => s.trim()).filter(Boolean)
        : [];

      return { year, rating, runtime, votes, genres, title };
    });

    // filter for each chart separately to keep robust
    const hasYear = data.filter((d) => Number.isFinite(d.year));
    const hasYearRating = data.filter((d) => Number.isFinite(d.year) && Number.isFinite(d.rating));
    const hasRuntimeRating = data.filter((d) => Number.isFinite(d.runtime) && Number.isFinite(d.rating));
    const hasRating = data.filter((d) => Number.isFinite(d.rating));
    const hasVotes = data.filter((d) => Number.isFinite(d.votes));
    const hasGenres = data.filter((d) => d.genres && d.genres.length);

    // chart sizing
    const W = 420;
    const H = 260;
    const margin = { top: 18, right: 18, bottom: 45, left: 55 };

    // ---------------- Chart 1: Movies per Year (bar) ----------------
    {
      const svgSel = '#chart1';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const yearly = d3.rollups(
        hasYear,
        (v) => v.length,
        (d) => d.year
      )
      .map(([year, count]) => ({ year, count }))
      .sort((a, b) => d3.ascending(a.year, b.year));

      const x = d3.scaleBand()
        .domain(yearly.map((d) => d.year))
        .range([0, innerW])
        .padding(0.15);

      const y = d3.scaleLinear()
        .domain([0, d3.max(yearly, (d) => d.count) || 1])
        .nice()
        .range([innerH, 0]);

      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).tickValues(x.domain().filter((d, i) => i % Math.ceil(x.domain().length / 6) === 0)))
        .selectAll('text')
        .attr('transform', 'rotate(-20)')
        .style('text-anchor', 'end');

      g.append('g').call(d3.axisLeft(y).ticks(6));

      g.selectAll('rect')
        .data(yearly)
        .join('rect')
        .attr('x', (d) => x(d.year))
        .attr('y', (d) => y(d.count))
        .attr('width', x.bandwidth())
        .attr('height', (d) => innerH - y(d.count));

      axisLabels(svg, W, H, margin, 'Year', 'Count');
    }

    // ---------------- Chart 2: Average Rating Over Time (line) ----------------
    {
      const svgSel = '#chart2';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const avgByYear = d3.rollups(
        hasYearRating,
        (v) => d3.mean(v, (d) => d.rating),
        (d) => d.year
      )
      .map(([year, avg]) => ({ year, avg }))
      .filter((d) => Number.isFinite(d.avg))
      .sort((a, b) => d3.ascending(a.year, b.year));

      const x = d3.scaleLinear()
        .domain(d3.extent(avgByYear, (d) => d.year) ?? [2000, 2001])
        .range([0, innerW]);

      const y = d3.scaleLinear()
        .domain([0, 10])
        .range([innerH, 0]);

      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).ticks(6).tickFormat(d3.format('d')));

      g.append('g').call(d3.axisLeft(y).ticks(5));

      const line = d3.line()
        .x((d) => x(d.year))
        .y((d) => y(d.avg));

      g.append('path')
        .datum(avgByYear)
        .attr('fill', 'none')
        .attr('stroke', 'currentColor')
        .attr('stroke-width', 2)
        .attr('d', line);

      axisLabels(svg, W, H, margin, 'Year', 'Avg Rating');
    }

    // ---------------- Chart 3: Runtime vs Rating (scatter) ----------------
    {
      const svgSel = '#chart3';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const x = d3.scaleLinear()
        .domain(d3.extent(hasRuntimeRating, (d) => d.runtime) ?? [60, 180])
        .nice()
        .range([0, innerW]);

      const y = d3.scaleLinear()
        .domain(d3.extent(hasRuntimeRating, (d) => d.rating) ?? [0, 10])
        .nice()
        .range([innerH, 0]);

      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).ticks(6));

      g.append('g').call(d3.axisLeft(y).ticks(6));

      g.selectAll('circle')
        .data(hasRuntimeRating.slice(0, 1500)) // protect performance
        .join('circle')
        .attr('cx', (d) => x(d.runtime))
        .attr('cy', (d) => y(d.rating))
        .attr('r', 2.2)
        .attr('opacity', 0.55);

      axisLabels(svg, W, H, margin, 'Runtime (min)', 'Rating');
    }

    // ---------------- Chart 4: Top 5 Genres (barh) ----------------
    {
      const svgSel = '#chart4';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const counts = new Map();
      for (const d of hasGenres) {
        for (const ge of d.genres) {
          counts.set(ge, (counts.get(ge) ?? 0) + 1);
        }
      }

      const top = Array.from(counts.entries())
        .map(([genre, count]) => ({ genre, count }))
        .sort((a, b) => d3.descending(a.count, b.count))
        .slice(0, 5)
        .reverse();

      const y = d3.scaleBand()
        .domain(top.map((d) => d.genre))
        .range([0, innerH])
        .padding(0.2);

      const x = d3.scaleLinear()
        .domain([0, d3.max(top, (d) => d.count) || 1])
        .nice()
        .range([0, innerW]);

      g.append('g').call(d3.axisLeft(y));
      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).ticks(5));

      g.selectAll('rect')
        .data(top)
        .join('rect')
        .attr('y', (d) => y(d.genre))
        .attr('x', 0)
        .attr('height', y.bandwidth())
        .attr('width', (d) => x(d.count));

      axisLabels(svg, W, H, margin, 'Count', 'Genre');
    }

    // ---------------- Chart 5: Rating Distribution (hist) ----------------
    {
      const svgSel = '#chart5';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const ratings = hasRating.map((d) => d.rating).filter(Number.isFinite);

      const x = d3.scaleLinear()
        .domain([0, 10])
        .range([0, innerW]);

      const bins = d3.bin()
        .domain(x.domain())
        .thresholds(10)(ratings);

      const y = d3.scaleLinear()
        .domain([0, d3.max(bins, (b) => b.length) || 1])
        .nice()
        .range([innerH, 0]);

      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).ticks(6));

      g.append('g').call(d3.axisLeft(y).ticks(6));

      g.selectAll('rect')
        .data(bins)
        .join('rect')
        .attr('x', (d) => x(d.x0))
        .attr('y', (d) => y(d.length))
        .attr('width', (d) => Math.max(0, x(d.x1) - x(d.x0) - 1))
        .attr('height', (d) => innerH - y(d.length));

      axisLabels(svg, W, H, margin, 'Rating', 'Movies');
    }

    // ---------------- Chart 6: Top 5 Most Voted Movies (barh) ----------------
    {
      const svgSel = '#chart6';
      const { svg, g, innerW, innerH } = makeChartFrame(svgSel, W, H, margin);

      const top = hasVotes
        .slice()
        .sort((a, b) => d3.descending(a.votes, b.votes))
        .slice(0, 5)
        .map((d) => ({
          name: d.title || 'Untitled',
          votes: d.votes
        }))
        .reverse();

      const y = d3.scaleBand()
        .domain(top.map((d) => d.name))
        .range([0, innerH])
        .padding(0.2);

      const x = d3.scaleLinear()
        .domain([0, d3.max(top, (d) => d.votes) || 1])
        .nice()
        .range([0, innerW]);

      g.append('g')
        .call(d3.axisLeft(y))
        .selectAll('text')
        .style('font-size', '10px');

      g.append('g')
        .attr('transform', `translate(0,${innerH})`)
        .call(d3.axisBottom(x).ticks(5).tickFormat(d3.format('~s')));

      g.selectAll('rect')
        .data(top)
        .join('rect')
        .attr('y', (d) => y(d.name))
        .attr('x', 0)
        .attr('height', y.bandwidth())
        .attr('width', (d) => x(d.votes));

      axisLabels(svg, W, H, margin, 'Votes', 'Movie');
    }
  });
</script>

<h1 class="title">A1: Summer Movies Dataset</h1>

<div class="grid">
  <div class="card">
    <h3>Movies per Year</h3>
    <svg id="chart1"></svg>
  </div>

  <div class="card">
    <h3>Average Rating Over Time</h3>
    <svg id="chart2"></svg>
  </div>

  <div class="card">
    <h3>Runtime vs Rating</h3>
    <svg id="chart3"></svg>
  </div>

  <div class="card">
    <h3>Top 5 Genres</h3>
    <svg id="chart4"></svg>
  </div>

  <div class="card">
    <h3>Rating Distribution</h3>
    <svg id="chart5"></svg>
  </div>

  <div class="card">
    <h3>Top 5 Most Voted Movies</h3>
    <svg id="chart6"></svg>
  </div>
</div>

<style>
  .title{
    text-align:center;
    margin: 18px 0 10px;
  }

  .grid{
    display:grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 22px;
    padding: 10px 18px 24px;
  }

  .card{
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    padding: 10px 10px 6px;
    background: #fff;
  }

  h3{
    text-align:center;
    margin: 6px 0 8px;
    font-size: 14px;
  }

  svg{
    width: 100%;
    height: auto;
    display:block;
  }

  @media (max-width: 1100px){
    .grid{ grid-template-columns: repeat(2, 1fr); }
  }
  @media (max-width: 720px){
    .grid{ grid-template-columns: 1fr; }
  }
</style>
