<script lang="ts">
	import { onMount, onDestroy } from "svelte";
	import {
		WebGLRenderer,
		Scene,
		PerspectiveCamera,
		Group,
		LinearToneMapping,
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
		ShaderMaterial,
		BufferGeometry,
		Raycaster,
		Vector2,
		MathUtils,
		type Intersection,
		Object3D,
		type Object3DEventMap,
	} from "three";
	import {
		CSS3DObject,
		CSS3DRenderer,
	} from "three/examples/jsm/renderers/CSS3DRenderer.js";
	import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
	import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
	import { PointerLockControls } from "three/examples/jsm/controls/PointerLockControls.js";
	import Vimeo from "@vimeo/player";

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
		let tvScreen: Mesh<BufferGeometry, ShaderMaterial> | undefined;
		let tvChanelControl: Mesh<BufferGeometry, MeshBasicMaterial>;
		let tvVolumeControl: Mesh<BufferGeometry, MeshBasicMaterial>;

		let fireshipProChanel = {
			degree: -4.24,
			src: "https://player.vimeo.com/video/599890291",
			name: "",
		};

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

						if (child.name === "screen") tvScreen = child;
						if (child.name === "volume_control") {
							tvVolumeControl = child;
							tvVolumeControl.rotation.z = -5.13;
						}
						if (child.name === "chanel_control") {
							tvChanelControl = child;
							fireshipProChanel.name = child.name;
							tvChanelControl.rotation.z = -4.24;
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
			if (!tvModel || !tvScreen) return;

			let tickStart = Date.now();
			let tickCurrent = tickStart;

			let shouldStopRendering = false;
			let pointerCoord = new Vector2(0.5);
			let prevPointerCoord = new Vector2(0.5);
			let tvModelTargetRotation = new Vector2();
			let selectedObject: Mesh | undefined;
			let intersects: Intersection<Object3D<Object3DEventMap>>[] = [];

			const tvScreenBoundingBox = new Box3().setFromObject(tvScreen);
			const tvScreenHeight =
				tvScreenBoundingBox.max.y - tvScreenBoundingBox.min.y;
			const tvScreenWidth =
				tvScreenBoundingBox.max.x - tvScreenBoundingBox.min.x;
			const iframeElementWidth = 400;

			const iframeElement: HTMLIFrameElement = document.createElement("iframe");

			iframeElement.src = fireshipProChanel.src;
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
			const iframePlayer = new Vimeo(iframeElement);

			const correctIframeSizes = (factor = 1) => {
				iframeElement.style.minHeight = `${
					iframeElementWidth * (tvScreenHeight / tvScreenWidth) * factor
				}px`;

				iframeElement.style.width = `${iframeElementWidth * factor}px`;
			};
			correctIframeSizes(1.15);
			const onIframeLoad = () => {
				correctIframeSizes();
				iframeElement.removeEventListener("load", onIframeLoad);
			};
			iframeElement.addEventListener("load", onIframeLoad);

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

			const cameraOrbit = new OrbitControls(camera, rendererCss.domElement);
			cameraOrbit.enableRotate = false;
			cameraOrbit.enableZoom = false;
			cameraOrbit.enablePan = false;
			cameraOrbit.target.set(0, 1.5, 0);
			const cameraLock = new PointerLockControls(camera, document.body);
			const raycaster = new Raycaster();

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

			tvScreen.position.add(new Vector3(0, 0, 0.001));
			tvScreen.material = new ShaderMaterial({
				transparent: true,
				vertexShader:
					"varying vec2 vUv;void main(){gl_Position=projectionMatrix*modelViewMatrix*vec4(position,1.0);vUv=uv;}",
				fragmentShader: `
        varying vec2 vUv;uniform float uTime;uniform float uDelta;uniform float uAlpha;
        float rand(vec2 uv){return fract(sin(dot(uv,vec2(12.9898,78.233)))*43758.5453);}
        void main(){vec2 uv=vUv;float rand_val=rand(uv+sin(uDelta*(uTime*0.0001)));gl_FragColor=vec4(mix(vec3(.0),vec3(rand_val),uAlpha),uAlpha);}
        `,
				uniforms: {
					uTime: { value: 0 },
					uDelta: { value: 0 },
					uAlpha: { value: 0 },
				},
			});

			tvModel.add(planeMask);
			cssScene.add(cssPlaneObject);
			scene.add(tvModel);

			const onpointerDown = () => {
				if (!tvModel) return;

				if (!intersects?.length || !(intersects[0]?.object instanceof Mesh))
					return;

				if (
					["volume_control", "chanel_control"].includes(
						intersects[0].object.name
					)
				) {
					selectedObject = intersects[0].object;
					if (selectedObject.name === "chanel_control") iframePlayer.pause();
					cameraLock.lock();
				}
			};

			const mapToBoundary = (
				value: number,
				minValue: number,
				maxValue: number
			) => {
				const newValue = Math.max(minValue, Math.min(maxValue, value));
				let normalizedValue =
					((newValue - minValue) / (maxValue - minValue)) * 2 - 1;
				if (Math.abs(normalizedValue) >= 0.99) {
					return 1;
				} else {
					return Math.abs(normalizedValue);
				}
			};

			const calculateTvRotation = (x: number, y: number) => {
				const rotationX = (y - prevPointerCoord.y) * 0.3;
				const rotationY = (x - prevPointerCoord.x) * 0.3;

				tvModelTargetRotation.x -= rotationX;
				tvModelTargetRotation.y += rotationY;
			};

			const interpolatePlaneRotation = () => {
				if (!tvModel) return;
				const ease = 0.1;

				tvModel.rotation.x = MathUtils.lerp(
					tvModel.rotation.x,
					tvModelTargetRotation.x,
					ease
				);
				tvModel.rotation.y = MathUtils.lerp(
					tvModel.rotation.y,
					tvModelTargetRotation.y,
					ease
				);
			};

			const onPointerMove = (e: PointerEvent) => {
				pointerCoord.x = (e.clientX / viewportWidth) * 2 - 1;
				pointerCoord.y = -(e.clientY / viewportHeight) * 2 + 1;

				raycaster.setFromCamera(pointerCoord, camera);
				intersects = raycaster.intersectObject(scene);

				if (canvasElement.parentElement)
					canvasElement.parentElement.style.cursor = "auto";
				if (
					intersects?.length &&
					canvasElement.parentElement &&
					["volume_control", "chanel_control"].includes(
						intersects[0].object.name
					)
				)
					canvasElement.parentElement.style.cursor = "pointer";

				if (!intersects.length)
					calculateTvRotation(pointerCoord.x, pointerCoord.y);
				else tvModelTargetRotation.set(0, 0);

				prevPointerCoord.set(pointerCoord.x, pointerCoord.y);

				if (!selectedObject || !tvChanelControl) return;

				selectedObject.rotation.z -= -e.movementY * 0.01;
				selectedObject.rotation.z = MathUtils.clamp(
					selectedObject.rotation.z,
					-Math.PI * 2,
					0
				);

				if (selectedObject.name === "volume_control") {
					const val = MathUtils.clamp(
						1 - (selectedObject.rotation.z + Math.PI * 2) / (Math.PI * 2),
						0,
						1
					);
					iframePlayer.setVolume(val);
				}
				if (typeof tvScreen?.material.uniforms?.uAlpha?.value === "number")
					tvScreen.material.uniforms.uAlpha.value = mapToBoundary(
						tvChanelControl.rotation.z,
						fireshipProChanel.degree - 0.5,
						fireshipProChanel.degree + 0.5
					);
			};

			const onPointerUp = (e: PointerEvent) => {
				selectedObject = undefined;
				cameraLock.unlock();
			};

			const render = () => {
				const tickCurrentTime = Date.now();
				const tickDelta = tickCurrentTime - tickCurrent;
				tickCurrent = tickCurrentTime;
				const tickElapsed = tickCurrent - tickStart;

				if (typeof tvScreen?.material.uniforms?.uTime?.value === "number")
					tvScreen.material.uniforms.uTime.value = tickElapsed;
				if (typeof tvScreen?.material.uniforms?.uDelta?.value === "number")
					tvScreen.material.uniforms.uDelta.value = tickDelta;

				cameraOrbit.update();

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

				interpolatePlaneRotation();

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

				const animationFrameId = requestAnimationFrame(render);
				if (shouldStopRendering) cancelAnimationFrame(animationFrameId);
			};

			render();

			window.addEventListener("pointerdown", onpointerDown);
			window.addEventListener("pointermove", onPointerMove);
			window.addEventListener("pointerup", onPointerUp);

			dispose = () => {
				window.removeEventListener("pointerdown", onpointerDown);
				window.removeEventListener("pointermove", onPointerMove);
				window.removeEventListener("pointerup", onPointerUp);
				cameraOrbit.dispose();
				cameraLock.dispose();
				shouldStopRendering = true;
			};
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
