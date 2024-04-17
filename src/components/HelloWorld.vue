<script setup>
/**
 * 投骰子
 */
import { ref } from "vue";
import * as THREE from "three";
// @ts-ignore
import { RoundedBoxGeometry } from "three/examples/jsm/geometries/RoundedBoxGeometry";
// @ts-ignore
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import * as CANNON from "cannon-es";
import useInitialize from "@/hooks/threejs/useInitialize";
import mp3 from "@/assets/dice.mp3";

const diceSize = 10; // 骰子的长宽高
const diceBevelRadius = 1.4; // 骰子的棱的曲面半径
const maxDiceNum = 10; // 最多骰子数量
const g = 300; // 重力加速度
const restitution = 0.27; // 物理世界的反弹系数
const cameraInitPosition = { x: 0, y: 40, z: 140 }; // 相机位置
const floorY = -60; // 地板的y位置
const maxDistance = 1000; // 轨道控制器的最远距离

let controls = null;
let diceList = []; // 存放所有骰子对象
let physicsWorld = null;
let isStillMoving = false;

// const globalContext = useGlobalContext() as Ref<GlobalContext>;
const containerRef = ref(null);
const diceNum = ref(5); // 投骰子的个数
diceNum.value = Number(prompt("请打开手机声音并输入骰子个数") || 5)
// 创建并添加地板
const createFloor = (scene) => {
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(300, 300),
    new THREE.ShadowMaterial({
      opacity: 0.1,
    })
  );
  floor.receiveShadow = true;
  floor.position.y = floorY;
  floor.quaternion.setFromAxisAngle(new THREE.Vector3(-1, 0, 0), Math.PI * 0.5);
  scene.add(floor);

  const floorBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane(),
  });
  // @ts-ignore
  floorBody.position.copy(floor.position);
  // @ts-ignore
  floorBody.quaternion.copy(floor.quaternion);
  physicsWorld?.addBody(floorBody);
};

// 创建骰子各点数对应的Canvas
const getDiceDotNumCanvas = (dotNum) => {
  const canvas = document.createElement("canvas");
  const size = 500;
  canvas.width = size;
  canvas.height = size;
  const ctx = canvas.getContext("2d");
  ctx.fillStyle = "#eeeeee";
  ctx.fillRect(0, 0, size, size);
  switch (dotNum) {
    case 1:
      ctx.fillStyle = "#d30704";
      ctx.arc(size / 2, size / 2, size * 0.2, 0, 2 * Math.PI);
      ctx.fill();
      break;
    case 2:
      ctx.fillStyle = "#153f87";
      ctx.arc(size / 2, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc(size / 2, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      break;
    case 3:
      ctx.fillStyle = "#153f87";
      ctx.arc((size * 8) / 11, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc(size / 2, size / 2, size * 0.12, 0, 2 * Math.PI);
      ctx.arc((size * 3) / 11, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      break;
    case 4:
      ctx.fillStyle = "#d30704";
      ctx.beginPath();
      ctx.arc((size * 3) / 11, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc((size * 8) / 11, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      ctx.beginPath();
      ctx.arc((size * 3) / 11, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc((size * 8) / 11, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      break;
    case 5:
      ctx.fillStyle = "#153f87";
      ctx.beginPath();
      ctx.arc((size * 3) / 11, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc((size * 8) / 11, (size * 3) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      ctx.beginPath();
      ctx.arc((size * 3) / 11, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.arc((size * 8) / 11, (size * 8) / 11, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      ctx.beginPath();
      ctx.arc(size / 2, size / 2, size * 0.12, 0, 2 * Math.PI);
      ctx.fill();
      break;
    case 6:
      ctx.fillStyle = "#153f87";
      ctx.beginPath();
      ctx.arc((size * 3) / 10, (size * 3) / 11, size * 0.1, 0, 2 * Math.PI);
      ctx.arc((size * 7) / 10, (size * 3) / 11, size * 0.1, 0, 2 * Math.PI);
      ctx.fill();
      ctx.beginPath();
      ctx.arc((size * 3) / 10, size / 2, size * 0.1, 0, 2 * Math.PI);
      ctx.arc((size * 7) / 10, size / 2, size * 0.1, 0, 2 * Math.PI);
      ctx.fill();
      ctx.beginPath();
      ctx.arc((size * 3) / 10, (size * 8) / 11, size * 0.1, 0, 2 * Math.PI);
      ctx.arc((size * 7) / 10, (size * 8) / 11, size * 0.1, 0, 2 * Math.PI);
      ctx.fill();
      break;
  }
  return canvas;
};

// 骰子几何体
const diceGeometry = new RoundedBoxGeometry(
  diceSize,
  diceSize,
  diceSize,
  1,
  diceBevelRadius
);

// 骰子材质
const materialList = [
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(1)),
    color: "#ffffff",
  }), // 右面
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(6)),
    color: "#ffffff",
  }), // 左面
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(3)),
    color: "#ffffff",
  }), // 上面
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(4)),
    color: "#ffffff",
  }), // 下面
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(5)),
    color: "#ffffff",
  }), // 后面
  new THREE.MeshStandardMaterial({
    map: new THREE.CanvasTexture(getDiceDotNumCanvas(2)),
    color: "#ffffff",
  }), // 前面
];

