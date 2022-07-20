<template lang="pug">
#container(ref="container")
</template>
<script lang="ts">
import {
  Color,
  LinearFilter,
  Mesh,
  OrthographicCamera,
  PlaneBufferGeometry,
  RepeatWrapping,
  Scene,
  ShaderMaterial,
  TextureLoader,
  Vector2,
  WebGLRenderer,
  WebGLRenderTarget,
} from "three";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { SavePass } from "three/examples/jsm/postprocessing/SavePass";
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass";
import { BlendShader } from "three/examples/jsm/shaders/BlendShader";
import { CopyShader } from "three/examples/jsm/shaders/CopyShader";
import { HorizontalBlurShader } from "three/examples/jsm/shaders/HorizontalBlurShader";
import { VerticalBlurShader } from "three/examples/jsm/shaders/VerticalBlurShader";
import { FXAAShader } from "three/examples/jsm/shaders/FXAAShader";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
// レンダラの初期化
const renderer = new WebGLRenderer();
renderer.shadowMap.enabled = true;
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setClearColor(0x000000);
let mouseX = 0.5;
let mouseY = 0.5;
renderer.domElement.addEventListener("mousemove", (e) => {
  mouseX = e.x / renderer.domElement.clientWidth;
  mouseY = e.y / renderer.domElement.clientHeight;
});

// UVスクローラを読み込み
const loader = new GLTFLoader();
const dracoLoader = new DRACOLoader();

dracoLoader.setDecoderPath("/draco/");
loader.setDRACOLoader(dracoLoader);
const scroller = await loader.loadAsync("/scroll-view.glb", (e) => {
  console.log((e.loaded / e.total) * 100);
});

const scrollerScene = new Scene();
scrollerScene.add(scroller.scene);
const uvScrollTexture = new TextureLoader().load("scroll.png");
uvScrollTexture.wrapS = RepeatWrapping;
uvScrollTexture.wrapT = RepeatWrapping;
const scrollSpeed = {
  points: 0.25,
  doors: 0.5,
  dispFrame: 0.6,
  platformLine: 0.25,
  floor: 0.35,
  welcome: 0.1,
  pillar: 0.2,
};
const materials: ShaderMaterial[] = [];
const startTime = Date.now();
scroller.scene.traverse((object) => {
  if (object.type === "Mesh") {
    const mesh = object as Mesh;
    const material = new ShaderMaterial({
      vertexShader: `
        varying vec2 v_UV;
        void main() {
	        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
          v_UV = uv;
        }`,
      fragmentShader: `
        varying vec2 v_UV;
        uniform sampler2D u_Texture;
        uniform float u_Time;
        void main() {
	        gl_FragColor = texture2D(u_Texture,vec2(v_UV.x+u_Time*${
            scrollSpeed[mesh.name]
          },1.-v_UV.y));
        }
        `,
      uniforms: {
        u_Texture: { value: null },
        u_Time: { value: 0 },
      },
    });
    materials.push(material);
    material.uniforms.u_Texture.value = uvScrollTexture;
    mesh.material = material;
  }
});
const scrollerCamera = scroller.cameras[0] as OrthographicCamera;
scrollerCamera.top = (scrollerCamera.top / 2) * (1080 / 1920);
scrollerCamera.bottom = (scrollerCamera.bottom / 2) * (1080 / 1920);
scrollerCamera.left = scrollerCamera.left / 2;
scrollerCamera.right = scrollerCamera.right / 2;
scrollerCamera.updateProjectionMatrix();
const renderTargetParameters = {
  minFilter: LinearFilter,
  magFilter: LinearFilter,
  stencilBuffer: false,
};
const scrollerRenderTarget = new WebGLRenderTarget(1920, 1080, {
  ...renderTargetParameters,
  depthBuffer: false,
});

// シーンの初期化
const scene = new Scene();
const geometory = new PlaneBufferGeometry(3.5555555, 2);
const material = new ShaderMaterial({
  vertexShader: `
  varying vec2 v_UV;
  void main() {
    v_UV = uv;
	  gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
  }`,
  fragmentShader: `
  varying vec2 v_UV;
  uniform sampler2D u_Scroller;
  uniform sampler2D u_AllOff;
  uniform sampler2D u_OnOff;
  uniform sampler2D u_DispOff;
  uniform vec3 u_DispColor;
  uniform vec2 u_Mouse;
  void main() {
    vec2 move = vec2(
      (u_Mouse.x-.5)*.5
      ,
      -(u_Mouse.y-.5)*.5
      );
    const float fac = .1;
    move*=fac;
    move = clamp(move,-fac,fac);
    vec2 uv = ((v_UV-.5)*(1.-fac)+.5)+move;
	  gl_FragColor = texture2D(u_AllOff,uv)+texture2D(u_OnOff,uv)*texture2D(u_Scroller,uv)+vec4(u_DispColor,1.0)*texture2D(u_DispOff,uv);
  }`,
  uniforms: {
    u_Scroller: { value: null },
    u_AllOff: { value: null },
    u_OnOff: { value: null },
    u_DispOff: { value: null },
    u_DispColor: { value: new Color(0x000000) },
    u_Mouse: { value: new Vector2(0.5, 0.5) },
  },
});
material.uniforms.u_Scroller.value = scrollerRenderTarget.texture;
const allOffTexture = new TextureLoader().load("alloff.png");
material.uniforms.u_AllOff.value = allOffTexture;
const onOffTexture = new TextureLoader().load("on-off.png");
material.uniforms.u_OnOff.value = onOffTexture;
const dispOffTexture = new TextureLoader().load("disp-off.png");
material.uniforms.u_DispOff.value = dispOffTexture;
const planeMesh = new Mesh(geometory, material);

