<script setup>
import Stats from 'stats.js';
import * as dat from 'three/examples/jsm/libs/lil-gui.module.min.js';
import * as THREE from 'three';
import { ref, onMounted } from 'vue';
import {
	acceleratedRaycast, computeBoundsTree, disposeBoundsTree,
	CENTER, SAH, AVERAGE, BVHHelper,
} from 'three-mesh-bvh';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

THREE.Mesh.prototype.raycast = acceleratedRaycast;
THREE.BufferGeometry.prototype.computeBoundsTree = computeBoundsTree;
THREE.BufferGeometry.prototype.disposeBoundsTree = disposeBoundsTree;

const bgColor = 0x131619;
const pointDist = 25;

const params = {
	// Raycasters
	raycasterCount: 150,
	raycasterSpeed: 1,
	raycasterNear: 0,
	raycasterFar: pointDist,

	// Mesh
	splitStrategy: CENTER,
	meshCount: 1,
	meshSpeed: 1,
	useBoundsTree: true,
	displayBVH: false,
	displayDepth: 10,
	displayParents: false,
};

let renderer, scene, stats, camera;
let geometry, material, bvhHelper, containerObj, mesh;
const knots = [];
const rayCasterObjects = [];

const raycaster = new THREE.Raycaster();
raycaster.firstHitOnly = true;

const sphere = new THREE.SphereGeometry(0.25, 20, 20);
const cylinder = new THREE.CylinderGeometry(0.01, 0.01);

const loader = new GLTFLoader();
// 创建 Draco Loader
const dracoLoader = new DRACOLoader();
// 设置解压路径（可以用 Three.js 官方提供的 JS 解码器）
dracoLoader.setDecoderPath("https://www.gstatic.com/draco/versioned/decoders/1.5.6/");
loader.setDRACOLoader(dracoLoader);

let lastFrameTime = null;

// 添加一个简单的几何体作为备用，以防模型加载失败
const defaultGeometry = new THREE.TorusKnotGeometry(1, 0.4, 400, 100);
const defaultMaterial = new THREE.MeshPhongMaterial({ color: 0xE91E63 });

function init() {

	// Renderer
	renderer = new THREE.WebGLRenderer({ antialias: true });
	renderer.setPixelRatio(window.devicePixelRatio);
	renderer.setSize(window.innerWidth, window.innerHeight);
	renderer.setClearColor(bgColor, 1);
	renderer.setAnimationLoop(render);
	document.body.appendChild(renderer.domElement);

	// Scene
	scene = new THREE.Scene();
	scene.fog = new THREE.Fog(bgColor, 40, 80);

	// Lights
	const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
	directionalLight.position.set(1, 1, 1);

	const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
	scene.add(directionalLight, ambientLight);

	// 创建 containerObj
	containerObj = new THREE.Object3D();
	containerObj.scale.setScalar(10);
	containerObj.rotation.x = containerObj.rotation.y = 10.989999999999943;
	scene.add(containerObj);

	// 先创建一个默认的 mesh 作为占位
	mesh = new THREE.Mesh(defaultGeometry, defaultMaterial);
	containerObj.add(mesh);
	knots.push(mesh);

	//model
	loader.load('./models/su7.glb', gltf => {
		const model = gltf.scene;
		
		// 移除默认的 mesh
		if (mesh && mesh.parent) {
			containerObj.remove(mesh);
			const index = knots.indexOf(mesh);
			if (index !== -1) knots.splice(index, 1);
		}
		
		// 添加新模型
		containerObj.add(model);
		
		// 更新 mesh 引用
		mesh = model;
		
		// 重新添加到 knots 数组
		knots.push(mesh);
		
		// 做BVH加速
		model.traverse((child) => {
			if (child.isMesh) {
				child.geometry.computeBoundsTree();
			}
		});
		
		// 更新 BVH 可视化
		if (params.displayBVH && bvhHelper) {
			if (bvhHelper) containerObj.remove(bvhHelper);
			bvhHelper = new BVHHelper(mesh);
			containerObj.add(bvhHelper);
		}
	}, undefined, (error) => {
		console.error('模型加载失败:', error);
	});

	// Camera
	camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
	camera.position.z = 60;

	// Stats
	stats = new Stats();
	document.body.appendChild(stats.dom);

	// GUI
	const gui = new dat.GUI();
	const rcFolder = gui.addFolder('Raycasters');
	rcFolder.add(params, 'raycasterCount', 1, 1000, 1).onChange(updateFromOptions);
	rcFolder.add(params, 'raycasterSpeed', 0, 20);
	rcFolder.add(params, 'raycasterNear', 0, pointDist).onChange(updateFromOptions);
	rcFolder.add(params, 'raycasterFar', 0, pointDist).onChange(updateFromOptions);
	rcFolder.open();

	const meshFolder = gui.addFolder('Mesh');
	meshFolder.add(params, 'useBoundsTree').onChange(updateFromOptions);
	meshFolder.add(params, 'splitStrategy', { CENTER, SAH, AVERAGE }).onChange(updateFromOptions);
	meshFolder.add(params, 'meshCount', 1, 300, 1).onChange(updateFromOptions);
	meshFolder.add(params, 'meshSpeed', 0, 20);
	meshFolder.add(params, 'displayBVH').onChange(updateFromOptions);
	meshFolder.add(params, 'displayParents').onChange(v => {

		if (bvhHelper) {

			bvhHelper.displayParents = v;
			bvhHelper.update();

		}

	});
	meshFolder.add(params, 'displayDepth', 1, 20, 1).onChange(v => {

		if (bvhHelper) {

			bvhHelper.depth = v;
			bvhHelper.update();

		}

	});
	meshFolder.open();

	window.addEventListener('resize', () => {

		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);

	});

	updateFromOptions();
}

