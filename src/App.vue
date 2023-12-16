<script setup>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
let scene, camera, renderer, analyser, uniforms;
import nihong from "./assets/田埂上的梦.mp3";

function startPlay() {
  init();
}
function init() {
  const fftSize = 2048;
  const overlay = document.getElementById("overlay");
  overlay.remove();
  const container = document.querySelector(".container");
  const width = window.innerWidth;
  const height = window.innerHeight;
  renderer = new THREE.WebGLRenderer({
    antialias: true,
  });
  renderer.setSize(width, height);
  renderer.setClearColor(0x000000);
  renderer.setPixelRatio(window.devicePixelRatio);
  container.appendChild(renderer.domElement);
  scene = new THREE.Scene();
  camera = new THREE.Camera();
  // 创建音频监听器
  const listener = new THREE.AudioListener();

  camera.add(listener);
  const soundSource = "./assets/20-20000.flac";
  const sound = new THREE.PositionalAudio(listener);
  const audioLoader = new THREE.AudioLoader();
  audioLoader.load(nihong, function (buffer) {
    sound.setBuffer(buffer);
    sound.setRefDistance(20);
    sound.play();
  });
  // 创建Audio对象
  const audio = new THREE.Audio(listener);
  // 创建音源文件

  // 创建HTML的HTMLAudioElement
  const mediaElement = new Audio(soundSource);

  //注意这里的Audio是浏览器的对象，不是Threejs的mediaElement.play()
  audio.setMediaElementSource(mediaElement);
  // 注意这里的audio才是上面Three创建的THREE.Audio()对象
  // 创建AudioAnalyser对象，使用AnalyserNode 去分析音频数据
  analyser = new THREE.AudioAnalyser(sound, fftSize);

  const data = analyser.getAverageFrequency();
  console.log(data, "const data = analyser.getAverageFrequency()");
  const format = renderer.capabilities.isWebGL2
    ? THREE.RedFormat
    : THREE.LuminanceFormat;
  console.log(analyser, format, "analyseranalyseranalyser");
  /* 这行代码主要是用于确定渲染器使用的颜色格式（color format）是THREE.RedFormat还是THREE.LuminanceFormat。
  根据渲染器的WebGL上下文（context）是否是WebGL2版本来决定使用哪种颜色格式。其目的是根据WebGL上下文的版本选择更加合适的颜色格式，在渲染时可以提高性能和效果。
  - 如果WebGL上下文是WebGL2，则使用THREE.RedFormat，表示使用R（红）通道作为颜色信息，没有G（绿）和B（蓝）通道。
  - 如果WebGL上下文不是WebGL2，则使用THREE.LuminanceFormat，表示使用亮度（Luminance）值作为颜色信息（通常是灰度），也就是只有一个通道。
   */
  uniforms = {
    tAudioData: {
      value: new THREE.DataTexture(analyser.data, fftSize / 2, 1, format),
    },
    /*** analyser.data表示音频数据源，它是一个一维数组，包含着频域数据； fftSize / 2表示频域采样点个数；1表示纹理在y轴方向上的采样点个数，因为它只包含一行数据；
     * format表示纹理数据类型，通常是THREE.LuminanceFormat或THREE.RGBAFormat。*/
    iResolution: {
      value: new THREE.Vector2(
        renderer.domElement.width,
        renderer.domElement.height,
      ),
    },
    iTime: { value: 0 },
    soundTextureYOffset: { value: 0 },
    soundTextureHeight: { value: 5 },
    soundTextureWidth: { value: 5 },
    uvScale: { value: new THREE.Vector2(1, 1) },
  };
  const material = new THREE.ShaderMaterial();
  material.uniforms = uniforms;
  material.vertexShader = `
  uniform vec2 uvScale;varying vec2 vUv;
   void main() { 
    vUv =  uv;
    gl_Position = vec4( position, 2.0 );}`;
  material.fragmentShader = `//Math constants
#define PI 3.14159
#define TWO_PI 6.28318

//Frequency range to which the halo reacts
//currently set to 0-520hz
#define FREQ 512.0

uniform vec2 iResolution;               // viewport resolution (in pixels)
uniform float iTime;                   // shader playback time (in seconds)
uniform sampler2D tAudioData;          // sound texture
uniform float soundTextureYOffset;      // Y offset of sound texture
uniform float soundTextureHeight;       // height of sound texture
uniform float soundTextureWidth;        // width of sound texture

varying vec2 vUv;

// Function to convert HSV color to RGB
vec3 hsv2rgb(vec3 c) {
    vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
    vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
    return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

// Function to calculate luma (brightness) of a color
float luma(vec3 color) {
    return dot(color, vec3(0.299, 0.587, 0.114));
}

// Function to get sound level from a specific angle
float getSound(float x) {
    vec2 tAudioDataUV = vec2((x + iTime) / soundTextureWidth, soundTextureYOffset / soundTextureHeight);
    return texture2D(tAudioData, tAudioDataUV).r;
}

void main(){
    vec2 uv = vUv;
    vec2 uvn = uv * 2.0 - 1.0;
    uvn.x *= iResolution.x / iResolution.y;
    
    // Calculate angle and normalize it
    float angle = atan(normalize(uvn).x, normalize(uvn).y);
    float angleNormalized = angle / PI;
    
    // Calculate halo
    float inner_halo = max(pow(1.0 - abs(length(uvn) - 0.6), 8.0), 0.0) * 2.0;
    inner_halo *= inner_halo;
    
    float outer_halo = max(pow(abs(length(uvn)), 4.0), 0.0) * 0.25;
    float halo = inner_halo + outer_halo;
    
    // Calculate sound frequency
    float freq = getSound(abs(angleNormalized) * 2.0) * 2.0;
    // Scale sound by 2.0 to match the original effect
    
    // Calculate color
    vec3 col = halo * hsv2rgb(vec3(angleNormalized * 0.5 + 0.5 - iTime * 0.1, 1.0, 1.0)) * freq;
    col = pow(col, vec3(2.0));
    col += max(length(col) / sqrt(3.0) - 1.0, 0.0);
    gl_FragColor = vec4(col, 1.0);
}
`;
  const geometry = new THREE.PlaneGeometry(5, 5);
  const mesh = new THREE.Mesh(geometry, material);
  console.log(mesh, "meshmeshmeshmeshmesh");
  scene.add(mesh);
  window.addEventListener("resize", onWindowResize);
  animate();
}
function onWindowResize() {
  renderer.setSize(window.innerWidth, window.innerHeight);
}
function animate() {
  requestAnimationFrame(animate);
  render();
}
function render() {
  analyser.getFrequencyData();
  uniforms.tAudioData.value.needsUpdate = true;
  uniforms.iTime.value += 0.0001;
  renderer.render(scene, camera);
}
</script>

<template>
  <div id="overlay">
    <button id="startButton" @click="startPlay">Play</button>
  </div>
  <div class="container"></div>
</template>

<style scoped>
#overlay {
  position: absolute;
  left: 20px;
  top: 20px;
  z-index: 888;
  width: 100vw;
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  background: rgba(0, 0, 0, 0.7);
  font-size: 16px;
}
#overlay button {
  background: transparent;
  border: 0;
  border: 1px solid rgb(255, 255, 255);
  border-radius: 4px;
  color: #ffffff;
  padding: 12px 18px;
  text-transform: uppercase;
  cursor: pointer;
}
@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
