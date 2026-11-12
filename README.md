# gorilla-horror

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>GORILLA HORROR V1</title>
<style>
*{box-sizing:border-box}
body{margin:0;overflow:hidden;background:#050907;color:white;font-family:Arial,sans-serif}
#hud{position:fixed;inset:0;pointer-events:none}
#top{position:absolute;top:14px;left:14px;text-shadow:0 2px 8px black;font-weight:bold}
#status{color:#9cff9c;font-size:13px;margin-top:6px}
#computer{
 position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);
 width:min(420px,92vw);background:#101914;border:3px solid #66836a;
 border-radius:12px;padding:18px;display:none;pointer-events:auto;
 box-shadow:0 0 35px #000;
}
#computer h2{margin:0 0 8px;color:#b9ffb5;letter-spacing:3px}
#computer p{color:#c5d0c5;font-size:13px}
#computer input{width:100%;padding:12px;margin:5px 0;background:#050b07;color:#baffb6;border:1px solid #6e956f;font-size:17px}
#computer .choice{padding:12px;margin:7px 0;background:#1b2a20;border:1px solid #526c56;cursor:pointer;color:#d9ffd5}
#computer .choice:hover{background:#304b35}
#computer small{color:#9aab9a}
#vr{position:absolute;bottom:20px;left:50%;transform:translateX(-50%);pointer-events:auto;
 padding:13px 24px;border:1px solid #9cff9c;background:#122318;color:#d9ffd5;
 font-weight:bold;border-radius:8px;font-size:16px}
#help{position:absolute;bottom:18px;left:14px;font-size:12px;color:#d1d8d1;text-shadow:0 2px 5px black}
#crosshair{position:absolute;left:50%;top:50%;width:5px;height:5px;background:white;border-radius:50%;opacity:.7}
#notice{position:absolute;bottom:70px;left:50%;transform:translateX(-50%);color:#caffc6;text-shadow:0 2px 5px black}
</style>
</head>
<body>
<div id="hud">
 <div id="top">GORILLA HORROR V1<div id="status">Starting...</div></div>
 <div id="crosshair"></div>
 <div id="notice"></div>
 <div id="help">WASD: move · Mouse: look · E: computer · Click: lock mouse</div>
 <button id="vr">ENTER VR</button>
 <div id="computer">
  <h2>GH COMPUTER</h2>
  <p>WELCOME TO GORILLA HORROR</p>
  <div class="choice" id="create">CREATE — MAKE A PRIVATE ROOM</div>
  <div class="choice" id="join">JOIN — ENTER A ROOM CODE</div>
  <p>ROOM CODE</p>
  <input id="roomCode" maxlength="8" placeholder="ENTER CODE">
  <p>PLAYER NAME</p>
  <input id="playerName" maxlength="16" placeholder="GORILLA">
  <p>COLOR CODE (3 DIGITS)</p>
  <input id="colorCode" maxlength="3" inputmode="numeric" value="123">
  <small>Use the keyboard to enter your name, room code, and color.</small>
  <div class="choice" id="save">SAVE NAME + COLOR</div>
  <div class="choice" id="close">CLOSE COMPUTER</div>
 </div>
</div>

<script type="module">
import * as THREE from "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js";
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
import {
 getDatabase, ref, set, update, onValue, onDisconnect, remove
} from "https://www.gstatic.com/firebasejs/10.12.2/firebase-database.js";

/* -------------------------------------------------------
   FIREBASE SETUP
   Replace these example values with your Firebase project
   configuration. Enable Realtime Database in Firebase.
-------------------------------------------------------- */
const firebaseConfig = {
 apiKey: "PASTE_YOUR_API_KEY",
 authDomain: "PASTE_YOUR_PROJECT.firebaseapp.com",
 databaseURL: "https://PASTE_YOUR_PROJECT-default-rtdb.firebaseio.com",
 projectId: "PASTE_YOUR_PROJECT",
 storageBucket: "PASTE_YOUR_PROJECT.appspot.com",
 messagingSenderId: "PASTE_YOUR_SENDER_ID",
 appId: "PASTE_YOUR_APP_ID"
};

let db=null, firebaseReady=false;
try{
 if(firebaseConfig.apiKey!=="PASTE_YOUR_API_KEY"){
  const app=initializeApp(firebaseConfig);
  db=getDatabase(app);
  firebaseReady=true;
 }
}catch(err){console.warn("Firebase setup needed:",err)}

const statusEl=document.getElementById("status");
const noticeEl=document.getElementById("notice");
const computer=document.getElementById("computer");
const roomInput=document.getElementById("roomCode");
const nameInput=document.getElementById("playerName");
const colorInput=document.getElementById("colorCode");

