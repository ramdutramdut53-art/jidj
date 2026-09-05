<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CyberStrike: Cyberpunk Shotgun Shooter</title>
  <style>
    body {
      margin: 0;
      overflow: hidden;
      background-color: #050508;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      user-select: none;
    }
    #canvas-container {
      width: 100vw;
      height: 100vh;
      display: block;
    }
    #ui-overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
      box-sizing: border-box;
      padding: 30px;
      display: flex;
      flex-direction: column;
      justify-content: space-between;
    }
    .hud-element {
      background: rgba(10, 15, 25, 0.75);
      border: 1px solid rgba(0, 255, 204, 0.3);
      border-left: 4px solid #00ffcc;
      backdrop-filter: blur(8px);
      padding: 15px 25px;
      border-radius: 4px;
      color: #fff;
      box-shadow: 0 0 15px rgba(0, 255, 204, 0.15);
    }
    #stats-hud {
      display: flex;
      gap: 30px;
    }
    .stat-val {
      font-size: 28px;
      font-weight: 800;
      color: #00ffcc;
      text-shadow: 0 0 10px rgba(0, 255, 204, 0.5);
    }
    .stat-label {
      font-size: 11px;
      text-transform: uppercase;
      letter-spacing: 2px;
      color: #88a0b0;
    }
    #weapon-hud {
      align-self: flex-end;
      text-align: right;
    }
    #ammo-val {
      color: #ff0055;
      text-shadow: 0 0 10px rgba(255, 0, 85, 0.5);
    }
    #crosshair {
      position: absolute;
      top: 50%;
      left: 50%;
      width: 20px;
      height: 20px;
      transform: translate(-50%, -50%);
      pointer-events: none;
    }
    #crosshair::before, #crosshair::after {
      content: '';
      position: absolute;
      background: #00ffcc;
      box-shadow: 0 0 8px #00ffcc;
    }
    #crosshair::before { top: 9px; left: 0; width: 20px; height: 2px; }
    #crosshair::after { top: 0; left: 9px; width: 2px; height: 20px; }
    
    #game-over-screen {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(5, 5, 10, 0.85);
      backdrop-filter: blur(12px);
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      color: #fff;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.5s ease;
    }
    #game-over-screen.active {
      opacity: 1;
      pointer-events: auto;
    }
    h1 {
      font-size: 64px;
      margin: 0;
      color: #ff0055;
      text-shadow: 0 0 20px rgba(255, 0, 85, 0.8);
      letter-spacing: 4px;
    }
    p { font-size: 18px; color: #a0a0c0; }
    .btn {
      margin-top: 25px;
      padding: 15px 40px;
      background: transparent;
      border: 2px solid #00ffcc;
      color: #00ffcc;
      font-size: 18px;
      font-weight: bold;
      text-transform: uppercase;
      letter-spacing: 2px;
      cursor: pointer;
      transition: all 0.2s;
      box-shadow: 0 0 15px rgba(0, 255, 204, 0.2);
    }
    .btn:hover {
      background: #00ffcc;
      color: #000;
      box-shadow: 0 0 30px rgba(0, 255, 204, 0.8);
    }
  </style>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>
  <div id="canvas-container"></div>
  <div id="crosshair"></div>
  
  <div id="ui-overlay">
    <div id="stats-hud" class="hud-element">
      <div>
        <div class="stat-label">Health</div>
        <div id="health-val" class="stat-val">100</div>
      </div>
      <div>
        <div class="stat-label">Score</div>
        <div id="score-val" class="stat-val">0</div>
      </div>
      <div>
        <div class="stat-label">Wave</div>
        <div id="wave-val" class="stat-val">1</div>
      </div>
    </div>
    
    <div id="weapon-hud" class="hud-element">
      <div class="stat-label">Weapon</div>
      <div id="weapon-name" class="stat-val" style="font-size:20px;">CYBER SHOTGUN</div>
      <div class="stat-label" style="margin-top:8px;">Shells Loaded</div>
      <div id="ammo-val" class="stat-val">8 / ∞</div>
    </div>
  </div>

  <div id="game-over-screen">
    <h1>SYSTEM CRITICAL</h1>
    <p>You were overrun by the rogue defense nodes.</p>
    <p>Final Score: <span id="final-score" style="color:#00ffcc; font-weight:bold;">0</span></p>
    <button class="btn" onclick="restartGame()">REBOOT</button>
  </div>

  <script>
    // --- GAME ENGINE SETUP ---
    const container = document.getElementById('canvas-container');
    const scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2(0x050508, 0.03);

    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    container.appendChild(renderer.domElement);

    // --- LIGHTING ---
    const ambientLight = new THREE.AmbientLight(0x1a1a2e, 0.8);
    scene.add(ambientLight);

    const dirLight = new THREE.DirectionalLight(0x00d2ff, 0.5);
    dirLight.position.set(20, 40, 20);
    dirLight.castShadow = true;
    dirLight.shadow.mapSize.width = 2048;
    dirLight.shadow.mapSize.height = 2048;
    scene.add(dirLight);

    // Dynamic player light
    const playerLight = new THREE.PointLight(0x00ffcc, 2, 12);
    scene.add(playerLight);

    // --- ARENA SETUP ---
    const arenaSize = 50;
    const gridHelper = new THREE.GridHelper(arenaSize, 50, 0x00ffcc, 0x112233);
    gridHelper.position.y = 0.01;
    scene.add(gridHelper);

    const floorGeo = new THREE.PlaneGeometry(arenaSize, arenaSize);
    const floorMat = new THREE.MeshStandardMaterial({ color: 0x0a0a10, roughness: 0.8, metalness: 0.2 });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    scene.add(floor);

    // Arena Boundary Walls (Visual)
    const wallMat = new THREE.MeshStandardMaterial({ color: 0x050515, emissive: 0x00ffff, emissiveIntensity: 0.05 });
    const wallGeo = new THREE.BoxGeometry(arenaSize, 4, 1);
    
    for (let i = 0; i < 4; i++) {
      const wall = new THREE.Mesh(wallGeo, wallMat);
      if (i === 0) wall.position.set(0, 2, -arenaSize/2);
      if (i === 1) wall.position.set(0, 2, arenaSize/2);
      if (i === 2) { wall.position.set(-arenaSize/2, 2, 0); wall.rotation.y = Math.PI/2; }
      if (i === 3) { wall.position.set(arenaSize/2, 2, 0); wall.rotation.y = Math.PI/2; }
      scene.add(wall);
    }

    // --- PLAYER CREATION ---
    const playerGroup = new THREE.Group();
    const bodyGeo = new THREE.CylinderGeometry(0.6, 0.6, 1.8, 16);
    const bodyMat = new THREE.MeshStandardMaterial({ color: 0x111122, metalness: 0.8, roughness: 0.2 });
    const playerBody = new THREE.Mesh(bodyGeo, bodyMat);
    playerBody.position.y = 0.9;
    playerBody.castShadow = true;
    playerGroup.add(playerBody);

    // Visor
    const visorGeo = new THREE.BoxGeometry(0.5, 0.2, 0.4);
    const visorMat = new THREE.MeshBasicMaterial({ color: 0x00ffcc });
    const visor = new THREE.Mesh(visorGeo, visorMat);
    visor.position.set(0, 1.3, 0.4);
    playerGroup.add(visor);

    // Shotgun Mesh
    const gunGroup = new THREE.Group();
    const barrelGeo = new THREE.BoxGeometry(0.15, 0.15, 1.2);
    const gunMat = new THREE.MeshStandardMaterial({ color: 0x333344, metalness: 0.9, roughness: 0.1 });
    const barrel = new THREE.Mesh(barrelGeo, gunMat);
    barrel.position.set(0.4, 0.9, 0.6);
    gunGroup.add(barrel);
    playerGroup.add(gunGroup);

    scene.add(playerGroup);

    // --- GAME STATE VARIABLES ---
    let player = {
      hp: 100,
      maxHp: 100,
      speed: 0.18,
      position: playerGroup.position,
      score: 0,
      wave: 1
    };

    let shotgun = {
      ammo: 8,
      maxAmmo: 8,
      reloading: false,
      lastShot: 0,
      fireRate: 650, // ms
      pellets: 8,
      spread: 0.18
    };

    let keys = {};
    let mousePos = new THREE.Vector3();
    let raycaster = new THREE.Raycaster();
    let mouseVec = new THREE.Vector2();
    let plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);

    let bullets = [];
    let enemies = [];
    let particles = [];
    let isGameOver = false;

    // --- CONTROLS LISTENERS ---
    window.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
    window.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);
    
    window.addEventListener('mousemove', e => {
      mouseVec.x = (e.clientX / window.innerWidth) * 2 - 1;
      mouseVec.y = -(e.clientY / window.innerHeight) * 2 + 1;
      
      // Dynamic crosshair movement
      const ch = document.getElementById('crosshair');
      ch.style.left = e.clientX + 'px';
      ch.style.top = e.clientY + 'px';
    });

    window.addEventListener('mousedown', e => {
      if (e.button === 0 && !isGameOver) shootShotgun();
    });

    window.addEventListener('keydown', e => {
      if (e.key.toLowerCase() === 'r') reloadShotgun();
    });

    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });

    // --- SHOOTING MECHANIC ---
    function shootShotgun() {
      const now = Date.now();
      if (now - shotgun.lastShot < shotgun.fireRate || shotgun.reloading) return;
      if (shotgun.ammo <= 0) {
        reloadShotgun();
        return;
      }

      shotgun.ammo--;
      shotgun.lastShot = now;
      updateHUD();

      // Gun recoil animation bounce
      gunGroup.position.z -= 0.3;

      // Muzzle Flash
      const flash = new THREE.PointLight(0xffaa00, 5, 5);
      flash.position.copy(playerGroup.position).add(new THREE.Vector3(0.4, 0.9, 0.8).applyQuaternion(playerGroup.quaternion));
      scene.add(flash);
      setTimeout(() => scene.remove(flash), 50);

      // Fire Pellets (Shotgun Spread)
      const gunTip = new THREE.Vector3(0.4, 0.9, 0.8).applyQuaternion(playerGroup.quaternion).add(playerGroup.position);

      for (let i = 0; i < shotgun.pellets; i++) {
        const bulletGeo = new THREE.SphereGeometry(0.08, 8, 8);
        const bulletMat = new THREE.MeshBasicMaterial({ color: 0xffcc00 });
        const bullet = new THREE.Mesh(bulletGeo, bulletMat);
        bullet.position.copy(gunTip);

        // Calculate direction with random spread
        const dir = new THREE.Vector3(0, 0, 1).applyQuaternion(playerGroup.quaternion);
        dir.x += (Math.random() - 0.5) * shotgun.spread;
        dir.z += (Math.random() - 0.5) * shotgun.spread;
        dir.normalize();

        bullets.push({
          mesh: bullet,
          dir: dir,
          speed: 0.8,
          life: 30 // frames
        });
        scene.add(bullet);
      }
    }

    function reloadShotgun() {
      if (shotgun.reloading || shotgun.ammo === shotgun.maxAmmo) return;
      shotgun.reloading = true;
      document.getElementById('ammo-val').innerText = "RELOADING...";
      setTimeout(() => {
        shotgun.ammo = shotgun.maxAmmo;
        shotgun.reloading = false;
        updateHUD();
      }, 1500);
    }

    // --- ENEMY SYSTEM ---
    function spawnEnemy() {
      if (isGameOver) return;

      const enemyGeo = new THREE.BoxGeometry(1.2, 1.2, 1.2);
      const enemyMat = new THREE.MeshStandardMaterial({ color: 0xff0055, emissive: 0x330011, roughness: 0.3 });
      const enemy = new THREE.Mesh(enemyGeo, enemyMat);
      
      // Spawn at random edge of arena
      const angle = Math.random() * Math.PI * 2;
      const radius = arenaSize / 2 - 2;
      enemy.position.set(Math.cos(angle) * radius, 0.6, Math.sin(angle) * radius);
      enemy.castShadow = true;

      enemies.push({
        mesh: enemy,
        hp: 3,
        speed: 0.05 + Math.random() * 0.03 + (player.wave * 0.005)
      });
      scene.add(enemy);
    }

    setInterval(() => {
      if (enemies.length < 5 + player.wave * 3) {
        spawnEnemy();
      }
    }, 1500);

    // --- PARTICLE EFFECTS ---
    function createBloodSplatter(pos, color = 0xff0055) {
      for (let i = 0; i < 12; i++) {
        const pGeo = new THREE.BoxGeometry(0.1, 0.1, 0.1);
        const pMat = new THREE.MeshBasicMaterial({ color: color });
        const p = new THREE.Mesh(pGeo, pMat);
        p.position.copy(pos);
        
        const vel = new THREE.Vector3(
          (Math.random() - 0.5) * 0.3,
          Math.random() * 0.2 + 0.1,
          (Math.random() - 0.5) * 0.3
        );

        particles.push({ mesh: p, vel: vel, life: 20 });
        scene.add(p);
      }
    }

    // --- GAME LOOP ---
    function animate() {
      if (isGameOver) return;
      requestAnimationFrame(animate);

      // 1. Player Movement Logic
      const moveVec = new THREE.Vector3();
      if (keys['w'] || keys['arrowup']) moveVec.z -= 1;
      if (keys['s'] || keys['arrowdown']) moveVec.z += 1;
      if (keys['a'] || keys['arrowleft']) moveVec.x -= 1;
      if (keys['d'] || keys['arrowright']) moveVec.x += 1;

      moveVec.normalize().multiplyScalar(player.speed);
      playerGroup.position.add(moveVec);

      // Restrict movement to arena bounds
      const bound = arenaSize / 2 - 1;
      playerGroup.position.x = Math.max(-bound, Math.min(bound, playerGroup.position.x));
      playerGroup.position.z = Math.max(-bound, Math.min(bound, playerGroup.position.z));

      // Light follows player
      playerLight.position.set(playerGroup.position.x, playerGroup.position.y + 3, playerGroup.position.z);

      // 2. Mouse Aiming / Rotation
      raycaster.setFromCamera(mouseVec, camera);
      const targetPoint = new THREE.Vector3();
      raycaster.ray.intersectPlane(plane, targetPoint);
      
      if (targetPoint) {
        playerGroup.lookAt(targetPoint.x, playerGroup.position.y, targetPoint.z);
      }

      // Gun recoil recovery
      gunGroup.position.z += (0 - gunGroup.position.z) * 0.1;

      // 3. Bullets Physics & Collision
      for (let i = bullets.length - 1; i >= 0; i--) {
        const b = bullets[i];
        b.mesh.position.addScaledVector(b.dir, b.speed);
        b.life--;

        // Check bullet vs enemy collision
        let hit = false;
        for (let j = enemies.length - 1; j >= 0; j--) {
          const e = enemies[j];
          if (b.mesh.position.distanceTo(e.mesh.position) < 0.8) {
            e.hp--;
            createBloodSplatter(b.mesh.position);
            
            // Knockback
            e.mesh.position.addScaledVector(b.dir, 0.2);

            if (e.hp <= 0) {
              createBloodSplatter(e.mesh.position, 0xffaa00);
              scene.remove(e.mesh);
              enemies.splice(j, 1);
              player.score += 100;
              
              // Wave progression
              if (player.score % 1000 === 0) {
                player.wave++;
              }
              updateHUD();
            }

            hit = true;
            break;
          }
        }

        if (hit || b.life <= 0) {
          scene.remove(b.mesh);
          bullets.splice(i, 1);
        }
      }

      // 4. Enemy AI & Collision
      for (let i = enemies.length - 1; i >= 0; i--) {
        const e = enemies[i];
        
        // Move towards player
        const dir = new THREE.Vector3().subVectors(playerGroup.position, e.mesh.position).normalize();
        dir.y = 0;
        e.mesh.position.addScaledVector(dir, e.speed);
        e.mesh.lookAt(playerGroup.position.x, e.mesh.position.y, playerGroup.position.z);

        // Enemy hits player
        if (e.mesh.position.distanceTo(playerGroup.position) < 1.0) {
          player.hp -= 0.5;
          createBloodSplatter(playerGroup.position, 0x00ffcc);
          updateHUD();

          if (player.hp <= 0) {
            triggerGameOver();
          }
        }
      }

      // 5. Update Particles
      for (let i = particles.length - 1; i >= 0; i--) {
        const p = particles[i];
        p.mesh.position.add(p.vel);
        p.life--;
        if (p.life <= 0) {
          scene.remove(p.mesh);
          particles.splice(i, 1);
        }
      }

      // Camera Top-Down Dynamic Follow
      camera.position.x = playerGroup.position.x;
      camera.position.z = playerGroup.position.z + 18;
      camera.position.y = 22;
      camera.lookAt(playerGroup.position);

      renderer.render(scene, camera);
    }

    function updateHUD() {
      document.getElementById('health-val').innerText = Math.max(0, Math.ceil(player.hp));
      document.getElementById('score-val').innerText = player.score;
      document.getElementById('wave-val').innerText = player.wave;
      if (!shotgun.reloading) {
        document.getElementById('ammo-val').innerText = `${shotgun.ammo} / ∞`;
      }
    }

    function triggerGameOver() {
      isGameOver = true;
      document.getElementById('final-score').innerText = player.score;
      document.getElementById('game-over-screen').classList.add('active');
    }

    function restartGame() {
      // Reset State
      player.hp = 100;
      player.score = 0;
      player.wave = 1;
      shotgun.ammo = shotgun.maxAmmo;
      shotgun.reloading = false;
      playerGroup.position.set(0, 0, 0);

      // Clear Entities
      enemies.forEach(e => scene.remove(e.mesh));
      bullets.forEach(b => scene.remove(b.mesh));
      particles.forEach(p => scene.remove(p.mesh));
      enemies = [];
      bullets = [];
      particles = [];

      document.getElementById('game-over-screen').classList.remove('active');
      isGameOver = false;
      updateHUD();
      animate();
    }

    // Start Game
    animate();
  </script>
</body>
</html>
