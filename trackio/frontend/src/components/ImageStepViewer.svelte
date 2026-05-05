<script>
  import { getMetricValues, getMediaUrl, fetchMediaBlob, isStaticMode } from "../lib/api.js";

  let { project, run, metricName, maxWidth = "100%", maxHeight = 320 } = $props();

  let points = $state([]);
  let loading = $state(false);
  let error = $state(null);
  let selectedIndex = $state(0);
  let resolvedUrls = $state(new Map());

  let hasData = $derived(points.length > 0);
  let selectedPoint = $derived(hasData ? points[selectedIndex] : null);
  let selectedValue = $derived(selectedPoint?.value ?? null);
  let selectedFilePath = $derived(selectedValue?.file_path ?? null);
  let selectedCaption = $derived(selectedValue?.caption ?? null);
  let selectedBoxes = $derived(selectedValue?.boxes ?? null);

  async function resolveUrl(filePath) {
    if (!filePath) return "";
    if (resolvedUrls.has(filePath)) return resolvedUrls.get(filePath);

    let url;
    if (await isStaticMode()) {
      url = await fetchMediaBlob(filePath);
    } else {
      url = getMediaUrl(filePath);
    }
    resolvedUrls = new Map(resolvedUrls).set(filePath, url);
    return url;
  }

  let selectedUrl = $state("");

  $effect(() => {
    selectedFilePath;
    (async () => {
      selectedUrl = await resolveUrl(selectedFilePath);
    })();
  });

  async function load() {
    if (!project || !run || !metricName) {
      points = [];
      selectedIndex = 0;
      error = null;
      return;
    }

    loading = true;
    error = null;
    try {
      const rows = await getMetricValues(project, run, metricName);
      const images = (Array.isArray(rows) ? rows : []).filter(
        (r) => r?.value && typeof r.value === "object" && r.value._type === "trackio.image",
      );

      points = images;
      selectedIndex = images.length ? images.length - 1 : 0;
    } catch (e) {
      error = e;
      points = [];
      selectedIndex = 0;
    } finally {
      loading = false;
    }
  }

  $effect(() => {
    project;
    run;
    metricName;
    load();
  });

  function clampIndex(next) {
    const max = points.length - 1;
    if (max < 0) return 0;
    return Math.max(0, Math.min(max, next));
  }

  function onSliderInput(e) {
    selectedIndex = clampIndex(parseInt(e.currentTarget.value ?? "0", 10));
  }

  function formatStep(step) {
    if (step === null || step === undefined) return "";
    return String(step);
  }

  function listBoxLayers(boxes) {
    if (!boxes || typeof boxes !== "object" || Array.isArray(boxes)) return [];
    const keys = Object.keys(boxes);
    keys.sort();
    return keys;
  }

  function normalizeColor(c) {
    if (!c) return "#22c55e";
    if (typeof c !== "string") return "#22c55e";
    if (c.startsWith("#") || c.startsWith("rgb")) return c;
    return c;
  }

  function parseBox(b) {
    if (!b || typeof b !== "object") return null;
    const position = b.position;
    if (!position || typeof position !== "object") return null;
    const { minX, minY, maxX, maxY } = position;
    const x0 = Number(minX);
    const y0 = Number(minY);
    const x1 = Number(maxX);
    const y1 = Number(maxY);
    if ([x0, y0, x1, y1].some((v) => Number.isNaN(v))) return null;
    return {
      x: x0,
      y: y0,
      w: Math.max(0, x1 - x0),
      h: Math.max(0, y1 - y0),
      label: typeof b.class_id === "number" || typeof b.class_id === "string" ? String(b.class_id) : "",
      color: normalizeColor(b.box_caption || b.color),
    };
  }

  let overlayBoxes = $derived.by(() => {
    const layers = listBoxLayers(selectedBoxes);
    if (layers.length === 0) return [];
    const first = selectedBoxes[layers[0]];
    const data = first?.box_data;
    if (!Array.isArray(data)) return [];
    return data.map(parseBox).filter(Boolean);
  });
</script>

