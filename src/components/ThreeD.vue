<template>
  <div class="container">
    <div ref="sceneContainer" class="scene-container"></div>
    <div class="controls-container">
      <select v-model="selectedElement">
        <option
          v-for="element in elements"
          :value="element"
          :key="element.symbol"
        >
          {{ element.symbol }} - {{ element.name }}
        </option>
      </select>
      <button @click="addSphere">Add Sphere</button>
      <div class="coordinates">
        <div v-for="(sphere, index) in spheres" :key="sphere.uuid">
          <p>
            {{ sphere.name }}:
            <span v-if="editingSphereIndex !== index">
              x={{ sphere.position.x.toFixed(2) }}, y={{
                sphere.position.y.toFixed(2)
              }}, z={{ sphere.position.z.toFixed(2) }}
            </span>
            <span v-else>
              X:
              <input
                v-model.number="sphere.position.x"
                type="number"
                step="0.01"
              />
              Y:
              <input
                v-model.number="sphere.position.y"
                type="number"
                step="0.01"
              />
              Z:
              <input
                v-model.number="sphere.position.z"
                type="number"
                step="0.01"
              />
            </span>
          </p>
          <button v-if="editingSphereIndex !== index" @click="startEdit(index)">
            수정하기
          </button>
          <template v-else>
            <button @click="updateSpherePosition(index)">저장</button>
            <button @click="cancelEdit()">취소</button>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