// 创建并添加所有骰子
const createDice = (scene) => {
  for (let i = 0; i < maxDiceNum; i++) {
    // 创建骰子
    const dice = new THREE.Mesh(diceGeometry, materialList);
    dice.castShadow = true;
    // 添加到场景中
    scene.add(dice);
    const body = new CANNON.Body({
      mass: 1,
      shape: new CANNON.Box(
        new CANNON.Vec3(diceSize / 2, diceSize / 2, diceSize / 2)
      ),
      sleepTimeLimit: 0.1,
    });
    physicsWorld?.addBody(body);
    diceList.push({ mesh: dice, body });
  }
};

// 投骰子
const throwDice = () => {
  if (isStillMoving) return;
  if (!diceNum.value) {
    // message.warning("page.threeJs3D.enterDiceNum");
    console.log("page.threeJs3D.enterDiceNum");
    return;
  }
  isStillMoving = true;
  diceList.forEach(
    (item, index) => {
      if (index <= (diceNum.value) - 1) {
        let x = 10;
        let y = 24;
        let z = -Math.floor(index / 2) * diceSize * 1.5;
        if ((index + 1) % 2 === 0) {
          x += diceSize * 1.5;
        }
        item.body.position = new CANNON.Vec3(x, y, z);
        // @ts-ignore
        item.mesh.position.copy(item.body.position);
        item.mesh.rotation.set(
          1 * Math.PI * Math.random(),
          0,
          2 * Math.PI * Math.random()
        );
        // @ts-ignore
        item.body.quaternion.copy(item.mesh.quaternion);
        const force = 1 + 5 * Math.random();
        item.body.applyImpulse(new CANNON.Vec3(-16, force, -5));
      } else {
        // 隐藏其他不需要的骰子
        item.body.position = new CANNON.Vec3(
          0,
          floorY + diceSize / 2,
          maxDistance + diceSize * index
        );
        // @ts-ignore
        item.mesh.position.copy(item.body.position);
        item.mesh.rotation.set(0, 0, 0);
        // @ts-ignore
        item.body.quaternion.copy(item.mesh.quaternion);
      }
    }
  );
  setTimeout(() => {
    const music = new Audio(mp3);
    music.play();
  }, 500);
  setTimeout(() => {
    isStillMoving = false;
  }, 2000);
};

// 创建物理世界
const createPhysicsWorld = () => {
  physicsWorld = new CANNON.World({
    allowSleep: true,
    gravity: new CANNON.Vec3(0, -g, 0),
  });
  physicsWorld.defaultContactMaterial.restitution = restitution; // 反弹系数
};

const initializeHandle = (
  scene,
  camera,
  renderer
) => {
  if (containerRef.value) {
    scene.background = new THREE.Color("#224141");
    camera.position.set(
      cameraInitPosition.x,
      cameraInitPosition.y,
      cameraInitPosition.z
    );
    camera.lookAt(0, 0, 0);
    renderer.setClearColor("#224141");
    renderer.shadowMap.enabled = true;

    // 添加环境光
    const light = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(light);

    // 添加右上角灯光
    const rightTopLight = new THREE.PointLight(0xffffff, 1);
    rightTopLight.position.set(100, 200, -10);
    rightTopLight.castShadow = true;
    rightTopLight.shadow.camera.near = 5;
    rightTopLight.shadow.camera.far = 400;
    scene.add(rightTopLight);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableRotate = false; // 禁止旋转，只能缩放
    controls.maxDistance = maxDistance; // 最远距离

    // 创建物理世界
    createPhysicsWorld();

    // 创建并添加地板
    createFloor(scene);

    // 创建并添加骰子
    createDice(scene);

    // 投骰子
    throwDice();
  }
};

const renderHandle = () => {
  physicsWorld?.fixedStep();
  for (let item of diceList) {
    // @ts-ignore
    item.mesh.position.copy(item.body.position);
    // @ts-ignore
    item.mesh.quaternion.copy(item.body.quaternion);
  }
  controls?.update();
};

useInitialize(
  containerRef,
  initializeHandle,
  null,
  renderHandle
);

// watch(
//   () => globalContext.value.menuWidth,
//   () => {
//     resize();
//   }
// );
</script>

<template>
  <div class="container" ref="containerRef">
    <div class="btn">
      <div class="btn_inner" @click="throwDice">开扔</div>
    </div>
    <div class="numBox">
      <!-- <a-input-number
        :min="1"
        :max="maxDiceNum"
        :precision="0"
        :value="diceNum"
        :onChange="(value: number | null)=>{
            diceNum = value;
          }"
      /> -->
    </div>
  </div>
</template>

<style scoped lang="scss">
.container {
  width: 100vw;
  height: 100vh;
  background-color: #224141;
  overflow: hidden;
  position: relative;

  canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }

  .btn {
    position: absolute;
    top: 60px;
    left: 0;
    right: 0;
    width: 120px;
    height: 120px;
    border-radius: 50%;
    margin: 0 auto;
    padding: 10px;
    background-color: #bbb;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1;

    .btn_inner {
      display: flex;
      flex: 1;
      justify-content: center;
      align-items: center;
      font-size: 24px;
      font-weight: 600;
      height: 100%;
      border-radius: 50%;
      color: #fff;

      background-color: #1a6840;
      cursor: pointer;

      &:active {
        background-color: #6a9c89;
      }
    }
  }

  .numBox {
    position: absolute;
    top: 60px;
    right: 50px;
    display: flex;
    align-items: center;
    font-size: 18px;
    z-index: 1;

    :deep(.ant-input-number) {
      width: 120px !important;
      height: 45px !important;

      .ant-input-number-input-wrap {
        height: 45px !important;
        line-height: 45px !important;

        input {
          width: 120px;
          height: 45px !important;
        }
      }
    }
  }
}
</style>