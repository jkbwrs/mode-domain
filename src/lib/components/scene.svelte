<script lang="ts">
  import { onMount } from "svelte";
  import { PerspectiveCamera, WebGLRenderer, Scene, Color, Group, BoxGeometry, MeshStandardMaterial, Mesh, PointLight, Vector2 } from "three";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
  import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
  import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
  import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass.js";
  import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
  import { PixelShader } from "./PixelShader";
  import seedrandom from "seedrandom";
  import type { Object3D } from "three";

  export let input: string;
  export let isPremium: boolean = false;
  export let isVip: boolean = true;

  let rng = seedrandom(input);
  let canvas: HTMLCanvasElement;
  let renderer: WebGLRenderer;
  let camera: PerspectiveCamera;
  let scene: Scene;
  let controls: OrbitControls;
  let structure: Object3D;
  let composer: EffectComposer;
  let frameId: number;

  let numBoxes: number = 320 + Math.floor(rng() * 20 - 10); // reduced to 220 from 320
  let baseDistance: number = 0.035 + rng() * 0.005 - 0.0025;
  let radius: number = 0.96 + rng() * 0.04 - 0.02;
  let turns: number = 5 + Math.floor(rng() * 2 - 1);
  let angleStep = (turns * Math.PI * 2) / numBoxes;
  let boxHeight: number = 0.05;
  let boxWidth: number = boxHeight * 15;
  let boxDepth: number = boxHeight * 1;
  let pixelFactor: number = 1;
  let lightCount = 40; // reduced to 40 from 100

  let boxColor = "#dffe00";
  let neonColors = ["#000000", "#494949", "#d2d0cb"];

  let boxesLeft: Object3D;
  let boxesRight: Object3D;

  $: if (input) {
    rng = seedrandom(input);
    numBoxes = 220 + Math.floor(rng() * 20 - 10);
    baseDistance = 0.035 + rng() * 0.005 - 0.0025;
    radius = 0.96 + rng() * 0.04 - 0.08;
    turns = 5 + Math.floor(rng() * 2 - 1.4);
    angleStep = (turns * Math.PI * 2) / numBoxes;
    pixelFactor = 1 + Math.floor(rng() * 8);
    updateHelixAndLights();
  }

  function updateHelixAndLights() {
    if (scene && structure) {
      scene.remove(structure);
      structure = createBoxesAlongHelix();
      scene.add(structure);
      if (boxesLeft) {
        scene.remove(boxesLeft);
      }
      if (boxesRight) {
        scene.remove(boxesRight);
      }
      boxesLeft = createBoxPattern("left");
      boxesRight = createBoxPattern("right");
      scene.add(boxesLeft);
      scene.add(boxesRight);
    }

    scene.children = scene.children.filter(
      (child) => !(child instanceof PointLight)
    );

    const lightIntensity = 0.4 + rng() * 0.1 - 0.05;
    const lightDistance = 5000 + rng() * 1000 - 500;

    for (let colorIndex = 0; colorIndex < neonColors.length; colorIndex++) {
      const color = new Color(neonColors[colorIndex]);
      for (let i = 0; i < lightCount / neonColors.length; i++) {
        const light = new PointLight(
          color,
          lightIntensity,
          lightDistance
        );
        light.position.set(
          (rng() - 0.5) * radius * 2,
          rng() * numBoxes * baseDistance - ((numBoxes - 1) * baseDistance) / 2,
          (rng() - 0.5) * radius * 2
        );
        scene.add(light);
      }
    }
  }

  onMount(() => {
    init();
    animate();

    if (isPremium && !isVip) {
      renderer.setClearColor(new Color("rgb(2, 2, 2)"));
      boxColor = "#494949";
      neonColors = ["#000000", "#494949", "#d2d0cb"];
      updateHelixAndLights(); 
    } else if (!isPremium && isVip) {
      renderer.setClearColor(new Color("#dffe00"));
    }

    return () => {
      cancelAnimationFrame(frameId);
      renderer.dispose();
      controls.dispose();
      scene.clear();
    };
  });

  function createBoxesAlongHelix() {
    const structure = new Group();

    for (let i = 0; i < numBoxes; i++) {
      const theta = angleStep * i;
      const x = radius * Math.sin(theta);
      const y = i * baseDistance;
      const z = radius * Math.cos(theta);

      const boxGeometry = new BoxGeometry(boxWidth, boxHeight, boxDepth);
      const boxMaterial = new MeshStandardMaterial({
        color: boxColor,
        metalness: 0,
        roughness: 1,
      });

      const box = new Mesh(boxGeometry, boxMaterial);
      box.position.set(x, y, z);
      box.castShadow = true;
      box.receiveShadow = true;

      structure.add(box);
    }

    structure.position.y = -((numBoxes - 1) * baseDistance) / 2;
    return structure;
  }

  function addRandomLights() {
    const lightIntensity = isPremium ? 20 : 0.4 + rng() * 0.1 - 0.05;
    const lightDistance = 5000 + rng() * 1000 - 500;

    for (let colorIndex = 0; colorIndex < neonColors.length; colorIndex++) {
      const color = new Color(neonColors[colorIndex]);
      for (let i = 0; i < lightCount / neonColors.length; i++) {
        const light = new PointLight(
          color,
          lightIntensity,
          lightDistance
        );
        light.position.set(
          (rng() - 0.5) * radius * 2,
          rng() * numBoxes * baseDistance - ((numBoxes - 1) * baseDistance) / 2,
          (rng() - 0.5) * radius * 2
        );
        scene.add(light);
      }
    }
  }

  function createBoxPattern(side: "left" | "right"): Object3D {
    const pattern = new Group();
    const count = Math.floor(6 + rng() * 60);
    const size = 0.02 + rng() * 0.02;
    const separation = size * 0.5;
    const directionMultiplier = side === "left" ? -1 : 1;

    const safeDistanceFromHelix = radius + size;

    for (let i = 0; i < count; i++) {
      const boxGeometry = new BoxGeometry(size, size, size);
      const boxMaterial = new MeshStandardMaterial({ color: boxColor });
      const box = new Mesh(boxGeometry, boxMaterial);

      let posX = directionMultiplier * safeDistanceFromHelix;
      let posY =
        rng() * numBoxes * baseDistance - ((numBoxes - 1) * baseDistance) / 2;
      let posZ = 0;
      let theta =
        ((posY + ((numBoxes - 1) * baseDistance) / 2) / baseDistance) *
        angleStep;
      let helixX = radius * Math.sin(theta);

      if (side === "left") {
        while (posX > helixX - size) {
          posY -= separation;
          theta =
            ((posY + ((numBoxes - 1) * baseDistance) / 2) / baseDistance) *
            angleStep;
          helixX = radius * Math.sin(theta);
        }
      } else {
        while (posX < helixX + size) {
          posY -= separation;
          theta =
            ((posY + ((numBoxes - 1) * baseDistance) / 2) / baseDistance) *
            angleStep;
          helixX = radius * Math.sin(theta);
        }
      }

      box.position.set(posX, posY, posZ);
      pattern.add(box);
    }

    return pattern;
  }

  function init() {
    scene = new Scene();
    camera = new PerspectiveCamera(
      75,
      canvas.clientWidth / canvas.clientHeight,
      0.1,
      1000
    );
    renderer = new WebGLRenderer({ canvas, antialias: true, alpha: true });
    renderer.setSize(canvas.clientWidth, canvas.clientHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setClearColor(new Color("#000"));
    
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.maxPolarAngle = Math.PI;

    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = WebGLRenderer.PCFSoftShadowMap;

    addRandomLights();

    structure = createBoxesAlongHelix();
    scene.add(structure);

    boxesLeft = createBoxPattern("left");
    boxesRight = createBoxPattern("right");
    scene.add(boxesLeft);
    scene.add(boxesRight);

    camera.position.z = 3.1;

    composer = new EffectComposer(renderer);
    const renderPass = new RenderPass(scene, camera);
    composer.addPass(renderPass);

    const pixelPass = new ShaderPass(PixelShader);
    pixelPass.uniforms["resolution"].value = new Vector2(
      canvas.clientWidth * 2,
      canvas.clientHeight * 2
    );
    pixelPass.uniforms["pixelSize"].value = pixelFactor;
    composer.addPass(pixelPass);

    const bloomPass = new UnrealBloomPass({
      threshold: 0.02,
      strength: 0.02,
      radius: 0.02,
      exposure: 0.1,
    });

    if (isPremium) {
      composer.addPass(bloomPass)
    } else if (isPremium) {
      composer.addPass(bloomPass)
    }
  }

  function animate() {
    frameId = requestAnimationFrame(animate);
    controls.update();
    composer.render();
  }
</script>

<canvas bind:this={canvas} style="background-color: {isPremium ? '#D1F544' : '#000'}" width="600" height="600" />
