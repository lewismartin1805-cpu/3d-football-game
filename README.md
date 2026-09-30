const fieldLength = 80;
const fieldWidth = 48;
const fieldHalfLength = fieldLength / 2;
const fieldHalfWidth = fieldWidth / 2;

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x7eb1c8);
scene.fog = new THREE.Fog(0x86bcd3, 40, 160);

const camera = new THREE.PerspectiveCamera(55, window.innerWidth / window.innerHeight, 0.1, 500);
camera.position.set(0, 20, 30);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const clock = new THREE.Clock();

const ambientLight = new THREE.HemisphereLight(0xdfeaff, 0x204d3f, 1.25);
scene.add(ambientLight);

const sunLight = new THREE.DirectionalLight(0xfff7dd, 1.2);
sunLight.position.set(10, 30, 10);
sunLight.castShadow = true;
sunLight.shadow.mapSize.set(2048, 2048);
sunLight.shadow.camera.left = -60;
sunLight.shadow.camera.right = 60;
sunLight.shadow.camera.top = 60;
sunLight.shadow.camera.bottom = -60;
scene.add(sunLight);

const world = new THREE.Group();
scene.add(world);

const startBtn = document.getElementById('start-btn');
const resetBtn = document.getElementById('reset-btn');
const menuScreen = document.getElementById('menu-screen');
const hud = document.getElementById('hud');
const messageEl = document.getElementById('message');
const homeScoreLabel = document.getElementById('home-score');
const awayScoreLabel = document.getElementById('away-score');
const timerLabel = document.getElementById('timer');
const actionButtons = document.querySelectorAll('.action-btn');

const gameState = {
  started: false,
  score: { home: 0, away: 0 },
  timer: 90,
  matchOver: false,
  messageTimeout: 0,
  ballOwner: null,
  lastTouch: null,
  gameTime: 0,
};

const ball = {
  mesh: null,
  velocity: new THREE.Vector3(),
  radius: 0.42,
};

const homeTeam = [];
const awayTeam = [];
const allPlayers = [];