const scene=new THREE.Scene();
scene.background=new THREE.Color(0x07120c);
scene.fog=new THREE.FogExp2(0x07120c,0.035);

const camera=new THREE.PerspectiveCamera(75,innerWidth/innerHeight,.1,500);
camera.position.set(0,1.65,5);

const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.xr.enabled=true;
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.HemisphereLight(0x9bb99c,0x182016,1.5));
const moon=new THREE.DirectionalLight(0xc5d7c0,1.4);
moon.position.set(-8,15,6);moon.castShadow=true;scene.add(moon);

const materials={
 ground:new THREE.MeshStandardMaterial({color:0x17251a,roughness:1}),
 bark:new THREE.MeshStandardMaterial({color:0x30251b}),
 leaves:new THREE.MeshStandardMaterial({color:0x102719}),
 wood:new THREE.MeshStandardMaterial({color:0x493322}),
 metal:new THREE.MeshStandardMaterial({color:0x303b32,roughness:.7}),
 screen:new THREE.MeshStandardMaterial({color:0x7cff83,emissive:0x16451a,emissiveIntensity:1})
};

function box(w,h,d,mat,x,y,z){
 const m=new THREE.Mesh(new THREE.BoxGeometry(w,h,d),mat);
 m.position.set(x,y,z);m.castShadow=true;m.receiveShadow=true;scene.add(m);return m;
}
function cylinder(rt,rb,h,mat,x,y,z){
 const m=new THREE.Mesh(new THREE.CylinderGeometry(rt,rb,h,8),mat);
 m.position.set(x,y,z);m.castShadow=true;scene.add(m);return m;
}

/* Forest floor */
const floor=new THREE.Mesh(new THREE.PlaneGeometry(300,300),materials.ground);
floor.rotation.x=-Math.PI/2;floor.position.y=-.08;floor.receiveShadow=true;scene.add(floor);

/* Starting treehouse */
box(8,.35,8,materials.wood,0,2.8,0);
box(.25,3,.25,materials.bark,-3.5,1.3,-3.5);
box(.25,3,.25,materials.bark,3.5,1.3,-3.5);
box(.25,3,.25,materials.bark,-3.5,1.3,3.5);
box(.25,3,.25,materials.bark,3.5,1.3,3.5);
box(8,.25,.3,materials.wood,0,1.5,-3.8);
box(8,.25,.3,materials.wood,0,1.5,3.8);
box(.3,.25,8,materials.wood,-3.8,1.5,0);
box(.3,.25,8,materials.wood,3.8,1.5,0);

/* Computer desk and monitor */
box(2.4,.18,.9,materials.wood,0,1.3,-2.5);
box(.18,1.2,.18,materials.wood,-.9,.7,-2.5);
box(.18,1.2,.18,materials.wood,.9,.7,-2.5);
box(1.5,1,.14,materials.metal,0,2,-2.75);
const monitor=box(1.35,.78,.035,materials.screen,0,2,-2.65);
box(1.4,.08,.55,materials.metal,0,1.45,-2.5);
for(let i=0;i<10;i++)box(.09,.035,.09,materials.screen,-.55+i*.12,1.51,-2.3);

/* Forest trees */
for(let i=0;i<110;i++){
 const a=Math.random()*Math.PI*2;
 const r=9+Math.random()*75;
 const x=Math.cos(a)*r,z=Math.sin(a)*r;
 cylinder(.35,.55,4,materials.bark,x,1.9,z);
 const crown=new THREE.Mesh(new THREE.ConeGeometry(2.3,5,7),materials.leaves);
 crown.position.set(x,5.2,z);crown.castShadow=true;scene.add(crown);
}

/* Path from the treehouse into the forest */
for(let i=0;i<12;i++){
 const p=box(2,.08,2,materials.wood,0,.02,5+i*2);
 p.rotation.y=(Math.random()-.5)*.15;
}

/* Simple gorilla avatar */
function makeGorilla(color=0x493c36){
 const root=new THREE.Group();
 const fur=new THREE.MeshStandardMaterial({color,roughness:.9});
 const body=new THREE.Mesh(new THREE.SphereGeometry(.38,12,10),fur);
 body.position.y=.85;root.add(body);
 const head=new THREE.Mesh(new THREE.SphereGeometry(.28,12,10),fur);
 head.position.set(0,1.35,0);root.add(head);
 const eyeMat=new THREE.MeshStandardMaterial({color:0xffffff});
 const pupil=new THREE.MeshStandardMaterial({color:0x080808});
 for(const x of [-.11,.11]){
  const eye=new THREE.Mesh(new THREE.SphereGeometry(.055,8,8),eyeMat);
  eye.position.set(x,1.4,-.24);root.add(eye);
  const dot=new THREE.Mesh(new THREE.SphereGeometry(.025,8,8),pupil);
  dot.position.set(x,1.4,-.29);root.add(dot);
 }
 const arms=[];
 for(const side of [-1,1]){
  const arm=new THREE.Mesh(new THREE.CylinderGeometry(.12,.16,.8,8),fur);
  arm.position.set(side*.48,.85,-.05);arm.rotation.z=side*-.7;root.add(arm);arms.push(arm);
 }
 root.userData.arms=arms;
 return root;
}
const myGorilla=makeGorilla();
scene.add(myGorilla);

