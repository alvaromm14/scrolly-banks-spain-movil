<script>
  import * as d3 from "d3";
  import rawData from "$data/data.js";
  import Tooltip from "./components/Tooltip.svelte";
  import Scrolly from "./components/Scrolly.svelte";
  import { fade } from "svelte/transition";

  let width = 928;
  let height = 600;

  $: innerHeight = height;

  const hierarchicalData = rawData[0];

  $: pack = d3.pack().size([width, innerHeight]).padding(6);

  let root;

  const hierarchicalDataFlat = {
    ...rawData[0],
    children: rawData[0].children.map((banco) => ({
      ...banco,
      children: banco.children.flatMap((region) =>
        (region.children || []).map((pais) => ({
          ...pais,
          region: region.name,
        })),
      ),
    })),
  };

  $: if (pack && hierarchicalDataFlat) {
    root = pack(
      d3
        .hierarchy(hierarchicalDataFlat)
        .sum((d) => d.value || 0)
        .sort((a, b) => b.value - a.value),
    );
  }

  function getRegionColor(node) {
    if (!node.parent) return "#fff"; // root
    if (node.depth === 1) return "#eee"; // banco

    const region = node.data.region;
    if (node.data.name === "España") return "#f26c55";

    switch (region) {
      case "Europa":
        return "#559df2";
      case "Sudamérica":
        return "#f5b956";
      case "Norteamérica":
        return "#485fb3";
      default:
        return "#999";
    }
  }

  let hoveredData;

  let currentStep;

  window.addEventListener("DOMContentLoaded", (event) => {
    function updateIframeHeight() {
      const el = document.documentElement;
      const rect = el.getBoundingClientRect();
      const styles = window.getComputedStyle(el);
      const margin =
        parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
      const height = Math.ceil(rect.height + margin);

      window.parent.postMessage(
        {
          type: "resize-iframe",
          value: height,
        },
        "*",
      );
    }
    updateIframeHeight();

    if (window.ResizeObserver) {
      new ResizeObserver(() => {
        updateIframeHeight();
      }).observe(document.documentElement);
    } else {
      window.addEventListener("load", updateIframeHeight);
      window.addEventListener("resize", updateIframeHeight);
    }

    window.addEventListener(
      "message",
      (event) => {
        if (event.data.type === "request-resize") {
          updateIframeHeight();
        }
      },
      false,
    );
  });
</script>