function makePitch() {
  const grass = new THREE.Mesh(
    new THREE.BoxGeometry(fieldLength + 8, 1, fieldWidth + 8),
    new THREE.MeshStandardMaterial({ color: 0x27a455, roughness: 0.9, metalness: 0.08 })
  );
  grass.position.y = -0.5;
  grass.receiveShadow = true;
  world.add(grass);

  const pitch = new THREE.Mesh(
    new THREE.BoxGeometry(fieldLength, 0.2, fieldWidth),
    new THREE.MeshStandardMaterial({ color: 0x2bbd63, roughness: 0.96, metalness: 0.05 })
  );
  pitch.position.y = 0.05;
  pitch.receiveShadow = true;
  world.add(pitch);

  const lineMat = new THREE.LineBasicMaterial({ color: 0xffffff });

  const pitchOutline = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(-fieldHalfLength, 0.12, -fieldHalfWidth),
      new THREE.Vector3(fieldHalfLength, 0.12, -fieldHalfWidth),
      new THREE.Vector3(fieldHalfLength, 0.12, fieldHalfWidth),
      new THREE.Vector3(-fieldHalfLength, 0.12, fieldHalfWidth),
      new THREE.Vector3(-fieldHalfLength, 0.12, -fieldHalfWidth),
    ]),
    lineMat
  );
  world.add(pitchOutline);

  const centerLine = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(0, 0.14, -fieldHalfWidth),
      new THREE.Vector3(0, 0.14, fieldHalfWidth),
    ]),
    lineMat
  );
  world.add(centerLine);

  const centerCircle = new THREE.Mesh(
    new THREE.RingGeometry(6, 6.2, 48),
    new THREE.MeshBasicMaterial({ color: 0xffffff, side: THREE.DoubleSide })
  );
  centerCircle.rotation.x = -Math.PI / 2;
  centerCircle.position.y = 0.15;
  centerCircle.position.z = 0;
  world.add(centerCircle);

  const penaltyBoxGeometry = new THREE.BoxGeometry(30, 0.15, 18);
  const penaltyBox1 = new THREE.Mesh(penaltyBoxGeometry, new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.18 }));
  penaltyBox1.position.set(-fieldHalfLength + 15, 0.2, 0);
  world.add(penaltyBox1);

  const penaltyBox2 = penaltyBox1.clone();
  penaltyBox2.position.x = fieldHalfLength - 15;
  world.add(penaltyBox2);

  const goal1 = new THREE.Mesh(
    new THREE.BoxGeometry(3.6, 2.1, 0.3),
    new THREE.MeshStandardMaterial({ color: 0xf7f9ff, metalness: 0.2, roughness: 0.6 })
  );
  goal1.position.set(-fieldHalfLength - 1.8, 1.2, 0);
  world.add(goal1);

  const goal2 = goal1.clone();
  goal2.position.x = fieldHalfLength + 1.8;
  world.add(goal2);

  const netMat = new THREE.MeshStandardMaterial({ color: 0xf5efe7, transparent: true, opacity: 0.16 });
  const netFront1 = new THREE.Mesh(new THREE.BoxGeometry(0.12, 2, 6), netMat);
  netFront1.position.set(-fieldHalfLength - 0.2, 1.2, 0);
  world.add(netFront1);

  const netFront2 = netFront1.clone();
  netFront2.position.x = fieldHalfLength + 0.2;
  world.add(netFront2);

  const crowd = new THREE.Group();
  for (let i = 0; i < 220; i++) {
    const fan = new THREE.Mesh(
      new THREE.BoxGeometry(0.8, 1.2, 0.8),
      new THREE.MeshStandardMaterial({ color: new THREE.Color().setHSL(Math.random(), 0.8, 0.55) })
    );
    const angle = (i / 220) * Math.PI * 2;
    const radius = 52 + (i % 4) * 1.5;
    const x = Math.cos(angle) * radius;
    const z = Math.sin(angle) * radius;
    fan.position.set(x, 1.5 + (Math.random() * 0.7), z);
    fan.castShadow = true;
    fan.receiveShadow = true;
    crowd.add(fan);
  }
  world.add(crowd);

  const stadiumBase = new THREE.Mesh(
    new THREE.CylinderGeometry(70, 70, 12, 56),
    new THREE.MeshStandardMaterial({ color: 0x2d4050, roughness: 0.92, metalness: 0.12 })
  );
  stadiumBase.position.y = -6.8;
  world.add(stadiumBase);

  const ring = new THREE.Mesh(
    new THREE.TorusGeometry(52, 1.3, 24, 120),
    new THREE.MeshStandardMaterial({ color: 0x4f5b68, roughness: 0.75, metalness: 0.12 })
  );
  ring.rotation.x = Math.PI / 2;
  ring.position.y = 4.2;
  world.add(ring);
}

function createPlayer(teamColor, homeSide, index) {
  const player = new THREE.Group();

  const body = new THREE.Mesh(
    new THREE.CapsuleGeometry(0.45, 1.4, 5, 10),
    new THREE.MeshStandardMaterial({ color: teamColor, roughness: 0.7, metalness: 0.1 })
  );
  body.castShadow = true;
  body.position.y = 1.1;
  player.add(body);

  const head = new THREE.Mesh(
    new THREE.SphereGeometry(0.35, 20, 20),
    new THREE.MeshStandardMaterial({ color: 0xf1d2a5, roughness: 1 })
  );
  head.position.y = 2.15;
  head.castShadow = true;
  player.add(head);

  const jersey = new THREE.Mesh(
    new THREE.BoxGeometry(0.75, 0.8, 0.3),
    new THREE.MeshStandardMaterial({ color: teamColor, roughness: 0.8 })
  );
  jersey.position.y = 1.35;
  jersey.castShadow = true;
  player.add(jersey);

  player.position.y = 0.15;
  world.add(player);

  return {
    group: player,
    color: teamColor,
    homeSide,
    index,
    speed: homeSide ? 7.5 : 6.8,
    target: new THREE.Vector3(),
    mesh: body,
    isControlled: homeSide,
    maxSprint: homeSide ? 9.2 : 8.8,
  };
}

