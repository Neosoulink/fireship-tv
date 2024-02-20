<script lang="ts">
	import { onMount, onDestroy } from "svelte";
	import {
		WebGLRenderer,
		Scene,
		PerspectiveCamera,
		Group,
		LinearToneMapping,
		GridHelper,
		TextureLoader,
		MeshBasicMaterial,
		Mesh,
		LinearSRGBColorSpace,
		PlaneGeometry,
		NoBlending,
		DoubleSide,
		Box3,
		Color,
		Vector3,
		Quaternion,
	} from "three";
	import {
		CSS3DObject,
		CSS3DRenderer,
	} from "three/examples/jsm/renderers/CSS3DRenderer.js";
	import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
	import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

	let canvasElement: HTMLCanvasElement;
	let dispose = () => {};

	onMount(() => {
		let viewportWidth = window.innerWidth;
		let viewportHeight = window.innerHeight;
		let viewportAspect = viewportWidth / viewportHeight;

		let scene = new Scene();

		let gltfLoader = new GLTFLoader();
		let textureLoader = new TextureLoader();
		let tvModel: Group | undefined;
		let tvScreen: Mesh | undefined;

		/**
		 * Initialize Model & Text.
		 * @param cb Callback triggered after all resources are loaded
		 */
		const loadResources = (cb: () => void) => {
			gltfLoader?.load("/tv.glb", (model) => {
				tvModel = model.scene;

				textureLoader?.load("/tv.png", (texture) => {
					texture.flipY = false;
					texture.colorSpace = LinearSRGBColorSpace;
					const mat = new MeshBasicMaterial({
						map: texture,
						blending: NoBlending,
					});

					tvModel?.children.forEach((child) => {
						if (!(child instanceof Mesh)) return;
						child.material = mat;

						if (child.name === "screen") {
							tvScreen = child;
							tvScreen.visible = false;
						}
					});

					cb();
				});
			});
		};

		/**
		 * Initialize the 3D experience.
		 */
		const init = () => {
			if (!tvModel || !tvScreen)
				return console.warn("🚧 Unable to resolve model");

			const tvScreenBoundingBox = new Box3().setFromObject(tvScreen);
			const tvScreenHeight =
				tvScreenBoundingBox.max.y - tvScreenBoundingBox.min.y;
			const tvScreenWidth =
				tvScreenBoundingBox.max.x - tvScreenBoundingBox.min.x;
			const iframeElementWidth = 400;

			const iframeElement: HTMLIFrameElement = document.createElement("iframe");
			iframeElement.src = "https://player.vimeo.com/video/599890291";
			iframeElement.allowFullscreen = true;
			iframeElement.allow = "autoplay";
			iframeElement.setAttribute("webkitallowfullscreen", "true");
			iframeElement.setAttribute("mozallowfullscreen", "true");
			iframeElement.setAttribute("frameborder", "0");
			iframeElement.style.aspectRatio = "16/9";
			iframeElement.style.border = "none";
			iframeElement.style.top = "0";
			iframeElement.style.left = "0";
			iframeElement.style.zIndex = "9999";
			const correctIframeSizes = (factor = 1) => {
				iframeElement.style.minHeight = `${
					iframeElementWidth * (tvScreenHeight / tvScreenWidth) * factor
				}px`;

				iframeElement.style.width = `${iframeElementWidth * factor}px`;
			};
			correctIframeSizes(1.15);
			iframeElement.onload = () => correctIframeSizes();

			const camera = new PerspectiveCamera(35, viewportAspect, 1, 100);
			camera.position.set(0, 1.5, 10);
			camera.lookAt(0, 1.5, 0);
			const renderer = new WebGLRenderer({
				canvas: canvasElement,
				antialias: true,
				alpha: true,
			});
			renderer.outputColorSpace = LinearSRGBColorSpace;
			renderer.toneMapping = LinearToneMapping;
			renderer.toneMappingExposure = 0.975;
			renderer.setClearAlpha(0);
			renderer.setSize(viewportWidth, viewportHeight);
			renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

			const gridHelper = new GridHelper(10, 10);

			const cssFactor = 1000;
			const cssScene = new Scene();
			const cssCamera = new PerspectiveCamera(
				camera.fov,
				camera.aspect,
				camera.near * cssFactor,
				camera.far * cssFactor
			);
			const rendererCss = new CSS3DRenderer();
			rendererCss.setSize(viewportWidth, viewportHeight);
			rendererCss.domElement.style.width = "100%";
			rendererCss.domElement.style.height = "100%";
			rendererCss.domElement.style.top = "0";
			rendererCss.domElement.style.left = "0";
			rendererCss.domElement.style.position = "fixed";
			canvasElement.parentElement?.insertBefore(
				rendererCss.domElement,
				canvasElement
			);
			const cameraControls = new OrbitControls(camera, rendererCss.domElement);
			cameraControls.enableDamping = true;
			cameraControls.target.set(0, 1.5, 0);

			const cssPlaneObject = new CSS3DObject(iframeElement);
			cssPlaneObject.scale
				.set(1, 1, 1)
				.multiplyScalar(cssFactor / (iframeElementWidth / tvScreenHeight));

			const planeMask = new Mesh(
				new PlaneGeometry(tvScreenWidth, tvScreenHeight),
				new MeshBasicMaterial({
					opacity: 0,
					color: new Color("black"),
					blending: NoBlending,
					side: DoubleSide,
				})
			);
			planeMask.position.copy(tvScreen.position).add(new Vector3(0, 0, -0.02));

			// ===

			tvModel.add(planeMask);
			cssScene.add(cssPlaneObject);
			scene.add(gridHelper, tvModel);

			const render = () => {
				cameraControls.update();

				planeMask.updateMatrixWorld();
				const planeMaskWorldMatrix = planeMask.matrixWorld;
				const planeMaskPosition = new Vector3();
				const planeMaskScale = new Vector3();
				const planeMaskQuaternion = new Quaternion();
				planeMaskWorldMatrix.decompose(
					planeMaskPosition,
					planeMaskQuaternion,
					planeMaskScale
				);

				cssPlaneObject.quaternion.copy(planeMaskQuaternion);
				cssPlaneObject.position
					.copy(planeMaskPosition)
					.multiplyScalar(cssFactor);
				const scaleFactor =
					iframeElementWidth /
					(planeMask.geometry.parameters.width * planeMaskScale.x);
				cssPlaneObject.scale
					.set(1, 1, 1)
					.multiplyScalar(cssFactor / scaleFactor);

				cssCamera.quaternion.copy(camera.quaternion);
				cssCamera.position.copy(camera.position).multiplyScalar(cssFactor);

				rendererCss.render(cssScene, cssCamera);

				renderer.render(scene, camera);

				requestAnimationFrame(render);
			};

			render();

			dispose = () => {};
		};

		loadResources(init);
	});

	onDestroy(() => dispose?.());
</script>

<main>
	<canvas bind:this={canvasElement} />
</main>

<style>
	:global(body) {
		overflow: hidden;
		margin: 0;
		padding: 0;
		position: relative;
	}
	main {
		width: 100vw;
		height: 100vh;
		overflow: hidden;
	}

	canvas {
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		position: fixed;
		opacity: 1;
		z-index: 0;
		pointer-events: none;
	}
</style>
