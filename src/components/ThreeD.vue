<template>
  <div ref="sceneContainer" class="scene-container">
    <div ref="coordinates" class="coordinates"></div>
  </div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

export default {
  name: "ThreeDView",
  mounted() {
    this.initThree();
  },
  methods: {
    initThree() {
      // Renderer 설정
      const renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setClearColor(0xffffff, 1);
      this.$refs.sceneContainer.appendChild(renderer.domElement);

      // 씬 생성
      const scene = new THREE.Scene();

      // 카메라 생성
      const camera = new THREE.PerspectiveCamera(
        32,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
      );
      camera.position.z = 5;
      this.camera = camera; // 카메라를 this에 저장

      // OrbitControls 설정
      const controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true;
      controls.dampingFactor = 0.25;
      controls.enableZoom = true;

      // 광원 추가
      const light = new THREE.DirectionalLight(0xffffff, 1);
      light.position.copy(camera.position); // 광원 위치를 카메라 위치로 설정
      scene.add(light);
      this.light = light; // 광원을 this에 저장

      // 바닥 생성
      const floorGeometry = new THREE.PlaneGeometry(100, 100);
      const floorMaterial = new THREE.MeshBasicMaterial({
        color: 0xaaaaaa,
        side: THREE.DoubleSide,
      });
      const floor = new THREE.Mesh(floorGeometry, floorMaterial);
      floor.rotation.x = Math.PI / 2;
      floor.position.y = -1; // 구체 아래에 바닥 배치
      scene.add(floor);

      // 벽 생성
      const wallGeometry = new THREE.PlaneGeometry(100, 100);
      const wallMaterial = new THREE.MeshBasicMaterial({
        color: 0xcccccc,
        side: THREE.DoubleSide,
      });
      const wall = new THREE.Mesh(wallGeometry, wallMaterial);
      wall.position.z = -10; // 카메라 뒤에 벽 배치
      scene.add(wall);

      // 텍스처에 글자 추가
      const createTextTexture = (text) => {
        const canvas = document.createElement("canvas");
        const context = canvas.getContext("2d");
        const size = 256;

        canvas.width = size;
        canvas.height = size;

        // 배경 색상 설정
        context.fillStyle = "white";
        context.fillRect(0, 0, size, size);

        // 텍스트 스타일 설정
        context.fillStyle = "black";
        context.font = "48px Arial";
        context.textAlign = "center";
        context.textBaseline = "middle";

        // 텍스트 그리기
        context.fillText(text, size / 2, size / 2);

        return new THREE.CanvasTexture(canvas);
      };

      const texture = createTextTexture("   Na   "); // 구체 표면에 나오는 글씨

      // 구체 생성
      const geometry = new THREE.SphereGeometry(0.4, 64, 64); // SphereGeometry 생성하여 구체를 만듦
      // 재질 생성
      const material = new THREE.MeshPhongMaterial({
        map: texture,
        shininess: 100, // 제일 맨들맨들하게, 반사 잘되게
      });
      const sphere = new THREE.Mesh(geometry, material); // 이 둘을 합쳐서
      scene.add(sphere); // 씬에 생성

      // Raycaster와 Mouse Vector 설정
      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();
      const plane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);

      // 드래그 상태 변수
      let isDragging = false;
      let selectedObject = null;
      let offset = new THREE.Vector3();

      const onMouseDown = (event) => {
        // 마우스 좌표를 정규화된 디바이스 좌표로 변환
        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        // Raycaster로 교차하는 객체 찾기
        raycaster.setFromCamera(mouse, camera);
        const intersects = raycaster.intersectObject(sphere);

        if (intersects.length > 0) {
          controls.enabled = false; // 드래그하는 동안 카메라 움직임 비활성화
          isDragging = true;
          selectedObject = intersects[0].object;

          // 평면과의 교차점을 이용하여 오프셋 계산
          const intersectPoint = intersects[0].point;
          offset.copy(intersectPoint).sub(selectedObject.position);

          // 평면을 클릭 위치와 카메라 방향으로 설정
          plane.setFromNormalAndCoplanarPoint(
            camera.getWorldDirection(plane.normal),
            intersectPoint
          );
        }
      };

      const onMouseMove = (event) => {
        if (!isDragging) return;

        // 마우스 좌표를 정규화된 디바이스 좌표로 변환
        mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        // Raycaster로 새로운 위치 계산
        raycaster.setFromCamera(mouse, camera);
        const planeIntersect = new THREE.Vector3();
        raycaster.ray.intersectPlane(plane, planeIntersect);

        if (planeIntersect) {
          selectedObject.position.copy(planeIntersect).sub(offset);
          updateCoordinates();
        }
      };

      const onMouseUp = () => {
        controls.enabled = true; // 드래그가 끝나면 카메라 움직임 활성화
        isDragging = false;
        selectedObject = null;
      };

      // 이벤트 리스너 설정
      window.addEventListener("mousedown", onMouseDown, false);
      window.addEventListener("mousemove", onMouseMove, false);
      window.addEventListener("mouseup", onMouseUp, false);

      // 개체의 좌표값을 화면에 표시
      const updateCoordinates = () => {
        const { x, y, z } = sphere.position;
        this.$refs.coordinates.innerText = `Coordinates: x=${x.toFixed(
          2
        )}, y=${y.toFixed(2)}, z=${z.toFixed(2)}`;
      };

      // 애니메이션 루프
      const animate = () => {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);

        // 카메라 위치를 기반으로 광원 위치 업데이트
        light.position.copy(camera.position);
      };
      animate();

      // 윈도우 리사이즈 이벤트 핸들러
      window.addEventListener("resize", () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      });
    },
  },
};
</script>

<style>
.scene-container {
  width: 100%;
  height: 100vh;
  position: relative;
  background-color: white;
}
.coordinates {
  position: absolute;
  top: 10px;
  left: 10px;
  background-color: rgba(255, 255, 255, 0.7);
  padding: 5px;
  border-radius: 5px;
  font-family: Arial, sans-serif;
  font-size: 14px;
}
</style>
