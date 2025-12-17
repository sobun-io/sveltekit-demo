<script>
  import { onDestroy, onMount } from 'svelte';
  import CreativeEngine from '@cesdk/engine';
  import { removeBackground } from '@imgly/background-removal';
	import { page } from '$app/stores';

  /** @type {HTMLDivElement | null} */
  let canvasContainer = null;
  /** @type {any} */
  let engine = null;
  /** @type {number | null} */
  let imageBlockId = null;
  /** @type {number | null} */
  let videoBlockId = null;
  /** @type {number | null} */
  let pageBlockId = null;

  const engineConfig = {
    license: 'YOUR_CESDK_LICENSE_KEY',
  };
  /** @type {{ fillId: number | null; previewUrl: string; objectUrl: string }} */
  let fillState = {
    fillId: null,
    previewUrl: '',
    objectUrl: ''
  };
  let originalImageUri = '';
  let thumbnailEngine = null;

  // Shared removal configuration so quality tweaks live in one place
  const backgroundConfig = {
    output: { format: 'image/png', quality: 1 },
    model: 'isnet'
  };

  function updateFillState(patch) {
    fillState = { ...fillState, ...patch };
  }

  function withBlobUrl(blob, currentUrl, onApply) {
    if (currentUrl) {
      URL.revokeObjectURL(currentUrl);
    }
    const nextUrl = URL.createObjectURL(blob);
    onApply(nextUrl);
    return nextUrl;
  }

  function disposeObjectUrls(...urls) {
    for (const url of urls) {
      if (url) {
        URL.revokeObjectURL(url);
      }
    }
  }

  function applyPreview(blob) {
    withBlobUrl(blob, fillState.previewUrl, (url) => {
      updateFillState({ previewUrl: url });
    });
  }

  function applyFill(blob) {
    if (!engine || fillState.fillId == null) return;
    return withBlobUrl(blob, fillState.objectUrl, (url) => {
      
      engine.block.setString(fillState.fillId, 'fill/image/imageFileURI', url);
      updateFillState({ objectUrl: url });
    });
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

  function toPdfBlob(data) {
    return data instanceof Blob ? data : new Blob([data], { type: 'application/pdf' });
  }

  async function exportPageAsPdf(filename) {
    if (!engine || pageBlockId == null) return;
    const exportResult = await engine.block.export(pageBlockId, 'application/pdf');
    downloadBlob(toPdfBlob(exportResult), filename);
  }

  async function getThumbnailEngine() {
    if (thumbnailEngine) return thumbnailEngine;
    thumbnailEngine = await CreativeEngine.init({ ...engineConfig, container: null });
    return thumbnailEngine;
  }

  async function generateThumbnail(sceneData) {
    const thumbEngine = await getThumbnailEngine();
    await thumbEngine.scene.loadFromString(sceneData);
    const [page] = thumbEngine.scene.getPages();
    if (!page) return null;

    const thumbnail = await thumbEngine.block.export(page, 'image/jpeg', {
      targetWidth: 200,
      targetHeight: 200,
      quality: 0.7
    });

    return thumbnail instanceof Blob ? thumbnail : new Blob([thumbnail], { type: 'image/jpeg' });
  }

  function reportBatchMetrics(batchMetrics) {
    const entry = {
      timestamp: new Date().toISOString(),
      ...batchMetrics
    };
    console.table([entry]);
    return fetch('/api/logs', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(entry)
    }).catch((error) => {
      console.warn('Failed to report batch metrics', error);
    });
  }

  onMount(async () => {
    engine = await CreativeEngine.init(engineConfig);

    if (canvasContainer && engine.element) {
      canvasContainer.appendChild(engine.element);
    }

    let scene = engine.scene.get();
    if (!scene) {
      scene = engine.scene.create();
      const page = engine.block.create('page');
      engine.block.appendChild(scene, page);
    }

    const [page] = engine.block.findByType('page');
    const targetPage = page ?? engine.block.create('page');
    if (!page) {
      engine.block.appendChild(scene, targetPage);
    }
    pageBlockId = targetPage;

    imageBlockId = engine.block.create('graphic');
    engine.block.setShape(imageBlockId, engine.block.createShape('rect'));
    const fillId = engine.block.createFill('image');
    updateFillState({ fillId });
    engine.block.setFill(imageBlockId, fillId);

    const demoImageUrl = new URL(
      `${import.meta.env.BASE_URL}green-screen.png`,
      window.location.origin
    ).href;
    originalImageUri = demoImageUrl;
    engine.block.setString(fillId, 'fill/image/imageFileURI', demoImageUrl);

    engine.block.appendChild(targetPage, imageBlockId);
    engine.scene.zoomToBlock(targetPage);
  });


  onDestroy(() => {
    disposeObjectUrls(fillState.previewUrl, fillState.objectUrl);
    updateFillState({ previewUrl: '', objectUrl: '', fillId: null });
    engine?.dispose();
    thumbnailEngine?.dispose();
    engine = null;
    thumbnailEngine = null;
  });

  async function newScene() {
  if (!engine) return;

  // Clear object URLs and local state
  disposeObjectUrls(fillState.previewUrl, fillState.objectUrl);
  updateFillState({ previewUrl: '', objectUrl: '', fillId: null });
  imageBlockId = null;
  videoBlockId = null;

  // Destroy existing scene and create a fresh empty one
  const currentScene = engine.scene.get();
  if (currentScene) {
    engine.block.destroy(currentScene);
  }


  const newScene = engine.scene.create();
  const newPage = engine.block.create('page');
  engine.block.appendChild(newScene, newPage);
  pageBlockId = newPage;

  // If you want to keep the engine element visible, zoom to the new page
  engine.scene.zoomToBlock(newPage);
  }

