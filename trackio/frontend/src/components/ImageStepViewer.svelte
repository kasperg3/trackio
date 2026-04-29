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
        <img
          src={selectedUrl}
          alt={selectedCaption || metricName}
          style:max-width={maxWidth}
          style:max-height={`${maxHeight}px`}
          loading="lazy"
        />
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

