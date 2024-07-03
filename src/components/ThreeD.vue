<template>
  <div class="container">
    <div ref="sceneContainer" class="scene-container"></div>
    <div class="controls-container">
      <input v-model="inputString" placeholder="예: I -1 -1 -1" @keyup.enter="addSphere" />
      <button @click="addSphere">Add Sphere</button>
      <div class="coordinates">
        <div v-for="(sphere, index) in spheres" :key="sphere.uuid">
          <p>
            {{ sphere.name }}:
            x={{ sphere.position.x.toFixed(2) }}, y={{ sphere.position.y.toFixed(2) }}, z={{ sphere.position.z.toFixed(2) }}
          </p>
        </div>
      </div>
      <div class="selected-spheres">
        <h3>Selected Spheres:</h3>
        <ul>
          <li v-for="(sphere, index) in selectedSpheres" :key="sphere.uuid">
            {{ sphere.name }}
          </li>
        </ul>
      </div>
    </div>
    <div class="axes-container" ref="axesContainer"></div>
    <div class="instructions">
      <p>두 구체를 선택후 'b' or 'Ctrl+b' 누르면 결합이 생성/제거됩니다</p>
      <p>구체를 선택한 후 'Delete' 키를 누르면 선택된 구체가 삭제됩니다</p>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import * as THREE from "three";
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { ArrowHelper } from 'three';
export default {
  name: "ThreeDView",
  setup() {
    const inputString = ref('');
    const spheres = ref([]);
    const selectedSpheres = ref([]);
    const bonds = ref([]);
    const sceneContainer = ref(null);
    const axesContainer = ref(null);
    let isDraggingSphere = false;

    const elements = ref([
  { symbol: "H", name: "Hydrogen", color: 0xFFFFFF },
    { symbol: "Li", name: "Lithium", color: 0xFF69B4 },
  { symbol: "Be", name: "Beryllium", color: 0x32CD32 },
  { symbol: "B", name: "Boron", color: 0xFF8C00 },
  { symbol: "C", name: "Carbon", color: 0x696969 },
  { symbol: "N", name: "Nitrogen", color: 0x1E90FF },
  { symbol: "O", name: "Oxygen", color: 0xFF4500 },
  { symbol: "F", name: "Fluorine", color: 0x7CFC00 },
  { symbol: "Ne", name: "Neon", color: 0xFF1493 },
  { symbol: "Na", name: "Sodium", color: 0x4169E1 },
  { symbol: "Mg", name: "Magnesium", color: 0x00FF00 },
  { symbol: "Al", name: "Aluminum", color: 0xA9A9A9 },
  { symbol: "Si", name: "Silicon", color: 0xA52A2A },
  { symbol: "P", name: "Phosphorus", color: 0xFFD700 },
  { symbol: "S", name: "Sulfur", color: 0xFFFF00 },
  { symbol: "Cl", name: "Chlorine", color: 0x7FFF00 },
  { symbol: "Ar", name: "Argon", color: 0x1E90FF },
  { symbol: "K", name: "Potassium", color: 0x9370DB },
  { symbol: "Ca", name: "Calcium", color: 0x4682B4 },
  { symbol: "Sc", name: "Scandium", color: 0x00FA9A },
  { symbol: "Ti", name: "Titanium", color: 0xB0C4DE },
  { symbol: "V", name: "Vanadium", color: 0x7B68EE },
  { symbol: "Cr", name: "Chromium", color: 0xDC143C },
  { symbol: "Mn", name: "Manganese", color: 0xFF6347 },
  { symbol: "Fe", name: "Iron", color: 0xB22222 },
  { symbol: "Co", name: "Cobalt", color: 0x6495ED },
  { symbol: "Ni", name: "Nickel", color: 0x708090 },
  { symbol: "Cu", name: "Copper", color: 0xD2691E },
  { symbol: "Zn", name: "Zinc", color: 0x2E8B57 },
  { symbol: "Ga", name: "Gallium", color: 0x00CED1 },
  { symbol: "Ge", name: "Germanium", color: 0xFF00FF },
  { symbol: "As", name: "Arsenic", color: 0x8A2BE2 },
  { symbol: "Se", name: "Selenium", color: 0xFF4500 },
  { symbol: "Br", name: "Bromine", color: 0x8B0000 },
  { symbol: "Kr", name: "Krypton", color: 0x2F4F4F },
  { symbol: "Rb", name: "Rubidium", color: 0x8B4513 },
  { symbol: "Sr", name: "Strontium", color: 0x2E8B57 },
  { symbol: "Y", name: "Yttrium", color: 0x00BFFF },
  { symbol: "Zr", name: "Zirconium", color: 0x4682B4 },
  { symbol: "Nb", name: "Niobium", color: 0xD2B48C },
  { symbol: "Mo", name: "Molybdenum", color: 0xC71585 },
  { symbol: "Tc", name: "Technetium", color: 0x40E0D0 },
  { symbol: "Ru", name: "Ruthenium", color: 0xDAA520 },
  { symbol: "Rh", name: "Rhodium", color: 0xB0E0E6 },
  { symbol: "Pd", name: "Palladium", color: 0xFF69B4 },
  { symbol: "Ag", name: "Silver", color: 0xC0C0C0 },
  { symbol: "Cd", name: "Cadmium", color: 0x00FF7F },
  { symbol: "In", name: "Indium", color: 0x4682B4 },
  { symbol: "Sn", name: "Tin", color: 0x708090 },
  { symbol: "Sb", name: "Antimony", color: 0x8B008B },
  { symbol: "Te", name: "Tellurium", color: 0xFFD700 },
  { symbol: "I", name: "Iodine", color: 0xFF1493 },
  { symbol: "Xe", name: "Xenon", color: 0xFF4500 },
  { symbol: "Cs", name: "Cesium", color: 0xFFD700 },
  { symbol: "Ba", name: "Barium", color: 0x32CD32 },
  { symbol: "La", name: "Lanthanum", color: 0x00FF00 },
  { symbol: "Ce", name: "Cerium", color: 0xFF6347 },
  { symbol: "Pr", name: "Praseodymium", color: 0x4682B4 },
  { symbol: "Nd", name: "Neodymium", color: 0x1E90FF },
  { symbol: "Pm", name: "Promethium", color: 0xFF4500 },
  { symbol: "Sm", name: "Samarium", color: 0xFFD700 },
  { symbol: "Eu", name: "Europium", color: 0x32CD32 },
  { symbol: "Gd", name: "Gadolinium", color: 0x00FF00 },
  { symbol: "Tb", name: "Terbium", color: 0xFF6347 },
  { symbol: "Dy", name: "Dysprosium", color: 0x4682B4 },
  { symbol: "Ho", name: "Holmium", color: 0x1E90FF },
  { symbol: "Er", name: "Erbium", color: 0xFF4500 },
  { symbol: "Tm", name: "Thulium", color: 0xFFD700 },
  { symbol: "Yb", name: "Ytterbium", color: 0x32CD32 },
  { symbol: "Lu", name: "Lutetium", color: 0x00FF00 },
  { symbol: "Hf", name: "Hafnium", color: 0xFF6347 },
  { symbol: "Ta", name: "Tantalum", color: 0x4682B4 },
  { symbol: "W", name: "Tungsten", color: 0x1E90FF },
  { symbol: "Re", name: "Rhenium", color: 0xFF4500 },
  { symbol: "Os", name: "Osmium", color: 0xFFD700 },
  { symbol: "Ir", name: "Iridium", color: 0x32CD32 },
  { symbol: "Pt", name: "Platinum", color: 0x00FF00 },
  { symbol: "Au", name: "Gold", color: 0xFFD700 },
  { symbol: "Hg", name: "Mercury", color: 0xFF6347 },
  { symbol: "Tl", name: "Thallium", color: 0x4682B4 },
  { symbol: "Pb", name: "Lead", color: 0x1E90FF },
  { symbol: "Bi", name: "Bismuth", color: 0xFF4500 },
  { symbol: "Po", name: "Polonium", color: 0xFFD700 },
  { symbol: "At", name: "Astatine", color: 0x32CD32 },
  { symbol: "Rn", name: "Radon", color: 0x00FF00 },
  { symbol: "Fr", name: "Francium", color: 0xFF6347 },
  { symbol: "Ra", name: "Radium", color: 0x4682B4 },
  { symbol: "Ac", name: "Actinium", color: 0x1E90FF },
  { symbol: "Th", name: "Thorium", color: 0xFF4500 },
  { symbol: "Pa", name: "Protactinium", color: 0xFFD700 },
  { symbol: "U", name: "Uranium", color: 0x32CD32 },
  { symbol: "Np", name: "Neptunium", color: 0x00FF00 },
  { symbol: "Pu", name: "Plutonium", color: 0xFF6347 },
  { symbol: "Am", name: "Americium", color: 0x4682B4 },
  { symbol: "Cm", name: "Curium", color: 0x1E90FF },
  { symbol: "Bk", name: "Berkelium", color: 0xFF4500 },
  { symbol: "Cf", name: "Californium", color: 0xFFD700 },
  { symbol: "Es", name: "Einsteinium", color: 0x32CD32 },
  { symbol: "Fm", name: "Fermium", color: 0x00FF00 },
  { symbol: "Md", name: "Mendelevium", color: 0xFF6347 },
  { symbol: "No", name: "Nobelium", color: 0x4682B4 },
  { symbol: "Lr", name: "Lawrencium", color: 0x1E90FF },
  { symbol: "Rf", name: "Rutherfordium", color: 0xFF4500 },
  { symbol: "Db", name: "Dubnium", color: 0xFFD700 },
  { symbol: "Sg", name: "Seaborgium", color: 0x32CD32 },
  { symbol: "Bh", name: "Bohrium", color: 0x00FF00 },
  { symbol: "Hs", name: "Hassium", color: 0xFF6347 },
  { symbol: "Mt", name: "Meitnerium", color: 0x4682B4 },
  { symbol: "Ds", name: "Darmstadtium", color: 0x1E90FF },
  { symbol: "Rg", name: "Roentgenium", color: 0xFF4500 },
  { symbol: "Cn", name: "Copernicium", color: 0xFFD700 },
  { symbol: "Nh", name: "Nihonium", color: 0x32CD32 },
  { symbol: "Fl", name: "Flerovium", color: 0x00FF00 },
  { symbol: "Mc", name: "Moscovium", color: 0xFF6347 },
  { symbol: "Lv", name: "Livermorium", color: 0x4682B4 },
  { symbol: "Ts", name: "Tennessine", color: 0x1E90FF },
  { symbol: "Og", name: "Oganesson", color: 0xFF4500 },
    ]);

    let scene, camera, renderer, light, controls;
    let axesScene, axesCamera, axesRenderer, axesHelper;

    const createSphereTexture = (text) => {
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

    const deleteSphere = (sphere) => {
      bonds.value = bonds.value.filter(bond => {
        if (bond.sphere1 === sphere || bond.sphere2 === sphere) {
          scene.remove(bond.bond);
          if (bond.bond.geometry) bond.bond.geometry.dispose();
          if (bond.bond.material) bond.bond.material.dispose();
          return false;
        }
        return true;
      });

      scene.remove(sphere);
      if (sphere.geometry) sphere.geometry.dispose();
      if (sphere.material) {
        if (sphere.material.map) sphere.material.map.dispose();
        sphere.material.dispose();
      }

      spheres.value = spheres.value.filter(s => s !== sphere);
      selectedSpheres.value = selectedSpheres.value.filter(s => s !== sphere);

      renderer.renderLists.dispose();

      requestAnimationFrame(() => {
        scene.updateMatrixWorld(true);
        camera.updateProjectionMatrix();
        renderer.clear();
        renderer.render(scene, camera);
      });

      cleanScene();
    };

    const cleanScene = () => {
      const objectsToRemove = [];
      scene.traverse((object) => {
        if (object.type === 'Mesh' && !spheres.value.includes(object)) {
          objectsToRemove.push(object);
        }
      });

      objectsToRemove.forEach((object) => {
        scene.remove(object);
        if (object.geometry) object.geometry.dispose();
        if (object.material) {
          if (object.material.map) object.material.map.dispose();
          object.material.dispose();
        }
      });

      renderer.renderLists.dispose();
      renderer.render(scene, camera);
    };

    const initThree = () => {
      const onKeydown = (event) => {
        if (event.key === 'Delete') {
          event.preventDefault();
          if (selectedSpheres.value.length > 0) {
            const sphereToDelete = selectedSpheres.value[selectedSpheres.value.length - 1];
            deleteSphere(sphereToDelete);
            selectedSpheres.value.pop();

            requestAnimationFrame(() => {
              scene.updateMatrixWorld(true);
              camera.updateProjectionMatrix();
              renderer.clear();
              renderer.render(scene, camera);
              cleanScene();
            });
          }
        } else if (event.key === 'b' || (event.ctrlKey && event.key === 'b')) {
          event.preventDefault();
          if (selectedSpheres.value.length === 2) {
            toggleBond();
          }
        } else if (event.key === 'c') {
          event.preventDefault();
          removeAllBonds();
        }
      };

      window.addEventListener('keydown', onKeydown);

      // 메인 씬 설정
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
      camera.position.set(0, 5, 5);
      camera.lookAt(new THREE.Vector3(0, 0, 0));

      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setClearColor(0xcccccc, 0.5);
      sceneContainer.value.appendChild(renderer.domElement);

      controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.25;
      controls.screenSpacePanning = false;
      controls.maxPolarAngle = Math.PI / 2;

      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
      scene.add(ambientLight);

      light = new THREE.DirectionalLight(0xffffff, 0.5);
      light.position.set(5, 5, 5);
      scene.add(light);

      // 축 화살표를 위한 씬 설정
      axesScene = new THREE.Scene();
      axesCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0.1, 10);
      axesCamera.position.set(0, 0, 2);
      axesCamera.lookAt(new THREE.Vector3(0, 0, 0));

      axesRenderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
      axesRenderer.setSize(300, 300);
      axesRenderer.setClearColor(0x000000, 0);
      axesContainer.value.appendChild(axesRenderer.domElement);

      // 축 화살표 생성
      const axesHelperSize = 0.8; // 크기 조정
      const arrowHeadSize = 0.2; // 화살표 머리 크기
      const arrowHeadWidth = 0.1; // 화살표 머리 너비
      const xAxis = new ArrowHelper(
  new THREE.Vector3(1, 0, 0),
  new THREE.Vector3(0, 0, 0),
  axesHelperSize,
  0xff0000,
  arrowHeadSize,
  arrowHeadWidth
);
axesScene.add(xAxis);

// Y축 (녹색)
const yAxis = new ArrowHelper(
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(0, 0, 0),
  axesHelperSize,
  0x00ff00,
  arrowHeadSize,
  arrowHeadWidth
);
axesScene.add(yAxis);

// Z축 (파란색)
const zAxis = new ArrowHelper(
  new THREE.Vector3(0, 0, 1),
  new THREE.Vector3(0, 0, 0),
  axesHelperSize,
  0x0000ff,
  arrowHeadSize,
  arrowHeadWidth
);
axesScene.add(zAxis);

// 축 레이블 추가
const addAxisLabel = (text, position) => {
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');
  const size = 256; // 캔버스 크기 설정
  canvas.width = size;
  canvas.height = size;

  context.font = 'Bold 100px Arial';
  context.fillStyle = 'black';
  context.textAlign = 'center';
  context.textBaseline = 'middle';
  context.fillText(text, size / 2, size / 2); // 텍스트를 중앙에 배치

  const texture = new THREE.CanvasTexture(canvas);
  const spriteMaterial = new THREE.SpriteMaterial({ map: texture });
  const sprite = new THREE.Sprite(spriteMaterial);
  sprite.position.copy(position);
  sprite.scale.set(0.2, 0.2, 0.2); // 스케일 조정
  axesScene.add(sprite);
};

addAxisLabel('X', new THREE.Vector3(axesHelperSize + 0.1, 0, 0));
addAxisLabel('Y', new THREE.Vector3(0, axesHelperSize + 0.1, 0));
addAxisLabel('Z', new THREE.Vector3(0, 0, axesHelperSize + 0.1));
      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();
      const plane = new THREE.Plane();

      let isDragging = false;
      let selectedObject = null;
      let offset = new THREE.Vector3();

      const onMouseDown = (event) => {
        if (event.button === 0) {
          mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
          mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

          raycaster.setFromCamera(mouse, camera);
          const intersects = raycaster.intersectObjects(spheres.value);

          if (intersects.length > 0) {
            const clickedSphere = intersects[0].object;
            isDragging = true;
            isDraggingSphere = true;
            selectedObject = clickedSphere;
            controls.enabled = false;
            const intersectPoint = intersects[0].point;
            offset.copy(intersectPoint).sub(selectedObject.position);
            plane.setFromNormalAndCoplanarPoint(
              camera.getWorldDirection(plane.normal),
              intersectPoint
            );
          }
        }
      };

      const onMouseMove = (event) => {
        if (!isDragging) return;

        if (selectedObject && isDraggingSphere) {
          mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
          mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

          raycaster.setFromCamera(mouse, camera);
          const planeIntersect = new THREE.Vector3();
          raycaster.ray.intersectPlane(plane, planeIntersect);

          if (planeIntersect) {
            selectedObject.position.copy(planeIntersect).sub(offset);
            updateBonds();
          }
        }
      };

      const onMouseUp = () => {
        isDragging = false;
        isDraggingSphere = false;
        selectedObject = null;
        controls.enabled = true;
      };

      const onContextMenu = (event) => {
        event.preventDefault();

        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        const intersects = raycaster.intersectObjects(spheres.value);

        if (intersects.length > 0) {
          const clickedSphere = intersects[0].object;

          const index = selectedSpheres.value.findIndex(s => s.uuid === clickedSphere.uuid);
          if (index === -1) {
            if (selectedSpheres.value.length < 2) {
              selectedSpheres.value.push(clickedSphere);
              clickedSphere.material.opacity = 0.5;
              clickedSphere.material.transparent = true;
            } else {
              alert("2개 까지만 선택이 가능합니다.");
            }
          } else {
            selectedSpheres.value.splice(index, 1);
            clickedSphere.material.opacity = 1;
            clickedSphere.material.transparent = false;
          }
        }
      };

      window.addEventListener("mousedown", onMouseDown, false);
      window.addEventListener("contextmenu", onContextMenu, false);
      window.addEventListener("mousemove", onMouseMove, false);
      window.addEventListener("mouseup", onMouseUp, false);

      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();

        // 메인 씬 렌더링
        renderer.render(scene, camera);

        // 카메라 회전을 동기화
        axesCamera.position.copy(camera.position);
        axesCamera.position.sub(controls.target);
        axesCamera.position.setLength(2);
        axesCamera.lookAt(axesScene.position);

        // 축 화살표 씬 렌더링
        axesRenderer.render(axesScene, axesCamera);

        light.position.copy(camera.position);
      };
      animate();

      window.addEventListener("resize", () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
        controls.update();
      });
    };

    const addSphere = () => {
      const parts = inputString.value.split(' ');
      if (parts.length !== 4) {
        alert('올바른 형식으로 입력해주세요. 예: I -1 -1 -1');
        return;
      }

      const symbol = parts[0];
      const x = parseFloat(parts[1]);
      const y = parseFloat(parts[2]);
      const z = parseFloat(parts[3]);

      if (isNaN(x) || isNaN(y) || isNaN(z)) {
        alert('올바른 좌표값을 입력해주세요.');
        return;
      }

      const element = elements.value.find(el => el.symbol === symbol);
      if (!element) {
        alert('올바른 원소 기호를 입력해주세요.');
        return;
      }

      const texture = createSphereTexture(symbol);

      const geometry = new THREE.SphereGeometry(0.4, 64, 64);
      const material = new THREE.MeshPhongMaterial({
        color: element.color,
        map: texture,
        shininess: 100,
        transparent: true,
        opacity: 1,
      });

      const sphere = new THREE.Mesh(geometry, material);
      sphere.position.set(x, y, z);
      sphere.name = `${symbol}`;

      scene.add(sphere);
      spheres.value.push(sphere);

      inputString.value = '';
      controls.update();
      renderer.render(scene, camera);
    };

    const toggleBond = () => {
      const [sphere1, sphere2] = selectedSpheres.value;

      const existingBondIndex = bonds.value.findIndex(
        bond => (bond.sphere1 === sphere1 && bond.sphere2 === sphere2) ||
          (bond.sphere1 === sphere2 && bond.sphere2 === sphere1)
      );

      if (existingBondIndex !== -1) {
        const bondToToggle = bonds.value[existingBondIndex];
        bondToToggle.bond.visible = !bondToToggle.bond.visible;
      } else {
        const start = sphere1.position;
        const end = sphere2.position;
        const direction = new THREE.Vector3().subVectors(end, start);
        const distance = direction.length();
        const sphereRadius = 0.4;

        const bondLength = Math.max(0.001, distance - 2 * sphereRadius);

        const cylinderGeometry = new THREE.CylinderGeometry(0.1, 0.1, bondLength, 32);
        const material = new THREE.MeshPhongMaterial({ color: 0xD3D3D3 });
        const cylinder = new THREE.Mesh(cylinderGeometry, material);

        const midpoint = new THREE.Vector3().addVectors(start, end).multiplyScalar(0.5);
        cylinder.position.copy(midpoint);
        cylinder.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction.normalize());

        scene.add(cylinder);
        bonds.value.push({
          bond: cylinder,
          sphere1: sphere1,
          sphere2: sphere2
        });
      }

      selectedSpheres.value.forEach(sphere => {
        sphere.material.opacity = 1;
        sphere.material.transparent = false;
      });
      selectedSpheres.value = [];

      renderer.render(scene, camera);
    };

    const updateBonds = () => {
      bonds.value.forEach(bondObj => {
        const start = bondObj.sphere1.position;
        const end = bondObj.sphere2.position;
        const direction = new THREE.Vector3().subVectors(end, start);
        const distance = direction.length();
        const sphereRadius = 0.4;

        const bondLength = Math.max(0.001, distance - 2 * sphereRadius);

        const midpoint = new THREE.Vector3().addVectors(start, end).multiplyScalar(0.5);
        bondObj.bond.position.copy(midpoint);
        bondObj.bond.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), direction.normalize());
        bondObj.bond.scale.y = bondLength / bondObj.bond.geometry.parameters.height;
      });
    };

    const removeAllBonds = () => {
      bonds.value.forEach(bondObj => {
        scene.remove(bondObj.bond);
        bondObj.bond.geometry.dispose();
        bondObj.bond.material.dispose();
      });
      bonds.value = [];
      renderer.renderLists.dispose();
      renderer.render(scene, camera);
    };

    onMounted(() => {
      initThree();
    });

    return {
      inputString,
      spheres,
      selectedSpheres,
      sceneContainer,
      addSphere,
      axesContainer
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
  position: relative;
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
  max-height: 200px;
}

.selected-spheres {
  margin-top: 10px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.9);
  font-family: Arial, sans-serif;
  font-size: 14px;
}

.instructions {
  position: absolute;
  bottom: 10px;
  left: 10px;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 10px;
  border-radius: 5px;
  font-family: Arial, sans-serif;
  font-size: 14px;
}

.axes-container {
  position: absolute;
  bottom: 160px;
  left: 10px;
  width: 300px; /* 크기를 300px로 조정 */
  height: 300px; /* 크기를 300px로 조정 */
  z-index: 10;
  /* background-color: rgba(255, 255, 255, 0.9); */
  border-radius: 5px;
}
</style>
