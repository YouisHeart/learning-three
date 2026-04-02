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
import { MeshSurfaceSampler } from 'three/examples/jsm/math/MeshSurfaceSampler.js' 
import { gsap } from "gsap";
import { acceleratedRaycast, computeBoundsTree, disposeBoundsTree } from 'three-mesh-bvh';

THREE.Mesh.prototype.raycast = acceleratedRaycast;
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;

const cont = ref(null);

let camera, renderer, controls, model, pointsGeometry;
const positions = [];
const colors = [];
const progresses = [];
const tempPosition = new THREE.Vector3();
const tempColor = new THREE.Color();
const colorA = new THREE.Color(0x2a43ff);
const colorB = new THREE.Color(0xff0000);
let sampledAny = false;

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

  new HDRLoader().load("./hdr/indoor.hdr", (tex) => {
    tex.mapping = THREE.EquirectangularReflectionMapping;
    scene.background = tex;
    scene.environment = tex;
  });


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

 gltfLoader.load("./models/su7.glb", (gltf) => {
  model = gltf.scene;
  model.visible = false;
  scene.add(model);

    // 居中
  // const box = new THREE.Box3().setFromObject(model);
  // const center = box.getCenter(new THREE.Vector3());
  // model.position.sub(center);

  model.updateMatrixWorld(true,true);

  // ✅ 在这里再做 traverse
  model.traverse((child) => {
    if (child.isMesh && child.geometry) {
       child.geometry.computeBoundsTree();
    } 
    // 真正得网格必须是Mesh，geometry.getAttribute("position")必须有顶点属性
    if (!child.isMesh || !child.geometry.getAttribute("position")) {
      return;
    }

    // 在模型表面随机取点（不是顶点，而是表面均匀分布），build会计算每个三角面的面积，让采样更均匀（面积大的面被选中的概率更高）
    const sampler = new MeshSurfaceSampler(child).build();
    const sampleCount = 3000;

    for (let i = 0; i < sampleCount; i++) {
      sampler.sample(tempPosition); // 随机在模型表面取一个点，存到tempPosition里
      child.localToWorld(tempPosition); // 把局部坐标转换到世界坐标
      positions.push(tempPosition.x, tempPosition.y, tempPosition.z); // 把点的位置存进数组

      // progress = 车头到车位方向归一化，假设x轴式车头方向
      const minX = -2;
      const maxX = 2;
      const progress = THREE.MathUtils.clamp((tempPosition.x - minX) / (maxX - minX), 0, 1); 

      const t = THREE.MathUtils.clamp((tempPosition.x + 1.5) / 3, 0, 1);
      // tempColor.copy(colorA).lerp(colorB, t);
      // tempColor.setHSL(0.6 - 0.6 * t, 1.0, 0.5); // 根据点的位置计算颜色，这里是从蓝色到红色的渐变
      // colors.push(tempColor.r, tempColor.g, tempColor.b);
      colors.push(0.36, 0.149, 0.68);
      progress.push(progress);
    }

    sampledAny = true;
  });

  if (!sampledAny) {
    console.warn("No mesh found for sampling.");
    return;
  }

  let t = 0; // 0 ~ 1 动画进度
gsap.to({ val: 0 }, {
  val: 1,
  duration: 5,
  onUpdate: function() {
    t = this.targets()[0].val;
  }
});

  // 点云生成也放这里
  pointsGeometry = new THREE.BufferGeometry();
  pointsGeometry.setAttribute(
    "position",
    new THREE.Float32BufferAttribute(new Float32Array(positions), 3)
  );
  pointsGeometry.setAttribute(
    "color",
    new THREE.Float32BufferAttribute(new Float32Array(colors), 3)
  );

  const pointsMaterial = new THREE.PointsMaterial({
    size: 0.0005,
    vertexColors: true,
    transparent: true,
    opacity: 0.8,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
    sizeAttenuation: true,
  });

  const points = new THREE.Points(pointsGeometry, pointsMaterial);
  scene.add(points);
});

  const clock = new THREE.Clock()
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