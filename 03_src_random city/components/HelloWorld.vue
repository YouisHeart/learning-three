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
import { gsap } from "gsap";

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
  controls = new OrbitControls(camera,renderer.domElement);
  controls.enableDamping = true;

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

  const groundGeometry = new THREE.PlaneGeometry(600, 600);
  const groundMaterial = new THREE.MeshPhongMaterial({
  color: 0x4cd1ff,
  transparent: true,
  opacity: 0.8,
  flatShading: true,
  side: THREE.DoubleSide,
  });

  const ground = new THREE.Mesh(groundGeometry, groundMaterial);
  ground.rotation.x = Math.PI / 2;
  scene.add(ground)

  const geometry = new THREE.BoxGeometry(10, 10, 10);
  const textureLoader = new THREE.TextureLoader();
  const texture = textureLoader.load("./texture/HighRiseNight0058_1_download600.jpg");
  texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
  texture.repeat.set(5,5);

  const buildings = [];

  for (let i = 0; i < 100; i++) {
  const material = new THREE.MeshPhongMaterial({
      // color: colors[Math.floor(Math.random() * 3)],
      // flatShading: true,
      map: texture
    });

  const building = new THREE.Mesh(geometry, material);

    buildings.push(building);
    scene.add(building);
  }

function startAnimation() {
  function animateLoop() {
    buildings.forEach((building) => {
      const duration = Math.random() * 0.6 + 0.3;
      const specialHeight = Math.random() < 0.1 ? 15 : 0;

      gsap.to(building.scale, {
        duration,
        x: 1 + Math.random() * 3,
        y: 1 + Math.random() * 20 + specialHeight,
        z: 1 + Math.random() * 3,
      });

      gsap.to(building.position, {
        duration,
        x: -200 + Math.random() * 400,
        z: -200 + Math.random() * 400,
        ease: "power2.inOut",
      });
    });

    setTimeout(animateLoop, 800); // 循环触发
  }

  animateLoop(); // ❗必须调用
}
startAnimation()


  // ================= loop =================
  const clock = new THREE.Clock()
  renderer.setAnimationLoop(() => {
    const delta = clock.getDelta();
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