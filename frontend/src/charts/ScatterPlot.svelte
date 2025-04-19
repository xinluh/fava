<script lang="ts">
  import { extent } from "d3-array";
  import { axisBottom, axisLeft } from "d3-axis";
  import { quadtree } from "d3-quadtree";
  import { scalePoint, scaleUtc } from "d3-scale";

  import { day } from "../format";

  import Axis from "./Axis.svelte";
  import { scatterplotScale } from "./helpers";
  import type { ScatterPlotDatum } from "./scatterplot";
  import type { TooltipFindNode } from "./tooltip";
  import { domHelpers, positionedTooltip } from "./tooltip";

  export let data: ScatterPlotDatum[];
  export let width: number;

  const today = new Date();
  const margin = {
    top: 10,
    right: 10,
    bottom: 30,
    left: 70,
  };
  const height = 250;
  $: innerWidth = width - margin.left - margin.right;
  $: innerHeight = height - margin.top - margin.bottom;

  // Scales
  $: dateExtent = extent(data, (d) => d.date);
  $: x = scaleUtc()
    .domain(dateExtent[0] ? dateExtent : [0, 1])
    .range([0, innerWidth]);
  $: y = scalePoint()
    .padding(1)
    .domain(data.map((d) => d.type))
    .range([innerHeight, 0]);

  // Axes
  $: xAxis = axisBottom(x).tickSizeOuter(0);
  $: yAxis = axisLeft(y)
    .tickPadding(6)
    .tickSize(-innerWidth)
    .tickFormat((d) => d);

  /** Quadtree for hover. */
  $: quad = quadtree(
    data,
    (d) => x(d.date),
    (d) => y(d.type) ?? 0
  );

  function tooltipText(d: ScatterPlotDatum) {
    return [domHelpers.t(d.description), domHelpers.em(day(d.date) + (d.meta?.end ? ' - ' + d.meta.end : ''))];
  }

  const tooltipFindNode: TooltipFindNode = (xPos, yPos) => {
    const d = quad.find(xPos, yPos);
    return d && [x(d.date), y(d.type) ?? 0, tooltipText(d)];
  };
</script>

<svg {width} {height}>
  <g
    use:positionedTooltip={tooltipFindNode}
    transform={`translate(${margin.left},${margin.top})`}
  >
    <Axis x axis={xAxis} {innerHeight} />
    <Axis y axis={yAxis} />
    <g>
      {#each data as dot}
        <circle
          r="5"
          fill={scatterplotScale(dot.type)}
          cx={x(dot.date)}
          cy={y(dot.type)}
          class:desaturate={dot.date > today}
        />
        {#if dot.meta?.end}
          <line class="event-line"
            x1={x(dot.date)}
            x2={x(new Date(dot.meta.end))}
            y1={y(dot.type)}
            y2={y(dot.type)}
            stroke={scatterplotScale(dot.type)}
          />
        <circle
          r="3"
          fill={scatterplotScale(dot.type)}
          cx={x(new Date(dot.meta.end))}
          cy={y(dot.type)}
          class:desaturate={dot.date > today}
        />
        {/if}
      {/each}
    </g>
  </g>
</svg>

<style>
  svg > g {
    pointer-events: all;
  }

  .desaturate {
    filter: saturate(50%);
  }

  .event-line {
    stroke-width: 3px;
  }
</style>
