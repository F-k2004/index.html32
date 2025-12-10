<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>🌀 کریستال — انفجار 3D واقعی + نور حجمی</title>
<style>
  :root { color-scheme: dark; }
  html,body { height:100%; margin:0; background:#000011; overflow:hidden; font-family:system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial; }
  #glCanvas { width:100vw; height:100vh; display:block; }
  .ui { position: absolute; right: 18px; top: 18px; background: rgba(255,255,255,0.04); padding:12px; border-radius:10px; backdrop-filter: blur(6px); color:#cfeffd; box-shadow:0 6px 20px rgba(0,0,0,0.6); }
  .i h3{ margin:0 0 6px 0; color:#9be8ff; font-size:15px; }
  .ui button{ background:linear-gradient(90deg,#7df9ff,#9b6bff); border:none; padding:8px 12px; border-radius:8px; color:#001219; font-weight:700; cursor:pointer; margin-right:6px; }
  .ui label { font-size:13px; display:block; margin-top:8px; color:#bfeffb; }
  .ui input[type=range]{ width:170px; }
  .footer { position:absolute; left:18px; bottom:18px; color:#a7dff6; font-size:13px; background:rgba(255,255,255,0.03); padding:8px 12px; border-radius:8px; }
</style>
</head>
<body>
<canvas id="glCanvas"></canvas>

<div class="ui" role="region" aria-label="کنترل‌ها">
  <h3>🌀 کریستال 3D — انفجار رنگین‌کمانی</h3>
  <div style="margin-top:8px;">
    <button id="explodeBtn">💥 انفجار</button>
    <button id="resetBtn">♻️ بازنشانی</button>
  </div>
  <label>قدرت انفجار: <span id="pwrText">1.0</span></label>
  <input id="power" type="range" min="0" max="2.0" step="0.01" value="1.0">
  <label>شدت نور حجمی: <span id="volText">0.6</span></label>
  <input id="vol" type="range" min="0" max="1.4" step="0.01" value="0.6">
  <label>سرعت پالس انرژی: <span id="spdText">1.0</span></label>
  <input id="speed" type="range" min="0.2" max="3.0" step="0.05" value="1.0">
</div>

<div class="footer">Rendered with WebGL raymarching — کلیک کن، صبر کن، از انفجار لذت ببر ✨</div>

<script id="vs" type="x-shader/x-vertex">
attribute vec2 aPos;
varying vec2 vUv;
void main(){
  vUv = aPos * 0.5 + 0.5;
  gl_Position = vec4(aPos, 0.0, 1.0);
}
</script>

<script id="fs" type="x-shader/x-fragment">
precision highp float;
uniform vec2 uRes;
uniform float uTime;
uniform float uExplode;     // 0..1
uniform float uPower;       // explosion power
uniform float uVolum;       // volumetric light intensity
uniform float uSpeed;       // speed multiplier
varying vec2 vUv;

// ----------------------- utils -----------------------
mat3 rotX(float a){ float s=sin(a), c=cos(a); return mat3(1.,0.,0., 0.,c,-s, 0.,s,c); }
mat3 rotY(float a){ float s=sin(a), c=cos(a); return mat3(c,0.,s, 0.,1.,0., -s,0.,c); }

float hash21(vec2 p){ p = fract(p*vec2(123.34, 456.21)); p += dot(p, p+45.32); return fract(p.x*p.y); }

float sdOcta(vec3 p){ p = abs(p); return (p.x + p.y + p.z - 1.0) * 0.57735027; } // octa-like

// combine several "facets" to form crystal-like SDF
float crystalSDF(vec3 p){
  // multiple scaled octahedrons rotated to make a faceted crystal
  float d = sdOcta(p);
  d = min(d, sdOcta(p * vec3(1.0, 0.9, 1.1) + vec3(0.0,0.2,0.0)));
  d = min(d, sdOcta(p * vec3(1.0,1.2,0.85) - vec3(0.0,0.15,0.0)));
  // carve with planes to produce irregular facets
  d = max(d, dot(p, normalize(vec3(0.1, 0.6, 0.8))) - 0.6);
  return d;
}

// displace geometry into shards using noise + explode factor
vec3 shardOffset(vec3 p, float seed){
  // small 3D noise-like function (sum of sines)
  float n = sin(p.x*3.2 + seed*12.3) * 0.3 + sin(p.y*4.1 + seed*6.7)*0.25 + sin(p.z*2.7 + seed*9.1)*0.2;
  vec3 off = normalize(vec3(sin(seed*12.1), cos(seed*7.3), sin(seed*3.3))) * n * 0.6;
  return off;
}

// a fractured crystal SDF: we evaluate SDF of many shards displaced outward by explode amount
float fracturedCrystal(vec3 p, float explode){
  // shard count (pseudo)
  float best = 1e5;
  // sample few shard seeds
  for(int i=0;i<7;i++){
    float seed = float(i) * 0.618 + 0.23;
    // compute local shard center by rotating and scaling
    vec3 q = p - shardOffset(p + seed*3.14, seed) * explode * uPower;
    // scale shards slightly by seed
    q *= 1.0 + 0.08 * sin(seed*10.0 + uTime*0.2*uSpeed);
    float d = crystalSDF(q);
    // simulate cracks by adding small noise planes
    d = max(d, dot(q, normalize(vec3(sin(seed*2.1), cos(seed*1.7), sin(seed*0.9)))) - 0.55);
    best = min(best, d);
  }
  return best;
}

// estimate normal
vec3 estNormal(vec3 p){
  float e = 0.0015;
  return normalize(vec3(
    fracturedCrystal(p+vec3(e,0,0), uExplode) - fracturedCrystal(p-vec3(e,0,0), uExplode),
    fracturedCrystal(p+vec3(0,e,0), uExplode) - fracturedCrystal(p-vec3(0,e,0), uExplode),
    fracturedCrystal(p+vec3(0,0,e), uExplode) - fracturedCrystal(p-vec3(0,0,e), uExplode)
  ));
}

// soft shadow (ray marching along light)
float softShadow(vec3 ro, vec3 rd, float mint, float maxt){
  float res = 1.0;
  float t = mint;
  for(int i=0;i<32;i++){
    float h = fracturedCrystal(ro + rd * t, uExplode);
    if(h < 0.0005) return 0.0;
    res = min(res, 8.0 * h / t);
    t += clamp(h, 0.01, 0.2);
    if(t > maxt) break;
  }
  return clamp(res, 0.0, 1.0);
}

// ambient occlusion (small)
float ao(vec3 p, vec3 n){
  float s = 0.0;
  float sca = 1.0;
  for(int i=1;i<=5;i++){
    float hr = float(i)*0.02;
    s += (hr - fracturedCrystal(p + n * hr, uExplode)) * sca;
    sca *= 0.7;
  }
  return clamp(1.0 - s*6.0, 0.0, 1.0);
}

// raymarch main
float trace(vec3 ro, vec3 rd, out vec3 pos, out int steps){
  float t = 0.01;
  for(int i=0;i<120;i++){
    vec3 p = ro + rd * t;
    float d = fracturedCrystal(p, uExplode);
    if(d < 0.001){
      pos = p;
      steps = i;
      return t;
    }
    t += clamp(d*0.7, 0.01, 0.4);
    if(t > 60.0) break;
  }
  steps = 120;
  pos = ro + rd * t;
  return -1.0;
}

// volumetric glow: sample along view ray accumulating emissive contribution
vec3 volumetric(vec3 ro, vec3 rd){
  float sum = 0.0;
  vec3 col = vec3(0.0);
  float t = 0.0;
  for(int i=0;i<42;i++){
    vec3 p = ro + rd * t;
    // energy field proportional to negative SDF (inside) and nearby bright surfaces
    float d = fracturedCrystal(p, uExplode);
    float energy = exp(-d*6.0) * (1.0 + 0.5*sin(uTime*0.6 + length(p.xy)*0.05));
    // color shift (rainbow through hue)
    float hue = mod(uTime*20.0 + length(p.xy)*0.15, 360.0);
    vec3 c = vec3(0.3, 0.6, 1.0); // base icy-blue
    // tint by hue subtly
    c = mix(c, vec3(abs(sin(hue)), abs(sin(hue+2.0)), abs(sin(hue+4.0))), 0.12);
    col += c * energy * 0.12;
    sum += energy;
    t += 0.6;
  }
  return col * uVolum;
}

// main color
void main(){
  vec2 uv = (vUv * 2.0 - 1.0) * vec2(uRes.x/uRes.y, 1.0);
  // camera
  vec3 camPos = vec3(0.0, 0.6, -3.6 + uExplode * 1.2); // pull back a bit during explode
  vec3 target = vec3(0.0, 0.2, 0.0);
  vec3 forward = normalize(target - camPos);
  vec3 right = normalize(cross(vec3(0.0,1.0,0.0), forward));
  vec3 up = cross(forward, right);
  // jitter camera slowly
  float ang = uTime*0.12*uSpeed;
  mat3 camRot = rotY(ang) * rotX(sin(uTime*0.05)*0.08);
  camPos = camRot * camPos;
  forward = normalize(target - camPos);
  right = normalize(cross(vec3(0.0,1.0,0.0), forward));
  up = cross(forward, right);

  vec3 rd = normalize(forward + uv.x * right * 1.2 + uv.y * up * 1.0);
  vec3 ro = camPos;

  // trace
  vec3 pos; int steps;
  float dist = trace(ro, rd, pos, steps);

  vec3 finalColor = vec3(0.0);

  // volumetric ambient glow
  finalColor += volumetric(ro, rd) * 0.9;

  if(dist > 0.0){
    vec3 n = estNormal(pos);
    // lighting
    vec3 lightPos = vec3(2.5, 4.0, -1.0);
    vec3 L = normalize(lightPos - pos);
    float diff = max(0.0, dot(n, L));
    float spec = pow(max(0.0, dot(reflect(-L, n), normalize(ro - pos))), 36.0);

    // shadow
    float sh = softShadow(pos + n*0.02, L, 0.02, 8.0);

    // base ice color (hue shifts slightly)
    float hue = mod(190.0 + uTime*8.0 + length(pos.xy)*2.0 + uExplode*40.0, 360.0);
    vec3 base = vec3(0.2, 0.6, 1.0) * 0.9;
    base = mix(base, vec3(1.0, 0.9, 0.8), 0.12 * sin(uTime*0.8));

    // fresnel
    float fres = pow(1.0 - max(0.0, dot(n, -rd)), 2.4);

    vec3 color = base * diff * (0.8 + 0.6*sh) + vec3(1.0) * spec * 0.9;
    color += fres * 0.5;

    // ambient occlusion
    float a = ao(pos, n);

    finalColor += color * a;

    // edge glow: brighten outline
    float edge = smoothstep(0.0, 0.04, abs(fract(length(n.xy)*10.0)-0.5));
    finalColor += vec3(0.5,0.8,1.0) * edge * 0.6 * uExplode;
  } else {
    // background star-like faint
    finalColor += vec3(0.01,0.02,0.04);
  }

  // tonemapping + gamma
  finalColor = 1.0 - exp(-finalColor * vec3(1.2));
  finalColor = pow(finalColor, vec3(0.95));

  gl_FragColor = vec4(clamp(finalColor, 0.0, 1.0), 1.0);
}
</script>

<script>
(function(){
  const canvas = document.getElementById('glCanvas');
  const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
  if(!gl){ alert('مرورگرت از WebGL پشتیبانی نمی‌کند'); return; }

  // compile shader
  function compile(id, type){
    const src = document.getElementById(id).text.trim();
    const sh = gl.createShader(type);
    gl.shaderSource(sh, src);
    gl.compileShader(sh);
    if(!gl.getShaderParameter(sh, gl.COMPILE_STATUS)){
      console.error(gl.getShaderInfoLog(sh));
      throw new Error('Shader compile error: ' + id);
    }
    return sh;
  }

  const vs = compile('vs', gl.VERTEX_SHADER);
  const fs = compile('fs', gl.FRAGMENT_SHADER);

  const prog = gl.createProgram();
  gl.attachShader(prog, vs);
  gl.attachShader(prog, fs);
  gl.bindAttribLocation(prog, 0, 'aPos');
  gl.linkProgram(prog);
  if(!gl.getProgramParameter(prog, gl.LINK_STATUS)){
    console.error(gl.getProgramInfoLog(prog));
    throw new Error('Program link error');
  }

  gl.useProgram(prog);

  // fullscreen quad
  const quad = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, quad);
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1,-1, 1,-1, -1,1, 1,1]), gl.STATIC_DRAW);
  gl.enableVertexAttribArray(0);
  gl.vertexAttribPointer(0,2,gl.FLOAT,false,0,0);

  // uniforms
  const uRes = gl.getUniformLocation(prog,'uRes');
  const uTime = gl.getUniformLocation(prog,'uTime');
  const uExplode = gl.getUniformLocation(prog,'uExplode');
  const uPower = gl.getUniformLocation(prog,'uPower');
  const uVolum = gl.getUniformLocation(prog,'uVolum');
  const uSpeed = gl.getUniformLocation(prog,'uSpeed');

  // resize
  function resize(){
    canvas.width = innerWidth;
    canvas.height = innerHeight;
    gl.viewport(0,0,canvas.width,canvas.height);
  }
  addEventListener('resize', resize);
  resize();

  // UI controls
  const explodeBtn = document.getElementById('explodeBtn');
  const resetBtn = document.getElementById('resetBtn');
  const powerR = document.getElementById('power');
  const volR = document.getElementById('vol');
  const spdR = document.getElementById('speed');
  const pwrText = document.getElementById('pwrText');
  const volText = document.getElementById('volText');
  const spdText = document.getElementById('spdText');

  let explode = 0.0;
  let explodeTarget = 0.0;
  let explodeTime = 0.0;
  let power = parseFloat(powerR.value);
  let volum = parseFloat(volR.value);
  let speed = parseFloat(spdR.value);

  powerR.addEventListener('input', ()=>{ power = parseFloat(powerR.value); pwrText.textContent = power.toFixed(2); });
  volR.addEventListener('input', ()=>{ volum = parseFloat(volR.value); volText.textContent = volum.toFixed(2); });
  spdR.addEventListener('input', ()=>{ speed = parseFloat(spdR.value); spdText.textContent = speed.toFixed(2); });

  explodeBtn.addEventListener('click', ()=> {
    explodeTarget = 1.0;
    explodeTime = performance.now();
  });
  resetBtn.addEventListener('click', ()=> {
    explodeTarget = 0.0;
    explodeTime = performance.now();
  });

  // auto small pulses
  let lastPulse = performance.now();
  function maybePulse(){
    const now = performance.now();
    if(now - lastPulse > 4200 + Math.random()*3000){
      explodeTarget = 1.0;
      explodeTime = now;
      lastPulse = now;
      // auto reset slightly later
      setTimeout(()=>{ explodeTarget = 0.0; explodeTime = performance.now(); }, 900 + Math.random()*800);
    }
  }

  // render loop
  let start = performance.now();
  function render(){
    maybePulse();
    const now = performance.now();
    const t = (now - start) * 0.001;

    // ease explode
    const dt = (now - explodeTime) * 0.001;
    explode += (explodeTarget - explode) * 0.08;

    gl.uniform2f(uRes, canvas.width, canvas.height);
    gl.uniform1f(uTime, t);
    gl.uniform1f(uExplode, explode);
    gl.uniform1f(uPower, power);
    gl.uniform1f(uVolum, volum);
    gl.uniform1f(uSpeed, speed);

    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
    requestAnimationFrame(render);
  }

  // start
  requestAnimationFrame(render);

})();
</script>
</body>
</html>
