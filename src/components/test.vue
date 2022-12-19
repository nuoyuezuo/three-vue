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
      new Three.MeshBasicMaterial()
  )
  plane.position.set(0,0,-6)
  plane.receiveShadow = true;
  //scene.add(plane)
  
  
  //生成模型
  const gltfloader = new GLTFLoader()
  gltfloader.load('./model/city.glb',(gltf)=>{
      
  
      gltf.scene.traverse((item)=>{
          if(item.type == "Mesh"){
              const cityMaterial = new Three.MeshBasicMaterial({
                  color: new Three.Color(0x00fff)
              })
              
                      //修改材质
              item.material.onBeforeCompile = (shader) =>{
                  shader.fragmentShader = shader.fragmentShader.replace(
                      '#include <dithering_fragment>',
                      `
                          #include <dithering_fragment>
                          //#end#
                      `
                  )
  
                  modify(shader,item);
                  //addSpread(shader);
                  //addLightLine(shader)
                  totop(shader)
              }
              //item.material = cityMaterial
          }
      })
      scene.add(gltf.scene)
  
      //添加飞线
      const flyLine = new FlyLine();
      scene.add(flyLine.mesh)
      scene.add(flyLine.mesh2)
      scene.add(flyLine.mesh3)
  })
  
  class FlyLine{
      constructor(){
          let linePoints = [
              new Three.Vector3(0,0,0),
              new Three.Vector3(5,4,0),
              new Three.Vector3(10,0,0),
          ]
  
          let linePoints2 = [
              new Three.Vector3(0,0,0),
              new Three.Vector3(3,3,3),
              new Three.Vector3(5,2,10),
          ]
  
          let linePoints3 = [
              new Three.Vector3(0,0,0),
              new Three.Vector3(-5,4,0),
              new Three.Vector3(-8,0,0),
          ]
          //创建曲线
          this.lineCure = new Three.CatmullRomCurve3(linePoints);
          this.lineCure2 = new Three.CatmullRomCurve3(linePoints2);
          this.lineCure3 = new Three.CatmullRomCurve3(linePoints3);
          //管道集合体
          this.geometry = new Three.TubeBufferGeometry(this.lineCure,100,0.3,2,false)
          this.geometry2 = new Three.TubeBufferGeometry(this.lineCure2,100,0.3,2,false)
          this.geometry3 = new Three.TubeBufferGeometry(this.lineCure3,100,0.3,2,false)
          
          //创建纹理
          const textloader = new Three.TextureLoader();
          this.texture = textloader.load('./textures/z_11.png');
          this.texture.repeat.set(1,2)
          this.texture.wrapS = Three.RepeatWrapping;
          this.texture.wrapT = Three.MirroredRepeatWrapping
  
          //材质
          this.material = new Three.MeshBasicMaterial({
              map: this.texture,
              transparent:true,
          })
  
          //创建飞线物体
          this.mesh = new Three.Mesh(
              this.geometry,
              this.material
          )
  
          this.mesh2 = new Three.Mesh(
              this.geometry2,
              this.material
          )
  
  
          this.mesh3 = new Three.Mesh(
              this.geometry3,
              this.material
          )
  
  
          //动画
  
          gsap.to(this.texture.offset,{
              x:-1,
              duration:1,
              repeat:-1,
              ease:"none",
          })
      }
  }
  
  function addSpread(shader){
  
      //扩散中心
      shader.uniforms.uSpreadCenter = {
          value:new Three.Vector2(0,0)
      }
      //扩散时间
      shader.uniforms.uSpreadTime = {value:0};
      //kuandu 
      shader.uniforms.uSpreadWidth = {value:40}
  
      shader.fragmentShader = shader.fragmentShader.replace(
          '#include <common>',
  
          `
          #include <common>
  
          uniform vec2 uSpreadCenter;
          uniform float uSpreadTime;
          uniform float uSpreadWidth;
          `
      );
  
      shader.fragmentShader = shader.fragmentShader.replace(
          '//#end#',
  
          `
          float spreadRadius = distance(vPosition.xz,uSpreadCenter);
          //扩散范围
          float spreadIndex = -(spreadRadius-uSpreadTime)*(spreadRadius-uSpreadTime)+ uSpreadWidth;
  
          if( spreadIndex > 0.0){
              gl_FragColor = mix(gl_FragColor,vec4(1,1,1,1),spreadIndex/uSpreadWidth);
          }
          //#end#
          `
      );
      gsap.to(shader.uniforms.uSpreadTime,{
          value: 500,
          duration:2,
          ease:"none",
          repeat:-1
      })
  }
  //光带
  function addLightLine(shader){
  
  //扩散中心
  shader.uniforms.uSpreadCenter = {
      value:new Three.Vector2(0,0)
  }
  //扩散时间
  shader.uniforms.uLightLineTime = {value:-800};
  //kuandu 
  shader.uniforms.uLightLineWidth = {value:40}
  
  shader.fragmentShader = shader.fragmentShader.replace(
      '#include <common>',
  
      `
      #include <common>
  
      uniform float uLightLineTime;
      uniform float uLightLineWidth;
      `
  );
  
  shader.fragmentShader = shader.fragmentShader.replace(
      '//#end#',
  
      `
      //扩散范围
      float LightLineMix = -(vPosition.x+vPosition.z-uLightLineTime)*(vPosition.x+vPosition.z-uLightLineTime)+ uLightLineWidth;
  
      if( LightLineMix > 0.0){
          gl_FragColor = mix(gl_FragColor,vec4(1,0.8,0.8,1),LightLineMix/uLightLineWidth);
      }
      //#end#
      `
  );
  gsap.to(shader.uniforms.uLightLineTime,{
      value: 500,
      duration:2,
      ease:"none",
      repeat:-1
  })
  }
  
  function totop(shader){
  
  //扩散中心
  shader.uniforms.uTotopCenter = {
      value:new Three.Vector2(0,0)
  }
  //扩散时间
  shader.uniforms.uTotopTime = {value:0};
  //kuandu 
  shader.uniforms.uTotopWidth = {value:40}
  
  shader.fragmentShader = shader.fragmentShader.replace(
      '#include <common>',
  
      `
      #include <common>
  
      uniform float uTotopTime;
      uniform float uTotopWidth;
      `
  );
  
  shader.fragmentShader = shader.fragmentShader.replace(
      '//#end#',
  
      `
      //扩散范围
      float TotopMix = -(vPosition.y-uTotopTime)*(vPosition.y-uTotopTime)+ uTotopWidth;
  
      if( TotopMix > 0.0){
          gl_FragColor = mix(gl_FragColor,vec4(0.8,0.8,1,1),TotopMix/uTotopWidth);
      }
      //#end#
      `
  );
  gsap.to(shader.uniforms.uTotopTime,{
      value: 500,
      duration:5,
      ease:"none",
      repeat:-1
  })
  }
  
  function modify(shader,item){
      item.geometry.computeBoundingBox()
              let {max,min} = item.geometry.boundingBox
              let uHeight = max.y - min.y
  
      shader.uniforms.uTopColor = {
                      value:new Three.Color("#aaaeff")
                  }
                  shader.uniforms.uHeight = {
                      value:uHeight,
                  }
  
                  shader.vertexShader = shader.vertexShader.replace(
                      "#include <common>",
                      `
                          #include <common>
                          varying vec3 vPosition;
                      `
                  );
  
                  shader.vertexShader = shader.vertexShader.replace(
                      "#include <begin_vertex>",
                      `
                          #include <begin_vertex>
                          vPosition = position ;
                      `
                  );
  
                  shader.fragmentShader = shader.fragmentShader.replace(
                      '#include <common>',
                      `
                          #include <common>
                          uniform vec3 uTopColor;
                          uniform float uHeight;
                          varying vec3 vPosition;
  
                      `
                  )
                  shader.fragmentShader = shader.fragmentShader.replace(
                      '//#end',
                      `
                          vec4 distGradColor = gl_FragColor;
  
                          //设置混合百分比
                          float gradMix = (vPosition.y + uHeight/2.0)/uHeight;
                          //计算混合颜色
                          vec3 gradMixColor = mix(distGradColor.xyz,uTopColor,gradMix);
                          gl_FragColor = vec4(gradMixColor,1);
                          //#end#
                      `
                  )
  }
  
  
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