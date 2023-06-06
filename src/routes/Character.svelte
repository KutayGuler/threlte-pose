<script lang="ts">
  import { GLTF, useGltfAnimations } from "@threlte/extras";
  import { useFrame } from "@threlte/core";
  import { webcam } from "../store";
  import {
    SupportedModels,
    createDetector,
    type PoseDetector,
    type BlazePoseTfjsModelConfig,
    type Pose,
  } from "@tensorflow-models/pose-detection";
  import * as tf from "@tensorflow/tfjs";
  import type { Bone, Object3D, SkinnedMesh } from "three";

  const { gltf } = useGltfAnimations();
  const MULTIPLIER = -50;

  // TODO: get rotation

  let detector: PoseDetector;
  let poses: Array<Pose>;

  async function initPoseDetectionModel() {
    let loadingText = document.getElementById("loading-text") as HTMLElement;
    loadingText.textContent = "Loading Pose Detection Model...";

    tf.setBackend("webgl");
    //tf.setBackend('webgpu');
    console.log("tfjs backend:", tf.getBackend());
    const detectorConfig: BlazePoseTfjsModelConfig = {
      runtime: "tfjs",
      modelType: "full",
      enableSmoothing: true,
    };

    if (detector != null) {
      detector.dispose();
    }

    try {
      detector = await createDetector(
        SupportedModels.BlazePose,
        detectorConfig
      );
      makeObjectAndChildrenWritable($gltf.scene.children[0], 0);
      start();
    } catch (error) {
      // @ts-expect-error
      detector = null;
      alert(error);
    }
    loadingText.textContent = "Loaded.";
    setTimeout(() => loadingText.classList.toggle("transparent"), 3000);
  }

  async function get3DKeypoints() {
    if (!detector) return undefined;

    const config = {
      maxPoses: 1,
      flipHorizontal: false,
    };
    poses = await detector.estimatePoses($webcam, config);
    //poses = await detector.estimatePosesGPU(webCam);

    let map = new Map();
    console.log(poses[0]);

    if (poses[0] && poses[0].keypoints3D) {
      for (let { x, y, z, name } of poses[0].keypoints3D) {
        map.set(name, {
          x: x * MULTIPLIER,
          y: y * MULTIPLIER,
          z: (z || 1) * MULTIPLIER,
        });
      }
    }

    return map;
  }

  const MIXAMO_TO_BLAZE = {
    mixamorigRightShoulder: "right_shoulder",
    mixamorigLeftShoulder: "left_shoulder",
    mixamorigHead: "nose",
  };

  function makeObjectAndChildrenWritable(
    obj: Object3D | SkinnedMesh | Bone,
    depth: number
  ) {
    Object.defineProperty(obj, "position", {
      writable: true,
    });
    Object.defineProperty(obj, "rotation", {
      writable: true,
    });

    for (let child of obj.children) {
      makeObjectAndChildrenWritable(child, depth + 1);
    }
  }

  function updateObjectAndChildren(
    obj: Object3D | SkinnedMesh | Bone,
    keypoints: Map<string, string>
  ) {
    // @ts-expect-error
    obj.position =
      keypoints.get(
        MIXAMO_TO_BLAZE[obj.name as keyof typeof MIXAMO_TO_BLAZE]
      ) || obj.position;

    for (let child of obj.children) {
      updateObjectAndChildren(child, keypoints);
    }
  }

  // not sure about making this function async
  const { start } = useFrame(async (ctx) => {
    const keypoints = await get3DKeypoints();
    if ($gltf && keypoints) {
      updateObjectAndChildren($gltf.scene.children[0], keypoints);
    }
  });

  initPoseDetectionModel();
</script>

<GLTF
  bind:gltf={$gltf}
  url="https://threejs.org/examples/models/gltf/Xbot.glb"
/>