/* Name label */
function makeLabel(text){
 const canvas=document.createElement("canvas");
 canvas.width=512;canvas.height=128;
 const ctx=canvas.getContext("2d");
 ctx.clearRect(0,0,512,128);
 ctx.font="bold 48px Arial";ctx.textAlign="center";
 ctx.fillStyle="white";ctx.strokeStyle="#101010";ctx.lineWidth=8;
 ctx.strokeText(text,256,78);ctx.fillText(text,256,78);
 const tex=new THREE.CanvasTexture(canvas);
 const sprite=new THREE.Sprite(new THREE.SpriteMaterial({map:tex,transparent:true}));
 sprite.scale.set(1.5,.38,1);sprite.position.y=1.9;
 return sprite;
}

let playerName=localStorage.getItem("gh_name")||"GORILLA";
let colorCode=localStorage.getItem("gh_color")||"123";
nameInput.value=playerName;colorInput.value=colorCode;
let roomId=null,playerId="p_"+Math.random().toString(36).slice(2,11);
let remotePlayers={};
let roomListener=null;
let lastSend=0;

function colorFromCode(code){
 const n=Math.max(0,Math.min(999,parseInt(code)||0));
 return new THREE.Color(`hsl(${(n*37)%360},55%,45%)`);
}
function updateMyColor(){
 const c=colorFromCode(colorCode);
 myGorilla.traverse(o=>{if(o.isMesh && o.material && o.material.color)o.material.color.copy(c)});
}
updateMyColor();

function say(message){
 noticeEl.textContent=message;
 setTimeout(()=>{if(noticeEl.textContent===message)noticeEl.textContent=""},3000);
}
function setStatus(message){statusEl.textContent=message}

/* Firebase room handling */
async function createRoom(){
 if(!firebaseReady){say("Add your Firebase settings first!");return}
 roomId=Math.random().toString(36).slice(2,7).toUpperCase();
 await joinRoom(roomId);
}
async function joinRoom(code){
 if(!firebaseReady){say("Firebase is not connected yet.");return}
 code=(code||"").trim().toUpperCase();
 if(!code){say("Enter a room code first.");return}
 roomId=code;
 const playerRef=ref(db,`rooms/${roomId}/players/${playerId}`);
 await set(playerRef,{
  name:playerName,color:colorCode,
  x:camera.position.x,y:camera.position.y,z:camera.position.z,
  ry:camera.rotation.y,updated:Date.now()
 });
 onDisconnect(playerRef).remove();
 if(roomListener)roomListener();
 roomListener=onValue(ref(db,`rooms/${roomId}/players`),snap=>{
  const players=snap.val()||{};
  for(const id of Object.keys(remotePlayers)){
   if(!players[id]){
    scene.remove(remotePlayers[id].group);
    delete remotePlayers[id];
   }
  }
  for(const [id,data] of Object.entries(players)){
   if(id===playerId)continue;
   if(!remotePlayers[id]){
    const group=makeGorilla(colorFromCode(data.color||"123").getHex());
    const label=makeLabel(data.name||"GORILLA");
    group.add(label);scene.add(group);
    remotePlayers[id]={group};
   }
   const p=remotePlayers[id].group;
   p.position.set(data.x||0,(data.y||0)-1.5,data.z||0);
   p.rotation.y=data.ry||0;
  }
 });
 setStatus("ROOM: "+roomId+" · "+(firebaseReady?"ONLINE":"OFFLINE"));
 say("Joined room "+roomId);
}
async function leaveRoom(){
 if(roomId && db){
  await remove(ref(db,`rooms/${roomId}/players/${playerId}`));
  if(roomListener)roomListener();
 }
 roomId=null;roomListener=null;
 for(const id of Object.keys(remotePlayers)){scene.remove(remotePlayers[id].group);delete remotePlayers[id]}
 setStatus("Not in a room");
}

/* Computer interactions */
document.getElementById("create").onclick=createRoom;
document.getElementById("join").onclick=()=>joinRoom(roomInput.value);
document.getElementById("save").onclick=()=>{
 playerName=nameInput.value.trim().slice(0,16)||"GORILLA";
 colorCode=(colorInput.value.replace(/\D/g,"").slice(0,3)||"123").padStart(3,"0");
 nameInput.value=playerName;colorInput.value=colorCode;
 localStorage.setItem("gh_name",playerName);
 localStorage.setItem("gh_color",colorCode);
 updateMyColor();say("Name and color saved!");
};
document.getElementById("close").onclick=()=>computer.style.display="none";

