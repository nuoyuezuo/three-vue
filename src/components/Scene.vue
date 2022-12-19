<!--  -->
<template>
  <div class="scene" ref="sceneDiv"></div>
</template>

<script  setup>
import { reactive,toRefs,onBeforeMount,onMounted,defineComponent, render,ref} from 'vue'
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader"
import * as Three from 'three';
import gsap from "gsap";
import * as dat from "dat.gui"
import { minify } from 'terser';
//声明场景元素div
let sceneDiv = ref(null)

//创建gui
const gui = new dat.GUI()

//初始化场景
const scene = new Three.Scene()

//创建相机
const camera = new Three.PerspectiveCamera(
    75,
    window.innerHeight / window.innerHeight,
    1,
    50000
);
//相机位置
camera.position.set(0,0,10)
scene.add(camera)

//辅助坐标系
const axesHelper = new Three.AxesHelper(5);
scene.add(axesHelper)

//平面
const plane = new Three.Mesh(
    new Three.PlaneGeometry(20,20),
    new Three.MeshBasicMaterial({
        side:Three.DoubleSide
    })
)
plane.position.set(0,0,0)
plane.receiveShadow = true;
plane.rotateX(-Math.PI * 0.5) 
scene.add(plane)


//初始化渲染器
const renderer = new Three.WebGLRenderer();
//设置渲染尺寸大小
renderer.setSize(window.innerWidth,window.innerHeight);
renderer.shadowMap.enabled = true

//监听屏幕变化

window.addEventListener("resize",()=>{
    //更新摄像头
    camera.aspect = window.innerWidth / window.innerHeight
    //更细摄像机矩阵投影矩阵
    camera.updateProjectionMatrix();

    //更新渲染器
    renderer.setSize(window.innerWidth,window.innerHeight)
    //设置渲染器的像素比例
    renderer.setPixelRatio(window.devicePixelRatio)
})

//控制器
const controls = new OrbitControls(camera,renderer.domElement)
//控制器阻尼
controls.enbaleDamping = true;


onMounted(()=>{
    sceneDiv.value.appendChild(renderer.domElement)
    animate()
})
const clock = new Three.Clock()
function animate(){
    controls.update();
    const time = clock.getElapsedTime();
    requestAnimationFrame(animate);
    renderer.render(scene,camera)
}
</script>
<style scoped>
.scene{
    width: 100vw;
    height: 100vh;
    position: absolute;
    z-index: 100;
    left: 0;
    top: 0;
}
</style>