function setupFormation() {
  const homePositions = [
    { x: -16, z: 18 },
    { x: -8, z: 10 },
    { x: 0, z: 18 },
    { x: 10, z: 10 },
    { x: 18, z: 18 },
    { x: -14, z: 0 },
    { x: -4, z: 0 },
    { x: 4, z: 0 },
    { x: 14, z: 0 },
    { x: -8, z: -12 },
    { x: 8, z: -12 },
  ];

  const awayPositions = [
    { x: -16, z: -18 },
    { x: -8, z: -10 },
    { x: 0, z: -18 },
    { x: 10, z: -10 },
    { x: 18, z: -18 },
    { x: -14, z: 0 },
    { x: -4, z: 0 },
    { x: 4, z: 0 },
    { x: 14, z: 0 },
    { x: -8, z: 12 },
    { x: 8, z: 12 },
  ];

  for (let i = 0; i < 11; i++) {
    const homePlayer = createPlayer(0x3dd9c4, true, i);
    homePlayer.group.position.set(homePositions[i].x, 0, homePositions[i].z);
    homePlayer.target.copy(homePlayer.group.position);
    homeTeam.push(homePlayer);
    allPlayers.push(homePlayer);

    const awayPlayer = createPlayer(0xff6b7d, false, i);
    awayPlayer.group.position.set(awayPositions[i].x, 0, awayPositions[i].z);
    awayPlayer.target.copy(awayPlayer.group.position);
    awayTeam.push(awayPlayer);
    allPlayers.push(awayPlayer);
  }
}

function createBall() {
  const geom = new THREE.SphereGeometry(0.42, 24, 24);
  const mat = new THREE.MeshStandardMaterial({ color: 0xf8f8f8, roughness: 0.85, metalness: 0.12 });
  const sphere = new THREE.Mesh(geom, mat);
  sphere.castShadow = true;
  sphere.receiveShadow = true;
  sphere.position.set(0, 0.42, 0);
  world.add(sphere);
  ball.mesh = sphere;
}

function showMessage(text, duration = 1.2) {
  messageEl.textContent = text;
  messageEl.classList.remove('hidden');
  gameState.messageTimeout = duration;
}

function setHud() {
  homeScoreLabel.textContent = String(gameState.score.home);
  awayScoreLabel.textContent = String(gameState.score.away);
  const secs = Math.max(0, gameState.timer);
  const minutes = Math.floor(secs / 60);
  const seconds = secs % 60;
  timerLabel.textContent = `${minutes}:${String(seconds).padStart(2, '0')}`;
}

function resetMatch() {
  gameState.score.home = 0;
  gameState.score.away = 0;
  gameState.timer = 90;
  gameState.matchOver = false;
  gameState.started = false;
  gameState.ballOwner = null;
  gameState.lastTouch = null;
  gameState.gameTime = 0;

  for (let i = 0; i < homeTeam.length; i++) {
    const homePos = { x: [ -16, -8, 0, 10, 18, -14, -4, 4, 14, -8, 8 ][i], z: [ 18, 10, 18, 10, 18, 0, 0, 0, 0, -12, -12 ][i] };
    const awayPos = { x: [ -16, -8, 0, 10, 18, -14, -4, 4, 14, -8, 8 ][i], z: [ -18, -10, -18, -10, -18, 0, 0, 0, 0, 12, 12 ][i] };

    homeTeam[i].group.position.set(homePos.x, 0, homePos.z);
    awayTeam[i].group.position.set(awayPos.x, 0, awayPos.z);
  }

  ball.mesh.position.set(0, 0.42, 0);
  ball.velocity.set(0, 0, 0);
  messageEl.classList.add('hidden');
  menuScreen.classList.remove('hidden');
  hud.classList.add('hidden');
  setHud();
}

