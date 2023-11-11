<script lang="ts">
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js';

    const params = {
        segments: 80,
        edgeRadius: .12,
        isAnimate: false,
        xyz: 0,
        stime: 0,
        ctime: 0,
        spinTime: 3000,
        stoppedTime: 0,
        trigger: 0,
        demoModeStopTime: 0,
    };

    let canvasEL:HTMLCanvasElement;
    let startBtn:HTMLButtonElement;
    let rollResult:HTMLParagraphElement;
    let gift:HTMLParagraphElement;
    let appBody:HTMLDivElement;
    
    let renderer, scene, camera, diceMesh, physicsWorld, intervalID1, intervalID2, intervalID3, body;
    let CANNON;

    const changeCanvas = (canvas:HTMLCanvasElement, ctx:any, isWin:boolean) => {
        window.devicePixelRatio = 10;
        const size = 300;
        canvas.style.width = size + "px"; 
        canvas.style.height = size + "px";
        
        const scale = window.devicePixelRatio;
        canvas.width = Math.floor(size * scale); 
        canvas.height = Math.floor(size * scale); 
        
        if(ctx) {
            ctx.font = 'bold 300pt Arial';
            ctx.fillStyle = 'white';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = 'black';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            if(!isWin) {
                ctx.fillText("SORRY", canvas.width / 2, canvas.height / 2);
            } else {
                ctx.fillText("Starbucks", canvas.width / 2, canvas.height * 2 / 5);
                ctx.fillText("Americano", canvas.width / 2, canvas.height * 3 / 5);
            }
            ctx.scale(scale, scale);
        }
    };

    const createBoxGeometry = () => {
        let boxGeometry = new THREE.BoxGeometry(1, 1, 1, params.segments, params.segments, params.segments);

        const positionAttr = boxGeometry.attributes.position;

        const subCubeHalfSize = .5 - params.edgeRadius;


        for (let i = 0; i < positionAttr.count; i++) {

            let position = new THREE.Vector3().fromBufferAttribute(positionAttr, i);

            const subCube = new THREE.Vector3(Math.sign(position.x), Math.sign(position.y), Math.sign(position.z)).multiplyScalar(subCubeHalfSize);
            const addition = new THREE.Vector3().subVectors(position, subCube);

            if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.y) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
                addition.normalize().multiplyScalar(params.edgeRadius);
                position = subCube.add(addition);
            } else if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.y) > subCubeHalfSize) {
                addition.z = 0;
                addition.normalize().multiplyScalar(params.edgeRadius);
                position.x = subCube.x + addition.x;
                position.y = subCube.y + addition.y;
            } else if (Math.abs(position.x) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
                addition.y = 0;
                addition.normalize().multiplyScalar(params.edgeRadius);
                position.x = subCube.x + addition.x;
                position.z = subCube.z + addition.z;
            } else if (Math.abs(position.y) > subCubeHalfSize && Math.abs(position.z) > subCubeHalfSize) {
                addition.x = 0;
                addition.normalize().multiplyScalar(params.edgeRadius);
                position.y = subCube.y + addition.y;
                position.z = subCube.z + addition.z;
            }

            positionAttr.setXYZ(i, position.x, position.y, position.z);
        }


        boxGeometry.deleteAttribute('normal');
        // boxGeometry.deleteAttribute('uv');
        boxGeometry = BufferGeometryUtils.mergeVertices(boxGeometry);

        boxGeometry.computeVertexNormals();

        return boxGeometry;
    };

    const createDiceMesh = () => {
        
        const geometry = createBoxGeometry();

        let canvas1 = document.createElement('canvas'), ctx1 = canvas1.getContext('2d');  //  right
        let canvas2 = document.createElement('canvas'), ctx2 = canvas2.getContext('2d');  //  left
        let texture1 = new THREE.Texture(canvas1);
        let texture2 = new THREE.Texture(canvas2);

        changeCanvas(canvas1, ctx1, true);
        changeCanvas(canvas2, ctx2, false);

        texture1.needsUpdate = true;
        texture2.needsUpdate = true;

        const material = [
            new THREE.MeshStandardMaterial({map: texture2, metalness: 0, roughness: 0, color: 0xeeeeee}),
            new THREE.MeshStandardMaterial({map: texture2, metalness: 0, roughness: 0, color: 0xeeeeee}),
            new THREE.MeshStandardMaterial({map: texture1, metalness: 0, roughness: 0, color: 0xeeeeee}),
            new THREE.MeshStandardMaterial({map: texture2, metalness: 0, roughness: 0, color: 0xeeeeee}),
            new THREE.MeshStandardMaterial({map: texture1, metalness: 0, roughness: 0, color: 0xeeeeee}),
            new THREE.MeshStandardMaterial({map: texture1, metalness: 0, roughness: 0, color: 0xeeeeee}),
        ];
        
        diceMesh = new THREE.Mesh(geometry, material);
        diceMesh.castShadow = true;

        return diceMesh;
    };

    const getRandomInt = (min:number, max:number) => {
        return Math.floor(Math.random() * (max - min)) + min;
    };

    const waitSomeTodoAgain = () => {
        let ctime = performance.now();

        if(ctime - params.stoppedTime >= 2) {
            clearInterval(intervalID2);

            initPhysics();
            initScene();
            initDice();

            params.stime = performance.now();

            rollResult.style.display = 'none';
            rollResult.innerHTML = "";
            
            gift.style.display = 'none';
            gift.innerHTML = "";

            animate();
        }
    }

    const goBackDemoMode = () => {
        let ctime = performance.now();

        if(ctime - params.demoModeStopTime > 3000) {
            clearInterval(intervalID3);
            initPhysics();
            initScene();
            initDice();
            startBtn.style.display = 'none';
            params.isAnimate = true;
            params.stime = performance.now();
            animate();
        }
    }

    const determineXYZ = () => {
        let rand = getRandomInt(0, 59);
        params.xyz = rand % 6;
    };

    const animate = () => {
        if(params.isAnimate == true) {
            let ctime = performance.now();
            let etime = ctime - params.stime;

            physicsWorld.step(1 / 60);
            
            if(etime < params.spinTime) {    
                body.velocity.set(0, 0.3, 0);
        
                if(params.xyz == 0) {
                    body.angularVelocity.set(Math.PI * 3, 0, 0);
                } else if(params.xyz == 1) {
                    body.angularVelocity.set(0, Math.PI * 3, 0);
                } else if(params.xyz == 2) {
                    body.angularVelocity.set(0, 0, Math.PI * 3);
                } else if(params.xyz == 3) {
                    body.angularVelocity.set(-Math.PI * 3, 0, 0);
                } else if(params.xyz == 4) {
                    body.angularVelocity.set(0, -Math.PI * 3, 0);
                } else {
                    body.angularVelocity.set(0, 0, -Math.PI * 3);
                }
            } else {
                clearInterval(intervalID1);
                body.angularVelocity.set(0, 0, 0);
            }
        
            diceMesh.position.copy(body.position)
            diceMesh.quaternion.copy(body.quaternion)
        
            renderer.render(scene, camera);
            requestAnimationFrame(animate);
        }
    };

    // const rollDice = () => {
        
    // };

    const initPhysics = () => {
        //  create physicsWorld
        physicsWorld = new CANNON.World({
            allowSleep: true,
            gravity: new CANNON.Vec3(0, -20, 0),
        })
        physicsWorld.defaultContactMaterial.restitution = .3;
    }

    const initRender = () => {
        //  create renderer
        renderer = new THREE.WebGLRenderer({
            alpha: true,
            antialias: true,
            canvas: canvasEL
        });

        renderer.shadowMap.enabled = true
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 10));
        renderer.setSize(300, 400);
        renderer.setClearColor(0xffffff);
    };

    const initScene = () => {
        //  create new scene
        scene = new THREE.Scene();

        //  add camera
        camera = new THREE.PerspectiveCamera(45, 300 / 400, .1, 300)
        camera.position.set(2, 1.5, 2);
        camera.lookAt(0, -0.5, 0);
        camera.aspect = 300 / 400;
        camera.updateProjectionMatrix();

        //  add lights 
        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(5, 5, 5).normalize();

        const ambientLight = new THREE.AmbientLight(0xffffff, 1);

        const topLight = new THREE.PointLight(0xffffff, 1);
        topLight.position.set(10, 15, 0);
        topLight.castShadow = true;
        topLight.shadow.mapSize.width = 2048;
        topLight.shadow.mapSize.height = 2048;
        topLight.shadow.camera.near = 3;
        topLight.shadow.camera.far = 400;

        const hemisphereLight = new THREE.HemisphereLight(0xffffff, 0x000000);
        hemisphereLight.position.set(1, 1, 1);

        scene.add(directionalLight);
        scene.add(ambientLight);
        scene.add(topLight);
        scene.add(hemisphereLight);
    }

    const initDice = () => {
        //  add floor
        const floor = new THREE.Mesh(
            new THREE.PlaneGeometry(1000, 1000),
            new THREE.ShadowMaterial({
                opacity: .1,
            }),
        )

        floor.receiveShadow = true;
        floor.position.y = -1;
        floor.quaternion.setFromAxisAngle(new THREE.Vector3(-1, 0, 0), Math.PI * .5);
        scene.add(floor);

        const floorBody = new CANNON.Body({
            type: CANNON.Body.STATIC,
            shape: new CANNON.Plane(),
        });
        
        floorBody.position.copy(floor.position);
        floorBody.quaternion.copy(floor.quaternion);
        physicsWorld.addBody(floorBody);

        diceMesh = createDiceMesh();
        scene.add(diceMesh);

        body = new CANNON.Body({
            mass: 1,
            shape: new CANNON.Box(new CANNON.Vec3(.5, .5, .5)),
            sleepTimeLimit: .5
        });

        physicsWorld.addBody(body);
        
        
        body.addEventListener('sleep', (e) => {
            body.allowSleep = false;

            const euler = new CANNON.Vec3();
            e.target.quaternion.toEuler(euler);

            const eps = .1;
            let isZero = (angle:any) => Math.abs(angle) < eps;
            let isHalfPi = (angle:any) => Math.abs(angle - .5 * Math.PI) < eps;
            let isMinusHalfPi = (angle:any) => Math.abs(.5 * Math.PI + angle) < eps;
            let isPiOrMinusPi = (angle:any) => (Math.abs(Math.PI - angle) < eps || Math.abs(Math.PI + angle) < eps);

            let isWin = true;

            if (isZero(euler.z)) {
                if (isZero(euler.x)) {
                    // alert("top");
                    isWin = true;
                } else if (isHalfPi(euler.x)) {
                    // alert("back");
                    isWin = true;
                } else if (isMinusHalfPi(euler.x)) {
                    // alert("front");
                    isWin = true;
                } else if (isPiOrMinusPi(euler.x)) {
                    // alert("bottom");
                    isWin = false;
                } else {
                    body.allowSleep = true;
                }
            } else if (isHalfPi(euler.z)) {
                // alert("right");
                isWin = false;
            } else if (isMinusHalfPi(euler.z)) {
                // alert("left");
                isWin = false;
            } else {
                body.allowSleep = true;
            }

            params.isAnimate = false;
            
            rollResult.style.display = 'block';            
            gift.style.display = 'block';

            if(isWin) {
                rollResult.innerHTML = "WIN";
                gift.innerHTML = "Starbucks Americano";
            } else {
                rollResult.innerHTML = "SORRY";
                rollResult.style.color = "red";
                gift.innerHTML = "Have a nice day!";
            }

            params.stoppedTime = performance.now();
            intervalID2 = setInterval(waitSomeTodoAgain, 1000);

        });

        
        params.isAnimate = true;
    
        intervalID1 = setInterval(determineXYZ, 1000);
        params.xyz = getRandomInt(0, 59) % 6;

        rollResult.style.color = "green";

        renderer.render(scene, camera);
    }

    const init = () => {
        import('https://cdn.skypack.dev/cannon-es')
        
        .then((module) => {

            CANNON = module;
            initPhysics();
            initRender();
            initScene();
            initDice();

            params.isAnimate = true;
            animate();

            startBtn.addEventListener('click', () => {
                startBtn.style.display = 'none';
                params.stime = performance.now();
                clearInterval(intervalID1);
                clearInterval(intervalID3);
                initPhysics();
                initScene();
                initDice();
                params.isAnimate = true;
                animate();
            });

            appBody.addEventListener('mouseover', () => {
                if(params.isAnimate) {
                    params.isAnimate = false;
                    clearInterval(intervalID1);
                    startBtn.style.display = 'block';

                    params.demoModeStopTime = performance.now();
                    intervalID3 = setInterval(goBackDemoMode, 1000);
                }
            });

        })



        .catch((error) => {
            console.error('Error loading external file:', error);
        });
    };

	onMount(() => {
		init();
	});

</script>

<div bind:this={appBody} class="h-[580px] bg-white-500 text-center" id="app_body">
    <p bind:this={rollResult} class="text-4xl hidden text-green-600" id="roll_result"></p>
    <canvas bind:this={canvasEL} id="dice"></canvas>
    <button bind:this={startBtn} class="absolute play-button hidden" id="start_btn">Start</button>
    <p bind:this={gift} class="text-4xl hidden text-green-600" id="gift"></p>
</div>

<style>
    .play-button {
		@apply w-[150px] bottom-28 justify-center bg-[#e94747] text-white px-[10px] py-[13px] text-[18px] rounded-2xl border-none cursor-pointer transition-all;
	}
	.play-button:hover {
		background-color: #b63838;
	}
    #dice {
        position: absolute;
        top: -5%;
        left: 50%;
		transform: translate(-50%, 50%);
    }
    #start_btn {
        z-index: 1;
		transform: translate(-50%, 50%);
    }
    #roll_result {
        position: absolute;
        top: 20%;
        left: 50%;
		transform: translate(-50%, -50%);
    }
    #gift {
        position: absolute;
        top: 80%;
        left: 50%;
		transform: translate(-50%, -50%);
    }

</style>