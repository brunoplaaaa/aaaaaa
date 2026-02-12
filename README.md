<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>WebXR Reacción Química - CORREGIDO</title>
    
    <!-- Three.js core (sin extras problemáticos) -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: 'Arial', sans-serif;
            background: black;
            touch-action: none;
        }
        #info-panel {
            position: absolute;
            bottom: 30px;
            left: 30px;
            width: 280px;
            background: rgba(10, 20, 30, 0.9);
            backdrop-filter: blur(5px);
            border-left: 6px solid #00ccff;
            border-radius: 12px;
            padding: 18px 22px;
            color: white;
            font-size: 16px;
            box-shadow: 0 8px 20px rgba(0,0,0,0.5);
            border: 1px solid rgba(255,255,255,0.2);
            transition: opacity 0.3s ease, visibility 0.3s;
            pointer-events: none;
            z-index: 100;
        }
        #info-panel h3 {
            margin: 0 0 10px 0;
            color: #88ddff;
            font-weight: 600;
            font-size: 20px;
            border-bottom: 1px solid #4488aa;
            padding-bottom: 6px;
        }
        .highlight {
            color: #ffaa66;
            font-weight: bold;
        }
        .hidden-panel {
            opacity: 0;
            visibility: hidden;
        }
        .visible-panel {
            opacity: 1;
            visibility: visible;
        }
        #status {
            position: absolute;
            top: 30px;
            right: 30px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 12px 18px;
            border-radius: 30px;
            font-size: 16px;
            border: 1px solid #00ccff;
            backdrop-filter: blur(2px);
            pointer-events: none;
            z-index: 200;
        }
        #instructions {
            position: absolute;
            top: 30px;
            left: 30px;
            background: rgba(0,0,0,0.6);
            color: #ccc;
            padding: 12px 18px;
            border-radius: 8px;
            font-size: 14px;
            pointer-events: none;
            z-index: 200;
        }
        #ar-button {
            position: absolute;
            bottom: 30px;
            right: 30px;
            background: #00ccff;
            color: black;
            padding: 16px 24px;
            border-radius: 40px;
            font-size: 18px;
            font-weight: bold;
            border: none;
            box-shadow: 0 4px 10px rgba(0,0,0,0.3);
            pointer-events: auto;
            z-index: 300;
            cursor: pointer;
        }
        canvas {
            display: block;
        }
    </style>