function startMatch() {
  menuScreen.classList.add('hidden');
  hud.classList.remove('hidden');
  gameState.started = true;
  gameState.timer = 90;
  gameState.matchOver = false;
  ball.mesh.position.set(0, 0.42, 0);
  ball.velocity.set(0, 0, 0);
  gameState.ballOwner = homeTeam[0];
  gameState.lastTouch = homeTeam[0];
  showMessage('Kick-off!', 1.5);
  setHud();
}

function determineClosestPlayerToBall(team) {
  let best = null;
  let bestDist = Infinity;
  for (const p of team) {
    const dist = p.group.position.distanceTo(ball.mesh.position);
    if (dist < bestDist) {
      bestDist = dist;
      best = p;
    }
  }
  return best;
}

function getNearestOpponent(player) {
  const targetTeam = player.homeSide ? awayTeam : homeTeam;
  let nearest = null;
  let nearestDist = Infinity;
  for (const p of targetTeam) {
    const dist = p.group.position.distanceTo(player.group.position);
    if (dist < nearestDist) {
      nearestDist = dist;
      nearest = p;
    }
  }
  return nearest;
}

function updateBallPhysics(delta) {
  if (!ball.mesh) return;

  if (gameState.ballOwner) {
    const owner = gameState.ballOwner;
    const offset = new THREE.Vector3(0.6, 0.34, 0.4);
    if (owner.homeSide) {
      offset.x = 0.65;
      offset.z = -0.2;
    } else {
      offset.x = -0.65;
      offset.z = 0.2;
    }

    const ownerPos = owner.group.position.clone();
    const targetBallPos = ownerPos.clone().add(offset);
    ball.mesh.position.lerp(targetBallPos, 0.25);
    ball.velocity.set(0, 0, 0);
    return;
  }

  ball.velocity.y -= 17 * delta;
  const nextPos = ball.mesh.position.clone().addScaledVector(ball.velocity, delta);

  if (Math.abs(nextPos.x) > fieldHalfLength + 5) {
    ball.velocity.x *= -0.75;
    nextPos.x = THREE.MathUtils.clamp(nextPos.x, -fieldHalfLength - 5, fieldHalfLength + 5);
  }

  if (Math.abs(nextPos.z) > fieldHalfWidth + 5) {
    ball.velocity.z *= -0.75;
    nextPos.z = THREE.MathUtils.clamp(nextPos.z, -fieldHalfWidth - 5, fieldHalfWidth + 5);
  }

  if (nextPos.y < 0.42) {
    nextPos.y = 0.42;
    ball.velocity.y *= -0.55;
    ball.velocity.x *= 0.97;
    ball.velocity.z *= 0.97;
  }

  ball.mesh.position.copy(nextPos);

  for (const p of allPlayers) {
    const playerPos = p.group.position;
    const dist = playerPos.distanceTo(ball.mesh.position);
    if (dist < 1.2 && ball.velocity.lengthSq() < 60) {
      gameState.ballOwner = p;
      gameState.lastTouch = p;
      ball.velocity.set(0, 0, 0);
      break;
    }
  }

  if (ball.mesh.position.z > fieldHalfWidth + 4 && Math.abs(ball.mesh.position.x) < 7) {
    gameState.score.home += 1;
    resetAfterGoal('Home scores!');
  }

  if (ball.mesh.position.z < -fieldHalfWidth - 4 && Math.abs(ball.mesh.position.x) < 7) {
    gameState.score.away += 1;
    resetAfterGoal('Away scores!');
  }
}

