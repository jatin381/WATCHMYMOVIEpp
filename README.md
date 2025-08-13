<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>🎬 Watch Together</title>
<style>
  body { font-family: Arial, sans-serif; text-align: center; background: #111; color: #fff; margin:0; padding:0; }
  h1, h2 { color: #ffd166; }
  button, input { padding: 10px; margin: 5px; border:none; border-radius:5px; }
  button { background:#ffd166; cursor:pointer; }
  #player { margin-top: 20px; }
  #emojiContainer { position: fixed; bottom: 10px; left: 50%; transform: translateX(-50%); font-size: 2rem; }
  #controls { margin-top: 15px; }
</style>
</head>
<body>

<h1>🎬 Watch Together</h1>

<div id="startScreen">
  <h2>Create or Join Room</h2>
  <input id="roomCode" placeholder="Room Code"><br>
  <input id="roomPass" placeholder="Password (optional)"><br>
  <input id="userName" placeholder="Your Name"><br>
  <button onclick="createRoom()">Create Room</button>
  <button onclick="joinRoom()">Join Room</button>
</div>

<div id="roomScreen" style="display:none;">
  <h2 id="roomTitle"></h2>
  <div id="hostControls" style="display:none;">
    <input id="ytLink" placeholder="YouTube Link">
    <button onclick="loadVideo()">Load</button>
  </div>
  <div id="player"></div>
  <div id="controls">
    <button onclick="toggleMic()">🎤 Mic</button>
    <button onclick="sendEmoji('❤️')">❤️</button>
    <button onclick="sendEmoji('😂')">😂</button>
    <button onclick="sendEmoji('👍')">👍</button>
    <button onclick="shareScreen()">🖥 Share Screen</button>
  </div>
  <div id="emojiContainer"></div>
  <video id="remoteVideo" autoplay playsinline style="width:200px;position:fixed;bottom:10px;right:10px;border:2px solid #fff;"></video>
</div>

<script src="https://www.youtube.com/iframe_api"></script>
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore, doc, setDoc, getDoc, onSnapshot, updateDoc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

let db, player, currentUser={}, isHost=false, peer, localStream;

// TODO: apna config yahan daalo
const firebaseConfig = {
  apiKey: "AIzaSyXXXXXX",
  authDomain: "XXXX.firebaseapp.com",
  projectId: "XXXX",
  storageBucket: "XXXX.appspot.com",
  messagingSenderId: "XXXX",
  appId: "XXXX"
};

const app = initializeApp(firebaseConfig);
db = getFirestore(app);

// Room create
window.createRoom = async () => {
  const code = document.getElementById('roomCode').value.trim();
  const pass = document.getElementById('roomPass').value.trim();
  const name = document.getElementById('userName').value.trim();
  if(!code || !name) return alert('Fill all');
  await setDoc(doc(db,"rooms",code), {password:pass||"",host:name,video:"",time:0,playing:false,emojis:[]});
  currentUser={name,code}; isHost=true;
  enterRoom();
};

// Join room
window.joinRoom = async () => {
  const code = document.getElementById('roomCode').value.trim();
  const pass = document.getElementById('roomPass').value.trim();
  const name = document.getElementById('userName').value.trim();
  if(!code || !name) return alert('Fill all');
  const ref = doc(db,"rooms",code);
  const snap = await getDoc(ref);
  if(!snap.exists()) return alert("No room");
  if(snap.data().password && snap.data().password !== pass) return alert("Wrong password");
  currentUser={name,code}; isHost=false;
  enterRoom();
};

// Enter room UI + listeners
function enterRoom(){
  document.getElementById('startScreen').style.display='none';
  document.getElementById('roomScreen').style.display='block';
  document.getElementById('roomTitle').innerText="Room: "+currentUser.code;
  if(isHost) document.getElementById('hostControls').style.display='block';
  const ref = doc(db,"rooms",currentUser.code);
  onSnapshot(ref, snap=>{
    const data = snap.data();
    if(!player && data.video) loadYouTubePlayer(data.video);
    if(player && !isHost){
      if(Math.abs(player.getCurrentTime()-data.time) > 1) player.seekTo(data.time);
      data.playing ? player.playVideo() : player.pauseVideo();
    }
    renderEmojis(data.emojis);
  });
}

// Extract YouTube ID
function extractID(url){
  const m = url.match(/(?:v=|youtu\.be\/)([^&]+)/);
  return m ? m[1] : url;
}

// Load video (host)
window.loadVideo = async () => {
  const id = extractID(document.getElementById('ytLink').value);
  await updateDoc(doc(db,"rooms",currentUser.code), {video:id,time:0,playing:false});
  loadYouTubePlayer(id);
};

// YouTube player
window.onYouTubeIframeAPIReady = ()=>{};
function loadYouTubePlayer(videoId){
  player = new YT.Player("player", {
    videoId,
    events: {
      onStateChange: e=>{
        if(isHost){
          updateDoc(doc(db,"rooms",currentUser.code), {
            time: player.getCurrentTime(),
            playing: e.data===YT.PlayerState.PLAYING
          });
        }
      }
    }
  });
}

// Emojis
window.sendEmoji = async (emoji)=>{
  const ref = doc(db,"rooms",currentUser.code);
  const snap = await getDoc(ref);
  const data = snap.data();
  data.emojis.push(emoji);
  if(data.emojis.length>10) data.emojis.shift();
  await updateDoc(ref,{emojis:data.emojis});
};
function renderEmojis(list){
  document.getElementById('emojiContainer').innerHTML=list.map(e=>`<span>${e}</span>`).join(" ");
}

// Voice chat (WebRTC)
window.toggleMic = async ()=>{
  if(localStream){
    localStream.getTracks().forEach(t=>t.stop());
    localStream=null;
    return;
  }
  localStream = await navigator.mediaDevices.getUserMedia({audio:true});
  initPeer();
};
function initPeer(){
  peer = new RTCPeerConnection();
  localStream.getTracks().forEach(track=>peer.addTrack(track,localStream));
  peer.ontrack = e=>{ document.getElementById('remoteVideo').srcObject = e.streams[0]; };
}

// Screen share
window.shareScreen = async ()=>{
  const screenStream = await navigator.mediaDevices.getDisplayMedia({video:true});
  screenStream.getTracks().forEach(track=>peer.addTrack(track,screenStream));
};

</script>
</body>
</html>