<section>
  <div class="sticky">
    <div
      class="chart-container"
      bind:clientWidth={width}
      bind:clientHeight={height}
    >
      <svg {width} {height}>
        <!-- Bancos -->
        {#each root.descendants().filter((d) => d.depth === 1) as banco}
          <g transform="translate({banco.x}, {banco.y})">
            <circle transition:fade r={banco.r} fill="#eee" />
          </g>
        {/each}

        <!-- España desde el paso 3 -->
        {#if currentStep >= 3}
          {#each root
            .descendants()
            .filter((d) => d.depth === 2 && d.data.name === "España") as pais}
            <g transform="translate({pais.x}, {pais.y})">
              <circle
                transition:fade
                on:mouseover={() =>
                  currentStep > 5 ? (hoveredData = pais) : null}
                on:mouseout={() => (hoveredData = null)}
                r={pais.r}
                fill={getRegionColor(pais)}
                stroke="#555"
                stroke-width={hoveredData
                  ? pais === hoveredData
                    ? "1"
                    : "0"
                  : "0"}
                opacity={hoveredData ? (pais === hoveredData ? 1 : 0.3) : 1}
              />

              {#if currentStep < 5}
                <!-- Mostrar el porcentaje en el paso 2 -->
                <text
                  transition:fade
                  class="pais"
                  font-size={Math.min(14, pais.r / 1.5)}
                  text-anchor="middle"
                  alignment-baseline="middle"
                >
                  {#if pais.parent.data.name === "BBVA"}
                    34%
                  {:else if pais.parent.data.name === "Santander"}
                    29%
                  {:else}
                    {Math.round((pais.value / pais.parent.value) * 100)}%
                  {/if}
                </text>
              {/if}
            </g>
          {/each}
        {/if}

        <!-- Resto del mundo desde el paso 5 -->
        {#if currentStep >= 5}
          {#each root
            .descendants()
            .filter((d) => d.depth === 2 && d.data.name !== "España") as pais}
            <g transform="translate({pais.x}, {pais.y})">
              <circle
                transition:fade
                on:mouseover={() =>
                  currentStep > 5 ? (hoveredData = pais) : null}
                on:mouseout={() => (hoveredData = null)}
                r={pais.r}
                fill={getRegionColor(pais)}
                stroke="#555"
                stroke-width={hoveredData
                  ? pais === hoveredData
                    ? "1"
                    : "0"
                  : "0"}
                opacity={hoveredData ? (pais === hoveredData ? 1 : 0.3) : 1}
              />
              {#if pais.r > 32}
                <text class="pais" font-size={Math.min(14, pais.r / 1.5)}>
                  {pais.data.name}
                </text>
              {/if}
            </g>
          {/each}
        {/if}

        <!-- Bancos labels -->
        {#if currentStep > 0}
          {#each root.descendants().filter((d) => d.depth === 1) as banco}
            <g transform="translate({banco.x}, {banco.y})">
              <text
                transition:fade
                class="banco"
                font-size={Math.min(14, banco.r / 1.5)}
                transform={currentStep > 2
                  ? `translate(0, ${banco.data.name === "Iberc." || banco.data.name === "Cajamar" || banco.data.name === "Bankinter" ? banco.r * -1 : banco.r * 0.9})`
                  : `translate(0, -2)`}
                style="transition: transform 700ms ease;"
                text-anchor="middle"
                alignment-baseline="middle"
              >
                {banco.data.name}
              </text>

              {#if currentStep < 3}
                <text
                  transition:fade
                  class="activos"
                  font-size={Math.min(13, banco.r / 2)}
                  dy="1em"
                  text-anchor="middle"
                >
                  {banco.value.toLocaleString("es-ES", { useGrouping: true })}
                </text>
              {/if}
            </g>
          {/each}
        {/if}
      </svg>
      <Tooltip {hoveredData} {width} />
    </div>
  </div>

  <div class="steps">
    <Scrolly bind:value={currentStep}>
      {#each [0, 1, 2, 3, 4, 5, 6] as i}
        <div class="step" class:active={currentStep === i}>
          {#if i === 0}
            <div class="step-content">
              <p>
                Estos son los <span style="font-weight: bold"
                  >beneficios después de impuestos</span
                > que obtuvieron los diez principales bancos españoles en todo el
                mundo en 2024.
              </p>
            </div>
          {:else if i === 1}
            <div class="step-content">
              <p>
                En total, la banca española se embolsó 36.000 millones de euros,
                el equivalente a cerca de 700€ por español. <br /><br /> El
                <span style="font-weight: bold">Santander</span>
                y el <span style="font-weight: bold">BBVA</span> acapararon por sí
                solos más de dos terceras partes.
              </p>
            </div>
          {:else if i === 5}
            <div class="step-content">
              <p>
                El resto provino del extranjero, concretamente el 24% de <span
                  style="color: #485fb3; font-weight: bold">Norteamérica</span
                >, el 13% de
                <span style="color: #f5b956; font-weight: bold">Sudamérica</span
                >
                y el 12% de
                <span style="color: #559df2; font-weight: bold">Europa</span>.
                <br /><br />El 3% restante fue facturado en
                <span style="color: #999; font-weight: bold">otros países</span>
                como Turquía o Marruecos. Explora esos equilibrios en el gráfico
                interactivo.
              </p>
            </div>
          {:else if i === 3}
            <div class="step-content">
              <p>
                <span style="color: #f28b79; font-weight: bold">España</span> no
                supuso ni la mitad de esos beneficios. Solo aportó de media el 49%,
                una cifra que desciende al 29% en el caso del Santander y al 34%
                en el del BBVA.
              </p>
            </div>
          {/if}
        </div>
      {/each}
    </Scrolly>
  </div>
</section>

<style>
  .chart-container {
    user-select: none;
    height: 100vh;
  }
  svg {
    display: block;
  }
  text {
    pointer-events: none;
    font-weight: 600;
  }

  .banco,
  .activos {
    text-shadow:
      -1.5px -1.5px 0 #fff,
      1.5px -1.5px 0 #fff,
      -1.5px 1.5px 0 #fff,
      1.5px 1.5px 0 #fff;
    font-weight: 800;
    background-color: white;
    alignment-baseline: middle;
    text-transform: uppercase;
  }

  .activos {
    font-weight: 400;
  }
  .pais {
    font-weight: 400;
    fill: white;
    text-anchor: middle;
    alignment-baseline: middle;
  }

  .step {
    height: 90vh;
    opacity: 0.3;
    transition: opacity 300ms ease;

    display: flex;
    justify-content: center;
    place-items: center;
  }

  .step.active {
    opacity: 1;
  }

  .step-content {
    background: white;
    padding: 20px;
    border-radius: 8px;
    max-width: 500px;
    width: 100%;
    box-shadow: rgba(0, 0, 0, 0.16) 0px 10px 36px 0px;
    line-height: 1.2;
  }

  section {
    position: relative;
  }

  .steps {
    position: relative;
    z-index: 2;
    pointer-events: none;
  }

  .sticky {
    position: sticky;
    top: 0;
    z-index: 1;
    height: 90vh;
    top: 5vh;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .chart-container {
    width: 100%;
    height: 100%;
    max-width: 750px;
    max-height: 750px;
  }
</style>