async function verticalLayout() {
    if (!engine) return;

    // sceneLayout expects a string (Free|VerticalStack|HorizontalStack|DepthStack), not an object
    const verticalScene = engine.scene.create("VerticalStack");
    const [stack] = engine.block.findByType('stack');
    // Pages go in the stack
    const page1 = engine.block.create('page');
    engine.block.setWidth(page1, 800);
    engine.block.setHeight(page1, 600);
    engine.block.appendChild(stack, page1);

    const page2 = engine.block.create('page');
    engine.block.setWidth(page2, 800);
    engine.block.setHeight(page2, 600);
    engine.block.appendChild(stack, page2);

    engine.block.setFloat(stack, 'stack/spacing', 100);
    //engine.block.getFloat(stack, 'stack/spacing');
    console.log('Stack spacing is', engine.block.getFloat(stack, 'stack/spacing'));
    //engine.block.setBool(stack, 'stack/spacingInScreenspace', true);
    const layout = engine.scene.getLayout(verticalScene);
    engine.scene.setLayout('Free');
    engine.block.setFloat(page1, 'position/x', 100);
    engine.block.setFloat(page1, 'position/y', 100);
    engine.block.setFloat(page2, 'position/x', 100);
    engine.block.setFloat(page2, 'position/y', 800);
    // Ensure we have a fill to reuse across blocks
    let fillId = fillState.fillId;
    if (fillId == null) {
      fillId = engine.block.createFill('image');
      updateFillState({ fillId });
      engine.block.setString(fillId, 'fill/image/imageFileURI', originalImageUri);
    }

    // Add blocks
    //const block1 = engine.block.create('graphic');
    //engine.block.setShape(block1, engine.block.createShape('rect'));
    //engine.block.setFill(block1, fillId);
    //engine.block.setPositionX(block1, 100);
    //engine.block.setPositionY(block1, 100);
    //engine.block.appendChild(page, block1);

    const block2 = engine.block.create('graphic');
    engine.block.setShape(block2, engine.block.createShape('rect'));
    engine.block.setFill(block2, fillId);
    engine.block.setPositionX(block2, 100);
    engine.block.setPositionY(block2, 100);
    engine.block.appendChild(page2, block2);

    //Reorder pages in the stack
    // Store existing stack
    const [stackScene] = engine.block.findByType('stack');
    // Move page #1 to index 1 (after page 2)
    //engine.block.insertChild(stackScene, page1, 1); // Insert at index 2

    //const [stackScene] = engine.block.findByType('stack');
    //const newPage = engine.block.create('page');
    //engine.block.setWidth(newPage, 800);
    //engine.block.setHeight(newPage, 600);
    //engine.block.appendChild(stackScene, newPage);

    //const block3 = engine.block.create('graphic');
    //engine.block.setFill(block3, fillId);
    //engine.block.appendChild(scene, block3);

    //const block1 = [engine.block.create('graphic'), engine.block.create('graphic'), engine.block.create('graphic')];
    //for (const block of blocks) {
      //engine.block.setShape(block, engine.block.createShape('rect'));
      //engine.block.setFill(block, fillId);
      //engine.block.appendChild(page, block);
    //}

    //engine.scene.zoomToBlock(page);
  }

  async function magazineLayout() {
    const magazine = engine.scene.create('HorizontalStack');
    const [stack] = engine.block.findByType('stack');

    // Pages go in the stack
    const page1 = engine.block.create('page');
    engine.block.setWidth(page1, 800);
    engine.block.setHeight(page1, 1200);
    engine.block.appendChild(stack, page1);

    const page2 = engine.block.create('page');
    engine.block.setWidth(page2, 800);
    engine.block.setHeight(page2, 1200);
    engine.block.appendChild(stack, page2);

    engine.block.setFloat(stack, 'stack/spacing', 0);

    // Ensure we have a fill to reuse across blocks
    let fillId = fillState.fillId;
    if (fillId == null) {
      fillId = engine.block.createFill('image');
      updateFillState({ fillId });
      engine.block.setString(fillId, 'fill/image/imageFileURI', originalImageUri);
    }

    const cover = engine.block.create('graphic');
    engine.block.setShape(cover, engine.block.createShape('star'));
    engine.block.setFill(cover, fillId);
    engine.block.scale(cover, 6.0);
    engine.block.setPositionX(cover, 100);
    engine.block.setPositionY(cover, 350);
    engine.block.appendChild(page2, cover);

    const title = engine.block.create('text');
    engine.block.replaceText(title, 'Meow Magazine');
    engine.block.scale(title, 2.0);
    engine.block.setPositionX(title, 100);
    engine.block.setPositionY(title, 350);
    engine.block.appendChild(page2, title);

    const back = engine.block.create('graphic');
    engine.block.setFill(back, fillId);
    engine.block.setShape(back, engine.block.createShape('rect'));
    engine.block.appendChild(page1, back);
    engine.block.fillParent(back, true);

  
    



  }

  async function autoSize() {
    // Fill 50% of parent width, 100% of parent height
    if (!engine || imageBlockId == null) return;
    engine.block.setSize(imageBlockId, 50, 100, { sizeMode: 'Auto' });
    //engine.block.setWidthMode(imageBlockId, 'Percent');
    //engine.block.setHeightMode(imageBlockId, 'Percent');
    //engine.block.setWidth(imageBlockId, 0.5);
    //engine.block.setHeight(imageBlockId, 1.0);

    // Get computed dimensions (actual pixel values after layout)
    const computedWidth = engine.block.getFloat(imageBlockId,"lastFrame/width");
    const computedHeight = engine.block.getFloat(imageBlockId, "lastFrame/height");

    console.log(`Rendered size: ${computedWidth} x ${computedHeight}`);
  }

  async function addBg() {
    if (!engine) return;
    if (pageBlockId == null) {
      const scene = engine.scene.get() ?? engine.scene.create();
      const [existingPage] = engine.block.findByType('page');
      pageBlockId = existingPage ?? engine.block.create('page');
      if (!existingPage) {
        engine.block.appendChild(scene, pageBlockId);
      }
    }
    const bgBlockUrl = new URL(
      `${import.meta.env.BASE_URL}bg.jpg`,
      window.location.origin
    ).href;

    // Create background image that fills page
    // create image fill
    const imageFill = engine.block.createFill('image');
    engine.block.setString(imageFill, 'fill/image/imageFileURI', bgBlockUrl);

    // apply fill to the page itself
    engine.block.setFill(pageBlockId, imageFill);
    engine.block.setContentFillMode(pageBlockId, 'Cover'); // optional: fit behavior

    //const bgBlock = engine.block.create('graphic');
    //engine.block.setShape(bgBlock, engine.block.createShape('rect'));
    //const imageFill = engine.block.createFill('image');
    //engine.block.setString(imageFill, 'fill/image/imageFileURI', bgBlockUrl);
    //engine.block.setFill(bgBlock, pageBlockId, imageFill);
    //engine.block.setContentFillMode(bgBlock, 'Cover');

    // Add to page
    //engine.block.appendChild(pageBlockId, bgBlock);
    //engine.block.fillParent(bgBlock, true);

    // Send to back so other content appears on top
    //engine.block.sendToBack(bgBlock);
    //console.log('Added page background', { pageBlockId, bgBlockUrl, bgBlock });

  }

  async function flipImage() {
    if (!engine || imageBlockId == null) return;
    engine.block.setFlipHorizontal(imageBlockId, true);
    engine.block.setFlipVertical(imageBlockId, true);
  }

  async function resetFlip() {
    if (!engine || imageBlockId == null) return;
    const isFlipped = engine.block.getFlipHorizontal(imageBlockId);
    engine.block.setFlipHorizontal(imageBlockId, !isFlipped);
  }

  async function resizeImage() {
    if (!engine || imageBlockId == null) return;
    engine.block.setWidthMode(imageBlockId, 'Percent');
    engine.block.setHeightMode(imageBlockId, 'Percent');
    engine.block.setWidth(imageBlockId, 1);
    engine.block.setHeight(imageBlockId, 0.5625);
  }

  async function removeBackgroundForFill(engineInstance, state, config = backgroundConfig) {
    if (!engineInstance || state.fillId == null) return null;
    const source = engineInstance.block.getString(state.fillId, 'fill/image/imageFileURI');
    if (!source) return null;
    const result = await removeBackground(source, config);
    return result;
  }

  async function removeBg() {
    try {
      const newBlob = await removeBackgroundForFill(engine, fillState);
      if (!newBlob) return;

      applyPreview(newBlob);
      applyFill(newBlob);
    } catch (error) {
      console.error('Failed to remove background', error);
    }
  }
  async function rotateImage() {
    // Rotate the image by 90°
    //engine.block.setRotation(imageBlockId, Math.PI / 2);
    const toRadians = (degrees) => (degrees * Math.PI) / 180;
    const toDegrees = (radians) => (radians * 180) / Math.PI;

    const targetRadians = toRadians(45); // 0.785398...
    engine.block.setRotation(imageBlockId, targetRadians);

    console.log('Image rotation is', toDegrees(targetRadians), '°');
  }

  async function greenScreen() {
    if (!engine || imageBlockId == null || pageBlockId == null) return;

    /* Previous approach kept for reference:
    // Apply Green Screen effect
    const chromaKeyEffect = engine.block.createEffect('green_screen');
    // Key out blue instead of green
    //engine.block.setFloat(chromaKeyEffect, 'green_screen/colorMatch', 0.25);
    engine.block.setColor(
      chromaKeyEffect,
      'green_screen/fromColor',
      { r: 0.3608, g: 0.6314, b: 0.4157, a: 1 }
    );
    //engine.block.setFloat(chromaKeyEffect, 'green_screen/smoothness', 0.1);
    //engine.block.setFloat(chromaKeyEffect, 'green_screen/spill', 0.2);
    //engine.block.appendEffect(imageBlockId, chromaKeyEffect);
    
    // Create virtual background
    const bgBlock = engine.block.create('graphic');
    engine.block.setFill(bgBlock, backgroundImageFill);
    engine.block.appendChild(page, bgBlock);
    engine.block.sendToBack(bgBlock);
    const videoBlock = engine.block.create('graphic');
    engine.block.setFill(videoBlock, videoFill);
    const chromaKey = engine.block.createEffect('green_screen');
    */

    const chromaKeyEffect = engine.block.createEffect('green_screen');
    engine.block.setColor(
      chromaKeyEffect,
      'effect/green_screen/fromColor',
      { r: 0.3608, g: 0.6314, b: 0.4157, a: 1 }
    );
    engine.block.appendEffect(imageBlockId, chromaKeyEffect);

    if (!videoBlockId) {
      videoBlockId = engine.block.create('graphic');
      engine.block.setShape(videoBlockId, engine.block.createShape('rect'));
      const videoFill = engine.block.createFill('video');
      engine.block.setString(
        videoFill,
        'fill/video/fileURI',
        'https://cdn.img.ly/assets/demo/v2/ly.img.video/videos/pexels-drone-footage-of-a-surfer-barrelling-a-wave-12715991.mp4'
      );
      engine.block.setFill(videoBlockId, videoFill);
      engine.block.setWidthMode(videoBlockId, 'Percent');
      engine.block.setHeightMode(videoBlockId, 'Percent');
      engine.block.setWidth(videoBlockId, 1);
      engine.block.setHeight(videoBlockId, 1);
      engine.block.appendChild(pageBlockId, videoBlockId);
      engine.block.sendToBack(videoBlockId);
      engine.block.bringToFront(imageBlockId);
    }
  }

  async function batchProcess() {
    if (!engine) return;
    const startedAt = performance.now();
    const sceneData = await engine.scene.saveToString();
    const thumbnail = await generateThumbnail(sceneData);
    if (thumbnail) {
      applyPreview(thumbnail);
    }
    reportBatchMetrics({
      action: 'thumbnail_generation',
      durationMs: Math.round(performance.now() - startedAt),
      success: Boolean(thumbnail)
    });
  }

  async function batchExport() {
    if (!engine || pageBlockId == null || fillState.fillId == null) return;

    const currentUri = engine.block.getString(fillState.fillId, 'fill/image/imageFileURI');

    if (originalImageUri) {
      engine.block.setString(fillState.fillId, 'fill/image/imageFileURI', originalImageUri);
      await exportPageAsPdf('image-original.pdf');
    }

    if (currentUri) {
      engine.block.setString(fillState.fillId, 'fill/image/imageFileURI', currentUri);
      await exportPageAsPdf('image-edited.pdf');
    }
  }

  async function exportCompressedImage() {
    if (!engine || imageBlockId == null) return;

    const scaledImage = {
      mimeType: 'image/png',
      pngCompressionLevel: 5,
      targetWidth: 740,
      targetHeight: 740
    };

    const pngData = await engine.block.export(imageBlockId, scaledImage);

    downloadBlob(pngData,
      'image-compressed.png');
  }

