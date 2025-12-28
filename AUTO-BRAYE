<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<title>CivilBIM Cameroon</title>
<meta name="viewport" content="width=device-width,initial-scale=1.0" />

<!-- React -->
<script src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>

<!-- Three.js + helpers -->
<script src="https://unpkg.com/three@0.154.0/build/three.min.js"></script>
<script src="https://unpkg.com/three@0.154.0/examples/js/controls/OrbitControls.js"></script>

<!-- PDF / Excel -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>

<!-- Firebase (optional login) -->
<script src="https://www.gstatic.com/firebasejs/10.13.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.13.0/firebase-auth-compat.js"></script>

<style>
body{margin:0;font-family:sans-serif}
#top{position:fixed;top:0;width:100%;background:#fff;padding:6px;z-index:10}
#toolbar{position:fixed;bottom:0;width:100%;display:flex;background:#0d6efd;z-index:10}
#toolbar button{flex:1;padding:10px;color:#fff;background:transparent;border:none;font-weight:bold}
#canvas{width:100vw;height:100vh;display:block}
#login{position:fixed;right:10px;top:40px;background:#fff;padding:6px;border-radius:4px;z-index:15}
</style>
</head>
<body>
<div id="top">
Concrete: <span id="vol">0</span> m³ | Cost: <span id="cost">0</span> XAF |
Foundation : <span id="found">0</span> m³
</div>
<canvas id="canvas"></canvas>
<div id="login">
<input id="email" placeholder="Email"/><br/>
<input id="pwd" placeholder="Password" type="password"/><br/>
<button onclick="login()">Login</button>
</div>
<div id="toolbar">
  <button onclick="mode='DRAW'">Wall</button>
  <button onclick="mode='COLUMN'">Column</button>
  <button onclick="mode='DOOR'">Door</button>
  <button onclick="mode='WINDOW'">Window</button>
  <button onclick="mode='WATER'">Water</button>
  <button onclick="mode='POWER'">Power</button>
  <button onclick="saveProject()">Save</button>
  <button onclick="loadProject()">Load</button>
  <button onclick="exportPDF()">PDF</button>
  <button onclick="exportBOQ()">BOQ</button>
  <button onclick="recordVideo()">Video</button>
</div>

<script>
/* ---------- Firebase login (optional) ---------- */
const firebaseConfig={apiKey:"YOUR_KEY",authDomain:"YOUR_DOMAIN",projectId:"YOUR_ID"};
firebase.initializeApp(firebaseConfig);
const auth=firebase.auth();
function login(){
  const email=document.getElementById("email").value;
  const pwd=document.getElementById("pwd").value;
  auth.signInWithEmailAndPassword(email,pwd)
    .then(()=>alert("Login successful")).catch(e=>alert(e.message));
}

/* ---------- Three.js scene ---------- */
let mode="DRAW";
const scene=new THREE.Scene();
scene.background=new THREE.Color(0xf0f0f0);
const camera=new THREE.PerspectiveCamera(60,window.innerWidth/window.innerHeight,0.1,1000);
camera.position.set(8,6,8);
const renderer=new THREE.WebGLRenderer({canvas:document.getElementById("canvas"),antialias:true});
renderer.setSize(window.innerWidth,window.innerHeight);
const controls=new THREE.OrbitControls(camera,renderer.domElement);
scene.add(new THREE.AmbientLight(0xffffff,0.8));
const dir=new THREE.DirectionalLight(0xffffff,0.6);dir.position.set(10,20,10);scene.add(dir);
const grid=new THREE.GridHelper(30,30);scene.add(grid);

let walls=[],columns=[],openings=[],symbols=[];
let startPoint=null;
const raycaster=new THREE.Raycaster(),mouse=new THREE.Vector2();

