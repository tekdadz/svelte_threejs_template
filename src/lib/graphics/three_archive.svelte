<script>
	import { onMount } from 'svelte';
	import { xPlane, yPlane, zPlane, animationSpeed } from '$lib/store/store';
	import { page } from '$app/stores';
	import { afterNavigate } from '$app/navigation';

	import * as THREE from 'three';

	// import vertexShader from './shaders/perlin/perlinVertex.glsl';
	// import fragmentShader from './shaders/perlin/perlinFragment.glsl';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { ImprovedNoise } from 'three/addons/math/ImprovedNoise.js';

	// import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
	import WebGL from 'three/addons/capabilities/WebGL.js';

	if ( WebGL.isWebGL2Available() === false ) {
		document.body.appendChild( WebGL.getWebGL2ErrorMessage() );
	}

	const INITIAL_THRESHOLD = 0.5;
	const INITIAL_OPACITY = 0.8;
	const INITIAL_STEPS = 200;

	let container;
  let controls;

	let camera, scene, renderer;
	let mesh;

	// let width = window.innerWidth > 1440 ? window.innerWidth * 0.8125 : window.innerWidth;
	let width = window.innerWidth;
	let height = window.innerHeight;

	const clock = new THREE.Clock();

  // ---------------------------------------------------------------------------
	// Texture -------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	const size = 128;
	const scale = 12;
	const data = new Uint8Array( size * size * size );

	let i = 0;
	const perlin = new ImprovedNoise();
	const vector = new THREE.Vector3();

	for ( let z = 0; z < size; z ++ ) {

		for ( let y = 0; y < size; y ++ ) {

			for ( let x = 0; x < size; x ++ ) {

				vector.set( x, y, z ).divideScalar( size );

				const d = perlin.noise( vector.x * scale, vector.y * scale, vector.z * scale );

				data[ i ++ ] = d * size + size;

			}

		}

	}

	console.log(data)

	const texture = new THREE.Data3DTexture( data, size, size, size );
	texture.format = THREE.RedFormat;
	texture.minFilter = THREE.LinearFilter;
	texture.magFilter = THREE.LinearFilter;
	texture.unpackAlignment = 1;
	texture.needsUpdate = true;

	// ---------------------------------------------------------------------------
	// Planes --------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	// Add these to the top, near other variables
	let planes = [];

  // ---------------------------------------------------------------------------
	// Material ------------------------------------------------------------------
  // ---------------------------------------------------------------------------

	const vertexShader = /* glsl */`
		in vec3 position;

		uniform mat4 modelMatrix;
		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;
		uniform vec3 cameraPos;

		out vec3 vOrigin;
		out vec3 vDirection;

		void main() {
			vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );

			vOrigin = vec3( inverse( modelMatrix ) * vec4( cameraPos, 1.0 ) ).xyz;
			vDirection = position - vOrigin;

			gl_Position = projectionMatrix * mvPosition;
		}
	`;

	const fragmentShader = /* glsl */`
		precision highp float;
		precision highp sampler3D;

		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;

		in vec3 vOrigin;
		in vec3 vDirection;

		out vec4 color;

		uniform sampler3D map;

		uniform float threshold;
		uniform float steps;
		uniform float baseOpacity;
		uniform float time;

		vec2 hitBox( vec3 orig, vec3 dir ) {
			const vec3 box_min = vec3( - 0.5 );
			const vec3 box_max = vec3( 0.5 );
			vec3 inv_dir = 1.0 / dir;
			vec3 tmin_tmp = ( box_min - orig ) * inv_dir;
			vec3 tmax_tmp = ( box_max - orig ) * inv_dir;
			vec3 tmin = min( tmin_tmp, tmax_tmp );
			vec3 tmax = max( tmin_tmp, tmax_tmp );
			float t0 = max( tmin.x, max( tmin.y, tmin.z ) );
			float t1 = min( tmax.x, min( tmax.y, tmax.z ) );
			return vec2( t0, t1 );
		}

		float noise(vec3 position) {
				// A simple gradient noise function
				position *= 1.0;
				vec3 ip = floor(position);
				vec3 fp = fract(position);
				float n = mix(mix(dot(ip, vec3(1.0, 57.0, 113.0)), dot(ip + vec3(1.0, 0.0, 0.0), vec3(1.0, 57.0, 113.0)), fp.x),
											mix(dot(ip + vec3(0.0, 1.0, 0.0), vec3(1.0, 57.0, 113.0)), dot(ip + vec3(1.0, 1.0, 0.0), vec3(1.0, 57.0, 113.0)), fp.x), fp.y);
				return fract(atan(n) * 437585.453);
		}

		float sample1( vec3 p ) {
				return texture(map, p).r;
		}

		void main() {
    color = vec4(0.0);
    vec3 rayDir = normalize(vDirection);
    vec2 bounds = hitBox(vOrigin, rayDir);

    if (bounds.x > bounds.y) discard;
    bounds.x = max(bounds.x, 0.0);

    vec3 p = vOrigin + bounds.x * rayDir;
    vec3 inc = 1.0 / abs(rayDir);
    float delta = min(inc.x, min(inc.y, inc.z));

    // float animatedSteps = steps - sin(time) * 5.0;
    delta /= steps;

    for (float t = bounds.x; t < bounds.y; t += delta) {
        vec3 pos = p + 0.5;
        float n = noise(pos * 10.0 - time * 1.0);
        float animatedThreshold = threshold + n * 0.1;

        float d = sample1(pos);

        if (d > animatedThreshold) {
            vec3 colorShift = vec3(0.75) + 0.5 * cos(pos + vec3(4, 1, -4) + n); // vec3 colorShift = vec3(0.75) + 0.5 * cos(time + pos + vec3(4, 1, -4) + n);
						
            color.rgb = colorShift;
            color.a = baseOpacity;
            break;
        }

        p += rayDir * delta;
    }

    if (color.a == 0.0) discard;
}
`;

	// Plane shaders
	const planeVertexShader = /* glsl */`
		in vec3 position;
		in vec2 uv;

		uniform mat4 modelViewMatrix;
		uniform mat4 projectionMatrix;

		out vec2 vUv;

		void main() {
			vUv = uv;
			gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
		}
	`;

	const planeFragmentShader = /* glsl */`
    precision highp float;
    precision highp sampler3D;

    in vec2 vUv;
    out vec4 color;

    uniform sampler3D map;
    uniform float slicePosition;
    uniform float threshold;
    uniform int sliceAxis; // 0 = X, 1 = Y, 2 = Z
    uniform float time;
    uniform float baseOpacity;

    float noise(vec3 position) {
        // Same noise function as used in the main shader
        vec3 ip = floor(position);
        vec3 fp = fract(position);
        float n = mix(mix(dot(ip, vec3(1.0, 57.0, 113.0)), dot(ip + vec3(1.0, 0.0, 0.0), vec3(1.0, 57.0, 113.0)), fp.x),
                      mix(dot(ip + vec3(0.0, 1.0, 0.0), vec3(1.0, 57.0, 113.0)), dot(ip + vec3(1.0, 1.0, 0.0), vec3(1.0, 57.0, 113.0)), fp.x), fp.z);
        // return fract(sin(n) * 4375.85453);
				return sin(n) * cos(n);
    }

    void main() {
        vec3 samplePos;

        if (sliceAxis == 0) {
            samplePos = vec3(slicePosition, vUv.y, 1.0 - vUv.x);
        } else if (sliceAxis == 1) {
            samplePos = vec3(vUv.x, slicePosition, vUv.y);
        } else {
            samplePos = vec3(vUv.x, vUv.y, slicePosition);
        }

        float n = noise(samplePos + sin(time * 1.0));
        float animatedThreshold = threshold + n * 0.05;

        // Sample the volume texture at the animated position
        float sampledValue = texture(map, samplePos).r;
        bool isAboveThreshold = sampledValue > animatedThreshold;

				bool isAboveThreshold2 = sampledValue > animatedThreshold / 1.05;
				bool isAboveThreshold3 = sampledValue > animatedThreshold / 1.1;
				bool isAboveThreshold4 = sampledValue > animatedThreshold / 1.15;

        // Use similar color calculation as in the main shader
        if (isAboveThreshold) {
					color = vec4(0.168, 0.168, 0.168, 1.0);
        } else if (isAboveThreshold2) {
					vec3 colorShift = vec3(0.5) + 0.2 * cos( samplePos + vec3(4, 1, -4) + n); // vec3 colorShift = vec3(0.5) + 0.2 * cos(time + samplePos + vec3(4, 1, -4) + n);
            color.rgb = colorShift;
            color.a = baseOpacity;
				} else if (isAboveThreshold3) {
					discard;
					} else if (isAboveThreshold4) {
						vec3 colorShift = vec3(0.5) + 0.2 * cos(samplePos + vec3(4, 1, -4) + n); // vec3 colorShift = vec3(0.5) + 0.2 * cos(time + samplePos + vec3(4, 1, -4) + n);
            color.rgb = colorShift;
            color.a = baseOpacity;
					} else {
						discard;
        }
    }
`;






	init();
	animate();

	function init() {
		camera = new THREE.PerspectiveCamera(20, width / height, 0.01, 100);
		camera.position.x = 3; 	
		camera.position.y = 2; 	
		camera.position.z = 3; 	

		scene = new THREE.Scene();

		setScene();

		renderer = new THREE.WebGLRenderer({ antialias: false });
		renderer.setPixelRatio(window.devicePixelRatio);
		renderer.setSize(width, height);

		// renderer.setClearColor(0x232323, 0);
		renderer.autoClearColor = false;
		renderer.autoClearDepth = true;
		renderer.autoClearStencil = true;

		controls = new OrbitControls(camera, renderer.domElement);
		controls.target.set(0, 0, 0);  // Explicitly set target to origin
		controls.update();
		// don't allow pan
		controls.enablePan = false;

		

		onMount(() => {
			container.appendChild(renderer.domElement);

			const unsubscribeX = xPlane.subscribe(({ min, max, value, visible }) => {
      const regularizedValue = regularizeValue(value, min, max);
      updatePlanePosition(planes[0], 'x', regularizedValue);
			planes[0].visible = visible;
    });

			const unsubscribeY = yPlane.subscribe(({ min, max, value, visible }) => {
				const regularizedValue = regularizeValue(value, min, max);
				updatePlanePosition(planes[1], 'y', regularizedValue);
				planes[1].visible = visible;
			});

			const unsubscribeZ = zPlane.subscribe(({ min, max, value, visible }) => {
				const regularizedValue = regularizeValue(value, min, max);
				updatePlanePosition(planes[2], 'z', regularizedValue);
				planes[2].visible = visible;
			});

			return () => {
				unsubscribeX();
				unsubscribeY();
				unsubscribeZ();
			};
		});
	}

	function setHome () {
		const geometry = new THREE.BoxGeometry( 1.001, 1.001, 1.001 );
			const material = new THREE.RawShaderMaterial( {
				glslVersion: THREE.GLSL3,
				uniforms: {
					time: { value: 0.0 },
					map: { value: texture },
					cameraPos: { value: new THREE.Vector3() },
					threshold: { value: INITIAL_THRESHOLD },
					steps: { value: INITIAL_STEPS },
					baseOpacity: { value: INITIAL_OPACITY },
				},
				vertexShader,
				fragmentShader,
				side: THREE.BackSide,
			} );

			mesh = new THREE.Mesh( geometry, material );
			scene.add( mesh );

			// Add box helper
			// const box = new THREE.BoxHelper( mesh, 0x88ff88);
			// scene.add( box );

			for (let i = 0; i < 3; i++) {
				const planeGeom = new THREE.PlaneGeometry(1, 1);
				const planeMat = new THREE.RawShaderMaterial({
						glslVersion: THREE.GLSL3,
						uniforms: {
								map: { value: texture },
								time: { value: 0.0 },
								cameraPos: { value: new THREE.Vector3() },
								threshold: { value: INITIAL_THRESHOLD },
								steps: { value: INITIAL_STEPS },
								baseOpacity: { value: 1.0 },
								slicePosition: { value: 0.5 },
								sliceAxis: { value: null },
						},
						vertexShader: planeVertexShader,
						fragmentShader: planeFragmentShader,
						side: THREE.DoubleSide,
				});
				const plane = new THREE.Mesh(planeGeom, planeMat);
				planes.push(plane);
				scene.add(plane);
			}

				// Align planes
				planes[0].rotateY(Math.PI / 2);
				planes[1].rotateX(Math.PI / 2);

				planes[0].material.uniforms.sliceAxis.value = 0;
				planes[1].material.uniforms.sliceAxis.value = 1;
				planes[2].material.uniforms.sliceAxis.value = 2;

				planes[0].visible = xPlane.visible;
				planes[1].visible = yPlane.visible;
				planes[2].visible = zPlane.visible;

				window.addEventListener( 'resize', onWindowResize );
	}

	function regularizeValue(actualValue, minActual, maxActual) {
    const normalized = (actualValue - minActual) / (maxActual - minActual);
    return normalized - 0.5;
  }

	function updatePlanePosition(plane, axis, position) {
    plane.position[axis] = position;
    plane.material.uniforms.slicePosition.value = position + 0.5;
	}

	function setScene () {
		if ($page.url.pathname == '/') {
			setHome();
		} else {
			setHome();
		}
	}

	afterNavigate (onNavigate);
	function onNavigate() {

		for( var i = scene.children.length - 1; i >= 0; i--) { 
				let obj = scene.children[i];
				scene.remove(obj); 
		}

		setScene();

	}

	function onWindowResize() {
		// let width = window.innerWidth > 1440 ? window.innerWidth * 0.8125 : window.innerWidth;
		let width = window.innerWidth;
		let height = window.innerHeight;

		camera.aspect = width / height;
		camera.updateProjectionMatrix();

		renderer.setSize(width, height);
	}

	function animate() {
		requestAnimationFrame(animate);
		mesh.material.uniforms.cameraPos.value.copy( camera.position );
			mesh.material.uniforms.time.value = clock.getElapsedTime() * 0.0125 * $animationSpeed;

		planes.forEach(plane => {
			plane.material.uniforms.cameraPos.value.copy( camera.position );
			plane.material.uniforms.time.value = clock.getElapsedTime() * 0.0125 * $animationSpeed;
		});
		renderer.render( scene, camera );
	}
</script>

<div bind:this={container} class:geometry={true} />

<style>
	.geometry {
		position: absolute;
		overflow: hidden;
		z-index: 1;
	}
</style>