</head>
<body>
    <div id="info-panel" class="hidden-panel">
        <h3 id="particle-title">Oxígeno (O)</h3>
        <p><span class="highlight">Número atómico:</span> <span id="atomic-number">8</span></p>
        <p><span class="highlight">Masa atómica:</span> <span id="atomic-mass">15.999 u</span></p>
        <p><span class="highlight">Electronegatividad:</span> <span id="electronegativity">3.44</span></p>
        <p><span class="highlight">Descubrimiento:</span> <span id="discovery">1774 (Priestley/Scheele)</span></p>
        <p style="margin-top:12px; font-style:italic; color:#aaa;" id="extra-info">Esencial para la respiración celular.</p>
    </div>

    <div id="status">🌍 Buscando superficies...</div>
    <div id="instructions">
        👆 Toca en una superficie para colocar Oxígeno (rojo)<br>
        👆 Toca en otra superficie para colocar Hidrógeno (blanco)<br>
        🖐️ Arrastra con 1 dedo para mover<br>
        ✌️ Dos dedos para escalar<br>
        👆👆 Doble tap para info
    </div>
    <button id="ar-button">Iniciar AR</button>

    <script>
        (async function() {
            // --- VARIABLES GLOBALES ---
            const infoPanel = document.getElementById('info-panel');
            const statusDiv = document.getElementById('status');
            const arButton = document.getElementById('ar-button');

            // --- ESTADO DE LA APLICACIÓN ---
            let xrSession = null;
            let xrReferenceSpace = null;
            let xrHitTestSource = null;
            let lastHitPose = null;        // Guarda la última pose válida del hit test
            let objectsPlaced = 0;          // 0: nada, 1: oxígeno, 2: hidrógeno, 3: agua
            let isDragging = false;
            let selectedObject = null;
            let initialPinchDistance = 0;
            let initialScale = 1;
            let lastTapTime = 0;
            let showDetailedInfo = false;

            // --- MODELOS 3D ---
            let oxygenModel = null;
            let hydrogenModel = null;
            let waterModel = null;

            // --- Three.js setup ---
            const scene = new THREE.Scene();
            const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.01, 20);
            
            const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.xr.enabled = true;
            document.body.appendChild(renderer.domElement);

            // --- ILUMINACIÓN ---
            const ambientLight = new THREE.AmbientLight(0xffffff, 1.0);
            scene.add(ambientLight);
            const light1 = new THREE.DirectionalLight(0xffeedd, 1.2);
            light1.position.set(1, 1, 1);
            scene.add(light1);
            const light2 = new THREE.DirectionalLight(0x99ccff, 0.8);
            light2.position.set(-1, 0.5, 1);
            scene.add(light2);
            const backLight = new THREE.DirectionalLight(0xccddff, 0.5);
            backLight.position.set(0, 0, -1);
            scene.add(backLight);

            // --- RED DE AYUDA (opcional, solo para depuración) ---
            const gridHelper = new THREE.GridHelper(5, 20, 0x88ccff, 0x446688);
            gridHelper.visible = false;
            scene.add(gridHelper);

            // --- FUNCIÓN AUXILIAR: Sprite con texto (compatible con WebXR) ---
            function createLabelSprite(text, bgColor = '#333333', textColor = '#ffffff') {
                const canvas = document.createElement('canvas');
                canvas.width = 256;
                canvas.height = 128;
                const ctx = canvas.getContext('2d');
                
                // Fondo redondeado
                ctx.fillStyle = bgColor;
                const radius = 20;
                ctx.beginPath();
                ctx.moveTo(radius, 10);
                ctx.lineTo(canvas.width - radius, 10);
                ctx.quadraticCurveTo(canvas.width - 10, 10, canvas.width - 10, radius);
                ctx.lineTo(canvas.width - 10, canvas.height - radius);
                ctx.quadraticCurveTo(canvas.width - 10, canvas.height - 10, canvas.width - radius, canvas.height - 10);
                ctx.lineTo(radius, canvas.height - 10);
                ctx.quadraticCurveTo(10, canvas.height - 10, 10, canvas.height - radius);
                ctx.lineTo(10, radius);
                ctx.quadraticCurveTo(10, 10, radius, 10);
                ctx.closePath();
                ctx.fill();
                
                // Borde
                ctx.strokeStyle = 'rgba(255,255,255,0.5)';
                ctx.lineWidth = 2;
                ctx.stroke();
                
                // Texto
                ctx.font = 'bold 40px Arial';
                ctx.fillStyle = textColor;
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText(text, canvas.width / 2, canvas.height / 2);
                
                const texture = new THREE.CanvasTexture(canvas);
                const material = new THREE.SpriteMaterial({ map: texture, depthTest: false, depthWrite: false });
                const sprite = new THREE.Sprite(material);
                sprite.scale.set(0.4, 0.2, 1);
                return sprite;
            }

            // --- CONSTRUCTORES DE MODELOS ---
            function createAtom(color, radius, orbitalColor, orbitalRadius = radius * 1.8, orbitalTube = 0.025, nombre = 'Átomo') {
                const group = new THREE.Group();
                
                // Núcleo
                const sphereGeo = new THREE.SphereGeometry(radius, 32, 16);
                const sphereMat = new THREE.MeshStandardMaterial({ 
                    color, 
                    roughness: 0.25, 
                    metalness: 0.3, 
                    emissive: color, 
                    emissiveIntensity: 0.1 
                });
                const sphere = new THREE.Mesh(sphereGeo, sphereMat);
                group.add(sphere);
                
                // Orbitales (anillos)
                const torusGeo = new THREE.TorusGeometry(orbitalRadius, orbitalTube, 16, 32);
                const torusMat = new THREE.MeshStandardMaterial({ 
                    color: orbitalColor, 
                    emissive: orbitalColor, 
                    emissiveIntensity: 0.2, 
                    transparent: true, 
                    opacity: 0.5, 
                    side: THREE.DoubleSide 
                });
                const torus = new THREE.Mesh(torusGeo, torusMat);
                torus.rotation.x = Math.PI / 2;
                torus.rotation.z = 0.3;
                group.add(torus);
                
                const torus2 = new THREE.Mesh(torusGeo, torusMat.clone());
                torus2.rotation.y = Math.PI / 2;
                torus2.rotation.x = 0.5;
                torus2.scale.set(1.1, 1.1, 1.1);
                group.add(torus2);
                
                // Etiqueta (sprite)
                const label = createLabelSprite(nombre, '#1a3a4a', '#ffffff');
                label.position.y = radius * 2.5;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'atom';
                return group;
            }

            function createH2() {
                const group = new THREE.Group();
                const hGeo = new THREE.SphereGeometry(0.1, 24, 12);
                const hMat = new THREE.MeshStandardMaterial({ 
                    color: 0xffffff, 
                    roughness: 0.3, 
                    metalness: 0.1, 
                    emissive: 0x446688, 
                    emissiveIntensity: 0.1 
                });
                
                const h1 = new THREE.Mesh(hGeo, hMat);
                h1.position.x = -0.15;
                group.add(h1);
                
                const h2 = new THREE.Mesh(hGeo, hMat.clone());
                h2.position.x = 0.15;
                group.add(h2);
                
                const bondGeo = new THREE.CylinderGeometry(0.03, 0.03, 0.3, 6);
                const bondMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, roughness: 0.6, metalness: 0.8 });
                const bond = new THREE.Mesh(bondGeo, bondMat);
                bond.rotation.z = Math.PI / 2;
                bond.position.x = 0;
                group.add(bond);
                
                const label = createLabelSprite('Hidrógeno (H₂)', '#1a3a4a', '#aaddff');
                label.position.y = 0.35;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'h2';
                return group;
            }

            function createH2O() {
                const group = new THREE.Group();
                
                // Oxígeno
                const oGeo = new THREE.SphereGeometry(0.16, 32, 16);
                const oMat = new THREE.MeshStandardMaterial({ 
                    color: 0xff3333, 
                    roughness: 0.25, 
                    metalness: 0.2, 
                    emissive: 0x331111, 
                    emissiveIntensity: 0.2 
                });
                const oxygen = new THREE.Mesh(oGeo, oMat);
                group.add(oxygen);
                
                // Hidrógenos
                const hGeo = new THREE.SphereGeometry(0.1, 24, 12);
                const hMat = new THREE.MeshStandardMaterial({ 
                    color: 0xffffff, 
                    roughness: 0.3, 
                    metalness: 0.1, 
                    emissive: 0x446688, 
                    emissiveIntensity: 0.1 
                });
                
                const h1 = new THREE.Mesh(hGeo, hMat);
                h1.position.set(0.2, 0.15, 0);
                group.add(h1);
                
                const h2 = new THREE.Mesh(hGeo, hMat.clone());
                h2.position.set(0.2, -0.15, 0);
                group.add(h2);
                
                // Enlaces
                const bondGeo = new THREE.CylinderGeometry(0.035, 0.035, 0.25, 6);
                const bondMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.5, metalness: 0.8 });
                
                const bond1 = new THREE.Mesh(bondGeo, bondMat);
                bond1.position.set(0.1, 0.075, 0);
                bond1.rotation.z = -0.7;
                group.add(bond1);
                
                const bond2 = new THREE.Mesh(bondGeo, bondMat);
                bond2.position.set(0.1, -0.075, 0);
                bond2.rotation.z = 0.7;
                group.add(bond2);
                
                const label = createLabelSprite('Agua (H₂O)', '#1a4a5a', '#88ddff');
                label.position.y = 0.4;
                group.add(label);
                
                group.userData.label = label;
                group.userData.type = 'water';
                return group;
            }

            // --- INICIALIZAR MODELOS Y AÑADIRLOS A LA ESCENA (ocultos) ---
            function initModels() {
                oxygenModel = createAtom(0xff5555, 0.16, 0x88ccff, 0.28, 0.025, 'Oxígeno (O)');
                oxygenModel.visible = false;
                scene.add(oxygenModel);

                hydrogenModel = createH2();
                hydrogenModel.visible = false;
                scene.add(hydrogenModel);

                waterModel = createH2O();
                waterModel.visible = false;
                scene.add(waterModel);
            }
            initModels();

            // --- ACTUALIZAR PANEL DE INFORMACIÓN ---
            function updateInfoPanelContent() {
                if (waterModel.visible) {
                    document.getElementById('particle-title').innerText = 'Agua (H₂O)';
                    document.getElementById('atomic-number').innerText = '10 (total)';
                    document.getElementById('atomic-mass').innerText = '18.015 u';
                    document.getElementById('electronegativity').innerText = 'O: 3.44 / H: 2.20';
                    document.getElementById('discovery').innerText = 'Conocida desde la antigüedad';
                    document.getElementById('extra-info').innerText = 'Molécula polar, disolvente universal.';
                } else if (oxygenModel.visible) {
                    document.getElementById('particle-title').innerText = 'Oxígeno (O)';
                    document.getElementById('atomic-number').innerText = '8';
                    document.getElementById('atomic-mass').innerText = '15.999 u';
                    document.getElementById('electronegativity').innerText = '3.44 (Pauling)';
                    document.getElementById('discovery').innerText = '1774 - Priestley, Scheele';
                    document.getElementById('extra-info').innerText = 'Gas incoloro, esencial para la vida.';
                } else if (hydrogenModel.visible) {
                    document.getElementById('particle-title').innerText = 'Hidrógeno (H₂)';
                    document.getElementById('atomic-number').innerText = '1';
                    document.getElementById('atomic-mass').innerText = '1.008 u';
                    document.getElementById('electronegativity').innerText = '2.20';
                    document.getElementById('discovery').innerText = '1766 - Henry Cavendish';
                    document.getElementById('extra-info').innerText = 'Elemento más abundante del universo.';
                } else {
                    document.getElementById('particle-title').innerText = 'Sin objetos';
                    document.getElementById('atomic-number').innerText = '--';
                    document.getElementById('atomic-mass').innerText = '--';
                    document.getElementById('electronegativity').innerText = '--';
                    document.getElementById('discovery').innerText = '--';
                    document.getElementById('extra-info').innerText = 'Coloca oxígeno e hidrógeno en el suelo.';
                }
            }

            function toggleDetailedInfo() {
                showDetailedInfo = !showDetailedInfo;
                if (showDetailedInfo) {
                    updateInfoPanelContent();
                    infoPanel.classList.remove('hidden-panel');
                    infoPanel.classList.add('visible-panel');
                } else {
                    infoPanel.classList.remove('visible-panel');
                    infoPanel.classList.add('hidden-panel');
                }
            }

            // --- LÓGICA DE REACCIÓN QUÍMICA ---
            function checkFusion() {
                if (oxygenModel.visible && hydrogenModel.visible) {
                    const distance = oxygenModel.position.distanceTo(hydrogenModel.position);
                    if (distance < 0.3) {
                        // Crear agua en el punto medio
                        waterModel.position.copy(oxygenModel.position).add(hydrogenModel.position).multiplyScalar(0.5);
                        waterModel.quaternion.copy(oxygenModel.quaternion).slerp(hydrogenModel.quaternion, 0.5);
                        waterModel.scale.setScalar((oxygenModel.scale.x + hydrogenModel.scale.x) / 2);
                        
                        oxygenModel.visible = false;
                        hydrogenModel.visible = false;
                        waterModel.visible = true;
                        
                        objectsPlaced = 3; // agua
                        statusDiv.innerHTML = '💧 ¡Reacción! Se ha formado agua.';
                    }
                }
                
                // Si no se cumple la condición, ocultar agua (por si acaso)
                if (!oxygenModel.visible || !hydrogenModel.visible) {
                    waterModel.visible = false;
                }
            }

            // --- WEBXR: INICIAR SESIÓN AR ---
            async function startAR() {
                if (!navigator.xr) {
                    alert('WebXR no está disponible en este navegador. Prueba con Chrome Android.');
                    return;
                }

                try {
                    const session = await navigator.xr.requestSession('immersive-ar', {
                        requiredFeatures: ['hit-test', 'local-floor']
                    });
                    xrSession = session;
                    
                    await renderer.xr.setSession(session);
                    
                    xrReferenceSpace = await session.requestReferenceSpace('local-floor');
                    
                    const viewerSpace = await session.requestReferenceSpace('viewer');
                    xrHitTestSource = await session.requestHitTestSource({
                        space: viewerSpace
                    });

                    arButton.style.display = 'none';
                    statusDiv.innerHTML = '✅ AR iniciado. Toca una superficie para colocar Oxígeno.';
                    objectsPlaced = 0;

                    // --- MANEJAR TOQUES PARA COLOCACIÓN (solo cuando NO se arrastra) ---
                    const canvas = renderer.domElement;
                    
                    const onTouchStart = (e) => {
                        if (e.touches.length === 1 && !isDragging) {
                            // Usar la última pose de hit test disponible
                            if (lastHitPose) {
                                const pos = new THREE.Vector3(lastHitPose.transform.position.x, lastHitPose.transform.position.y, lastHitPose.transform.position.z);
                                
                                if (objectsPlaced === 0) {
                                    oxygenModel.position.copy(pos);
                                    oxygenModel.visible = true;
                                    oxygenModel.scale.setScalar(1);
                                    objectsPlaced = 1;
                                    statusDiv.innerHTML = '✅ Oxígeno colocado. Toca otra superficie para colocar Hidrógeno.';
                                } else if (objectsPlaced === 1) {
                                    hydrogenModel.position.copy(pos);
                                    hydrogenModel.visible = true;
                                    hydrogenModel.scale.setScalar(1);
                                    objectsPlaced = 2;
                                    statusDiv.innerHTML = '✅ Hidrógeno colocado. Acerca los objetos para la reacción.';
                                }
                                // Si ya hay agua, no hacemos nada
                            }
                        }
                    };

                    canvas.addEventListener('touchstart', onTouchStart, { passive: false });

                    // --- LOOP DE ANIMACIÓN XR ---
                    const onXRFrame = (time, frame) => {
                        if (!xrSession) return;
                        
                        // Solicitar el próximo frame
                        xrSession.requestAnimationFrame(onXRFrame);
                        
                        // Obtener resultados de hit test
                        if (frame && xrHitTestSource) {
                            const hitTestResults = frame.getHitTestResults(xrHitTestSource);
                            if (hitTestResults.length > 0) {
                                const pose = hitTestResults[0].getPose(xrReferenceSpace);
                                if (pose) {
                                    lastHitPose = pose;
                                }
                            }
                        }
                        
                        // Verificar fusión
                        checkFusion();
                    };
                    
                    xrSession.requestAnimationFrame(onXRFrame);
                    
                } catch (e) {
                    console.error(e);
                    alert('Error al iniciar AR: ' + e.message);
                }
            }

            // --- BOTÓN DE INICIO ---
            arButton.addEventListener('click', startAR);

            // --- EVENTOS TÁCTILES PARA INTERACCIÓN (ARRastrar y escalar) ---
            function setupTouchEvents() {
                const canvas = renderer.domElement;

                canvas.addEventListener('touchstart', (e) => {
                    if (e.touches.length === 1) {
                        // Seleccionar objeto para arrastrar
                        const touch = e.touches[0];
                        const coords = new THREE.Vector2(
                            (touch.clientX / window.innerWidth) * 2 - 1,
                            -(touch.clientY / window.innerHeight) * 2 + 1
                        );
                        const raycaster = new THREE.Raycaster();
                        raycaster.setFromCamera(coords, camera);
                        
                        const objects = [];
                        if (oxygenModel.visible) objects.push(oxygenModel);
                        if (hydrogenModel.visible) objects.push(hydrogenModel);
                        if (waterModel.visible) objects.push(waterModel);
                        
                        const intersects = raycaster.intersectObjects(objects, true);
                        
                        if (intersects.length > 0) {
                            let obj = intersects[0].object;
                            // Subir hasta el grupo padre (el modelo)
                            while (obj.parent && obj.parent !== scene) {
                                obj = obj.parent;
                            }
                            selectedObject = obj;
                            isDragging = true;
                            e.preventDefault();
                        }
                    } else if (e.touches.length === 2) {
                        // Iniciar pinch para escalar
                        const dx = e.touches[0].clientX - e.touches[1].clientX;
                        const dy = e.touches[0].clientY - e.touches[1].clientY;
                        initialPinchDistance = Math.sqrt(dx * dx + dy * dy);
                        
                        // Escala inicial del objeto visible (solo uno)
                        if (oxygenModel.visible) initialScale = oxygenModel.scale.x;
                        else if (hydrogenModel.visible) initialScale = hydrogenModel.scale.x;
                        else if (waterModel.visible) initialScale = waterModel.scale.x;
                        e.preventDefault();
                    }
                }, { passive: false });

                canvas.addEventListener('touchmove', (e) => {
                    if (e.touches.length === 1 && isDragging && selectedObject) {
                        // Arrastrar: proyectar toque sobre el plano del suelo (Y=0)
                        const touch = e.touches[0];
                        const coords = new THREE.Vector2(
                            (touch.clientX / window.innerWidth) * 2 - 1,
                            -(touch.clientY / window.innerHeight) * 2 + 1
                        );
                        const raycaster = new THREE.Raycaster();
                        raycaster.setFromCamera(coords, camera);
                        
                        const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
                        const target = new THREE.Vector3();
                        if (raycaster.ray.intersectPlane(plane, target)) {
                            selectedObject.position.copy(target);
                            // Asegurar que quede sobre el suelo
                            selectedObject.position.y = 0;
                        }
                        e.preventDefault();
                    } else if (e.touches.length === 2) {
                        // Escalar con dos dedos
                        const dx = e.touches[0].clientX - e.touches[1].clientX;
                        const dy = e.touches[0].clientY - e.touches[1].clientY;
                        const distance = Math.sqrt(dx * dx + dy * dy);
                        const scaleFactor = distance / initialPinchDistance;
                        const newScale = initialScale * scaleFactor;
                        
                        if (oxygenModel.visible) oxygenModel.scale.setScalar(newScale);
                        if (hydrogenModel.visible) hydrogenModel.scale.setScalar(newScale);
                        if (waterModel.visible) waterModel.scale.setScalar(newScale);
                        e.preventDefault();
                    }
                }, { passive: false });

                canvas.addEventListener('touchend', (e) => {
                    if (e.touches.length === 0) {
                        isDragging = false;
                        selectedObject = null;
                    }
                    
                    // Detectar doble tap
                    const currentTime = new Date().getTime();
                    const tapLength = currentTime - lastTapTime;
                    if (tapLength < 300 && tapLength > 0) {
                        toggleDetailedInfo();
                        e.preventDefault();
                    }
                    lastTapTime = currentTime;
                });
            }
            setupTouchEvents();

            // --- LOOP DE RENDER (para modo no AR y también para CSS2D, pero ahora no usamos CSS2D) ---
            function animate() {
                if (!xrSession) {
                    // Solo renderizar normal si no estamos en AR
                    renderer.render(scene, camera);
                }
                requestAnimationFrame(animate);
            }
            animate();

            // --- REDIMENSIONAR ---
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });

            // Inicializar panel oculto
            infoPanel.classList.add('hidden-panel');
        })();
    </script>
</body>
</html>