function resetAfterGoal(text) {
  if (gameState.matchOver) return;
  gameState.matchOver = true;
  showMessage(text, 2.6);
  setTimeout(() => {
    gameState.matchOver = false;
    gameState.ballOwner = homeTeam[0];
    ball.mesh.position.set(0, 0.42, 0);
    ball.velocity.set(0, 0, 0);
    setHud();
  }, 1800);
}

function movePlayer(player, target, delta) {
  const toTarget = target.clone().sub(player.group.position);
  const distance = toTarget.length();
  if (distance > 0.01) {
    const dir = toTarget.normalize();
    const step = Math.min(distance, player.speed * delta);
    player.group.position.addScaledVector(dir, step);
  }
}

function updateHomeAI(delta) {
  if (!gameState.started || gameState.matchOver) return;

  const possession = gameState.ballOwner;

  for (let i = 0; i < homeTeam.length; i++) {
    const player = homeTeam[i];
    if (possession && possession === player) {
      player.group.position.x += 0.01;
      continue;
    }

    let target = new THREE.Vector3();
    if (possession) {
      target.copy(possession.group.position);
      if (player.homeSide && player !== possession) {
        const offsetX = (i % 2 === 0 ? -1.2 : 1.2) * 2;
        const offsetZ = -8 + (i * 0.9);
        target.x += offsetX;
        target.z += offsetZ;
      }
    } else {
      target.copy(ball.mesh.position);
      if (player.group.position.distanceTo(ball.mesh.position) < 5) {
        target.x += (Math.random() - 0.5) * 3;
        target.z += (Math.random() - 0.5) * 3;
      }
    }

    if (target.x > fieldHalfLength - 3) target.x = fieldHalfLength - 3;
    if (target.x < -fieldHalfLength + 3) target.x = -fieldHalfLength + 3;
    if (target.z > fieldHalfWidth - 2) target.z = fieldHalfWidth - 2;
    if (target.z < -fieldHalfWidth + 2) target.z = -fieldHalfWidth + 2;

    movePlayer(player, target, delta);
  }
}

function updateAwayAI(delta) {
  if (!gameState.started || gameState.matchOver) return;

  for (let i = 0; i < awayTeam.length; i++) {
    const player = awayTeam[i];
    if (gameState.ballOwner === player) {
      const target = new THREE.Vector3(0, 0, -fieldHalfWidth + 4);
      movePlayer(player, target, delta);
      continue;
    }

    let target = new THREE.Vector3();
    if (gameState.ballOwner) {
      target.copy(gameState.ballOwner.group.position);
      target.z *= -0.9;
      target.x *= 0.9;
      if (gameState.ballOwner.homeSide) {
        target.z += 5;
      }
    } else {
      target.copy(ball.mesh.position);
      target.z *= -1;
    }

    if (target.x > fieldHalfLength - 3) target.x = fieldHalfLength - 3;
    if (target.x < -fieldHalfLength + 3) target.x = -fieldHalfLength + 3;
    if (target.z > fieldHalfWidth - 2) target.z = fieldHalfWidth - 2;
    if (target.z < -fieldHalfWidth + 2) target.z = -fieldHalfWidth + 2;

    movePlayer(player, target, delta);
  }
}