<div class="image-step-viewer">
  {#if loading}
    <div class="viewer-placeholder">Loading…</div>
  {:else if error}
    <div class="viewer-placeholder">Failed to load image metric.</div>
  {:else if !hasData}
    <div class="viewer-placeholder">No images logged for this metric.</div>
  {:else}
    <div class="viewer-header">
      <div class="viewer-title">{metricName}</div>
      <div class="viewer-meta">
        <span>Step: {formatStep(selectedPoint.step)}</span>
        <span class="viewer-meta-sep">·</span>
        <span>{selectedIndex + 1}/{points.length}</span>
      </div>
    </div>
    <div class="viewer-image-wrap" style:max-height={`${maxHeight}px`}>
      {#if selectedUrl}
        <div class="viewer-stage">
          <img
            src={selectedUrl}
            alt={selectedCaption || metricName}
            style:max-width={maxWidth}
            style:max-height={`${maxHeight}px`}
            loading="lazy"
          />
          {#if overlayBoxes.length > 0}
            <div class="bbox-overlay">
              {#each overlayBoxes as b}
                <div
                  class="bbox"
                  style:left={`${b.x * 100}%`}
                  style:top={`${b.y * 100}%`}
                  style:width={`${b.w * 100}%`}
                  style:height={`${b.h * 100}%`}
                  style:border-color={b.color}
                >
                  {#if b.label}
                    <div class="bbox-label" style:background={b.color}>{b.label}</div>
                  {/if}
                </div>
              {/each}
            </div>
          {/if}
        </div>
      {:else}
        <div class="viewer-placeholder">Loading image…</div>
      {/if}
    </div>
    {#if selectedCaption}
      <div class="viewer-caption">{selectedCaption}</div>
    {/if}
    <div class="viewer-controls">
      <input
        class="viewer-slider"
        type="range"
        min="0"
        max={Math.max(0, points.length - 1)}
        step="1"
        value={selectedIndex}
        oninput={onSliderInput}
      />
    </div>
  {/if}
</div>

<style>
  .image-step-viewer {
    display: flex;
    flex-direction: column;
    gap: 10px;
    width: 100%;
    padding: 12px;
    border: 1px solid rgba(148, 163, 184, 0.35);
    border-radius: 10px;
    background: rgba(15, 23, 42, 0.02);
  }
  :global(body.dark) .image-step-viewer {
    background: rgba(148, 163, 184, 0.08);
    border-color: rgba(148, 163, 184, 0.25);
  }
  .viewer-header {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    align-items: baseline;
  }
  .viewer-title {
    font-weight: 600;
    font-size: 14px;
    word-break: break-word;
  }
  .viewer-meta {
    font-size: 12px;
    color: #64748b;
    display: flex;
    align-items: center;
    gap: 6px;
    white-space: nowrap;
  }
  :global(body.dark) .viewer-meta {
    color: rgba(226, 232, 240, 0.75);
  }
  .viewer-meta-sep {
    opacity: 0.7;
  }
  .viewer-image-wrap {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    overflow: hidden;
    border-radius: 8px;
    background: rgba(148, 163, 184, 0.12);
  }
  :global(body.dark) .viewer-image-wrap {
    background: rgba(2, 6, 23, 0.35);
  }
  .viewer-image-wrap img {
    width: auto;
    height: auto;
    object-fit: contain;
    border-radius: 6px;
  }
  .viewer-stage {
    position: relative;
    display: inline-block;
    max-width: 100%;
    max-height: 100%;
  }
  .bbox-overlay {
    position: absolute;
    inset: 0;
    pointer-events: none;
  }
  .bbox {
    position: absolute;
    border: 2px solid #22c55e;
    box-sizing: border-box;
  }
  .bbox-label {
    position: absolute;
    top: -20px;
    left: 0;
    padding: 2px 6px;
    border-radius: 4px;
    color: white;
    font-size: 11px;
    line-height: 1.2;
  }
  .viewer-caption {
    font-size: 12px;
    color: #334155;
  }
  :global(body.dark) .viewer-caption {
    color: rgba(226, 232, 240, 0.85);
  }
  .viewer-controls {
    display: flex;
    align-items: center;
  }
  .viewer-slider {
    width: 100%;
  }
  .viewer-placeholder {
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 120px;
    font-size: 13px;
    color: #64748b;
  }
  :global(body.dark) .viewer-placeholder {
    color: rgba(226, 232, 240, 0.75);
  }
</style>
