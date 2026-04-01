<script setup>
import * as THREE from "three";
import { AmbientLight, DirectionalLight } from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import Stats from "three/examples/jsm/libs/stats.module.js";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { HDRLoader } from "three/examples/jsm/loaders/HDRLoader.js";
import { KTX2Loader } from "three/examples/jsm/loaders/KTX2Loader.js";
import { onMounted, ref } from "vue";
import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';

const cont = ref(null);

let camera, renderer, controls;
const scene = new THREE.Scene();

onMounted(() => {
  init();
});

async function init() {
  // ================= renderer =================
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  cont.value.appendChild(renderer.domElement);

  // ================= camera =================
  camera = new THREE.PerspectiveCamera(
    70,
    window.innerWidth / window.innerHeight,
    0.05, 
    1000
  );
  camera.position.set(10, 20, 20);
  camera.lookAt(0,0,0)

  // ================= controls =================
  controls = new OrbitControls(camera, renderer.domElement);

  // ================= light =================
  const light = new DirectionalLight(0xffffff, 0.6);
  light.position.set(-0.13, 2.51, 2.27);
  light.castShadow = true;
  scene.add(light);

  scene.add(new AmbientLight(0xffffff, 3.0));

  // ================= HDR =================
  new HDRLoader().load("./hdr/indoor.hdr", (tex) => {
    tex.mapping = THREE.EquirectangularReflectionMapping;
    scene.background = tex;
    scene.environment = tex;
  });


  // ================= 模型 =================
  const gltfLoader = new GLTFLoader();
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath(
    "https://unpkg.com/three@0.180.0/examples/jsm/libs/draco/"
  );
  gltfLoader.setDRACOLoader(dracoLoader);

  const ktx2Loader = new KTX2Loader();
  ktx2Loader.setTranscoderPath(
    "https://unpkg.com/three@0.180.0/examples/jsm/libs/basis/"
  );
  ktx2Loader.detectSupport(renderer);
  gltfLoader.setKTX2Loader(ktx2Loader);

  const gltf = await gltfLoader.loadAsync("models/futuristic_city.glb");
  gltf.scene.traverse((child) => {
    if (child.isMesh) {
      const oldMat = child.material;

      child.castShadow = true;
      child.receiveShadow = true;

      // 替换材质
      child.material = new THREE.MeshToonMaterial({
        map:oldMat.map, 
        color: 0x03e06d,
        shininess: 30,
        flatShading: true
      });
    }
  });
  // 缩小
  gltf.scene.scale.set(0.001,0.001,0.001) 
  scene.add(gltf.scene);

  // ================= loop =================
  renderer.setAnimationLoop(() => {
    controls.update();
    renderer.render(scene, camera);
  });
  window.addEventListener("resize", resize);
}

function resize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>

<template>
  <div ref="cont" style="width:100vw;height:100vh;"></div>
</template>