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
            x={{ sphere.position.x.toFixed(2) }}, y={{
              sphere.position.y.toFixed(2)
            }}, z={{ sphere.position.z.toFixed(2) }}
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
    <div class="instructions">
      <p>두 구체를 선택후 'b' or 'Ctrl+b' 누르면 결합이 생성/제거됩니다</p>
      <p>구체를 선택한 후 'Delete' 키를 누르면 선택된 구체가 삭제됩니다</p>
    </div>
    <div class="axes-container">
      <div class="axis-label">X</div>
      <div class="axis-label">Y</div>
      <div class="axis-label">Z</div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import * as THREE from "three";
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

export default {
  name: "ThreeDView",
  setup() {
    const inputString = ref('');
    const spheres = ref([]);
    const selectedSpheres = ref([]);
    const bonds = ref([]);
    const sceneContainer = ref(null);
    let isDraggingSphere = false;

    const elements = ref([
      { symbol: "H", name: "Hydrogen", color: 0xFFFFFF },
      { symbol: "He", name: "Helium", color: 0xFFD700 },
    ]);

    let scene, camera, renderer, light, controls;

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

    const createLabelTexture = (text) => {
      const canvas = document.createElement('canvas');
      const context = canvas.getContext('2d');
      const size = 256;

      canvas.width = size;
      canvas.height = size;

      context.clearRect(0, 0, size, size);

      context.fillStyle = 'black';
      context.font = '24px Arial';
      context.textAlign = 'center';
      context.textBaseline = 'middle';

      context.fillText(text, size / 2, size / 2);

      return new THREE.CanvasTexture(canvas);
    };

    const deleteSphere = (sphere) => {
      // 구체와 관련된 모든 결합 제거
      bonds.value = bonds.value.filter(bond => {
        if (bond.sphere1 === sphere || bond.sphere2 === sphere) {
          scene.remove(bond.bond);
          if (bond.bond.geometry) bond.bond.geometry.dispose();
          if (bond.bond.material) bond.bond.material.dispose();
          return false;
        }
        return true;
      });

      // 구체 제거
      scene.remove(sphere);
      if (sphere.geometry) sphere.geometry.dispose();
      if (sphere.material) {
        if (sphere.material.map) sphere.material.map.dispose();
        sphere.material.dispose();
      }

      // spheres 배열에서 제거
      spheres.value = spheres.value.filter(s => s !== sphere);

      // 선택된 구체 목록에서 제거
      selectedSpheres.value = selectedSpheres.value.filter(s => s !== sphere);

      // Three.js의 캐시 초기화
      renderer.renderLists.dispose();

      console.log('Sphere deleted:', sphere.name);
      console.log('Remaining spheres:', spheres.value.length);
      console.log('Scene children count:', scene.children.length);

      // 씬 다시 렌더링
      requestAnimationFrame(() => {
        scene.updateMatrixWorld(true);
        camera.updateProjectionMatrix();
        renderer.clear();
        renderer.render(scene, camera);
      });

      // 씬 정리
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

      console.log('Scene cleaned');
      console.log('Scene children count after cleaning:', scene.children.length);
    };

    const initThree = () => {
      const onKeydown = (event) => {
        if (event.key === 'Delete') {
          event.preventDefault();
          console.log('Delete key pressed');
          if (selectedSpheres.value.length > 0) {
            const sphereToDelete = selectedSpheres.value[selectedSpheres.value.length - 1];
            deleteSphere(sphereToDelete);
            selectedSpheres.value.pop();

            // 씬 강제 업데이트
            requestAnimationFrame(() => {
              scene.updateMatrixWorld(true);
              camera.updateProjectionMatrix();
              renderer.clear();
              renderer.render(scene, camera);
              cleanScene();  // 추가된 부분
            });

            console.log('Deletion complete');
            console.log('Remaining spheres:', spheres.value.length);
            console.log('Scene children count:', scene.children.length);
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

      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setClearColor(0xcccccc, 0.5);

      sceneContainer.value.appendChild(renderer.domElement);
      scene = new THREE.Scene();

      camera = new THREE.PerspectiveCamera(
        45,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );
      camera.position.set(0, 5, 5);
      camera.lookAt(new THREE.Vector3(0, 0, 0));
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

      // X, Y, Z 축을 나타내는 화살표와 레이블 추가
      const length = 1;
      const hexX = 0xff0000;
      const hexY = 0x00ff00;
      const hexZ = 0x0000ff;

      const arrowX = new THREE.ArrowHelper(new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 0, 0), length, hexX);
      const arrowY = new THREE.ArrowHelper(new THREE.Vector3(0, 1, 0), new THREE.Vector3(0, 0, 0), length, hexY);
      const arrowZ = new THREE.ArrowHelper(new THREE.Vector3(0, 0, 1), new THREE.Vector3(0, 0, 0), length, hexZ);

      const labelX = new THREE.Sprite(new THREE.SpriteMaterial({ map: createLabelTexture('X'), transparent: true }));
      const labelY = new THREE.Sprite(new THREE.SpriteMaterial({ map: createLabelTexture('Y'), transparent: true }));
      const labelZ = new THREE.Sprite(new THREE.SpriteMaterial({ map: createLabelTexture('Z'), transparent: true }));

      labelX.position.set(1.2, 0, 0);
      labelY.position.set(0, 1.2, 0);
      labelZ.position.set(0, 0, 1.2);

      scene.add(arrowX);
      scene.add(arrowY);
      scene.add(arrowZ);

      scene.add(labelX);
      scene.add(labelY);
      scene.add(labelZ);

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

          console.log('Selected spheres:', selectedSpheres.value.map(s => s.name));
        }
      };

      window.addEventListener("mousedown", onMouseDown, false);
      window.addEventListener("contextmenu", onContextMenu, false);
      window.addEventListener("mousemove", onMouseMove, false);
      window.addEventListener("mouseup", onMouseUp, false);

      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();

        scene.updateMatrixWorld(true);
        camera.updateProjectionMatrix();
        renderer.clear();
        renderer.render(scene, camera);
        light.position.copy(camera.position);

        console.log('Animate - Scene children count:', scene.children.length);
        console.log('Animate - Spheres count:', spheres.value.length);
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

      console.log('Sphere added:', sphere.name);
      console.log('Scene children count:', scene.children.length);
      console.log('Spheres array length:', spheres.value.length);

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
        console.log('Bond visibility toggled:', bondToToggle.bond.visible);
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

        console.log('Bond created');
      }

      selectedSpheres.value.forEach(sphere => {
        sphere.material.opacity = 1;
        sphere.material.transparent = false;
      });
      selectedSpheres.value = [];

      renderer.render(scene, camera);

      console.log('Current bonds:', bonds.value);
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
  bottom: 10px;
  left: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.axis-label {
  background-color: rgba(255, 255, 255, 0.9);
  padding: 5px;
  border-radius: 3px;
  font-family: Arial, sans-serif;
  font-size: 14px;
}
</style>