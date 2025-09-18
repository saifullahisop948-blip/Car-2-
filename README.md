<!doctype html>
<html lang="hi">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>3D RC Car Endless Runner</title>
<style>
  html,body{height:100%;margin:0;background:#7fc9ff;overflow:hidden;font-family:Inter,system-ui,Arial}
  #container{width:100%;height:100%;position:relative}
  #uiTop{position:absolute;left:12px;top:12px;color:#012639;font-weight:700;background:rgba(255,255,255,0.85);padding:8px 12px;border-radius:10px}
  #uiRight{position:absolute;right:12px;top:12px;color:#012639;font-weight:700;background:rgba(255,255,255,0.85);padding:8px 12px;border-radius:10px}
  #controls{position:absolute;left:50%;transform:translateX(-50%);bottom:18px;display:flex;gap:10px}
  .btn{width:84px;height:56px;border-radius:12px;background:linear-gradient(#fff,#e7e7e7);display:flex;align-items:center;justify-content:center;font-weight:800;user-select:none;cursor:pointer;box-shadow:0 6px 12px rgba(0,0,0,0.18)}
  #loseOverlay{position:absolute;inset:0;display:none;align-items:center;justify-content:center;background:rgba(2,6,23,0.6);color:#fff;flex-direction:column;gap:14px;z-index:30}
  #loseOverlay .title{font-size:36px;font-weight:800;color:#ff6b6b}
  #loseOverlay button{padding:10px 16px;border-radius:10px;border:0;background:#ff5a5a;color:#fff;font-weight:700;cursor:pointer}
  @media(max-width:520px){ .btn{width:64px;height:48px;font-size:14px} #uiTop,#uiRight{font-size:13px} .title{font-size:28px} }
</style>
</head>
<body>
<div id="container"></div>

<div id="uiTop">Score: <span id="score">0</span></div>
<div id="uiRight">Speed: <span id="spd">6</span></div>

<div id="controls">
  <div class="btn" id="leftBtn">◀ Left</div>
  <div class="btn" id="rightBtn">Right ▶</div>
</div>

<div id="loseOverlay" role="dialog" aria-modal="true">
  <div class="title">YOU LOSE</div>
  <div id="loseScore">Score: 0</div>
  <button id="restartBtn">Restart</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.min.js"></script>
<script>
/*
3D RC car endless runner (no external 3D model).
Features:
- Stylized RC car built from primitives
- 3D road that loops infinitely
- Occasional single obstacles spawn (rocks)
- Left/right controls via buttons, keyboard, touch
- Collision => "YOU LOSE" overlay + restart
- Obstacles recycled to create loop
*/

const container = document.getElementById('container');
const loseOverlay = document.getElementById('loseOverlay');
const loseScore = document.getElementById('loseScore');
const scoreEl = document.getElementById('score');
const spdEl = document.getElementById('spd');
const restartBtn = document.getElementById('restartBtn');
const leftBtn = document.getElementById('leftBtn');
const rightBtn = document.getElementById('rightBtn');

let scene, camera, renderer, clock;
let car, wheels = [];
let roadSegments = [];
let obstacles = [];
let params = {
  speed: 6,
  maxX: 3.2,
  obstacleFreq: 1/1.6, // avg seconds between single obstacle spawns (tweak)
  spawnProbPerSec: 0.18, // fallback prob per second
};
let running = true;
let score = 0;

// init three
function init(){
  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x9fdfff, 0.015);

  camera = new THREE.PerspectiveCamera(60, innerWidth/innerHeight, 0.1, 200);
  camera.position.set(0, 5.4, 9.6);
  camera.lookAt(0,0,0);

  renderer = new THREE.WebGLRenderer({antialias:true});
  renderer.setPixelRatio(Math.min(2, devicePixelRatio || 1));
  renderer.setSize(innerWidth, innerHeight);
  container.appendChild(renderer.domElement);

  clock = new THREE.Clock();

  // lights
  const hemi = new THREE.HemisphereLight(0xffffff, 0x404060, 0.9);
  scene.add(hemi);
  const dir = new THREE.DirectionalLight(0xffffff, 0.9);
  dir.position.set(5,10,7);
  scene.add(dir);

  // sky gradient via large sphere
  const skyGeo = new THREE.SphereGeometry(120, 24, 12);
  const skyMat = new THREE.MeshBasicMaterial({color:0x9fdfff, side:THREE.BackSide});
  const sky = new THREE.Mesh(skyGeo, skyMat);
  scene.add(sky);

  // create infinite road using repeated segments
  createRoadSegments();

  // create RC car from primitives
  car = createRCCar();
  car.position.set(0, 0.35, 4.0);
  scene.add(car);

  // obstacles pool (reused)
  for(let i=0;i<8;i++){
    obstacles.push(createRock());
  }

  window.addEventListener('resize', onResize);
  animate();
}

// Helpers
function onResize(){
  camera.aspect = innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
}

function createRoadSegments(){
  const segCount = 6;
  const segLen = 12;
  const roadWidth = 7.2;
  for(let i=0;i<segCount;i++){
    const g = new THREE.PlaneGeometry(roadWidth, segLen, 1, 1);
    const mat = new THREE.MeshPhongMaterial({color:0x333333});
    const seg = new THREE.Mesh(g, mat);
    seg.rotation.x = -Math.PI/2;
    seg.position.z = -i * segLen + 10;
    seg.receiveShadow = true;
    // add dashed center line (texture-like)
    const centerGeo = new THREE.PlaneGeometry(0.6, segLen,1,1);
    const centerMat = new THREE.MeshBasicMaterial({color:0xffffff});
    const center = new THREE.Mesh(centerGeo, centerMat);
    center.rotation.x = -Math.PI/2;
    center.position.set(0, 0.01, seg.position.z);
    scene.add(center);
    roadSegments.push({seg, center, len:segLen});
    scene.add(seg);

    // side grass
    const grassLeft = new THREE.Mesh(new THREE.PlaneGeometry(6, segLen), new THREE.MeshBasicMaterial({color:0x1b6b3a}));
    grassLeft.rotation.x = -Math.PI/2; grassLeft.position.set(-roadWidth/2 - 3, 0.0, seg.position.z);
    scene.add(grassLeft);
    const grassRight = grassLeft.clone(); grassRight.position.x = roadWidth/2 + 3; scene.add(grassRight);
  }
}

// Stylized RC car primitive model
function createRCCar(){
  const group = new THREE.Group();

  // main body
  const bodyGeo = new THREE.BoxGeometry(1.6, 0.36, 2.2);
  const bodyMat = new THREE.MeshPhongMaterial({color:0xff3b3b, shininess:60});
  const body = new THREE.Mesh(bodyGeo, bodyMat);
  body.position.y = 0.35;
  body.castShadow = true;
  group.add(body);

  // cockpit / window
  const winGeo = new THREE.BoxGeometry(0.9, 0.22, 1.0);
  const winMat = new THREE.MeshPhongMaterial({color:0xbfe9ff, transparent:true, opacity:0.95});
  const win = new THREE.Mesh(winGeo, winMat); win.position.set(0, 0.52, 0.12);
  group.add(win);

  // roof stripe
  const stripeGeo = new THREE.BoxGeometry(0.18, 0.02, 1.6);
  const stripe = new THREE.Mesh(stripeGeo, new THREE.MeshPhongMaterial({color:0x222}));
  stripe.position.set(0, 0.70, 0.0);
  group.add(stripe);

  // wheels
  const wheelGeo = new THREE.CylinderGeometry(0.22, 0.22, 0.14, 16);
  const wheelMat = new THREE.MeshPhongMaterial({color:0x111});
  const offsets = [
    [-0.6,0.14,0.9],
    [0.6,0.14,0.9],
    [-0.6,0.14,-0.85],
    [0.6,0.14,-0.85],
  ];
  offsets.forEach(o=>{
    const w = new THREE.Mesh(wheelGeo, wheelMat);
    w.rotation.z = Math.PI/2;
    w.position.set(o[0], o[1], o[2]);
    w.castShadow = true;
    group.add(w);
    wheels.push(w);
  });

  // small antenna to signal RC
  const antGeo = new THREE.CylinderGeometry(0.02,0.02,0.9,8);
  const ant = new THREE.Mesh(antGeo, new THREE.MeshPhongMaterial({color:0x111}));
  ant.position.set(0.7, 0.9, 0.9);
  group.add(ant);

  return group;
}

// rock obstacle
function createRock(){
  const geo = new THREE.IcosahedronGeometry(0.4 + Math.random()*0.4, 0);
  const colorOptions = [0xc47f4b,0xa45f3a,0x7f6f6f,0xe0b86b,0x9bc24b];
  const mat = new THREE.MeshPhongMaterial({color: colorOptions[Math.floor(Math.random()*colorOptions.length)], flatShading:true});
  const rock = new THREE.Mesh(geo, mat);
  rock.castShadow = false;
  rock.visible = false;
  scene.add(rock);
  return rock;
}

// spawn single obstacle ahead at random x within road
let spawnAccumulator = 0;
function maybeSpawn(delta){
  spawnAccumulator += delta;
  // spawn chance lower: we prefer occasional single rocks
  const spawnEvery = 1.0 / params.obstacleFreq; // seconds average
  if(spawnAccumulator > 0.6){ // evaluate every 0.6s
    spawnAccumulator = 0;
    if(Math.random() < params.spawnProbPerSec * 0.6){
      spawnRockOnce();
    }
  }
}

function spawnRockOnce(){
  // find an inactive rock from pool
  const rock = obstacles.find(r=>!r.visible);
  if(!rock) return;
  rock.scale.setScalar(1 + Math.random()*0.6);
  const laneWidth = 7.2;
  const x = (Math.random()* (laneWidth - 1.2)) - (laneWidth/2 - 0.6);
  rock.position.set(x, 0.42, -60); // spawn far ahead
  rock.visible = true;
  rock.userData.speed = 1 + Math.random()*0.6;
}

// Recycle or move rocks
function updateRocks(delta, moveZ){
  for(const r of obstacles){
    if(!r.visible) continue;
    r.position.z += moveZ * r.userData.speed;
    // small rotation
    r.rotation.x += 0.01 + Math.random()*0.02;
    r.rotation.y += 0.005 + Math.random()*0.01;
    if(r.position.z > 30){
      r.visible = false;
      score += 10;
      scoreEl.textContent = score;
    }
  }
}

// road loop update
function updateRoad(moveZ){
  for(const item of roadSegments){
    item.seg.position.z += moveZ;
    item.center.position.z += moveZ;
    if(item.seg.position.z > 20){
      // move this segment far back to create infinite loop
      item.seg.position.z -= item.len * roadSegments.length;
      item.center.position.z = item.seg.position.z;
    }
  }
}

// collision sphere-vs-car-box approx
function checkCollision(){
  // car bounding box approx
  const carBox = new THREE.Box3().setFromObject(car);
  for(const r of obstacles){
    if(!r.visible) continue;
    // simple distance check between rock center and carBox
    const rockPos = r.position;
    // clamp rock x inside carBox x-range
    const closestX = Math.max(carBox.min.x, Math.min(rockPos.x, carBox.max.x));
    const closestY = Math.max(carBox.min.y, Math.min(rockPos.y, carBox.max.y));
    const closestZ = Math.max(carBox.min.z, Math.min(rockPos.z, carBox.max.z));
    const dx = rockPos.x - closestX;
    const dy = rockPos.y - closestY;
    const dz = rockPos.z - closestZ;
    const dist2 = dx*dx + dy*dy + dz*dz;
    const rRadius = (r.geometry.boundingSphere ? r.geometry.boundingSphere.radius : 0.5) * r.scale.x;
    if(dist2 < (rRadius + 0.18)*(rRadius + 0.18)){
      return true;
    }
  }
  return false;
}

// Controls
let leftPressed = false, rightPressed = false;
leftBtn.addEventListener('pointerdown', ()=>leftPressed = true);
leftBtn.addEventListener('pointerup', ()=>leftPressed = false);
leftBtn.addEventListener('pointercancel', ()=>leftPressed = false);
rightBtn.addEventListener('pointerdown', ()=>rightPressed = true);
rightBtn.addEventListener('pointerup', ()=>rightPressed = false);
rightBtn.addEventListener('pointercancel', ()=>rightPressed = false);

window.addEventListener('keydown', e=>{
  if(e.key === 'ArrowLeft' || e.key === 'a') leftPressed = true;
  if(e.key === 'ArrowRight' || e.key === 'd') rightPressed = true;
});
window.addEventListener('keyup', e=>{
  if(e.key === 'ArrowLeft' || e.key === 'a') leftPressed = false;
  if(e.key === 'ArrowRight' || e.key === 'd') rightPressed = false;
});

// touch drag steering
let touchX = null;
renderer && renderer.domElement && renderer.domElement.addEventListener('touchstart', (ev)=>{
  touchX = ev.touches[0].clientX;
},{passive:true});
renderer && renderer.domElement && renderer.domElement.addEventListener('touchmove', (ev)=>{
  if(touchX === null) return;
  const dx = ev.touches[0].clientX - touchX;
  car.position.x += dx * 0.008 * (innerWidth/800);
  touchX = ev.touches[0].clientX;
},{passive:true});
renderer && renderer.domElement && renderer.domElement.addEventListener('touchend', ()=>{ touchX = null; });

// game over
function lose(){
  running = false;
  loseScore.textContent = 'Score: ' + score;
  loseOverlay.style.display = 'flex';
}

// restart
restartBtn.addEventListener('click', ()=>{
  // reset state
  for(const r of obstacles) r.visible = false;
  score = 0; scoreEl.textContent = 0;
  car.position.set(0, 0.35, 4.0);
  running = true;
  loseOverlay.style.display = 'none';
  clock.start();
});

// main loop
function animate(){
  requestAnimationFrame(animate);
  if(!renderer) return;
  const delta = clock.getDelta();
  if(!running) { renderer.render(scene, camera); return; }

  // Move speed scaled by delta
  const moveZ = params.speed * delta * 10 * 0.6; // tuning for pleasant speed
  spdEl.textContent = Math.round(params.speed);

  // Input steering
  if(leftPressed) car.position.x -= 4.0 * delta;
  if(rightPressed) car.position.x += 4.0 * delta;
  car.position.x = Math.max(-params.maxX, Math.min(params.maxX, car.position.x));

  // small tilt effect
  car.rotation.z = THREE.MathUtils.lerp(car.rotation.z, - (car.position.x)*0.04, 0.12);

  // animate wheels rotate
  wheels.forEach(w=> w.rotation.x -= moveZ*0.8 );

  // update road
  updateRoad(moveZ);

  // spawn obstacles occasionally (one by one)
  maybeSpawn(delta);

  // move rocks towards camera (simulate car forward)
  updateRocks(delta, moveZ);

  // check collision
  if(checkCollision()){
    lose();
  }

  // camera follows slightly for parallax
  const camTargetX = THREE.MathUtils.lerp(camera.position.x, car.position.x*0.24, 0.06);
  camera.position.x = camTargetX;
  camera.lookAt(car.position.x*0.18, 0.6, 0);

  renderer.render(scene, camera);
}

// initialize bounding sphere for rocks (used in collision)
function prepareRocksBounding(){
  for(const r of obstacles){
    r.geometry.computeBoundingSphere();
  }
}

init();
prepareRocksBounding();

</script>
</body>
</html># Car-2-
A car game with obstacles 
