<script lang="ts">
  import { Canvas } from "@threlte/core";
  import Scene from "./Scene.svelte";
  import { onMount } from "svelte";
  import { webcam } from "../store";
  let camStream: MediaStream;
  let webCam: HTMLVideoElement;

  function initWebCam() {
    let [longSide, shortSide] = [800, 600];
    console.log("initWebCam...");

    if (camStream) {
      camStream.getVideoTracks().forEach((camera) => {
        camera.stop();
      });
    }

    webCam.id = "webcam";
    webCam.autoplay = true;
    webCam.muted = true;
    webCam.playsInline = true;
    webCam.width = longSide;
    webCam.height = shortSide;
    webCam.style.transform = "scaleX(-1)";
    (document.getElementById("video-sec") as HTMLElement).appendChild(webCam);

    let facingMode = "user";

    const option = {
      video: {
        width: { min: 0, max: webCam.width },
        height: { min: 0, max: webCam.height },
        aspectRatio: webCam.width / webCam.height,
        facingMode: facingMode,
      },
      audio: false,
    };

    // Get image from camera
    navigator.mediaDevices
      .getUserMedia(option)
      .then((stream) => {
        console.log("streamed...");
        camStream = stream;
        webCam.srcObject = stream;
        webCam.play();
        $webcam = webCam;
      })
      .catch((e) => {
        alert("WEBCAM ERROR: " + e.message);
      });
  }

  let mounted = false;

  onMount(() => {
    webCam = document.createElement("video");
    initWebCam();
    mounted = true;
    // initWebCam();
    // initPoseDetectionModel();
    // tick();
  });

  $: console.log($webcam);
</script>

<h1 id="loading-text">Loading...</h1>
<main id="video-sec">
  {#if mounted && $webcam}
    <Canvas size={{ width: 800, height: 600 }}>
      <Scene />
    </Canvas>
  {/if}
</main>

<style>
  main {
    display: flex;
    flex-direction: row;
  }
</style>
