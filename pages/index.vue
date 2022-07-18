<template lang="pug">
#container(ref="container")
</template>

<script setup lang="ts">
import {
  AmbientLight,
  Color,
  IcosahedronBufferGeometry,
  LinearFilter,
  Mesh,
  MeshPhongMaterial,
  PerspectiveCamera,
  PointLight,
  Scene,
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
const container = ref<HTMLDivElement>(null);
let onWindowResize: () => void;
onMounted(() => {
  const scene = new Scene();

  // カメラを初期化
  const camera = new PerspectiveCamera(
    45, // fov
    window.innerWidth / window.innerHeight, // aspect
    1, // near
    500 // far
  );

  // 位置を初期化
  camera.position.z = 300;
  camera.position.y = 0;

  // ポイントライト
  const pointLight = new PointLight(
    0xffffff, //color
    1 //intensity
  );
  pointLight.position.set(-50, 100, 50);
  pointLight.castShadow = true;
  scene.add(pointLight);

  // アンビエントライト
  const ambientLight = new AmbientLight(0xffffff, 1);
  scene.add(ambientLight);

  // モデルを追加
  const geometory = new IcosahedronBufferGeometry(70);
  const material = new MeshPhongMaterial({
    color: 0x23afd3,
    specular: 0x050505,
    shininess: 100,
  });
  const model = new Mesh(geometory, material);
  model.position.x = -20;
  scene.add(model);

  // レンダラの初期化
  const renderer = new WebGLRenderer();
  renderer.shadowMap.enabled = true;
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0xffffff);
  container.value.appendChild(renderer.domElement);

  // コンポーザーの初期化
  const composer = new EffectComposer(renderer);

  // レンダーパス：3Dオブジェクトをレンダリングする(1)
  const renderPass = new RenderPass(scene, camera);

  // セーブパス：描画されたものを保存する => フレームバッファオブジェクト(3)
  const renderTargetParameters = {
    minFilter: LinearFilter,
    magFilter: LinearFilter,
    stencilBuffer: false,
  };
  const savePass = new SavePass(
    new WebGLRenderTarget(
      container.value.clientWidth,
      container.value.clientHeight,
      renderTargetParameters
    )
  );

  // ブレンドパス(シェーダパス) 色をブレンドする(2)
  const blendPass = new ShaderPass(
    BlendShader, // シェーダ
    "tDiffuse1" // TextureID？ => 前のパス(RenderPass)から受け取ったテクスチャを入れるスロット名,デフォルトは"tDiffuse"だが、このシェーダにはないので、上書き
  );
  blendPass.uniforms.tDiffuse2.value = savePass.renderTarget.texture;
  blendPass.uniforms.mixRatio.value = 0.8;

  const blurSize = 1;

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
  onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);

    horizontalBlurPass.uniforms.h.value = (1 / window.innerWidth) * blurSize;
    horizontalBlurPass.uniforms.h.value = (1 / window.innerWidth) * blurSize;
    fxaaPass.uniforms.resolution.value.x = 1 / window.innerWidth;
    fxaaPass.uniforms.resolution.value.y = 1 / window.innerHeight;
  };
  window.addEventListener("resize", onWindowResize, false);

  let counter = 0;
  function render() {
    composer.render();
    requestAnimationFrame(render);

    model.rotation.x += 0.02;
    model.rotation.y += -0.02;
    model.rotation.z += 0.03;
    model.position.x += Math.sin(counter);
    counter += 0.05;
  }
  render();
});
onUnmounted(() => {
  window.removeEventListener("resize", onWindowResize, false);
});
</script>
