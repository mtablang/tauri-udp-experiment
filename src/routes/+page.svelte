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

  let video1: HTMLVideoElement | null | undefined = $state();
  let video2: HTMLVideoElement | null | undefined = $state();
  let currentVideo: HTMLVideoElement | null | undefined = $state();
  let isFading = false; // Flag to prevent multiple triggers

  // let showSelectScreen = false;
  // let playingScreensaver = false;
  // let videoQueue = [];
  // let currentIndex = 0;

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
        showScreenSaverAfterQueue = true;
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
    console.log(`[startVideoQueue] titles`, titles);
    if (!titles.length) return;
    videoQueue = titles;
    currentIndex = 0;
    await playNextVideo();
  };

  const handleTimeUpdate = () => {
    if (!currentVideo) return;

    const remainingTime = currentVideo.duration - currentVideo.currentTime;
    console.log(`[handleTimeUpdate] remainingTime`, remainingTime);

    if (remainingTime <= 0.3 && !isFading) {
      // if (remainingTime <= 0.3) {
      isFading = true; // Set flag to avoid repeated calls
      console.log(`[handleTimeUpdate] Now playing next video since <= .3 second`);
      playNextVideo();
      // isFading = true; // Set flag to avoid repeated calls
      // fadeOutVideo();
    }
  };

  const playNextVideo = async () => {
    // Reset the fading flag
    isFading = false;

    console.log(
      '[playNextVideo] videoQueue and currentIndex',
      $state.snapshot(videoQueue),
      $state.snapshot(currentIndex)
    );
    if (currentIndex < videoQueue.length) {
      const nextVideoTitle = videoQueue[currentIndex];
      currentIndex++;
      // await playVideo(nextVideoTitle);
      await switchVideo(nextVideoTitle);
    } else {
      // Queue finished!
      console.log(
        '[playNextVideo] All videos in the queue have been played. Going to play _screensaver or _select'
      );

      // videoMode = false;
      if (showScreenSaverAfterQueue) {
        await switchVideo(`_screensaver`);
        // await switchVideo(`_intro`); // TEMP FOR QUICK VIDEO
      } else {
        showSelectImage = true;
        await displayImage(`_select`);
      }
    }
  };

  // const playVideo = async (videoTitle: string) => {
  //   console.log(`[playVideo] Now gonna play ${videoTitle}`);
  //   const file = await readFile(`${videoTitle}.mp4`, {
  //     baseDir: baseDir,
  //   });

  //   // Convert binary data to a Blob URL
  //   const blob = new Blob([new Uint8Array(file)], { type: 'video/mp4' });

  //   if (videoElement) {
  //     videoElement.src = URL.createObjectURL(blob);

  //     console.log(`videoElement.src`, videoElement.src);

  //     videoElement
  //       .play()
  //       .catch((err) => {
  //         console.error('video playback error:', err);
  //       })
  //       .then(() => {
  //         getCurrentWindow().setFullscreen(true);
  //       });
  //   }
  // };

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

  // const switchVideo = async (videoTitle: string) => {
  //   if (!currentVideo) {
  //     currentVideo = video1;
  //   }
  //   console.log(`[switchVideo], videoTitle`, videoTitle);
  //   const file = await readFile(`${videoTitle}.mp4`, { baseDir: baseDir });
  //   const blob = new Blob([new Uint8Array(file)], { type: 'video/mp4' });
  //   const nextVideo = currentVideo === video1 ? video2 : video1;

  //   console.log(`[switchVideo] nextVideo:`, nextVideo);
  //   console.log(`[switchVideo] currentVideo`, currentVideo);
  //   nextVideo!.src = URL.createObjectURL(blob);
  //   nextVideo!.load();

  //   // Start fade-in of next video
  //   nextVideo!.classList.add('fade-in');
  //   currentVideo!.classList.add('fade-out');

  //   // Wait for fade transition
  //   setTimeout(() => {
  //     currentVideo!.classList.remove('fade-out');
  //     nextVideo!.classList.remove('fade-in');
  //     nextVideo!.classList.add('current');
  //     currentVideo!.classList.remove('current');
  //     currentVideo = nextVideo;
  //     console.log(`going to switch to`, currentVideo === video1 ? 'video1' : 'video2');

  //     // Start playback
  //     currentVideo!.play().catch((err) => console.error('Playback error:', err));
  //   }, 300);
  // };

  const switchVideo = async (videoTitle: string) => {
    console.log(`[switchVideo]`);
    if (!currentVideo) {
      currentVideo = video1;
    }
    const file = await readFile(`${videoTitle}.mp4`, { baseDir });
    const blob = new Blob([new Uint8Array(file)], { type: 'video/mp4' });
    const nextVideo = currentVideo === video1 ? video2 : video1;

    if (nextVideo) {
      nextVideo!.src = URL.createObjectURL(blob);
      nextVideo!.load();

      // Apply fade-in and fade-out transitions
      nextVideo!.classList.add('fade-in');
      currentVideo!.classList.add('fade-out');

      nextVideo!.classList.add('current');
      currentVideo!.classList.remove('current');
      // Start playback
      setTimeout(() => {
        currentVideo!.classList.remove('fade-out');
        nextVideo!.classList.remove('fade-in');

        currentVideo = nextVideo;
        currentVideo!.play().catch((err) => console.error('Playback error:', err));
      }, 300);
    } else {
      console.log(`[switchVideo] no nextVideo`);
    }
  };

  onMount(() => {});
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
        <!-- <video bind:this={videoElement} class="video" onended={playNextVideo}></video> -->

        <video
          bind:this={video1}
          preload="auto"
          muted
          playsinline
          class="current"
          ontimeupdate={handleTimeUpdate}
        ></video>
        <video bind:this={video2} preload="auto" muted playsinline ontimeupdate={handleTimeUpdate}
        ></video>
      {:else}
        <img bind:this={imageElement} class="image" alt="Select image" />
      {/if}
    </div>
  {/if}
</main>

<style>
  :root {
    background: black;
    /* background: yellow; */
    color: #0f0f0f;
    overflow: hidden;

    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;

    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
  }

  .container {
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
    height: 100vh;
    width: 100vw;
    background: black;
    /* background: indigo; */
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
    position: relative;
  }
  video,
  img {
    position: absolute;
    height: 100%;
    width: 100%;
    object-fit: contain;
    object-position: center;
    z-index: 0; /* Default */
    visibility: hidden; /* Hide non-current videos */

    /* Remove inline element magic space to prevent overflows
    https://courses.joshwcomeau.com/css-for-js/01-rendering-logic-1/09-flow-layout#inline-elements-have-magic-space */
    display: block;
  }

  .fade-in {
    opacity: 1;
    z-index: 2;
    transition: opacity 0.3s linear;
  }

  .fade-out {
    opacity: 0;
    z-index: 1;
    transition: opacity 0.3s linear;
  }

  .current {
    z-index: 3; /* Always on top */
    visibility: visible; /* Make current video visible */
  }
</style>
