<script>
	import { T, useFrame, useRender, useThrelte } from '@threlte/core';
	import { ContactShadows, Float, Grid, OrbitControls } from '@threlte/extras';
	import Spaceship from './models/spaceship.svelte';
	import {
		Color,
		MeshBasicMaterial,
		PMREMGenerator,
		PlaneGeometry,
		Raycaster,
		SphereGeometry,
		Vector2,
		Vector3
	} from 'three';
	import { onMount } from 'svelte';
	import Stars from './Stars.svelte';
	import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
	import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
	import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
	import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

	let planeRef;
	let sphereRef;
	let spaceShipRef;

	const { scene, camera, renderer } = useThrelte();
	let intersectionPoint;
	let translY = 0;
	let translAccelleration = 0;
	let angleZ = 0;
	let angleAccelleration = 0;

	let pmrem = new PMREMGenerator(renderer);
	let envMapRT;

	const composer = new EffectComposer(renderer);
	composer.setSize(innerWidth, innerHeight);

	const setupEffectComposer = () => {
		const renderPass = new RenderPass(scene, camera.current);
		composer.addPass(renderPass);

		const bloomPass = new UnrealBloomPass(new Vector2(innerWidth, innerHeight), 0.275, 1, 0);
		composer.addPass(bloomPass);

		const outputPass = new OutputPass();
		composer.addPass(outputPass);
	};

	// takes control of the render loop, unlike useFrame
	// https://threlte.xyz/docs/reference/core/use-render
	useRender(({ scene }) => {
		if (intersectionPoint) {
			const targetY = intersectionPoint?.y || 0;
			translAccelleration += (targetY - translY) * 0.002; // stiffness
			translAccelleration *= 0.95; // damping
			translY += translAccelleration;

			const dir = intersectionPoint.clone().sub(new Vector3(0, translY, 0)).normalize();
			const dirCos = dir.dot(new Vector3(0, 1, 0));
			const angle = Math.acos(dirCos) - Math.PI * 0.5;
			angleAccelleration += (angle - angleZ) * 0.01; // stiffness
			angleAccelleration *= 0.85; // damping
			angleZ += angleAccelleration;
		}

		if (envMapRT) envMapRT.dispose();

		spaceShipRef.visible = false;
		scene.background = null;
		envMapRT = pmrem.fromScene(scene, 0, 0.1, 1000);
		scene.background = new Color('#598889').multiplyScalar(0.05);
		spaceShipRef.visible = true;

		spaceShipRef.traverse((child) => {
			if (child?.material?.envMapIntensity) {
				child.material.envMap = envMapRT.texture;
				child.material.envMapIntensity = 100;
				child.material.normalScale.set(0.3, 0.3);
			}
		});

		composer.render();
	});

	onMount(() => {
		setupEffectComposer();

		const raycaster = new Raycaster();
		const pointer = new Vector2();

		function onPointerMove(event) {
			pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
			pointer.y = -(event.clientY / window.innerHeight) * 2 + 1;

			raycaster.setFromCamera(pointer, $camera);
			const intersects = raycaster.intersectObject(planeRef);
			intersectionPoint = intersects[0]?.point;

			if (intersectionPoint) {
				// this prevents the spring motion to be different while the pointer
				// spans the x axis
				intersectionPoint.x = -3;
				sphereRef.position.copy(intersectionPoint);
			}
		}

		window.addEventListener('pointermove', onPointerMove);
		return () => {
			window.removeEventListener('pointermove', onPointerMove);
		};
	});
</script>

<T.PerspectiveCamera makeDefault position={[-5, 6, 10]} fov={25}>
	<OrbitControls enableDamping target={[0, 0, 0]} />
</T.PerspectiveCamera>

<T.DirectionalLight intensity={1.8} position={[0, 10, 0]} castShadow shadow.bias={-0.0001} />
<T.AmbientLight intensity={0.2} />

<Stars />

<Spaceship
	bind:ref={spaceShipRef}
	position={[0, translY, 0]}
	rotation={[angleZ, 0, angleZ, 'ZXY']}
/>

<T.Mesh renderOrder={2} bind:ref={planeRef} visible={false}>
	<T.PlaneGeometry args={[20, 20]} />
	<T.MeshBasicMaterial color={[1, 0, 1]} transparent opacity={0.25} />
</T.Mesh>

<T.Mesh position={[1, 2, 0]} bind:ref={sphereRef} visible={false}>
	<T.SphereGeometry args={[0.1, 20, 20]} />
	<T.MeshBasicMaterial color={[1, 0, 0]} />
</T.Mesh>