</script>
  
<div class="editor-container">
  <div class="canvas-container" bind:this="{canvasContainer}"></div>
  <div class="button-overlay">
    <button on:click={() => autoSize()}>Auto-size</button>
    <button on:click={() => addBg()}>Add Background</button>
    <button on:click={() => rotateImage()}>Rotate</button>
    <button on:click="{flipImage}">Flip</button>
    <button on:click="{resetFlip}">Reset Flip</button>
    <button on:click="{resizeImage}">Resize</button>
    <button on:click={() => removeBg()}>Remove BG</button>
    <button on:click={() => greenScreen()}>Green Screen</button>
    <button on:click={() => batchProcess()}>Generate Thumbnail</button>
    <button on:click={() => batchExport()}>Batch Export</button>
    <button on:click={() => exportCompressedImage()}>Compress + Export</button>
    <button on:click={() => newScene()}>New Scene</button>
    <button on:click={() => verticalLayout()}>Vertical Layout</button>
    <button on:click={() => magazineLayout()}>Magazine Layout</button>


    <button on:click={() => exportImage()}>Export</button>
    
  </div>
</div>

{#if fillState.previewUrl}
  <div class="preview">
    <h3>Export preview</h3>
    <img src="{fillState.previewUrl}" alt="Image has been exported" />
  </div>
{/if}

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
  
    .button-overlay {
      position: absolute;
      top: 20px;
      left: 20px;
    }
  
    .button-overlay button {
      border-radius: 8px;
      border: 1px solid #ccc;
      padding: 0.6em 1.2em;
      font-size: 1em;
      font-weight: 500;
      font-family: inherit;
      background-color: #ffffff;
      color: #1a1a1a;
      cursor: pointer;
      transition:
        border-color 0.25s,
        box-shadow 0.25s;
      box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
    }
  
    .button-overlay button:hover {
      border-color: #646cff;
      box-shadow: 0 4px 10px rgba(100, 108, 255, 0.2);
    }
  
    .button-overlay button:focus,
    .button-overlay button:focus-visible {
      outline: 2px solid #646cff;
      outline-offset: 2px;
    }

    .preview {
      margin: 16px;
      padding: 12px;
      background: #ffffff;
      border: 1px solid #e5e7eb;
      border-radius: 12px;
      max-width: 360px;
      box-shadow: 0 6px 16px rgba(15, 23, 42, 0.12);
    }

    .preview h3 {
      margin: 0 0 8px;
      font-size: 0.95rem;
      font-weight: 600;
      color: #1f2937;
    }

    .preview img {
      width: 100%;
      display: block;
      border-radius: 8px;
    }
  </style>
