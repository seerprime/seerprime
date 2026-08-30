<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>particle universe</title>
<style>
  html, body { margin: 0; padding: 0; overflow: hidden; background: #000; height: 100%; }
  canvas { display: block; }
  #hint {
    position: fixed; bottom: 16px; left: 50%; transform: translateX(-50%);
    color: rgba(255,255,255,0.35); font-family: monospace; font-size: 11px;
    letter-spacing: 2px; text-transform: uppercase; pointer-events: none;
  }
</style>
</head>
<body>
<div id="hint">drag to rotate · scroll to zoom</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 0.1, 4000);
camera.position.set(0, 90, 420);
camera.lookAt(0,0,0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setClearColor(0x000000, 1);
document.body.appendChild(renderer.domElement);

// ---------- soft glow sprite texture ----------
function makeGlowTexture(inner, outer, size) {
  size = size || 128;
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = size;
  const ctx = canvas.getContext('2d');
  const g = ctx.createRadialGradient(size/2, size/2, 0, size/2, size/2, size/2);
  g.addColorStop(0, inner);
  g.addColorStop(0.4, outer);
  g.addColorStop(1, 'rgba(0,0,0,0)');
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, size, size);
  return new THREE.CanvasTexture(canvas);
}

const starTex = makeGlowTexture('rgba(255,255,255,1)', 'rgba(255,255,255,0.35)');
const coreTex = makeGlowTexture('rgba(255,255,255,1)', 'rgba(255,150,70,0.55)');

const universe = new THREE.Group();
scene.add(universe);

// ---------- distant starfield: whole-universe backdrop ----------
function makeShell(count, rMin, rMax, size, color, opacity) {
  const positions = new Float32Array(count * 3);
  for (let i = 0; i < count; i++) {
    const r = rMin + Math.random() * (rMax - rMin);
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos(2 * Math.random() - 1);
    positions[i*3]   = r * Math.sin(phi) * Math.cos(theta);
    positions[i*3+1] = r * Math.sin(phi) * Math.sin(theta);
    positions[i*3+2] = r * Math.cos(phi);
  }
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const mat = new THREE.PointsMaterial({
    size: size, map: starTex, transparent: true, opacity: opacity,
    color: color, blending: THREE.AdditiveBlending, depthWrite: false, sizeAttenuation: true
  });
  return new THREE.Points(geo, mat);
}

const farStars = makeShell(7000, 900, 1700, 3.2, 0xffffff, 0.5);
const midStars = makeShell(4000, 500, 950, 2.4, 0xbfd4ff, 0.55);
scene.add(farStars, midStars);

// ---------- spiral galaxy of particles ----------
function makeGalaxy(count) {
  const positions = new Float32Array(count * 3);
  const colors = new Float32Array(count * 3);
  const arms = 4;
  const colorCore = new THREE.Color(0xffd39a);
  const colorMid = new THREE.Color(0xff8a4c);
  const colorOuter = new THREE.Color(0x4f7fe0);
  const tmp = new THREE.Color();

  for (let i = 0; i < count; i++) {
    const armIndex = i % arms;
    const t = Math.pow(Math.random(), 1.7);
    const radius = 18 + t * 260;
    const spin = radius * 0.02;
    const angle = (armIndex / arms) * Math.PI * 2 + spin;

    const spread = (1 - t) * 16 + 3;
    const rx = (Math.random() - 0.5) * spread;
    const ry = (Math.random() - 0.5) * spread * 0.5;
    const rz = (Math.random() - 0.5) * spread;

    const x = Math.cos(angle) * radius + rx;
    const y = ry;
    const z = Math.sin(angle) * radius + rz;

    positions[i*3] = x;
    positions[i*3+1] = y;
    positions[i*3+2] = z;

    if (t < 0.25) tmp.copy(colorCore).lerp(colorMid, t / 0.25);
    else tmp.copy(colorMid).lerp(colorOuter, (t - 0.25) / 0.75);

    colors[i*3] = tmp.r;
    colors[i*3+1] = tmp.g;
    colors[i*3+2] = tmp.b;
  }

  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));

  const mat = new THREE.PointsMaterial({
    size: 3.4, map: starTex, transparent: true, opacity: 0.9,
    vertexColors: true, blending: THREE.AdditiveBlending, depthWrite: false, sizeAttenuation: true
  });

  return new THREE.Points(geo, mat);
}