renderer.domElement.addEventListener("pointerdown",e=>{
 mouse.x=(e.clientX/window.innerWidth)*2-1;
 mouse.y=-(e.clientY/window.innerHeight)*2+1;
 raycaster.setFromCamera(mouse,camera);
 const hits=raycaster.intersectObjects(scene.children,true);
 if(!hits.length)return;
 const p=hits[0].point;

 if(mode==="DRAW"){
   if(!startPoint) startPoint=p;
   else{
     const len=startPoint.distanceTo(p);
     const mid=startPoint.clone().add(p).multiplyScalar(0.5);
     const angle=Math.atan2(p.x-startPoint.x,p.z-startPoint.z);
     const geo=new THREE.BoxGeometry(len,3,0.2);
     const mat=new THREE.MeshStandardMaterial({color:0xdcdcdc});
     const wall=new THREE.Mesh(geo,mat);
     wall.position.set(mid.x,1.5,mid.z);
     wall.rotation.y=angle;
     scene.add(wall);
     walls.push({len});
     startPoint=null;
     updateCost();
   }
 }else if(mode==="COLUMN"){
   const c=new THREE.Mesh(new THREE.BoxGeometry(0.25,3,0.25),
                          new THREE.MeshStandardMaterial({color:0x999999}));
   c.position.set(p.x,1.5,p.z);scene.add(c);columns.push(1);updateCost();
 }else if(mode==="DOOR"||mode==="WINDOW"){
   const h=mode==="DOOR"?2.1:1.2,w=mode==="DOOR"?0.9:1.2,y=mode==="DOOR"?1.05:1.6;
   const o=new THREE.Mesh(new THREE.BoxGeometry(w,h,0.1),
       new THREE.MeshStandardMaterial({color:mode==="DOOR"?0x7b4a12:0x87ceeb,opacity:0.6,transparent:true}));
   o.position.set(p.x,y,p.z);scene.add(o);openings.push(1);
 }else if(mode==="WATER"||mode==="POWER"){
   const s=new THREE.Mesh(new THREE.SphereGeometry(0.15,16,16),
       new THREE.MeshStandardMaterial({color:mode==="WATER"?"blue":"yellow"}));
   s.position.set(p.x,0.2,p.z);scene.add(s);symbols.push(1);
 }
});

function updateCost(){
 const wallVol=walls.reduce((s,w)=>s+w.len*3*0.2,0);
 const colVol=columns.length*0.25*0.25*3;
 const foundation=(wallVol+colVol)*0.3;
 const vol=wallVol+colVol;
 document.getElementById("vol").innerText=vol.toFixed(2);
 document.getElementById("found").innerText=foundation.toFixed(2);
 document.getElementById("cost").innerText=(vol*75000).toLocaleString();
}

/* ---------- Save / Load ---------- */
function saveProject(){
 localStorage.setItem("civilbim",JSON.stringify({walls,columns,openings,symbols}));
 alert("Project saved locally");
}
function loadProject(){
 const d=JSON.parse(localStorage.getItem("civilbim"));
 if(!d)return alert("No save found");
 location.reload();
}

/* ---------- PDF / BOQ ---------- */
function exportPDF(){
 const {jsPDF}=window.jspdf;const pdf=new jsPDF();
 pdf.text("CivilBIM Cameroon",10,10);
 pdf.text("Walls: "+walls.length,10,20);
 pdf.text("Columns: "+columns.length,10,30);
 pdf.text("Concrete: "+document.getElementById("vol").innerText+" m³",10,40);
 pdf.save("civilbim.pdf");
}
function exportBOQ(){
 const wb=XLSX.utils.book_new();
 const ws=XLSX.utils.aoa_to_sheet([
  ["Item","Qty","Unit"],
  ["Concrete",document.getElementById("vol").innerText,"m³"],
  ["Foundation",document.getElementById("found").innerText,"m³"]
 ]);
 XLSX.utils.book_append_sheet(wb,ws,"BOQ");
 XLSX.writeFile(wb,"BOQ.xlsx");
}

/* ---------- Screen-record walkthrough ---------- */
async function recordVideo(){
 const stream=await navigator.mediaDevices.getDisplayMedia({video:true});
 const rec=new MediaRecorder(stream);
 rec.ondataavailable=e=>{
   const a=document.createElement("a");
   a.href=URL.createObjectURL(e.data);a.download="walkthrough.webm";a.click();
 };
 rec.start();setTimeout(()=>rec.stop(),15000);
}

/* ---------- Render loop ---------- */
function animate(){requestAnimationFrame(animate);renderer.render(scene,camera);}animate();
window.addEventListener("resize",()=>{camera.aspect=innerWidth/innerHeight;
camera.updateProjectionMatrix();renderer.setSize(innerWidth,innerHeight);});
</script>
</body>
</html>