function addKnot() {
	// 使用默认几何体创建新的 mesh，而不是引用同一个 mesh
	const newMesh = new THREE.Mesh(defaultGeometry, defaultMaterial);
	newMesh.rotation.x = Math.random() * 10;
	newMesh.rotation.y = Math.random() * 10;
	knots.push(newMesh);
	containerObj.add(newMesh);
	return newMesh;
}

function addRaycaster() {

	const obj = new THREE.Object3D();
	const whiteMaterial = new THREE.MeshBasicMaterial({ color: 0xffffff });
	const origMesh = new THREE.Mesh(sphere, whiteMaterial);
	const hitMesh = new THREE.Mesh(sphere, whiteMaterial);
	hitMesh.scale.setScalar(0.25);
	origMesh.scale.setScalar(0.5);

	const cylinderMesh = new THREE.Mesh(cylinder, new THREE.MeshBasicMaterial({
		color: 0xffffff,
		transparent: true,
		opacity: 0.25
	}));
	cylinderMesh.rotation.z = Math.PI / 2;

	obj.add(cylinderMesh, origMesh, hitMesh);
	scene.add(obj);

	origMesh.position.set(pointDist, 0, 0);
	obj.rotation.x = Math.random() * 10;
	obj.rotation.y = Math.random() * 10;
	obj.rotation.z = Math.random() * 10;

	const origVec = new THREE.Vector3();
	const dirVec = new THREE.Vector3();
	const xDir = Math.random() - 0.5;
	const yDir = Math.random() - 0.5;
	const zDir = Math.random() - 0.5;

	rayCasterObjects.push({
		update: deltaTime => {

			obj.rotation.x += xDir * 0.0001 * params.raycasterSpeed * deltaTime;
			obj.rotation.y += yDir * 0.0001 * params.raycasterSpeed * deltaTime;
			obj.rotation.z += zDir * 0.0001 * params.raycasterSpeed * deltaTime;

			origMesh.updateMatrixWorld();
			origVec.setFromMatrixPosition(origMesh.matrixWorld);
			dirVec.copy(origVec).multiplyScalar(-1).normalize();

			raycaster.set(origVec, dirVec);
			const res = raycaster.intersectObject(containerObj, true);
			const length = res.length ? res[0].distance : pointDist;

			hitMesh.position.set(pointDist - length, 0, 0);

			const lineLength = res.length ? length - raycaster.near : length - raycaster.near - (pointDist - raycaster.far);
			cylinderMesh.position.set(pointDist - raycaster.near - (lineLength / 2), 0, 0);
			cylinderMesh.scale.set(1, lineLength, 1);

		},

		remove: () => scene.remove(obj)
	});

}