const raycaster=new THREE.Raycaster();
function toggleComputer(){
 computer.style.display=computer.style.display==="block"?"none":"block";
}
addEventListener("keydown",e=>{
 if(e.key.toLowerCase()==="e" && !e.repeat)toggleComputer();
 if(e.key==="Escape")computer.style.display="none";
});

/* Desktop movement */
const keys={};
addEventListener("keydown",e=>keys[e.key.toLowerCase()]=true);
addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);
let yaw=0,pitch=0,mouseLocked=false;
renderer.domElement.addEventListener("click",()=>{
 if(computer.style.display!=="block" && !renderer.xr.isPresenting)renderer.domElement.requestPointerLock?.();
});
document.addEventListener("pointerlockchange",()=>mouseLocked=document.pointerLockElement===renderer.domElement);
document.addEventListener("mousemove",e=>{
 if(!mouseLocked)return;
 yaw-=e.movementX*.002;
 pitch-=e.movementY*.002;
 pitch=Math.max(-1.3,Math.min(1.3,pitch));
 camera.rotation.set(pitch,yaw,0,"YXZ");
});

/* VR controllers: grip movement moves the player in the direction
   of the controller's swing. This is a basic V1 movement prototype. */
const controller0=renderer.xr.getController(0);
const controller1=renderer.xr.getController(1);
scene.add(controller0,controller1);
let previousHands=[new THREE.Vector3(),new THREE.Vector3()];
let handReady=[false,false];
function handleVRHands(){
 const hands=[controller0,controller1];
 let movement=new THREE.Vector3();
 hands.forEach((hand,i)=>{
  const pos=new THREE.Vector3();
  hand.getWorldPosition(pos);
  if(handReady[i]){
   const delta=previousHands[i].clone().sub(pos);
   delta.y=0;
   if(delta.length()<.35)movement.add(delta.multiplyScalar(1.5));
  }
  previousHands[i].copy(pos);handReady[i]=true;
 });
 if(movement.length()>0){
  camera.position.add(movement);
 }
}

/* ENTER VR */
document.getElementById("vr").onclick=async()=>{
 if(!navigator.xr){say("WebXR is not available in this browser.");return}
 try{
  const supported=await navigator.xr.isSessionSupported("immersive-vr");
  if(!supported){say("No compatible VR headset/session found.");return}
  const session=await navigator.xr.requestSession("immersive-vr",{optionalFeatures:["local-floor","bounded-floor"]});
  await renderer.xr.setSession(session);
  document.getElementById("vr").style.display="none";
  session.addEventListener("end",()=>document.getElementById("vr").style.display="block");
 }catch(err){say("Could not start VR: "+err.message)}
};

/* Main loop */
const clock=new THREE.Clock();
function animate(){
 const dt=Math.min(clock.getDelta(),.05);
 if(!renderer.xr.isPresenting && computer.style.display!=="block"){
  const forward=new THREE.Vector3(-Math.sin(yaw),0,-Math.cos(yaw));
  const right=new THREE.Vector3(Math.cos(yaw),0,-Math.sin(yaw));
  const speed=4*dt;
  if(keys.w)camera.position.addScaledVector(forward,speed);
  if(keys.s)camera.position.addScaledVector(forward,-speed);
  if(keys.a)camera.position.addScaledVector(right,-speed);
  if(keys.d)camera.position.addScaledVector(right,speed);
  camera.position.y=1.65;
 }
 if(renderer.xr.isPresenting)handleVRHands();

 myGorilla.position.set(camera.position.x,camera.position.y-1.5,camera.position.z);
 myGorilla.rotation.y=camera.rotation.y;

 if(roomId && db && Date.now()-lastSend>120){
  lastSend=Date.now();
  update(ref(db,`rooms/${roomId}/players/${playerId}`),{
   name:playerName,color:colorCode,
   x:camera.position.x,y:camera.position.y,z:camera.position.z,
   ry:camera.rotation.y,updated:Date.now()
  }).catch(()=>{});
 }
 renderer.render(scene,camera);
}
renderer.setAnimationLoop(animate);

addEventListener("resize",()=>{
 camera.aspect=innerWidth/innerHeight;
 camera.updateProjectionMatrix();
 renderer.setSize(innerWidth,innerHeight);
});

setStatus(firebaseReady?"Firebase ready · Not in a room":"Firebase setup needed");
say("Welcome to Gorilla Horror!");
</script>
</body>
</html>
