<script>
  import { fly, fade } from "svelte/transition";
  import { tick } from "svelte";

  export let hoveredData = null;
  export let width = 800;

  let tooltipWidth = 0;
  let xPosition = 0;
  let yPosition = 0;

  const xNudge = 10;
  const yNudge = 10;

  async function updatePosition() {
    await tick();
    if (hoveredData) {
      xPosition =
        hoveredData.x + tooltipWidth + xNudge > width
          ? hoveredData.x - tooltipWidth - xNudge
          : hoveredData.x + xNudge;

      yPosition = hoveredData.y + yNudge;
    }
  }

  $: if (hoveredData) {
    updatePosition();
  }
</script>

{#if hoveredData}
  <div
    class="tooltip"
    in:fly={{ y: 10, duration: 200, delay: 100 }}
    out:fade
    style="position:absolute; top:{yPosition}px; left:{xPosition}px"
    bind:clientWidth={tooltipWidth}
  >
    <div class="title-line">
      <h1>{hoveredData.data.name}</h1>
      <span class="bank">{hoveredData.parent?.data?.name ?? "—"}</span>
    </div>
    <p class="value">
      <em
        >Beneficios: {hoveredData.value.toLocaleString("es-ES", {
          useGrouping: true,
        })}M€</em
      >
    </p>
  </div>
{/if}

<style>
  .tooltip {
    background-color: white;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    padding: 8px;
    border-radius: 4px;
    max-width: 220px;
    pointer-events: none;
    transition:
      top 50ms ease,
      left 50ms ease;
    border: 1px solid #ddd;
  }

  .title-line {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    gap: 6px;
  }

  h1 {
    font-size: 0.9rem;
    font-weight: bold;
    color: #222;
    margin: 0;
  }

  .bank {
    background-color: #f28b79;
    color: white;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 600;
    white-space: nowrap;
    user-select: none;
    border: 1px solid rgba(0, 0, 0, 0.15);
    flex-shrink: 0;
  }

  .value {
    font-size: 0.8rem;
    color: #444;
    margin: 4px 0 0 0;
  }

  .value em {
    font-style: italic;
  }
</style>