scene.add(planeMesh);
const camera = new OrthographicCamera();
camera.position.z = 1;
camera.lookAt(0, 0, 0);
camera.top = 1;
camera.bottom = -1;
camera.left = -window.innerWidth / window.innerHeight;
camera.right = window.innerWidth / window.innerHeight;
camera.near = 0.1;
camera.far = 2;
camera.updateProjectionMatrix();

// コンポーザーの初期化
const composer = new EffectComposer(renderer);

// レンダーパス：描画
const renderPass = new RenderPass(scene, camera);

// セーブパス：描画されたものを保存する => フレームバッファオブジェクト(3)
const savePass = new SavePass(
  new WebGLRenderTarget(
    window.innerWidth,
    window.innerHeight,
    renderTargetParameters
  )
);

// ブレンドパス(シェーダパス) 色をブレンドする(2)
const blendPass = new ShaderPass(
  BlendShader, // シェーダ
  "tDiffuse1" // TextureID？ => 前のパス(RenderPass)から受け取ったテクスチャを入れるスロット名,デフォルトは"tDiffuse"だが、このシェーダにはないので、上書き
);
blendPass.uniforms.tDiffuse2.value = savePass.renderTarget.texture;
blendPass.uniforms.mixRatio.value = 0.75;

let blurSize = 5;

// 水平ブラー (4)
const horizontalBlurPass = new ShaderPass(HorizontalBlurShader);
horizontalBlurPass.uniforms.h.value = (1 / window.innerWidth) * blurSize;

// 垂直ブラー (5)
const verticalBlurPass = new ShaderPass(VerticalBlurShader);
verticalBlurPass.uniforms.v.value = (1 / window.innerHeight) * blurSize;

const fxaaPass = new ShaderPass(FXAAShader);
fxaaPass.uniforms.resolution.value.x = 1 / window.innerWidth;
fxaaPass.uniforms.resolution.value.y = 1 / window.innerHeight;

// アウトプットパス(6)
const outputPass = new ShaderPass(CopyShader);
outputPass.renderToScreen = true;

// パスを順番に追加
composer.addPass(renderPass);

composer.addPass(blendPass);
composer.addPass(savePass);
composer.addPass(horizontalBlurPass);
composer.addPass(verticalBlurPass);
composer.addPass(fxaaPass);
composer.addPass(outputPass);

// 画面の大きさ変更時にどうするか
const onWindowResize = () => {
  renderer.setSize(window.innerWidth, window.innerHeight);

  horizontalBlurPass.uniforms.h.value = (1 / window.innerWidth) * blurSize;
  verticalBlurPass.uniforms.v.value = (1 / window.innerHeight) * blurSize;
  fxaaPass.uniforms.resolution.value.x = 1 / window.innerWidth;
  fxaaPass.uniforms.resolution.value.y = 1 / window.innerHeight;
  camera.left = -window.innerWidth / window.innerHeight;
  camera.right = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
};
window.addEventListener("resize", onWindowResize, false);

let mouseCurrentX = mouseX;
let mouseCurrentY = mouseY;
</script>
<script setup lang="ts">
const container = ref<HTMLDivElement>(null);
const running = ref(true);
onMounted(async () => {
  container.value.appendChild(renderer.domElement);

  function render() {
    renderer.setRenderTarget(scrollerRenderTarget);
    renderer.render(scrollerScene, scrollerCamera);
    renderer.setRenderTarget(null);
    composer.render();
    if (running.value) requestAnimationFrame(render);

    horizontalBlurPass.uniforms.h.value = (1 / window.innerWidth) * blurSize;
    verticalBlurPass.uniforms.v.value = (1 / window.innerHeight) * blurSize;
    blurSize = Math.max(0, blurSize - 0.1);

    const time = (Date.now() - startTime) / 1000;
    for (const material of materials) {
      material.uniforms.u_Time.value = time;
    }
    material.uniforms.u_DispColor.value = new Color(
      Math.floor(Math.random() * 256) * 0x010000 +
        Math.floor(Math.random() * 256) * 0x000100 +
        Math.floor(Math.random() * 256) * 0x000001
    );
    material.uniforms.u_Mouse.value.x = mouseCurrentX =
      mouseX * 0.2 + mouseCurrentX * 0.8;
    material.uniforms.u_Mouse.value.y = mouseCurrentY =
      mouseY * 0.2 + mouseCurrentY * 0.8;
  }
  render();
});
onUnmounted(() => {
  running.value = false;
});
</script>