function updateFromOptions() {

	if (!raycaster) return;
	
	raycaster.near = params.raycasterNear;
	raycaster.far = params.raycasterFar;

	// Update raycaster count
	while (rayCasterObjects.length > params.raycasterCount) {

		rayCasterObjects.pop().remove();

	}

	while (rayCasterObjects.length < params.raycasterCount) {

		addRaycaster();

	}

	if (!mesh) return;

	// 获取当前的几何体
	const currentGeometry = mesh.isGroup || mesh.isScene ? null : mesh.geometry;
	if (!currentGeometry) return;

	// Update bounds tree
	if (
		!params.useBoundsTree && currentGeometry.boundsTree ||
		currentGeometry.boundsTree && params.splitStrategy !== currentGeometry.boundsTree.splitStrategy
	) {

		currentGeometry.disposeBoundsTree();

	}

	if (params.useBoundsTree && !currentGeometry.boundsTree) {

		console.time('computing bounds tree');
		currentGeometry.computeBoundsTree({
			maxLeafSize: 5,
			strategy: parseFloat(params.splitStrategy),
		});
		currentGeometry.boundsTree.splitStrategy = params.splitStrategy;
		console.timeEnd('computing bounds tree');

		if (bvhHelper) bvhHelper.update();

	}

	// Update knot count
	const oldLen = knots.length;
	while (knots.length > params.meshCount) {

		const removedMesh = knots.pop();
		if (removedMesh && removedMesh.parent) {
			containerObj.remove(removedMesh);
		}

	}

	while (knots.length < params.meshCount) {
		addKnot();
	}

	if (oldLen !== knots.length && knots.length > 0) {

		const lerp = (a, b, t) => a + (b - a) * t;
		const lerpAmt = (knots.length - 1) / 299;
		const dist = lerp(0, 2, lerpAmt);
		const scale = lerp(1, 0.2, lerpAmt);

		knots.forEach(c => {

			if (c) {
				c.scale.setScalar(scale);

				const vec3 = new THREE.Vector3(0, 1, 0);
				vec3.applyAxisAngle(new THREE.Vector3(1, 0, 0), Math.PI * Math.random());
				vec3.applyAxisAngle(new THREE.Vector3(0, 1, 0), 2 * Math.PI * Math.random());
				vec3.multiplyScalar(dist);

				c.position.copy(vec3);
			}

		});

	}

	// Update bounds visualization
	const shouldDisplayBounds = params.displayBVH && currentGeometry.boundsTree;
	if (bvhHelper && !shouldDisplayBounds) {

		containerObj.remove(bvhHelper);
		bvhHelper = null;

	}

	if (!bvhHelper && shouldDisplayBounds && mesh && !mesh.isGroup) {

		bvhHelper = new BVHHelper(mesh);
		containerObj.add(bvhHelper);

	}

}

function render() {

	if (!stats || !renderer || !scene || !camera) return;
	
	stats.begin();

	const currTime = window.performance.now();
	lastFrameTime = lastFrameTime || currTime;
	const deltaTime = currTime - lastFrameTime;

	// Update GUI settings
	if (bvhHelper) bvhHelper.visible = params.displayBVH;

	if (containerObj) {
		containerObj.rotation.x += 0.0001 * params.meshSpeed * deltaTime;
		containerObj.rotation.y += 0.0001 * params.meshSpeed * deltaTime;
		containerObj.children.forEach(c => {
			if (c && c.rotation) {
				c.rotation.x += 0.0001 * params.meshSpeed * deltaTime;
				c.rotation.y += 0.0001 * params.meshSpeed * deltaTime;
			}
		});
		containerObj.updateMatrixWorld();
	}

	rayCasterObjects.forEach(f => f.update(deltaTime));

	renderer.render(scene, camera);

	lastFrameTime = currTime;

	stats.end();

}

onMounted(() => {
	init();
});
</script>

<template>
	<div></div>
</template>