function triggerAction(action) {
  if (!gameState.started || gameState.matchOver) return;

  const owner = gameState.ballOwner;
  if (!owner || owner.homeSide === false) {
    showMessage('Press to regain possession', 1.1);
    return;
  }

  const attackDirection = new THREE.Vector3(0, 0.2, 1);
  const target = new THREE.Vector3();

  switch (action) {
    case 'pass': {
      const teammates = homeTeam.filter((p) => p !== owner);
      const receiver = teammates.reduce((best, p) => {
        const currDist = p.group.position.distanceTo(owner.group.position);
        const bestDist = best ? best.group.position.distanceTo(owner.group.position) : Infinity;
        return currDist < bestDist ? p : best;
      }, null) || homeTeam[0];
      target.copy(receiver.group.position).add(new THREE.Vector3(0, 0.5, 0));
      const dir = target.clone().sub(owner.group.position).normalize();
      ball.velocity.copy(dir.multiplyScalar(22));
      ball.velocity.y = 3.8;
      gameState.ballOwner = null;
      gameState.lastTouch = owner;
      showMessage('Pass played', 1);
      break;
    }

    case 'shoot': {
      const shotTarget = new THREE.Vector3(0, 0.4, fieldHalfWidth + 12);
      const dir = shotTarget.clone().sub(ball.mesh.position).normalize();
      ball.velocity.copy(dir.multiplyScalar(28));
      ball.velocity.y = 7.2;
      gameState.ballOwner = null;
      gameState.lastTouch = owner;
      showMessage('Shot taken!', 1);
      break;
    }

    case 'cross': {
      const targetX = owner.group.position.x + (Math.random() - 0.5) * 12;
      const crossTarget = new THREE.Vector3(targetX, 0.4, fieldHalfWidth + 3);
      const dir = crossTarget.clone().sub(ball.mesh.position).normalize();
      ball.velocity.copy(dir.multiplyScalar(19));
      ball.velocity.y = 8.5;
      gameState.ballOwner = null;
      gameState.lastTouch = owner;
      showMessage('Cross delivered', 1);
      break;
    }

    case 'tackle': {
      const nearest = getNearestOpponent(owner);
      if (nearest && nearest.group.position.distanceTo(owner.group.position) < 2.1) {
        const tackleDir = nearest.group.position.clone().sub(owner.group.position).normalize();
        ball.velocity.copy(tackleDir.multiplyScalar(16));
        ball.velocity.y = 3.2;
        gameState.ballOwner = null;
        gameState.lastTouch = owner;
        showMessage('Tackle won the ball', 1.2);
      } else {
        showMessage('No challenge nearby', 1.1);
      }
      break;
    }

    default:
      break;
  }
}

function animate() {
  requestAnimationFrame(animate);

  const delta = Math.min(clock.getDelta(), 0.032);

  if (gameState.started && !gameState.matchOver) {
    if (gameState.messageTimeout > 0) {
      gameState.messageTimeout -= delta;
      if (gameState.messageTimeout <= 0) {
        messageEl.classList.add('hidden');
      }
    }

    gameState.timer = Math.max(0, gameState.timer - delta);
    gameState.gameTime += delta;

    if (gameState.timer <= 0) {
      gameState.started = false;
      showMessage('Full time', 2.5);
      return;
    }

    updateHomeAI(delta);
    updateAwayAI(delta);
    updateBallPhysics(delta);

    if (gameState.ballOwner && gameState.ballOwner.homeSide === false) {
      const target = new THREE.Vector3(0, 0, -fieldHalfWidth + 2);
      movePlayer(gameState.ballOwner, target, delta);
    }

    if (gameState.ballOwner && gameState.ballOwner.homeSide === true) {
      const owner = gameState.ballOwner;
      const defendPos = owner.group.position.clone();
      defendPos.x = THREE.MathUtils.clamp(defendPos.x, -fieldHalfLength + 2, fieldHalfLength - 2);
      defendPos.z = THREE.MathUtils.clamp(defendPos.z, -fieldHalfWidth + 2, fieldHalfWidth - 2);
    }
  }

  camera.position.lerp(new THREE.Vector3(0, 22, 30), 0.05);
  camera.lookAt(0, 0, 0);
  renderer.render(scene, camera);
  setHud();
}

startBtn.addEventListener('click', startMatch);
resetBtn.addEventListener('click', resetMatch);
for (const btn of actionButtons) {
  btn.addEventListener('click', () => triggerAction(btn.dataset.action));
}

makePitch();
setupFormation();
createBall();
resetMatch();
animate();
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
