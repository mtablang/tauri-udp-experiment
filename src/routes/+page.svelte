<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import { onMount } from 'svelte';
  import { bind, send, unbind } from '@kuyoonjo/tauri-plugin-udp';
  import { listen } from '@tauri-apps/api/event';

  import { readFile, BaseDirectory } from '@tauri-apps/plugin-fs';

  import { getCurrentWindow } from '@tauri-apps/api/window';

  // let name = $state('');
  // let greetMsg = $state('');

  let setupMode = $state(true);
  let showSelectImage = $state(false); // whether video or image should be shown
  let showScreenSaverAfterQueue = $state(true); // whether screensaver or image should be shown after video queue is finished
  let broadcastAddress = $state('0.0.0.0');
  let port = $state('5001');
  let filesDirectory = $state('Documents');
  let baseDir = $state(BaseDirectory.Document);
  // let videoDirectoryPath = $state('fms');
  let videoElement: HTMLVideoElement; // Reference to the <video> element
  let imageElement: HTMLImageElement; // Reference to the <img> element

  let videoQueue: string[] = $state([]); // Array of video titles
  let currentIndex = $state(0);

  onMount(async () => {});

  const finishSetup = async () => {
    const id = 'unique-id';
    // await bind(id, '0.0.0.0:8080');
    console.log(`binding to ${$state.snapshot(broadcastAddress)}:${$state.snapshot(port)}`);
    await bind(id, `${broadcastAddress}:${port}`);

    switch (filesDirectory) {
      case 'Documents':
        baseDir = BaseDirectory.Document;
        break;
      case 'Downloads':
        baseDir = BaseDirectory.Download;
        break;
      case 'Desktop':
        baseDir = BaseDirectory.Desktop;
        break;
    }

    // await displayImage(`_select`);

    setupMode = false;

    await listen('plugin://udp', (x) => {
      const payloadDataJSON = String.fromCharCode(...x.payload.data);
      console.log(`payloadDataJSON`, payloadDataJSON);
      const payloadData = JSON.parse(payloadDataJSON)[0];
      console.log(`payloadData`, payloadData);

      if (payloadData.AddedProduct) {
        showSelectImage = false; // reset to false if ever it's been set as true before this

        let videoTitles = [];
        if (payloadData.Products && payloadData.Products.length > 0) {
          console.log(`payloadData.Products`, payloadData.Products);

          videoTitles = payloadData.Products.map((product) => {
            return product.ProductCode;
          });
          console.log(`[listen] videoTitles`, videoTitles);
        }

        // playVideo(videoTitle);

        if (payloadData.IsScreenSaverOn) {
          console.log(`IsScreenSaverOn = true so Going to show _screensaver video after the queue`);
          showScreenSaverAfterQueue = true;
        } else {
          console.log(`IsScreenSaverOn = false so Going to show _select image after the queue`);
          showScreenSaverAfterQueue = false;
        }
        startVideoQueue([`_intro`, ...videoTitles, `_outro`]);
      } else if (payloadData.IsScreenSaverOn) {
        console.log(`showing _screensaver right away`);
        showSelectImage = false;
        startVideoQueue(['_screensaver']);
      } else if (!payloadData.IsScreenSaverOn && !payloadData.AddedProduct) {
        console.log(`showing _select image right away`);
        displayImage(`_select`);
        showSelectImage = true;
      }
    });
  };

  // Function to start playing a queue of videos
  const startVideoQueue = async (titles: string[]) => {
    if (!titles.length) return;
    videoQueue = titles;
    currentIndex = 0;
    await playNextVideo();
  };

  const playNextVideo = async () => {
    console.log(
      '[playNextVideo] videoQueue and currentIndex',
      $state.snapshot(videoQueue),
      $state.snapshot(currentIndex)
    );
    if (currentIndex < videoQueue.length) {
      const nextVideoTitle = videoQueue[currentIndex];
      currentIndex++;
      await playVideo(nextVideoTitle);
    } else {
      // Queue finished!
      console.log(
        'All videos in the queue have been played. Going to play _screensaver or _select'
      );

      // videoMode = false;
      if (showScreenSaverAfterQueue) {
        await playVideo(`_screensaver`);
      } else {
        showSelectImage = true;
        await displayImage(`_select`);
      }
    }
  };

  const playVideo = async (videoTitle: string) => {
    console.log(`[playVideo] Now gonna play ${videoTitle}`);
    const file = await readFile(`${videoTitle}.mp4`, {
      baseDir: baseDir,
    });

    // Convert binary data to a Blob URL
    const blob = new Blob([new Uint8Array(file)], { type: 'video/mp4' });

    if (videoElement) {
      videoElement.src = URL.createObjectURL(blob);

      console.log(`videoElement.src`, videoElement.src);

      videoElement
        .play()
        .catch((err) => {
          console.error('video playback error:', err);
        })
        .then(() => {
          getCurrentWindow().setFullscreen(true);
        });
    }
  };

  const displayImage = async (imageTitle: string) => {
    console.log(`[displayImage] Now gonna display image ${imageTitle}`);

    const file = await readFile(`${imageTitle}.jpg`, {
      baseDir: baseDir,
    });

    // Convert binary data to a Blob URL
    const blob = new Blob([new Uint8Array(file)], { type: 'image/jpeg' });

    console.log(`file`, file);
    if (imageElement) {
      console.log(`imageElement`, $state.snapshot(imageElement));
      imageElement.src = URL.createObjectURL(blob);
      // console.log(`imageElement.src`, imageElement.src);
      getCurrentWindow().setFullscreen(true);
    } else {
      console.log(`no imageElement`);
    }
  };
