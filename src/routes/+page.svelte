<script lang="ts">
  import { invoke } from '@tauri-apps/api/core';
  import { onMount } from 'svelte';
  import { bind, send, unbind } from '@kuyoonjo/tauri-plugin-udp';
  import { listen } from '@tauri-apps/api/event';

  import { readFile, readTextFile, writeTextFile, BaseDirectory } from '@tauri-apps/plugin-fs';

  import { getCurrentWindow } from '@tauri-apps/api/window';

  let setupMode = $state(true);
  let showSelectImage = $state(false); // whether video or image should be shown
  let showScreenSaverAfterQueue = $state(true); // whether screensaver or image should be shown after video queue is finished
  let broadcastAddress = $state('0.0.0.0');
  let port = $state('5002');
  let filesDirectory = $state('Downloads');
  let baseDir = $state(BaseDirectory.Download);
  // let videoDirectoryPath = $state('fms');
  let videoElement: HTMLVideoElement; // Reference to the <video> element
  let imageElement: HTMLImageElement; // Reference to the <img> element

  let videoQueue: string[] = $state([]); // Array of video titles
  let currentVideoIndex = $state(0);

  let invisibleBoxClickCount = $state(0);
  let invisibleBoxCountdown: number | undefined = $state();

  let noCommandTimeout: number | undefined = $state();
  let queuedCommand = $state();

  onMount(async () => {
    // Load settings from file
    await loadPreviousSettings();

    // Setup UDP listener
    await finishSetup();

    // Set window to full screen right away
    getCurrentWindow().setFullscreen(true);
  });

  /**
   * Read settings file saved in $HOME/.giving-machine
   */
  const loadPreviousSettings = async () => {
    console.log(`[loadPreviousSettings]`);
    const givingMachineSettingsFileContents = await readTextFile(`.giving-machine`, {
      baseDir: BaseDirectory.Home,
    });

    console.log(
      '[loadPreviousSettings] givingMachineSettingsFileContents',
      givingMachineSettingsFileContents
    );

    // Parse the contents into an object
    const givingMachineSettings = Object.fromEntries(
      givingMachineSettingsFileContents
        .trim() // Remove leading/trailing whitespace
        .split('\n') // Split into lines
        .map((line: string) => line.split('=')) // Split each line into key and value
    );

    // Destructure the variables from the object
    const { PORT, BROADCAST_ADDRESS, FILES_DIRECTORY } = givingMachineSettings;

    if (PORT) {
      port = PORT;
    }
    if (BROADCAST_ADDRESS) {
      broadcastAddress = BROADCAST_ADDRESS;
    }
    if (FILES_DIRECTORY) {
      filesDirectory = FILES_DIRECTORY;
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
    }
  };

  /**
   * Write settings file to $HOME/.giving-machine
   */
  const writeSettingsToFile = async () => {
    const lines = [
      `PORT=${port}`,
      `ADDRESS=${broadcastAddress}`,
      `FILES_DIRECTORY=${filesDirectory}`,
    ];

    const contents = lines.join('\n'); // Join the lines with newline characters

    await writeTextFile('.giving-machine', contents, {
      baseDir: BaseDirectory.Home,
    });
  };

  /**
   * Show setup page
   */
  const showSetup = async () => {
    if (!setupMode) {
      // Unbind the UDP listener if showing the setup mode
      await unbind('giving-machine');
      setupMode = true;
    }
  };

  /**
   *   3 consecutive clicks of the hidden box should show the setup page
   */
  function handleBoxClick() {
    invisibleBoxClickCount++;
    if (invisibleBoxClickCount === 3) {
      invisibleBoxClickCount = 0;
      showSetup();
      console.log('Three-click event triggered!');
    }

    // Reset count after 1 second of inactivity
    clearTimeout(invisibleBoxCountdown);
    invisibleBoxCountdown = setTimeout(() => (invisibleBoxClickCount = 0), 1000);
  }

  /**
   * Finish the setup and start listening for UDP commands
   */
  const finishSetup = async () => {
    const id = 'giving-machine';

    console.log(
      `[finishSetup] binding to ${$state.snapshot(broadcastAddress)}:${$state.snapshot(port)}`
    );

    // await bind(id, '0.0.0.0:8080');
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

    setupMode = false;

    await writeSettingsToFile();

    // Listen to UDP commands
    await listen('plugin://udp', (message) => {
      clearTimeout(noCommandTimeout);

      const payloadDataJSON = String.fromCharCode(...message.payload.data);
      console.log(`[processUDPCommand] payloadDataJSON`, payloadDataJSON);
      const payloadData = JSON.parse(payloadDataJSON)[0];
      console.log(`[processUDPCommand] payloadData`, payloadData);

      // Check if the videoQueue is currently being played first
      // If it's empty, process the command right away
      if (videoQueue.length == 0) {
        console.log(
          `[listen] The videoQueue is now empty so we will process this command right away.`,
          payloadData
        );
        processUDPCommand(payloadData);
      } else {
        // If videoQueue is showing, we save this command as the next to be processed, overwriting any command that is queued before this
        console.log(
          `[listen] The videoQueue is still in process so we will queue this command.`,
          payloadData
        );
        if (queuedCommand) {
          console.log(
            `[listen] Additionally, we will completely overwrite this previously queued command:`,
            $state.snapshot(queuedCommand)
          );
        }
        queuedCommand = payloadData;
      }
    });
  };

  const processUDPCommand = (payloadData) => {
    if (payloadData.AddedProduct) {
      // If there is an AddedProduct, we queue the `_intro`, ...videoTitles, `_outro` sequence

      showSelectImage = false; // reset to false if ever it's been set as true before this

      let videoTitles = [];
      if (payloadData.Products && payloadData.Products.length > 0) {
        console.log(`[processUDPCommand] payloadData.Products`, payloadData.Products);

        videoTitles = payloadData.Products.map((product) => {
          return product.ProductCode;
        });
        console.log(`[processUDPCommand] videoTitles`, videoTitles);
      }

      if (payloadData.IsScreenSaverOn) {
        console.log(
          `[processUDPCommand] IsScreenSaverOn = true so Going to show _screensaver video after the queue`
        );
        showScreenSaverAfterQueue = true;
      } else {
        console.log(
          `[processUDPCommand] IsScreenSaverOn = false so Going to show _select image after the queue`
        );
        showScreenSaverAfterQueue = false;
      }
      startVideoQueue([`_intro`, ...videoTitles, `_outro`]);
    } else if (payloadData.IsScreenSaverOn) {
      // If there is no AddedProduct but there is an IsScreenSaverOn, show _screensaver right away

      console.log(`[processUDPCommand] showing _screensaver right away`);
      showScreenSaverAfterQueue = true;
      showSelectImage = false;
      playVideo('_screensaver');
    } else if (!payloadData.IsScreenSaverOn && !payloadData.AddedProduct) {
      // If there are no AddedProduct and IsScreenSaverOn, show _select image right away

      console.log(`[processUDPCommand] showing _select image right away`);
      displayImage(`_select`);
      showSelectImage = true;
    }
  };

  // Function to start playing a queue of videos
  const startVideoQueue = async (titles: string[]) => {
    console.log(`[startVideoQueue] titles`, titles);
    if (!titles.length) return;
    videoQueue = titles;
    currentVideoIndex = 0;
    await playNextVideo();
  };

  /**
   * Runs the next video in the videoQueue when the current video has ended.
   * If the queue has finished this will show the screensaver or the select image instead.
   */
  const playNextVideo = async () => {
    console.log(
      '[playNextVideo] videoQueue and currentIndex',
      $state.snapshot(videoQueue),
      $state.snapshot(currentVideoIndex)
    );
    if (currentVideoIndex < videoQueue.length) {
      const nextVideoTitle = videoQueue[currentVideoIndex];
      currentVideoIndex++;
      await playVideo(nextVideoTitle);
    } else {
      // Queue finished!
      console.log('[playNextVideo] All videos in the queue have been played. ');

      // Reset videoQueue and currentVideoIndex
      videoQueue = [];
      currentVideoIndex = 0;

      if (queuedCommand) {
        console.log(
          `[playNextVideo] There is a queued command so we will process this command now`,
          $state.snapshot(queuedCommand)
        );
        processUDPCommand($state.snapshot(queuedCommand));
        queuedCommand = undefined; // reset queuedCommand
      } else if (showScreenSaverAfterQueue) {
        console.log(`[playNextVideo] Will now show the screensaver`);
        await playVideo(`_screensaver`);
      } else {
        showSelectImage = true;
        console.log(`[playNextVideo] Will now show the _select image`);
        await displayImage(`_select`);
      }
    }
  };

  /**
   * Plays the mp4 video file with filename `videoTitle` in the base directory
   * @param videoTitle
   */
  const playVideo = async (videoTitle: string) => {
    console.log(`[playVideo] Now gonna play ${videoTitle}`);
    const file = await readFile(`${videoTitle}.mp4`, {
      baseDir: baseDir,
    });

    // Convert binary data to a Blob URL
    const blob = new Blob([new Uint8Array(file)], { type: 'video/mp4' });

    if (videoElement) {
      videoElement.src = URL.createObjectURL(blob);

      console.log(`[playVideo] videoElement.src`, videoElement.src);

      videoElement
        .play()
        .catch((err) => {
          console.error('[playVideo] video playback error:', err);
        })
        .then(() => {
          getCurrentWindow().setFullscreen(true);
        });
    }
  };

  /**
   * Displays jpg image file with filename `imageTitle` from the base directory
   * @param imageTitle
   */
  const displayImage = async (imageTitle: string) => {
    console.log(`[displayImage] Now gonna display image ${imageTitle}`);

    const file = await readFile(`${imageTitle}.jpg`, {
      baseDir: baseDir,
    });

    // Convert binary data to a Blob URL
    const blob = new Blob([new Uint8Array(file)], { type: 'image/jpeg' });

    console.log(`[displayImage] image file`, file);
    if (imageElement) {
      console.log(`[displayImage] imageElement`, $state.snapshot(imageElement));
      imageElement.src = URL.createObjectURL(blob);
      // console.log(`imageElement.src`, imageElement.src);
      getCurrentWindow().setFullscreen(true);

      // 5 minute countdown
      noCommandTimeout = setTimeout(() => {
        console.log(`[displayImage] 5 minutes have passed so we will show _screensaver now`);
        showSelectImage = false;
        showScreenSaverAfterQueue = true;
        playVideo(`_screensaver`);
      }, 300000);
    } else {
      console.error(`[displayImage] no imageElement`);
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
      <select bind:value={filesDirectory} name="files-dir" id="files-dir">
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

  <div class="hidden-square" onclick={handleBoxClick}></div>
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
  }
  video,
  img {
    height: 100%;
    width: 100%;
    object-fit: contain;
    object-position: center;

    /* Remove inline element magic space to prevent overflows
    https://courses.joshwcomeau.com/css-for-js/01-rendering-logic-1/09-flow-layout#inline-elements-have-magic-space */
    display: block;
  }

  .hidden-square {
    position: fixed;
    bottom: 10px;
    left: 10px;
    width: 100px;
    height: 100px;
    background-color: transparent;
    z-index: 1000;
    color: red;
  }

  /* Optional for testing */
  .hidden-square:hover {
    /* background-color: rgba(0, 0, 0, 0.1); */
  }
</style>
