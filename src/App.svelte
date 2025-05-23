<script lang="ts">
  import { onMount } from "svelte";
  import * as THREE from "three";

  import { GUI } from "three/addons/libs/lil-gui.module.min.js";
  import { OrbitControls } from "three/addons/controls/OrbitControls.js";
  import { NRRDLoader } from "three/addons/loaders/NRRDLoader.js";
  import { VolumeRenderShader1 } from "three/addons/shaders/VolumeShader.js";

  let threejsRootRef = $state<HTMLElement>();

  onMount(() => {
    let renderer, scene, camera, controls, material, volconfig, cmtextures;

    init();

    function init() {
      scene = new THREE.Scene();

      // Create renderer
      renderer = new THREE.WebGLRenderer();
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      document.body.appendChild(renderer.domElement);

      // Create camera (The volume renderer does not work very well with perspective yet)
      const h = 512; // frustum height
      const aspect = window.innerWidth / window.innerHeight;
      camera = new THREE.OrthographicCamera(
        (-h * aspect) / 2,
        (h * aspect) / 2,
        h / 2,
        -h / 2,
        1,
        2000
      );
      camera.position.set(-64, -64, 128);
      camera.up.set(0, 0, 1); // In our data, z is up

      // Create controls
      controls = new OrbitControls(camera, renderer.domElement);
      controls.addEventListener("change", render);
      controls.target.set(64, 128, 128);
      controls.minZoom = 0.5;
      controls.maxZoom = 4;
      controls.enablePan = false;
      controls.update();

      // scene.add( new AxesHelper( 128 ) );

      // Lighting is baked into the shader a.t.m.
      // let dirLight = new DirectionalLight( 0xffffff );

      // The gui for interaction
      volconfig = {
        clim1: 1680 / 2,
        clim2: 2000,
        renderstyle: "iso",
        isothreshold: 1680,
        colormap: "viridis",
      };
      const gui = new GUI();
      gui.add(volconfig, "clim1", 0, 3000, 0.01).onChange(updateUniforms);
      gui.add(volconfig, "clim2", 0, 3000, 0.01).onChange(updateUniforms);
      gui
        .add(volconfig, "colormap", { gray: "gray", viridis: "viridis" })
        .onChange(updateUniforms);
      gui
        .add(volconfig, "renderstyle", { mip: "mip", iso: "iso" })
        .onChange(updateUniforms);
      gui
        .add(volconfig, "isothreshold", 0, 1680, 0.01)
        .onChange(updateUniforms);

      // Load the data ...
      new NRRDLoader().load("./resample-025.nrrd", function (volume) {
        // new NRRDLoader().load("./stent.nrrd", function (volume) {
        console.log("loaded");

        // Texture to hold the volume. We have scalars, so we put our data in the red channel.
        // THREEJS will select R32F (33326) based on the THREE.RedFormat and THREE.FloatType.
        // Also see https://www.khronos.org/registry/webgl/specs/latest/2.0/#TEXTURE_TYPES_FORMATS_FROM_DOM_ELEMENTS_TABLE
        // TODO: look the dtype up in the volume metadata

        const uint16Array = volume.data;

        // Create a Float32Array of the same length
        const float32Array = new Float32Array(uint16Array.length);

        // Convert each element from Uint16Array to Float32Array
        for (let i = 0; i < uint16Array.length; i++) {
          float32Array[i] = uint16Array[i];
        }

        const texture = new THREE.Data3DTexture(
          float32Array,
          volume.xLength,
          volume.yLength,
          volume.zLength
        );
        texture.format = THREE.RedFormat;
        texture.type = THREE.FloatType;

        // texture.minFilter = texture.magFilter = THREE.LinearFilter;
        texture.unpackAlignment = 1;
        texture.needsUpdate = true;

        // Colormap textures
        cmtextures = {
          viridis: new THREE.TextureLoader().load("./cm_viridis.png", render),
          gray: new THREE.TextureLoader().load("./cm_gray.png", render),
        };

        // Material
        const shader = VolumeRenderShader1;

        const uniforms = THREE.UniformsUtils.clone(shader.uniforms);

        uniforms["u_data"].value = texture;
        uniforms["u_size"].value.set(
          volume.xLength,
          volume.yLength,
          volume.zLength
        );
        uniforms["u_clim"].value.set(volconfig.clim1, volconfig.clim2);
        uniforms["u_renderstyle"].value =
          volconfig.renderstyle == "mip" ? 0 : 1; // 0: MIP, 1: ISO
        uniforms["u_renderthreshold"].value = volconfig.isothreshold; // For ISO renderstyle
        uniforms["u_cmdata"].value = cmtextures[volconfig.colormap];

        material = new THREE.ShaderMaterial({
          uniforms: uniforms,
          vertexShader: shader.vertexShader,
          fragmentShader: shader.fragmentShader,
          side: THREE.BackSide, // The volume shader uses the backface as its "reference point"
        });

        texture.needsUpdate = true;

        // THREE.Mesh
        const geometry = new THREE.BoxGeometry(
          volume.xLength,
          volume.yLength,
          volume.zLength
        );
        geometry.translate(
          volume.xLength / 2 - 0.5,
          volume.yLength / 2 - 0.5,
          volume.zLength / 2 - 0.5
        );

        const mesh = new THREE.Mesh(geometry, material);
        scene.add(mesh);

        render();
      });

      window.addEventListener("resize", onWindowResize);
    }

    function updateUniforms() {
      material.uniforms["u_clim"].value.set(volconfig.clim1, volconfig.clim2);
      material.uniforms["u_renderstyle"].value =
        volconfig.renderstyle == "mip" ? 0 : 1; // 0: MIP, 1: ISO
      material.uniforms["u_renderthreshold"].value = volconfig.isothreshold; // For ISO renderstyle
      material.uniforms["u_cmdata"].value = cmtextures[volconfig.colormap];

      render();
    }

    function onWindowResize() {
      renderer.setSize(window.innerWidth, window.innerHeight);

      const aspect = window.innerWidth / window.innerHeight;

      const frustumHeight = camera.top - camera.bottom;

      camera.left = (-frustumHeight * aspect) / 2;
      camera.right = (frustumHeight * aspect) / 2;

      camera.updateProjectionMatrix();

      render();
    }

    function render() {
      renderer.render(scene, camera);
    }
  });
</script>

<!-- <div bind:this={threejsRootRef}></div>

<style>
  div {
    width: 500px;
    height: 500px;
  }
</style> -->