const galaxy = makeGalaxy(9000);
universe.add(galaxy);

// ---------- glowing warm core (echo of the reference cell image) ----------
const coreGeo = new THREE.BufferGeometry();
const coreCount = 1400;
const corePos = new Float32Array(coreCount * 3);
for (let i = 0; i < coreCount; i++) {
  const r = Math.pow(Math.random(), 0.5) * 26;
  const theta = Math.random() * Math.PI * 2;
  const phi = Math.acos(2 * Math.random() - 1);
  corePos[i*3]   = r * Math.sin(phi) * Math.cos(theta);
  corePos[i*3+1] = r * Math.sin(phi) * Math.sin(theta) * 0.6;
  corePos[i*3+2] = r * Math.cos(phi);
}
coreGeo.setAttribute('position', new THREE.BufferAttribute(corePos, 3));
const coreMat = new THREE.PointsMaterial({
  size: 6.5, map: coreTex, transparent: true, opacity: 0.95,
  color: 0xffb066, blending: THREE.AdditiveBlending, depthWrite: false, sizeAttenuation: true
});
const core = new THREE.Points(coreGeo, coreMat);
universe.add(core);

// a single bright point light-like sprite at the very center
const centerSprite = new THREE.Sprite(new THREE.SpriteMaterial({
  map: coreTex, color: 0xffffff, transparent: true, opacity: 1,
  blending: THREE.AdditiveBlending, depthWrite: false
}));
centerSprite.scale.set(30, 30, 1);
universe.add(centerSprite);

// ---------- interaction: drag to rotate, scroll to zoom ----------
let isDragging = false;
let prevX = 0, prevY = 0;
let rotX = 0.15, rotY = 0;
let targetRotX = rotX, targetRotY = rotY;

renderer.domElement.addEventListener('pointerdown', (e) => {
  isDragging = true; prevX = e.clientX; prevY = e.clientY;
});
window.addEventListener('pointerup', () => { isDragging = false; });
window.addEventListener('pointermove', (e) => {
  if (!isDragging) return;
  const dx = e.clientX - prevX;
  const dy = e.clientY - prevY;
  prevX = e.clientX; prevY = e.clientY;
  targetRotY += dx * 0.004;
  targetRotX += dy * 0.004;
  targetRotX = Math.max(-1.2, Math.min(1.2, targetRotX));
});

let dist = 420;
renderer.domElement.addEventListener('wheel', (e) => {
  dist += e.deltaY * 0.4;
  dist = Math.max(120, Math.min(1200, dist));
}, { passive: true });

function resize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
window.addEventListener('resize', resize);

let t = 0;
function animate() {
  requestAnimationFrame(animate);
  t += 0.0016;

  rotX += (targetRotX - rotX) * 0.06;
  rotY += (targetRotY - rotY) * 0.06;

  camera.position.x = dist * Math.sin(rotY) * Math.cos(rotX);
  camera.position.y = dist * Math.sin(rotX) + 60;
  camera.position.z = dist * Math.cos(rotY) * Math.cos(rotX);
  camera.lookAt(0, 0, 0);

  universe.rotation.y = t * 0.06;
  farStars.rotation.y = t * 0.008;
  midStars.rotation.y = -t * 0.015;
  core.rotation.y = -t * 0.15;

  const pulse = 1 + Math.sin(t * 2.2) * 0.08;
  centerSprite.scale.set(30 * pulse, 30 * pulse, 1);

  renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
