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
  let broadcastAddress = $state('0.0.0.0');
  let port = $state('5001');
  let videoDirectory = $state('Documents');
  let videoElement: HTMLVideoElement; // Reference to the <video> element

  onMount(async () => {});

  const finishSetup = async () => {
    const id = 'unique-id';
    // await bind(id, '0.0.0.0:8080');
    console.log(`binding to ${broadcastAddress}:${port}`);
    await bind(id, `${broadcastAddress}:${port}`);

    setupMode = false;

    await listen('plugin://udp', (x) => {
      const payloadDataJSON = String.fromCharCode(...x.payload.data);
      console.log(`payloadDataJSON`, payloadDataJSON);
      const payloadData = JSON.parse(payloadDataJSON)[0];
      console.log(`payloadData`, payloadData);

      if (payloadData.IsScreenSaverOn) {
        console.log(`payloadData.Products`, payloadData.Products);
        const videoTitle = payloadData.Products[0].ProductCode;
        console.log(`videoTitle`, videoTitle);
        playVideo(videoTitle);
      }
    });
  };

  const playVideo = async (videoTitle: string) => {
    let baseDir = BaseDirectory.Document;
    switch (videoDirectory) {
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
</script>

<main class="container">
  {#if setupMode}
    <div class="setup">
      Broadcast address <input bind:value={broadcastAddress} />
      <br />
      Port <input bind:value={port} />
      <br />
      Video directory
      <select bind:value={videoDirectory} name="pets" id="pet-select">
        <option value="Desktop">Desktop</option>
        <option value="Documents">Documents Folder</option>
        <option value="Downloads">Downloads Folder</option>
      </select>

      <button onclick={finishSetup} class="finish-setup-button">Finish Setup</button>
    </div>
  {:else}
    <video bind:this={videoElement}> </video>
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
    /* padding-top: 10vh; */
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
  }

  .setup {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }
  .finish-setup-button {
    margin-top: 40px;
    font-size: 1em;
  }

  /* .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: 0.75s;
  }

  .logo.tauri:hover {
    filter: drop-shadow(0 0 2em #24c8db);
  } 

  .row {
    display: flex;
    justify-content: center;
  }

  a {
    font-weight: 500;
    color: #646cff;
    text-decoration: inherit;
  }

  a:hover {
    color: #535bf2;
  }

  h1 {
    text-align: center;
  }

  input,
  button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #0f0f0f;
    background-color: #ffffff;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
  }

  button {
    cursor: pointer;
  }

  button:hover {
    border-color: #396cd8;
  }
  button:active {
    border-color: #396cd8;
    background-color: #e8e8e8;
  }

  input,
  button {
    outline: none;
  }

  #greet-input {
    margin-right: 5px;
  } */

  /* @media (prefers-color-scheme: dark) {
    :root {
      color: #f6f6f6;
      background-color: #2f2f2f;
    }

    a:hover {
      color: #24c8db;
    }

    input,
    button {
      color: #ffffff;
      background-color: #0f0f0f98;
    }
    button:active {
      background-color: #0f0f0f69;
    }
  } */
</style>
