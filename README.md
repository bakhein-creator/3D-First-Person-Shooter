<!DOCTYPE html>
<html lang="en">
<head>
   <meta charset="UTF-8">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   <title>3D First-Person Shooter</title>
   <style>
       body {
           margin: 0;
           overflow: hidden;
           font-family: 'Inter', sans-serif;
           background-color: #000;
       }
       canvas {
           display: block;
           touch-action: none;
           -webkit-user-select: none;
           -moz-user-select: none;
           -ms-user-select: none;
           user-select: none;
       }
       #crosshair {
           position: absolute;
           top: 50%;
           left: 50%;
           transform: translate(-50%, -50%);
           width: 20px;
           height: 20px;
           border: 2px solid #fff;
           border-radius: 50%;
           box-shadow: 0 0 10px #fff;
           pointer-events: none;
           z-index: 100;
       }
       #message-box {
           position: absolute;
           top: 50%;
           left: 50%;
           transform: translate(-50%, -50%);
           background: rgba(0, 0, 0, 0.8);
           color: #fff;
           padding: 2rem 3rem;
           border-radius: 1rem;
           text-align: center;
           font-size: 1.5rem;
           font-weight: bold;
           cursor: pointer;
           user-select: none;
           box-shadow: 0 4px 15px rgba(255, 255, 255, 0.2);
           transition: transform 0.3s ease-in-out;
       }
       #message-box:hover {
           transform: translate(-50%, -50%) scale(1.05);
       }
       #water-count, #enemies-count, #health-count {
           position: absolute;
           top: 10px;
           left: 10px;
           color: white;
           font-size: 1.5rem;
           font-weight: bold;
           background: rgba(0, 0, 0, 0.5);
           padding: 10px 20px;
           border-radius: 10px;
           z-index: 50;
       }
       #enemies-count {
           left: 160px;
       }
       #health-count {
           left: 310px;
           color: #ff4d4d;
       }
       #upgrade-message, #game-clear-message, #game-over-message {
           position: absolute;
           top: 20%;
           left: 50%;
           transform: translate(-50%, -50%);
           background: rgba(0, 0, 0, 0.8);
           color: gold;
           padding: 1rem 2rem;
           border-radius: 1rem;
           text-align: center;
           font-size: 2rem;
           font-weight: bold;
           box-shadow: 0 4px 20px rgba(255, 215, 0, 0.5);
           display: none;
           z-index: 100;
       }
       #game-over-message {
           color: #ff4d4d;
           box-shadow: 0 4px 20px rgba(255, 77, 77, 0.5);
       }
       #gemini-panel {
           position: absolute;
           top: 10px;
           right: 10px;
           background: rgba(0, 0, 0, 0.7);
           color: #fff;
           padding: 1rem;
           border-radius: 1rem;
           width: 300px;
           max-height: 250px;
           overflow-y: auto;
           display: none;
           z-index: 50;
           box-shadow: 0 4px 15px rgba(255, 255, 255, 0.2);
           font-size: 1rem;
       }
       #gemini-panel h3 {
           margin: 0 0 10px 0;
           font-size: 1.2rem;
           color: #4CAF50;
       }
       #gemini-button {
           background-color: #4CAF50;
           color: white;
           border: none;
           padding: 10px 20px;
           border-radius: 8px;
           cursor: pointer;
           font-size: 1rem;
           font-weight: bold;
           transition: background-color 0.3s ease;
           width: 100%;
           margin-top: 10px;
           box-shadow: 0 2px 5px rgba(0,0,0,0.2);
       }
       #gemini-button:hover {
           background-color: #45a049;
       }
       #gemini-button:disabled {
           background-color: #888;
           cursor: not-allowed;
       }
       #gemini-lore-text {
           white-space: pre-wrap;
           margin-top: 10px;
           display: none;
       }
   </style>
