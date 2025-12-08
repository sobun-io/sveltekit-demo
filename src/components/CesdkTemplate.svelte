<script>
  import { onDestroy, onMount } from 'svelte';
  import CreativeEngine from '@cesdk/engine';

  /** @type {HTMLDivElement | null} */
  let canvasContainer = null;
  /** @type {any} */
  let engine = null;
  const sceneUrl = 'https://cdn.img.ly/assets/demo/v2/ly.img.template/templates/cesdk_postcard_2.scene';
  let textVariables = {
    first_name: 'Ada',
    last_name: 'Lovelace',
    address: '12 Analytical Avenue',
    city: 'London',
  };

  onMount(async () => {
    const config = {
      license: '<YOUR_CESDK_LICENSE_KEY>',
    };

    engine = await CreativeEngine.init(config);
    if (canvasContainer && engine.element) {
      canvasContainer.appendChild(engine.element);
    }

    try {
      await engine.scene.loadFromURL(sceneUrl);
      applyTextVariables();
    } catch (error) {
      console.warn('Failed to load template scene.', error);
    }
  });

  onDestroy(() => {
    engine?.dispose();
    engine = null;
  });

  function applyTextVariables() {
    if (!engine) return;
    for (const [key, value] of Object.entries(textVariables)) {
      engine.variable.setString(key, value ?? '');
    }
  }

  function handleInput(key, event) {
    textVariables = {
      ...textVariables,
      [key]: event.target.value,
    };
    applyTextVariables();
  }

  function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
  }

  async function batchFill() {
    const records = await fetch('/api/cesdk-datasets.json').then((r) => r.json());
    if (!records.length || !records[0]?.variables) return;

    textVariables = {
      ...textVariables,
      ...records[0].variables,
    };
    applyTextVariables();
  }

  async function batchExport() {
    if (!engine) return;
    applyTextVariables();

    const scene = engine.scene.get();
    if (!scene) return;

    const exportResult = await engine.block.export(scene, 'application/pdf');
    const blob =
      exportResult instanceof Blob
        ? exportResult
        : new Blob([exportResult], { type: 'application/pdf' });

    downloadBlob(blob, 'template.pdf');
  }

</script>

<div class="editor-container">
  <div class="canvas-container" bind:this="{canvasContainer}"></div>
  <div class="form-overlay">
    <form on:submit|preventDefault>
      <h2>Template Variables</h2>
      {#each Object.entries(textVariables) as [key, value]}
        <label for="{key}">
          <span>{key.replace(/_/g, ' ')}</span>
          <input
            id="{key}"
            type="text"
            value="{textVariables[key] ?? ''}"
            on:input="{event => handleInput(key, event)}"
          />
        </label>
      {/each}
      <button type="button" on:click="{applyTextVariables}">Apply</button>
      <button type="button" on:click="{batchExport}">Download PDF</button>
      <button type="button" on:click="{batchFill}">Fill from API</button>
      
    </form>
  </div>
</div>

<style>
  .editor-container {
    width: 100vw;
    height: 100vh;
    position: relative;
  }

  .canvas-container {
    width: 100%;
    height: 100%;
  }

  .form-overlay {
    position: absolute;
    top: 16px;
    right: 16px;
    width: 240px;
    background: rgba(255, 255, 255, 0.96);
    border: 1px solid #d2d6dc;
    border-radius: 12px;
    padding: 16px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.14);
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .form-overlay h2 {
    margin: 0;
    font-size: 1rem;
    font-weight: 600;
    color: #1f2937;
  }

  .form-overlay label {
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 0.85rem;
    text-transform: capitalize;
    color: #4b5563;
  }

  .form-overlay input {
    border-radius: 8px;
    border: 1px solid #cbd5f5;
    padding: 0.45em 0.75em;
    font-size: 0.9rem;
    transition: border-color 0.2s, box-shadow 0.2s;
  }

  .form-overlay input:focus {
    outline: none;
    border-color: #6366f1;
    box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.25);
  }

  .form-overlay button {
    border-radius: 8px;
    border: none;
    background: #6366f1;
    color: #fff;
    font-weight: 600;
    font-size: 0.9rem;
    padding: 0.6em 1em;
    cursor: pointer;
    transition: background 0.2s ease-in-out;
  }

  .form-overlay button:hover {
    background: #4f46e5;
  }

  .form-overlay button:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.35);
  }

</style>
