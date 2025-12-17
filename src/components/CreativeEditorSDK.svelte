<script>
    import CreativeEditorSDK from '@cesdk/cesdk-js';
    import CreativeEngine from '@cesdk/engine';
    
    //import BackgroundRemovalPlugin from '@imgly/plugin-background-removal-web';
    import { onDestroy, onMount } from 'svelte';
  
    // reference to the container HTML element where CE.SDK will be initialized
    /** @type {HTMLDivElement | null} */
    let container = null;
    // where to keep track of the CE.SDK instance
    /** @type {any}*/
    let cesdk = null;
    /** @type {CreativeEngine | null}*/
    let engine = null;
  
    //const DEFAULT_TEMPLATE_URL = 'https://cdn.img.ly/assets/demo/v2/ly.img.template/templates/cesdk_postcard_2.scene';

    // deafult CreativeEditor SDK configuration
    /** @type {Record<string, any>}*/
    export let config = {};

    /** @type {'local'}*/
    const ON_UPLOAD = 'local';
    const defaultConfig = {
      license: 'YOUR_CESDKz', // replace it with a valid CE.SDK license key
      callbacks: { onUpload: ON_UPLOAD }, // enable local file uploads in the Asset Library
      // other default configs...
    };
  
    // accessing the component's props
    //const { el, children, class: _, config, ...props } = $props();
  
    // hook to initialize the CreativeEditorSDK component
    onMount(() => {
      // integrate the configs read from props with the default ones
      const ceSDKConfig = {
        ...defaultConfig,
        ...config,
      };

      if (!container) return;
  
      try {
        // initialize the CreativeEditorSDK instance in the container element
        // using the given config
        CreativeEditorSDK.create(container, ceSDKConfig).then(async instance => {
          cesdk = instance;
          engine = instance.engine;
  
          // do something with the instance of CreativeEditor SDK (e.g., populate
          // the asset library with default / demo asset sources)
          await Promise.all([
            cesdk.addDefaultAssetSources(),
            cesdk.addDemoAssetSources({ sceneMode: 'Design' }),
          ]);
          
          // Add custom layout button
          const caseAssetPath = (path) => `${import.meta.env.BASE_URL}${path.replace(/^\//, '')}`;
          // Remote assets hosted in the CESDK web examples repo
          const layoutBaseUrl = 'https://raw.githubusercontent.com/imgly/cesdk-web-examples/main/showcase-layouts/public/cases/layouts/';
          const layoutManifestUrl = `${layoutBaseUrl}CustomLayouts.json`;

          try {
            const layoutResponse = await fetch(layoutManifestUrl);
            if (!layoutResponse.ok) {
              throw new Error(`Failed to load layouts manifest (HTTP ${layoutResponse.status})`);
            }
            const layoutManifest = await layoutResponse.json();

            await cesdk.engine.asset.addLocalAssetSourceFromJSONString(
              JSON.stringify(layoutManifest),
              layoutBaseUrl
            );

            // Make the custom layouts visible in the Asset Library
            cesdk.ui.addAssetLibraryEntry({
              id: 'ly.img.layouts',
              sourceIds: ['ly.img.layouts'],
              title: 'Layouts',
              icon: caseAssetPath('/collage-small.svg'),
              gridBackgroundType: 'cover'
            });

            cesdk.ui.setDockOrder(
              [
        {
          id: 'ly.img.assetLibrary.dock',
          key: 'ly.img.layouts',
          label: 'libraries.ly.img.layouts.label',
          icon: ({ iconSize }) => {
            return iconSize === 'normal'
              ? caseAssetPath('/collage-small.svg')
              : caseAssetPath('/collage-large.svg');
          },
          entries: ['ly.img.layouts']
        },
        'ly.img.separator',
        ...cesdk.ui
          .getDockOrder()
          .filter(({ key }) => !['ly.img.template'].includes(key))
      ]
            );
          } catch (error) {
            console.warn('Failed to add custom layout source', error);
          }

          

          //await cesdk.addPlugin(
            //BackgroundRemovalPlugin({
              //ui: {
                //locations: ['canvasMenu'],
              //},
            //}),  
          //);
          await cesdk.createDesignScene();
          // Hide rotate handles on the control gizmo once the scene is ready
          //engine.editor.setSetting('controlGizmo/showRotateHandles', false);
                          
  
          // load a predefined template; fallback to a blank scene when loading fails
          //try {
            //await cesdk.engine?.scene?.loadFromURL(DEFAULT_TEMPLATE_URL);
          //} catch (error) {
            //console.warn('Falling back to a blank scene because loading the default template failed.', error);
            //await cesdk.createDesignScene();
          //}
          //engine.editor?.setSetting('controlGizmo/showRotateHandles', false);
          
          // Register a custom button component that exports the current scene as PDF
          //instance.ui.registerComponent(
          //  'convert.nav',
          //  ({ builder: { Button }, engine }) => {
          //    Button('convert-to-pdf', {
          //      label: 'Convert To PDF',
          //      icon: '@imgly/Download',
          //      color: 'accent',
          //      onClick: async () => {
          //        // Export the current scene as a PDF blob
          //        const scene = engine.scene.get();
          //        const blob = await engine.block.export(scene, { mimeType: 'application/pdf' });
          //        // Trigger download of the PDF blob
          //        const element = document.createElement('a');
          //        const downloadUrl = window.URL.createObjectURL(blob);
          //        element.setAttribute('href', downloadUrl);
          //        element.setAttribute('download', 'converted.pdf');
          //        element.style.display = 'none';
          //        element.click();
          //        element.remove();
          //        window.URL.revokeObjectURL(downloadUrl);
          //      },
          //    });
          //  },
          //);
          // Add the custom button at the end of the navigation bar
          //instance.ui.setNavigationBarOrder([
          //  ...instance.ui.getNavigationBarOrder(),
          //  'convert.nav',
          //]);
        });
      } catch (err) {
        console.warn(`CreativeEditor SDK failed to mount.`, { err });
      }
      
    });
  
    // hook to clean up when the component unmounts
    onDestroy(() => {
      try {
        // dispose of the CE.SDK instance if it exists
        if (cesdk) {
          cesdk.dispose();
          cesdk = null;
        }
      } catch (err) {
        // log error if CreativeEditor SDK fails to unmount
        console.warn(`CreativeEditor SDK failed to unmount.`, { err });
      }
    });
  </script>
  
  <!-- the container HTML element where the CE.SDK editor will be mounted -->
  <div id="cesdk_container" bind:this="{container}"></div>
  
  <style>
    /* styling for the CE.SDK container element to take full viewport size */
    #cesdk_container {
      height: 100%;
      width: 100%;
    }
  </style>