</script>

<main class="container">
  {#if setupMode}
    <div class="setup">
      Broadcast address <input bind:value={broadcastAddress} />
      <br />
      Port <input bind:value={port} />
      <br />
      Video directory
      <select bind:value={filesDirectory} name="pets" id="pet-select">
        <option value="Desktop">Desktop</option>
        <option value="Documents">Documents Folder</option>
        <option value="Downloads">Downloads Folder</option>
      </select>
      <br />
      <!-- Path <input bind:value={videoDirectoryPath} /> -->
      <button onclick={finishSetup} class="finish-setup-button">Finish Setup</button>
    </div>
  {:else}
    <div class="media-player">
      {#if !showSelectImage}
        <video bind:this={videoElement} class="video" onended={playNextVideo}></video>
      {:else}
        <img bind:this={imageElement} class="image" alt="Select image" />
      {/if}
    </div>
  {/if}

  <!-- <h1>Welcome to Tauri + Svelte</h1>

  <div class="row">
    <a href="https://vitejs.dev" target="_blank">
      <img src="/vite.svg" class="logo vite" alt="Vite Logo" />
    </a>
    <a href="https://tauri.app" target="_blank">
      <img src="/tauri.svg" class="logo tauri" alt="Tauri Logo" />
    </a>
    <a href="https://kit.svelte.dev" target="_blank">
      <img src="/svelte.svg" class="logo svelte-kit" alt="SvelteKit Logo" />
    </a>
  </div>
  <p>Click on the Tauri, Vite, and SvelteKit logos to learn more.</p>

  <form class="row" onsubmit={greet}>
    <input id="greet-input" placeholder="Enter a name..." bind:value={name} />
    <button type="submit">Greet___</button>
  </form>
  <p>{greetMsg}</p> -->
</main>

<style>
  /* .logo.vite:hover {
    filter: drop-shadow(0 0 2em #747bff);
  }

  .logo.svelte-kit:hover {
    filter: drop-shadow(0 0 2em #ff3e00);
  } */

  :root {
    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;

    color: #0f0f0f;
    background-color: #f6f6f6;

    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
  }

  .container {
    margin: 0;
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
    height: 100vh;
    width: 100vw;
    /* background: hotpink; */
    background: black;
  }

  .setup {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
    background: #eee;
  }
  .finish-setup-button {
    margin-top: 40px;
    font-size: 1em;
  }

  .media-player {
    height: 100%;
    width: 100%;
  }
  video,
  img {
    height: 100%;
    width: 100%;
    object-fit: contain;
  }
</style>
