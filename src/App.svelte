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
	import Vimeo from "@vimeo/player";

	let canvasElement: HTMLCanvasElement;
	let dispose = () => {};

	onMount(() => {
		let gltfLoader = new GLTFLoader();
		let textureLoader = new TextureLoader();

		let tvModel: Group | undefined;
		let tvScreen: Mesh<BufferGeometry, ShaderMaterial> | undefined;
		let tvChanelControl: Mesh<BufferGeometry, MeshBasicMaterial> | undefined;
		let tvVolumeControl: Mesh<BufferGeometry, MeshBasicMaterial> | undefined;
		let tvFireshipProChanel = {
			degree: -4.24,
			src: "https://player.vimeo.com/video/599890291",
		};
		let tvInitialVolumeDegree = -5.13;

		/**
		 * Initialize Model & Texture.
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
						if (child.name === "volume_control") tvVolumeControl = child;
						if (child.name === "chanel_control") tvChanelControl = child;
					});

					cb();
				});
			});
		};

		/** Initialize the experience.*/
		const init = () => {
			const parentElement = canvasElement.parentElement;

			// DATA
			if (!tvModel || !tvScreen || !parentElement) return;

			let viewportWidth = parentElement.clientWidth;
			let viewportHeight = parentElement.clientHeight;
			let viewportAspect = viewportWidth / viewportHeight;

			/* Scene camera */
			const scene = new Scene();

			const camera = new PerspectiveCamera(35, viewportAspect, 1, 100);
			camera.position.set(0, 1.5, 10);
			camera.lookAt(0, 1.5, 0);

			/* Experience renderer */
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

			const raycaster = new Raycaster();

			const pointerCoord = new Vector2(0.5);

			const tvScreenBoundingBox = new Box3().setFromObject(tvScreen);
			const tvScreenHeight =
				tvScreenBoundingBox.max.y - tvScreenBoundingBox.min.y;
			const tvScreenWidth =
				tvScreenBoundingBox.max.x - tvScreenBoundingBox.min.x;
			const tvTargetRotation = new Vector2();

			const iframeElementWidth = 400;
			const iframeElement: HTMLIFrameElement = document.createElement("iframe");
			iframeElement.src = tvFireshipProChanel.src;
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
			parentElement?.insertBefore(rendererCss.domElement, canvasElement);

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

			let tickStart = Date.now();
			let tickCurrent = tickStart;

			let shouldStopRendering = false;

			let intersects: Intersection<Object3D<Object3DEventMap>>[] = [];
			let selectedObject: Mesh | undefined;

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
					uAlpha: { value: 1 },
				},
			});

			tvModel.add(planeMask);
			cssScene.add(cssPlaneObject);
			scene.add(tvModel);

			// METHODS
			const mapToBoundary = (
				value: number,
				minValue: number,
				maxValue: number
			) => {
				let result =
					((Math.max(minValue, Math.min(maxValue, value)) - minValue) /
						(maxValue - minValue)) *
						2 -
					1;
				if (Math.abs(result) >= 0.99) return 1;
				return Math.abs(result);
			};
			const calculateTvRotation = () => {
				tvTargetRotation.x = pointerCoord.y * Math.PI * -0.09;
				tvTargetRotation.y = pointerCoord.x * Math.PI * 0.09;
			};
			const interpolateTvRotation = () => {
				if (!tvModel) return;
				tvModel.rotation.x += (tvTargetRotation.x - tvModel?.rotation.x) * 0.08;
				tvModel.rotation.y += (tvTargetRotation.y - tvModel?.rotation.y) * 0.08;
			};
			const correctIframeSizes = (factor = 1) => {
				iframeElement.style.minHeight = `${
					iframeElementWidth * (tvScreenHeight / tvScreenWidth) * factor
				}px`;

				iframeElement.style.width = `${iframeElementWidth * factor}px`;
			};
			correctIframeSizes(1.15);

			const onIframeLoad = () => {
				correctIframeSizes();
				if (
					tvScreen &&
					tvChanelControl &&
					typeof tvScreen.material?.uniforms?.uAlpha?.value === "number"
				) {
					tvChanelControl.rotation.z = tvFireshipProChanel.degree;
					tvScreen.material.uniforms.uAlpha.value = 0;
				}

				if (tvVolumeControl && iframePlayer) {
					tvVolumeControl.rotation.z = tvInitialVolumeDegree;
					iframePlayer.setVolume(
						MathUtils.clamp(
							1 - (tvVolumeControl.rotation.z + Math.PI * 2) / (Math.PI * 2),
							0,
							1
						)
					);
				}
				iframeElement.removeEventListener("load", onIframeLoad);
			};
			const onpointerDown = () => {
				if (
					!intersects?.length ||
					!(intersects[0]?.object instanceof Mesh) ||
					!["volume_control", "chanel_control"].includes(
						intersects[0].object.name
					)
				)
					return;
				selectedObject = intersects[0].object;
				if (selectedObject.name === "chanel_control") iframePlayer.pause();
			};
			const onPointerMove = (e: PointerEvent) => {
				pointerCoord.set(
					(e.clientX / viewportWidth) * 2 - 1,
					-(e.clientY / viewportHeight) * 2 + 1
				);

				raycaster.setFromCamera(pointerCoord, camera);
				intersects = raycaster.intersectObject(scene);

				if (!selectedObject) {
					parentElement.style.cursor = "auto";
					if (intersects.length) {
						if (
							["volume_control", "chanel_control"].includes(
								intersects[0].object.name
							)
						)
							parentElement.style.cursor = "pointer";
						tvTargetRotation.set(0, 0);
					}

					if (!intersects.length) calculateTvRotation();
				}

				if (selectedObject) {
					selectedObject.rotation.z -= -e.movementY * 0.01;
					selectedObject.rotation.z = MathUtils.clamp(
						selectedObject.rotation.z,
						-Math.PI * 2,
						0
					);

					if (selectedObject.name === "volume_control")
						iframePlayer.setVolume(
							MathUtils.clamp(
								1 - (selectedObject.rotation.z + Math.PI * 2) / (Math.PI * 2),
								0,
								1
							)
						);

					if (
						typeof tvScreen?.material.uniforms?.uAlpha?.value === "number" &&
						selectedObject.name === "chanel_control"
					)
						tvScreen.material.uniforms.uAlpha.value = mapToBoundary(
							selectedObject.rotation.z,
							tvFireshipProChanel.degree - 0.5,
							tvFireshipProChanel.degree + 0.5
						);
				}
			};
			const onPointerUp = (e: PointerEvent) => {
				selectedObject = undefined;
				parentElement.style.cursor = "auto";
			};
			const onPointerLeave = () => {
				tvTargetRotation.set(0, 0);
			};
			const onResize = () => {
				viewportWidth = parentElement.clientWidth;
				viewportHeight = parentElement.clientHeight;
				viewportAspect = viewportWidth / viewportHeight;

				rendererCss.setSize(viewportWidth, viewportHeight);
				cssCamera.updateProjectionMatrix();

				renderer.setViewport(0, 0, viewportWidth, viewportHeight);
				renderer.setSize(viewportWidth, viewportHeight);
				renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
				camera.aspect = viewportWidth / viewportHeight;
				camera.updateProjectionMatrix();
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

				interpolateTvRotation();

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

			// EVENTS
			iframeElement.addEventListener("load", onIframeLoad);

			parentElement?.addEventListener("pointerdown", onpointerDown);
			parentElement?.addEventListener("pointermove", onPointerMove);
			parentElement?.addEventListener("pointerup", onPointerUp);
			parentElement?.addEventListener("pointerleave", onPointerLeave);
			window?.addEventListener("resize", onResize);

			dispose = () => {
				parentElement?.removeEventListener("pointerdown", onpointerDown);
				parentElement?.removeEventListener("pointermove", onPointerMove);
				parentElement?.removeEventListener("pointerup", onPointerUp);
				parentElement?.removeEventListener("pointerleave", onPointerLeave);
				window?.removeEventListener("resize", onResize);

				shouldStopRendering = true;
			};

			render();
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
