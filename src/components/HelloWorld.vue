<script setup>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
let scene, camera, renderer, analyser, uniforms;

function startPlay() {
  init()
}
function init() {const fftSize = 2048;const overlay =
document.getElementById('overlay');overlay.remove();const container =
document.querySelector('.container');const width = window.innerWidthconst height
= window.innerHeightrenderer = new THREE.WebGLRenderer({ antialias: true
});renderer.setSize(width,
height);renderer.setClearColor(0x000000);renderer.setPixelRatio(window.devicePixelRatio);container.appendChild(renderer.domElement);scene
= new THREE.Scene();camera = new THREE.Camera();// 创建音频监听器const listener
= new THREE.AudioListener();// 创建Audio对象const audio = new
THREE.Audio(listener);// 创建音源文件const soundSource = '/mf.mp3'//
创建HTML的HTMLAudioElementconst mediaElement = new Audio(soundSource)
//注意这里的Audio是浏览器的对象，不是Threejs的mediaElement.play()audio.setMediaElementSource(mediaElement)
//注意这里的audio才是上面Three创建的THREE.Audio()对象//
创建AudioAnalyser对象，使用AnalyserNode 去分析音频数据analyser = new
THREE.AudioAnalyser(audio, fftSize)const format =
(renderer.capabilities.isWebGL2) ? THREE.RedFormat : THREE.LuminanceFormat;/*
这行代码主要是用于确定渲染器使用的颜色格式（color
format）是THREE.RedFormat还是THREE.LuminanceFormat。根据渲染器的WebGL上下文（context）是否是WebGL2版本来决定使用哪种颜色格式。其目的是根据WebGL上下文的版本选择更加合适的颜色格式，在渲染时可以提高性能和效果。-
如果WebGL上下文是WebGL2，则使用THREE.RedFormat，表示使用R（红）通道作为颜色信息，没有G（绿）和B（蓝）通道。-
如果WebGL上下文不是WebGL2，则使用THREE.LuminanceFormat，表示使用亮度（Luminance）值作为颜色信息（通常是灰度），也就是只有一个通道。
*/uniforms = {tAudioData: { value: new THREE.DataTexture(analyser.data, fftSize
/ 2, 1, format) },/***
analyser.data表示音频数据源，它是一个一维数组，包含着频域数据； fftSize /
2表示频域采样点个数；1表示纹理在y轴方向上的采样点个数，因为它只包含一行数据；format表示纹理数据类型，通常是THREE.LuminanceFormat或THREE.RGBAFormat。*/iResolution:
{ value: new THREE.Vector2(renderer.domElement.width,
renderer.domElement.height) },iTime: { value: 0 },soundTextureYOffset: { value:
0 },soundTextureHeight: { value: 5 },soundTextureWidth: { value: 5 },uvScale: {
value: new THREE.Vector2( 1, 1 ) },};
const material = newTHREE.ShaderMaterial();

material.uniforms = uniforms;
material.vertexShader =
`uniform vec2 uvScale;varying vec2 vUv; void main() { vUv = uv;gl_Position =
vec4( position, 2.0 );}`

const geometry = newTHREE.PlaneGeometry(5, 5)

const mesh = new THREE.Mesh(geometry,
material);scene.add(mesh);window.addEventListener('resize',
onWindowResize);animate(); }
function onWindowResize() {
  renderer.setSize(window.innerWidth, window.innerHeight);
} function animate() {
  requestAnimationFrame(animate); render();
}
function render() {
  analyser.getFrequencyData(); uniforms.tAudioData.value.needsUpdate = true; uniforms.iTime.value += 0.0001; renderer.render(scene, camera);
}
defineProps({
  msg: {
    type: String,
    required: true
  }
})
</script>

<template>
  <template>
    <div id="overlay">
      <button id="startButton" @click="startPlay">Play</button>
    </div>
    <div class="container"></div>
  </template>
</template>

<style scoped>
h1 {
  font-weight: 500;
  font-size: 2.6rem;
  position: relative;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