export default {
  name: "ThreeDView",
  setup() {
    const selectedElement = ref(null);
    const elements = ref([
      { symbol: "H", name: "Hydrogen", color: 0xffffff },
      { symbol: "C", name: "Carbon", color: 0x000000 },
      { symbol: "N", name: "Nitrogen", color: 0x0000ff },
      { symbol: "O", name: "Oxygen", color: 0xff0000 },
      { symbol: "F", name: "Fluorine", color: 0x00ff00 },
      { symbol: "Cl", name: "Chlorine", color: 0x00ff00 },
      { symbol: "Br", name: "Bromine", color: 0x8b0000 },
      { symbol: "I", name: "Iodine", color: 0x9400d3 },
      { symbol: "He", name: "Helium", color: 0x00ffff },
      { symbol: "Ne", name: "Neon", color: 0x00ffff },
      { symbol: "Ar", name: "Argon", color: 0x00ffff },
      { symbol: "Kr", name: "Krypton", color: 0x00ffff },
      { symbol: "Xe", name: "Xenon", color: 0x00ffff },
      { symbol: "P", name: "Phosphorus", color: 0xffa500 },
      { symbol: "S", name: "Sulfur", color: 0xffff00 },
      { symbol: "B", name: "Boron", color: 0xf5f5dc },
      { symbol: "Li", name: "Lithium", color: 0xee82ee },
      { symbol: "Na", name: "Sodium", color: 0xee82ee },
      { symbol: "K", name: "Potassium", color: 0xee82ee },
      { symbol: "Rb", name: "Rubidium", color: 0xee82ee },
      { symbol: "Cs", name: "Cesium", color: 0xee82ee },
      { symbol: "Fr", name: "Francium", color: 0xee82ee },
      { symbol: "Be", name: "Beryllium", color: 0x006400 },
      { symbol: "Mg", name: "Magnesium", color: 0x006400 },
      { symbol: "Ca", name: "Calcium", color: 0x006400 },
      { symbol: "Sr", name: "Strontium", color: 0x006400 },
      { symbol: "Ba", name: "Barium", color: 0x006400 },
      { symbol: "Ra", name: "Radium", color: 0x006400 },
      { symbol: "Ti", name: "Titanium", color: 0x808080 },
      { symbol: "Fe", name: "Iron", color: 0xff8c00 },
      { symbol: "Al", name: "Aluminum", color: 0xb0c4de },
      { symbol: "Si", name: "Silicon", color: 0x778899 },
      { symbol: "Ni", name: "Nickel", color: 0x708090 },
      { symbol: "Cu", name: "Copper", color: 0xb87333 },
      { symbol: "Zn", name: "Zinc", color: 0xcd5c5c },
      { symbol: "Ag", name: "Silver", color: 0xc0c0c0 },
      { symbol: "Pt", name: "Platinum", color: 0xe5e4e2 },
      { symbol: "Au", name: "Gold", color: 0xffd700 },
      { symbol: "Hg", name: "Mercury", color: 0xb0e0e6 },
      { symbol: "Pb", name: "Lead", color: 0x696969 },
      { symbol: "U", name: "Uranium", color: 0x4b0082 },
    ]);
    const spheres = ref([]);
    const editingSphereIndex = ref(null);
    const sceneContainer = ref(null);

    let scene, camera, renderer, controls, light;

    const initThree = () => {
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setClearColor(0xffffff, 1);
      sceneContainer.value.appendChild(renderer.domElement);

      scene = new THREE.Scene();

      camera = new THREE.PerspectiveCamera(
        32,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );
      camera.position.z = 5;

      controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.25;
      controls.enableZoom = true;

      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
      scene.add(ambientLight);

      light = new THREE.DirectionalLight(0xffffff, 0.5);
      light.position.set(5, 5, 5);
      scene.add(light);

      const floorGeometry = new THREE.PlaneGeometry(100, 100);
      const floorMaterial = new THREE.MeshBasicMaterial({
        color: 0xaaaaaa,
        side: THREE.DoubleSide,
      });
      const floor = new THREE.Mesh(floorGeometry, floorMaterial);
      floor.rotation.x = Math.PI / 2;
      floor.position.y = -1;
      scene.add(floor);

      const wallGeometry = new THREE.PlaneGeometry(100, 100);
      const wallMaterial = new THREE.MeshBasicMaterial({
        color: 0xcccccc,
        side: THREE.DoubleSide,
      });
      const wall = new THREE.Mesh(wallGeometry, wallMaterial);
      wall.position.z = -10;
      scene.add(wall);

      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();
      const plane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);

      let isDragging = false;
      let selectedObject = null;
      let offset = new THREE.Vector3();

      const onMouseDown = (event) => {
        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        const intersects = raycaster.intersectObjects(spheres.value);

        if (intersects.length > 0) {
          controls.enabled = false;
          isDragging = true;
          selectedObject = intersects[0].object;

          const intersectPoint = intersects[0].point;
          offset.copy(intersectPoint).sub(selectedObject.position);

          plane.setFromNormalAndCoplanarPoint(
            camera.getWorldDirection(plane.normal),
            intersectPoint
          );
        }
      };

      const onMouseMove = (event) => {
        if (!isDragging) return;

        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        const planeIntersect = new THREE.Vector3();
        raycaster.ray.intersectPlane(plane, planeIntersect);

        if (planeIntersect) {
          selectedObject.position.copy(planeIntersect).sub(offset);
        }
      };

      const onMouseUp = () => {
        controls.enabled = true;
        isDragging = false;
        selectedObject = null;
      };

      window.addEventListener("mousedown", onMouseDown, false);
      window.addEventListener("mousemove", onMouseMove, false);
      window.addEventListener("mouseup", onMouseUp, false);

      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
        light.position.copy(camera.position);
      };
      animate();

      window.addEventListener("resize", () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      });
    };

    const addSphere = () => {
      if (!selectedElement.value) return;

      const createTextTexture = (text) => {
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        const size = 256;

        canvas.width = size;
        canvas.height = size;

        context.fillStyle = 'white';
        context.fillRect(0, 0, size, size); 

        context.fillStyle = 'black';
        context.font = '48px Arial';
        context.textAlign = 'center';
        context.textBaseline = 'middle'; 

        context.fillText(text, size / 2, size / 2);

        return new THREE.CanvasTexture(canvas);
      };

      const texture = createTextTexture(selectedElement.value.symbol);

      const geometry = new THREE.SphereGeometry(0.4, 64, 64);
      const material = new THREE.MeshPhongMaterial({
        color: selectedElement.value.color,
        map: texture,
        shininess: 100,
      });

      const existingSpheres = spheres.value.filter((sphere) =>
        sphere.name.startsWith(selectedElement.value.symbol)
      );
      const number = existingSpheres.length + 1;
      const sphere = new THREE.Mesh(geometry, material);
      sphere.position.set(0, 0, 0);
      sphere.name = `${selectedElement.value.symbol}${number}`;

      scene.add(sphere);
      spheres.value.push(sphere);
    };

    const startEdit = (index) => {
      editingSphereIndex.value = index;
    };

    const cancelEdit = () => {
      editingSphereIndex.value = null;
    };

    const updateSpherePosition = (index) => {
      const sphere = spheres.value[index];
      sphere.position.set(
        sphere.position.x,
        sphere.position.y,
        sphere.position.z
      );
      editingSphereIndex.value = null;
    };

    onMounted(() => {
      initThree();
    });

    return {
      selectedElement,
      elements,
      spheres,
      editingSphereIndex,
      sceneContainer,
      addSphere,
      startEdit,
      cancelEdit,
      updateSpherePosition,
    };
  },
};
</script>

<style>
html,
body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.container {
  display: flex;
  width: 100%;
  height: 100vh;
}

.scene-container {
  width: 100%;
  height: 100vh;
  background-color: white;
}

.controls-container {
  position: absolute;
  top: 10px;
  right: 10px;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 10px;
  border-radius: 5px;
}

.coordinates {
  margin-top: 10px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.9);
  font-family: Arial, sans-serif;
  font-size: 14px;
  overflow-y: auto;
}
</style>