</head>
<body>
   <div id="crosshair"></div>
   <div id="message-box">클릭하여 시작</div>
   <div id="water-count">물: 0</div>
   <div id="enemies-count">적: 0 / 50</div>
   <div id="health-count">HP: 10</div>
   <div id="upgrade-message">총 업그레이드!</div>
   <div id="game-clear-message">게임 클리어!</div>
   <div id="game-over-message" style="display: none;">게임 오버!</div>


   <!-- Gemini AI Panel -->
   <div id="gemini-panel">
       <h3>✨적 정보 분석✨</h3>
       <button id="gemini-button">✨적 정보 받기</button>
       <p id="gemini-lore-text"></p>
   </div>


   <!-- Load Three.js library -->
   <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
   <script>
       // 게임 상태를 위한 전역 변수
       let scene, camera, renderer, gun, water = null;
       let player, canMove = false;
       let keys = { w: false, a: false, s: false, d: false };
       const raycaster = new THREE.Raycaster();
       let isShooting = false;
       let enemies = [];
       let shootCooldown = false;
       let enemySpeed = 0.05;
       let time = 0;
       let isZoomed = false;
       const normalFOV = 75;
       const zoomedFOV = 20;
       let waterCount = 0;
       let enemyCount = 1;
       let shootCooldownTime = 300;
       let gunLevel = 1;
       let totalEnemiesDefeated = 0;
       const enemiesToWin = 50;
       let health = 10;
       let isGameOver = false;


       // 플레이어 이동 및 점프 변수
       const moveSpeed = 0.2;
       let verticalVelocity = 0;
       let onGround = true;
       const gravity = 0.01;
       const jumpForce = 0.2;


       // Gemini API 변수
       let geminiButton = null;
       let geminiPanel = null;
       let geminiLoreText = null;
       let healthCountElement = null;


       // 게임 씬 초기화
       function init() {
           console.log('게임 초기화 중...');


           scene = new THREE.Scene();
           scene.background = new THREE.Color(0x87ceeb);


           camera = new THREE.PerspectiveCamera(normalFOV, window.innerWidth / window.innerHeight, 0.1, 1000);
           camera.position.set(0, 1.6, 0);


           renderer = new THREE.WebGLRenderer({ antialias: true });
           renderer.setSize(window.innerWidth, window.innerHeight);
           renderer.setPixelRatio(window.devicePixelRatio);
           document.body.appendChild(renderer.domElement);


           const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
           scene.add(ambientLight);
           const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
           directionalLight.position.set(5, 10, 7.5);
           directionalLight.castShadow = true;
           directionalLight.shadow.mapSize.width = 1024;
           directionalLight.shadow.mapSize.height = 1024;
           directionalLight.shadow.camera.near = 0.5;
           directionalLight.shadow.camera.far = 50;
           scene.add(directionalLight);
          
           const groundGeometry = new THREE.PlaneGeometry(50, 50);
           const groundMaterial = new THREE.MeshPhongMaterial({ color: 0x8f8f8f });
           const ground = new THREE.Mesh(groundGeometry, groundMaterial);
           ground.rotation.x = -Math.PI / 2;
           ground.receiveShadow = true;
           scene.add(ground);


           player = new THREE.Object3D();
           player.add(camera);
           scene.add(player);


           createGun(gunLevel);
           createEnemies(enemyCount);
          
           // Gemini UI 요소 초기화
           geminiButton = document.getElementById('gemini-button');
           geminiPanel = document.getElementById('gemini-panel');
           geminiLoreText = document.getElementById('gemini-lore-text');
           healthCountElement = document.getElementById('health-count');


           setupEventListeners();
           animate();


           console.log('게임 초기화 완료.');
       }


       // 주어진 레벨에 따라 총을 생성하는 함수
       function createGun(level) {
           if (gun) {
               camera.remove(gun);
           }
          
           gun = new THREE.Group();
          
           let rifleBody, rifleHandle, rifleStock, rifleBarrel;
           let glowingCore, scope, sideFin, sideFin2, secondBarrel;


           // --- 레벨별 총 모델 ---
           if (level === 1) {
               // 초기, 기본 총
               rifleBody = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.15, 0.8), new THREE.MeshBasicMaterial({ color: 0x333333 }));
               rifleHandle = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.2, 0.1), new THREE.MeshBasicMaterial({ color: 0x222222 }));
               rifleStock = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.1, 0.4), new THREE.MeshBasicMaterial({ color: 0x333333 }));
               rifleBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.9), new THREE.MeshBasicMaterial({ color: 0x555555 }));


               rifleHandle.position.set(0.05, -0.1, -0.3);
               rifleStock.position.set(0, 0.05, 0.2);
               rifleBarrel.position.set(0, 0, -0.8);
               rifleBarrel.rotation.x = Math.PI / 2;


               gun.add(rifleBody, rifleHandle, rifleStock, rifleBarrel);


           } else if (level === 2) {
               // 첫 번째 업그레이드: 빨간색 빛나는 코어
               rifleBody = new THREE.Mesh(new THREE.BoxGeometry(0.2, 0.15, 0.8), new THREE.MeshBasicMaterial({ color: 0x444444 }));
               rifleHandle = new THREE.Mesh(new THREE.BoxGeometry(0.05, 0.2, 0.1), new THREE.MeshBasicMaterial({ color: 0x222222 }));
               rifleStock = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.1, 0.4), new THREE.MeshBasicMaterial({ color: 0x444444 }));
               rifleBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.9), new THREE.MeshBasicMaterial({ color: 0x555555 }));
               glowingCore = new THREE.Mesh(new THREE.CylinderGeometry(0.015, 0.015, 0.8), new THREE.MeshBasicMaterial({ color: 0xff0000, transparent: true, opacity: 0.8 }));
               scope = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.3), new THREE.MeshBasicMaterial({ color: 0x222222 }));
               sideFin = new THREE.Mesh(new THREE.BoxGeometry(0.01, 0.1, 0.3), new THREE.MeshBasicMaterial({ color: 0x222222 }));
               sideFin2 = sideFin.clone();


               rifleHandle.position.set(0.05, -0.1, -0.3);
               rifleStock.position.set(0, 0.05, 0.2);
               rifleBarrel.position.set(0, 0, -0.8);
               rifleBarrel.rotation.x = Math.PI / 2;
               glowingCore.position.set(0, 0, -0.8);
               glowingCore.rotation.x = Math.PI / 2;
               scope.position.set(0, 0.1, -0.4);
               scope.rotation.z = Math.PI / 2;
               sideFin.position.set(0.1, 0.05, -0.5);
               sideFin2.position.x = -0.1;


               gun.add(rifleBody, rifleHandle, rifleStock, rifleBarrel, glowingCore, scope, sideFin, sideFin2);
          
           } else if (level === 3) {
               // 두 번째 업그레이드: 파란색 빛나는 코어, 이중 총열
               rifleBody = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.15, 0.9), new THREE.MeshBasicMaterial({ color: 0x6a0dad }));
               rifleHandle = new THREE.Mesh(new THREE.BoxGeometry(0.06, 0.2, 0.1), new THREE.MeshBasicMaterial({ color: 0x333333 }));
               rifleStock = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.12, 0.5), new THREE.MeshBasicMaterial({ color: 0x6a0dad }));
              
               // 이중 총열
               rifleBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.025, 0.025, 1.0), new THREE.MeshBasicMaterial({ color: 0x555555 }));
               secondBarrel = rifleBarrel.clone();
              
               glowingCore = new THREE.Mesh(new THREE.CylinderGeometry(0.018, 0.018, 0.9), new THREE.MeshBasicMaterial({ color: 0x00ffff, transparent: true, opacity: 0.8 }));
               scope = new THREE.Mesh(new THREE.CylinderGeometry(0.025, 0.025, 0.35), new THREE.MeshBasicMaterial({ color: 0x222222 }));
               sideFin = new THREE.Mesh(new THREE.BoxGeometry(0.015, 0.12, 0.4), new THREE.MeshBasicMaterial({ color: 0x444444 }));
               sideFin2 = sideFin.clone();


               rifleHandle.position.set(0.06, -0.1, -0.35);
               rifleStock.position.set(0, 0.05, 0.25);
              
               rifleBarrel.position.set(-0.05, 0.03, -0.8);
               rifleBarrel.rotation.x = Math.PI / 2;
               secondBarrel.position.set(0.05, 0.03, -0.8);
               secondBarrel.rotation.x = Math.PI / 2;


               glowingCore.position.set(0, 0, -0.85);
               glowingCore.rotation.x = Math.PI / 2;
               scope.position.set(0, 0.15, -0.45);
               scope.rotation.z = Math.PI / 2;
               sideFin.position.set(0.12, 0.06, -0.5);
               sideFin2.position.x = -0.12;


               gun.add(rifleBody, rifleHandle, rifleStock, rifleBarrel, secondBarrel, glowingCore, scope, sideFin, sideFin2);
          
           } else if (level >= 4) {
               // 최종, 궁극의 총
               rifleBody = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.18, 1.0), new THREE.MeshBasicMaterial({ color: 0xffd700 }));
               rifleHandle = new THREE.Mesh(new THREE.BoxGeometry(0.07, 0.22, 0.12), new THREE.MeshBasicMaterial({ color: 0x555555 }));
               rifleStock = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.15, 0.6), new THREE.MeshBasicMaterial({ color: 0xffd700 }));
              
               // 삼중 총열
               rifleBarrel = new THREE.Mesh(new THREE.CylinderGeometry(0.03, 0.03, 1.2), new THREE.MeshBasicMaterial({ color: 0x777777 }));
               let barrel2 = rifleBarrel.clone();
               let barrel3 = rifleBarrel.clone();


               glowingCore = new THREE.Mesh(new THREE.CylinderGeometry(0.025, 0.025, 1.1), new THREE.MeshBasicMaterial({ color: 0xffa500, transparent: true, opacity: 0.9 }));
               scope = new THREE.Mesh(new THREE.CylinderGeometry(0.03, 0.03, 0.4), new THREE.MeshBasicMaterial({ color: 0x333333 }));
               sideFin = new THREE.Mesh(new THREE.BoxGeometry(0.02, 0.15, 0.5), new THREE.MeshBasicMaterial({ color: 0x444444 }));
               sideFin2 = sideFin.clone();


               rifleHandle.position.set(0.07, -0.1, -0.4);
               rifleStock.position.set(0, 0.07, 0.3);


               rifleBarrel.position.set(0, 0, -0.9);
               rifleBarrel.rotation.x = Math.PI / 2;
               barrel2.position.set(-0.07, 0.04, -0.9);
               barrel2.rotation.x = Math.PI / 2;
               barrel3.position.set(0.07, 0.04, -0.9);
               barrel3.rotation.x = Math.PI / 2;


               glowingCore.position.set(0, 0, -0.95);
               glowingCore.rotation.x = Math.PI / 2;
               scope.position.set(0, 0.18, -0.5);
               scope.rotation.z = Math.PI / 2;
               sideFin.position.set(0.15, 0.08, -0.6);
               sideFin2.position.x = -0.15;


               gun.add(rifleBody, rifleHandle, rifleStock, rifleBarrel, barrel2, barrel3, glowingCore, scope, sideFin, sideFin2);
           }


           gun.position.set(0.25, -0.2, -0.5);
           camera.add(gun);
          
           const upgradeMessage = document.getElementById('upgrade-message');
           upgradeMessage.textContent = `총 업그레이드! (Lv.${level})`;
           upgradeMessage.style.display = 'block';


           setTimeout(() => {
               upgradeMessage.style.display = 'none';
           }, 1000);
       }


       // 단일 적 생성 함수
       function createSingleEnemy() {
           const newEnemy = new THREE.Group();
          
           const bodyMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
           const headMaterial = new THREE.MeshBasicMaterial({ color: 0xffa500 });
           const limbMaterial = new THREE.MeshBasicMaterial({ color: 0x0000ff });


           const torso = new THREE.Mesh(new THREE.BoxGeometry(1.0, 1.5, 0.5), bodyMaterial);
           torso.position.y = 1.25;
           newEnemy.add(torso);


           const head = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.8, 0.8), headMaterial);
           head.position.y = 2.2;
           newEnemy.add(head);


           const armGeometry = new THREE.CylinderGeometry(0.2, 0.2, 1.2);
           const leftArm = new THREE.Mesh(armGeometry, limbMaterial);
           leftArm.position.set(-0.6, 1.25, 0);
           leftArm.rotation.z = Math.PI / 2;
           newEnemy.add(leftArm);
           newEnemy.leftArm = leftArm;


           const rightArm = new THREE.Mesh(armGeometry, limbMaterial);
           rightArm.position.set(0.6, 1.25, 0);
           rightArm.rotation.z = -Math.PI / 2;
           newEnemy.add(rightArm);
           newEnemy.rightArm = rightArm;


           const legGeometry = new THREE.CylinderGeometry(0.2, 0.2, 1.2);
           const leftLeg = new THREE.Mesh(legGeometry, limbMaterial);
           leftLeg.position.set(-0.25, 0.6, 0);
           newEnemy.add(leftLeg);
           newEnemy.leftLeg = leftLeg;


           const rightLeg = new THREE.Mesh(legGeometry, limbMaterial);
           rightLeg.position.set(0.25, 0.6, 0);
           newEnemy.add(rightLeg);
           newEnemy.rightLeg = rightLeg;


           newEnemy.position.set(Math.random() * 20 - 10, 0, Math.random() * 20 - 10);
           scene.add(newEnemy);
           return newEnemy;
       }


       // 여러 적 생성 함수
       function createEnemies(count) {
           enemies.forEach(e => scene.remove(e));
           enemies = [];
           for (let i = 0; i < count; i++) {
               enemies.push(createSingleEnemy());
           }
       }
      
       // 특정 위치에 물체 드롭 함수
       function dropWater(position) {
           if (water) {
               scene.remove(water);
           }
          
           const waterGeometry = new THREE.CylinderGeometry(0.2, 0.2, 0.1, 12);
           const waterMaterial = new THREE.MeshBasicMaterial({ color: 0x00aaff });
           water = new THREE.Mesh(waterGeometry, waterMaterial);
          
           water.position.copy(position);
           water.position.y = 0.1;
           scene.add(water);
       }


       // 사용자 상호 작용을 위한 모든 이벤트 리스너 설정
       function setupEventListeners() {
           document.getElementById('message-box').addEventListener('click', () => {
               if (!canMove) {
                   document.body.requestPointerLock();
                   document.getElementById('message-box').style.display = 'none';
               }
           });


           document.addEventListener('pointerlockchange', () => {
               if (document.pointerLockElement === document.body) {
                   canMove = true;
                   console.log('포인터 잠금 활성화.');
               } else {
                   canMove = false;
                   console.log('포인터 잠금 비활성화.');
               }
           });


           document.addEventListener('mousemove', (event) => {
               if (canMove) {
                   const movementX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;
                   const movementY = event.movementY || event.mozMovementY || event.webkitMovementY || 0;


                   player.rotation.y -= movementX * 0.002;
                   camera.rotation.x -= movementY * 0.002;
                   camera.rotation.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, camera.rotation.x));
               }
           });


           document.addEventListener('keydown', (event) => {
               if (canMove) {
                   const key = event.key.toLowerCase();
                   if (keys.hasOwnProperty(key)) {
                       keys[key] = true;
                   }
                   if (event.key === ' ' && onGround) {
                       onGround = false;
                       verticalVelocity = jumpForce;
                   }
               }
           });
           document.addEventListener('keyup', (event) => {
               if (canMove) {
                   const key = event.key.toLowerCase();
                   if (keys.hasOwnProperty(key)) {
                       keys[key] = false;
                   }
               }
           });


           document.addEventListener('mousedown', (event) => {
               if (canMove && event.button === 0 && !isShooting && !shootCooldown) {
                   shoot();
               } else if (canMove && event.button === 2) {
                   event.preventDefault();
                   isZoomed = !isZoomed;
                   camera.fov = isZoomed ? zoomedFOV : normalFOV;
                   camera.updateProjectionMatrix();
               }
           });


           window.addEventListener('resize', () => {
               camera.aspect = window.innerWidth / window.innerHeight;
               camera.updateProjectionMatrix();
               renderer.setSize(window.innerWidth, window.innerHeight);
           });
          
           // Gemini 버튼에 대한 이벤트 리스너
           if (geminiButton) {
               geminiButton.addEventListener('click', generateEnemyLore);
           }
       }


       // 레이캐스터를 사용한 총격 함수
       function shoot() {
           if (totalEnemiesDefeated >= enemiesToWin || isGameOver) return;


           isShooting = true;
           shootCooldown = true;


           gun.rotation.x = -Math.PI / 16;
           setTimeout(() => {
               gun.rotation.x = 0;
               isShooting = false;
           }, 100);


           raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
           const intersects = raycaster.intersectObjects(scene.children, true);


           for (const intersect of intersects) {
               let hitEnemy = null;
               for (const enemy of enemies) {
                   if (intersect.object.parent === enemy) {
                       hitEnemy = enemy;
                       break;
                   }
               }


               if (hitEnemy) {
                   const enemyPosition = hitEnemy.position.clone();
                   scene.remove(hitEnemy);
                   enemies = enemies.filter(e => e !== hitEnemy);
                   dropWater(enemyPosition);
                  
                   totalEnemiesDefeated++;
                   document.getElementById('enemies-count').textContent = `적: ${totalEnemiesDefeated} / ${enemiesToWin}`;


                   const messageBox = document.getElementById('message-box');
                   messageBox.textContent = '적 처치!';
                   messageBox.style.display = 'block';


                   // 적 처치 후 Gemini 패널 보이기
                   geminiPanel.style.display = 'block';
                   geminiLoreText.style.display = 'none';
                   geminiButton.disabled = false;
                   geminiButton.textContent = '✨적 정보 받기';


                   setTimeout(() => {
                       messageBox.style.display = 'none';
                   }, 1000);


                   if (enemies.length === 0) {
                       createEnemies(enemyCount);
                   }
                   break;
               }
           }


           setTimeout(() => {
               shootCooldown = false;
           }, shootCooldownTime);
       }


       // 메인 애니메이션 루프
       function animate() {
           requestAnimationFrame(animate);
           if (totalEnemiesDefeated >= enemiesToWin) {
               canMove = false;
               document.getElementById('game-clear-message').style.display = 'block';
               return;
           }
           if (isGameOver) {
               return;
           }


           time += 0.05;


           // 적과의 충돌 감지
           for (let i = enemies.length - 1; i >= 0; i--) {
               const enemy = enemies[i];
               if (player.position.distanceTo(enemy.position) < 1.5) {
                   health--;
                   healthCountElement.textContent = `HP: ${health}`;
                  
                   // 데미지 메시지 일시적으로 표시
                   const messageBox = document.getElementById('message-box');
                   messageBox.textContent = '적에게 맞았습니다!';
                   messageBox.style.display = 'block';
                   setTimeout(() => { messageBox.style.display = 'none'; }, 1000);
                  
                   if (health <= 0) {
                       isGameOver = true;
                       document.getElementById('game-over-message').style.display = 'block';
                       canMove = false;
                       return;
                   }


                   // 충돌한 적 제거 및 새 적 생성
                   scene.remove(enemy);
                   enemies.splice(i, 1);
                   if (enemies.length === 0) {
                       createEnemies(enemyCount);
                   }
               }
           }


           if (canMove) {
               const forward = new THREE.Vector3();
               camera.getWorldDirection(forward);
              
               const moveForward = new THREE.Vector3(forward.x, 0, forward.z).normalize();
               const moveRight = new THREE.Vector3().crossVectors(camera.up, moveForward).normalize();


               if (keys.w) player.position.add(moveForward.multiplyScalar(moveSpeed));
               if (keys.s) player.position.add(moveForward.multiplyScalar(-moveSpeed));
               if (keys.a) player.position.add(moveRight.multiplyScalar(moveSpeed));
               if (keys.d) player.position.add(moveRight.multiplyScalar(-moveSpeed));
           }
          
           if (!onGround) {
               player.position.y += verticalVelocity;
               verticalVelocity -= gravity;
               if (player.position.y <= 1.6) {
                   player.position.y = 1.6;
                   onGround = true;
                   verticalVelocity = 0;
               }
           }


           enemies.forEach(enemy => {
               if (enemy.leftArm) {
                   const direction = new THREE.Vector3().subVectors(player.position, enemy.position).normalize();
                   enemy.position.add(direction.multiplyScalar(enemySpeed));


                   enemy.position.y = Math.sin(time * 5) * 0.1;


                   const walkCycle = Math.sin(time * 5);
                   const armSwingAngle = Math.PI / 4;
                   const legSwingAngle = Math.PI / 8;


                   enemy.leftArm.rotation.x = walkCycle * armSwingAngle;
                   enemy.rightLeg.rotation.x = -walkCycle * legSwingAngle;


                   enemy.rightArm.rotation.x = -walkCycle * armSwingAngle;
                   enemy.leftLeg.rotation.x = walkCycle * legSwingAngle;
               }
           });


           if (water) {
               if (player.position.distanceTo(water.position) < 1.0) {
                   scene.remove(water);
                   water = null;
                  
                   waterCount++;
                   document.getElementById('water-count').textContent = '물: ' + waterCount;


                   const messageBox = document.getElementById('message-box');
                   messageBox.textContent = '물 획득!';
                   messageBox.style.display = 'block';


                   if (waterCount >= 10) {
                       if (gunLevel < 4) {
                           waterCount = 0;
                       }
                      
                       document.getElementById('water-count').textContent = '물: ' + waterCount;
                       enemyCount++;
                       gunLevel++;
                       shootCooldownTime = Math.max(50, shootCooldownTime - 50);
                       createGun(gunLevel);
                       createEnemies(enemyCount);
                   }


                   setTimeout(() => {
                       messageBox.style.display = 'none';
                   }, 1000);
               }
           }


           renderer.render(scene, camera);
       }


       // Gemini API를 사용하여 적의 배경 지식을 생성하고 표시하는 함수
       async function generateEnemyLore() {
           geminiButton.disabled = true;
           geminiButton.textContent = '분석 중...';
           geminiLoreText.style.display = 'none';
           geminiLoreText.textContent = '';


           const userQuery = "3D 총 게임에서 방금 쓰러뜨린 적에 대한 짧고 기발한 설명을 만들어줘. 비현실적이고 코믹한 설정을 포함해줘. 한국어로 작성해줘.";
           const apiKey = "";
           const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;


           const payload = {
               contents: [{ parts: [{ text: userQuery }] }],
               systemInstruction: {
                   parts: [{ text: "당신은 게임의 게임 마스터입니다. 플레이어가 쓰러뜨린 적에 대한 독특한 설명을 제공하세요." }]
               }
           };


           try {
               const response = await fetch(apiUrl, {
                   method: 'POST',
                   headers: { 'Content-Type': 'application/json' },
                   body: JSON.stringify(payload)
               });


               if (!response.ok) {
                   throw new Error(`HTTP error! status: ${response.status}`);
               }


               const result = await response.json();
               const candidate = result.candidates?.[0];
               let text = "적의 정보를 찾을 수 없습니다.";


               if (candidate && candidate.content?.parts?.[0]?.text) {
                   text = candidate.content.parts[0].text;
               }


               geminiLoreText.textContent = text;
               geminiLoreText.style.display = 'block';
           } catch (error) {
               console.error('API 호출 중 오류 발생:', error);
               geminiLoreText.textContent = "적 정보 분석에 실패했습니다. 다시 시도해 주세요.";
               geminiLoreText.style.display = 'block';
           } finally {
               geminiButton.disabled = false;
               geminiButton.textContent = '✨적 정보 받기';
           }
       }


       window.onload = init;
   </script>
</body>
</html>


