<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SSCC QR-CODE GENERATOR</title>
<style>
body { font-family: Arial,sans-serif; margin:0; padding:0; display:flex; flex-direction:column; justify-content:center; align-items:center; min-height:100vh; transition:background 0.3s,color 0.3s; }

/* Tema chiaro */
body.light { background:#ffffff; color:#ff7b00; }
body.light .wrapper { background:#f5f5f5; }
body.light .controls button, body.light .qty-btn, body.light .input-row button { background:#ff7b00; color:#fff; border:2px solid #ff7b00; }
body.light .controls button:hover, body.light .qty-btn:hover, body.light .input-row button:hover { background:#fff; color:#ff7b00; }

/* Tema scuro */
body.dark { background:#000000; color:#00a6ff; }
body.dark .wrapper { background:#1b1b1b; }
body.dark .controls button, body.dark .qty-btn, body.dark .input-row button { background:#0056ff; color:#fff; border:2px solid #0056ff; }
body.dark .controls button:hover, body.dark .qty-btn:hover, body.dark .input-row button:hover { background:#000; color:#00a6ff; }

.wrapper { display:flex; flex-wrap:wrap; gap:20px; max-width:900px; width:90%; padding:20px; border-radius:12px; box-shadow:0 3px 15px rgba(0,0,0,0.3); justify-content:center; align-items:flex-start; }
.controls { flex:1 1 280px; display:flex; flex-direction:column; gap:10px; }
.input-row { display:flex; gap:8px; align-items:center; }
.input-row input { flex:1; padding:10px; border-radius:6px; border:none; font-size:1rem; }
.controls button, .qty-btn { padding:8px 12px; border-radius:6px; font-size:0.95rem; cursor:pointer; }
.qty-container { display:flex; gap:6px; justify-content:space-between; }
.history-title { margin-top:15px; font-size:0.95rem; font-weight:bold; }
.history { max-height:150px; overflow-y:auto; font-size:0.9rem; }
.history-item { padding:4px 0; border-bottom:1px solid #444; cursor:pointer; user-select:text; }
.history-item:hover { text-decoration:underline; }
.preview-card { flex:1 1 260px; background:#f8f8f8; color:#000; border-radius:10px; padding:10px; display:flex; flex-direction:column; align-items:center; }
#qrContainer { width:220px; height:220px; background:#fff; border-radius:8px; border:1px solid #ccc; display:flex; justify-content:center; align-items:center; margin-bottom:5px; }
.qr-placeholder { font-size:0.85rem; color:#888; text-align:center; }
#last4 { font-size:1.1rem; font-weight:bold; margin-top:2px; }
footer { width:100%; text-align:center; padding:10px 0; font-size:0.95rem; cursor:pointer; }

/* Dashboard */
#dashboardContainer { display:none; padding:20px; max-width:900px; width:90%; background:#f8f8f8; border-radius:10px; }
body.dark #dashboardContainer { background:#1b1b1b; color:#00a6ff; }
#dashboardContent div { margin-bottom:5px; }
</style>
</head>
<body class="light">

<div class="wrapper">

  <div class="controls">
    <label for="ssccInput">SSCC Nummer</label>
    <div class="input-row">
      <input id="ssccInput" type="text" placeholder="SSCC eingeben oder scannen...">
      <button id="clearBtn" title="Löschen">✖️</button>
    </div>

    <div class="qty-container">
      <button class="qty-btn" data-count="5">5</button>
      <button class="qty-btn" data-count="10">10</button>
      <button class="qty-btn" data-count="15">15</button>
      <button class="qty-btn" data-count="20">20</button>
    </div>

    <div class="history-title">Letzte SSCC</div>
    <div class="history" id="historyList"></div>
  </div>

  <div class="preview-card">
    <div id="qrContainer">
      <div class="qr-placeholder">📱 QR wird hier generiert</div>
    </div>
    <div id="last4">----</div>
  </div>

</div>

<footer id="themeToggle">Made with ❤️ by Shko 🌞 / 🌙  Thema wechseln</footer>

<!-- Dashboard nascosta -->
<div id="dashboardContainer">
  <h2>Dashboard Riservata</h2>
  <div id="dashboardLogin">
    <input type="email" id="loginEmail" placeholder="Email">
    <input type="password" id="loginPassword" placeholder="Password">
    <button id="loginBtn">Accedi</button>
  </div>
  <div id="dashboardContent" style="display:none;">
    <button id="logoutBtn">Esci</button>
    <h3>Ultime stampe</h3>
    <div id="stats"></div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-auth-compat.js"></script>

<script>
/* -------- Firebase -------- */
const firebaseConfig = {
  apiKey: "AIzaSyA5kXm_3HMWO0YY4h_FzGr9bOpHZ5J6oxg",
  authDomain: "server-06-0201.firebaseapp.com",
  projectId: "server-06-0201",
  storageBucket: "server-06-0201.firebasestorage.app",
  messagingSenderId: "436450955649",
  appId: "1:436450955649:web:44645c7d39942b44875880"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();
const auth = firebase.auth();

/* -------- Variabili -------- */
const ssccInput = document.getElementById('ssccInput');
const qrContainer = document.getElementById('qrContainer');
const last4El = document.getElementById('last4');
const historyList = document.getElementById('historyList');
const themeToggle = document.getElementById('themeToggle');
const qtyButtons = document.querySelectorAll('.qty-btn');
const dashboardContainer = document.getElementById('dashboardContainer');
const dashboardLogin = document.getElementById('dashboardLogin');
const dashboardContent = document.getElementById('dashboardContent');
const loginBtn = document.getElementById('loginBtn');
const logoutBtn = document.getElementById('logoutBtn');
const statsDiv = document.getElementById('stats');

let qrCode = null;
let ssccHistory = [];
let currentTheme = 'light';
let pcId = localStorage.getItem('pcId');
if(!pcId){ pcId = 'pc-'+Math.random().toString(36).substr(2,9); localStorage.setItem('pcId', pcId); }

/* -------- Tema -------- */
themeToggle.addEventListener('click', ()=>{
  currentTheme = currentTheme==='light'?'dark':'light';
  document.body.className=currentTheme;
});

/* -------- Cronologia -------- */
function addToHistory(sscc){
  if(!sscc) return;
  const now=new Date();
  const time=now.getHours().toString().padStart(2,'0')+':'+now.getMinutes().toString().padStart(2,'0');
  ssccHistory=ssccHistory.filter(item=>item.value!==sscc);
  ssccHistory.unshift({value:sscc,time});
  if(ssccHistory.length>6) ssccHistory.pop();
  renderHistory();
}
function renderHistory(){
  historyList.innerHTML='';
  ssccHistory.forEach(item=>{
    const div=document.createElement('div');
    div.className='history-item';
    div.textContent=`${item.value} (${item.time})`;
    div.addEventListener('click', ()=>{ ssccInput.value=item.value; generateQR(item.value); });
    historyList.appendChild(div);
  });
}

/* -------- QR -------- */
function generateQR(value){
  qrContainer.innerHTML=''; last4El.textContent='----';
  if(!value){ qrContainer.innerHTML='<div class="qr-placeholder">📱 QR wird hier generiert</div>'; return; }
  qrCode=new QRCode(qrContainer,{text:value,width:220,height:220});
  last4El.textContent=value.slice(-4);
  addToHistory(value);
}

/* -------- Stampa e Firestore -------- */
function printQR(copies=1){
  const img=qrContainer.querySelector('img')||qrContainer.querySelector('canvas');
  if(!img) return alert('QR non generato');
  const qrSrc=img.src||img.toDataURL('image/png');
  const codeText=last4El.textContent||'';
  const printWindow=window.open('','_blank','width=600,height=400');
  let bodyHTML='';
  for(let i=0;i<copies;i++){
    bodyHTML+=`<div class="label"><img src="${qrSrc}"><div class="code">${codeText}</div></div>`;
    // Salvataggio Firestore
    db.collection("printLogs").add({pcId:ssccInput.value, sscc:ssccInput.value, quantity:copies, timestamp:firebase.firestore.FieldValue.serverTimestamp()});
  }
  printWindow.document.write(`<html><head><title>Druck</title><style>@page{size:50mm 30mm;margin:0}body{margin:0;padding:0;display:flex;flex-wrap:wrap;justify-content:center;align-items:center;font-family:Arial,sans-serif}.label{width:50mm;height:30mm;display:flex;flex-direction:column;justify-content:center;align-items:center;page-break-inside:avoid;}img{width:40mm;height:40mm;}.code{font-size:9pt;font-weight:bold;margin-top:1mm;}</style></head><body>${bodyHTML}</body></html>`);
  printWindow.document.close(); printWindow.focus(); printWindow.print(); printWindow.close();
}

/* -------- Eventi -------- */
ssccInput.addEventListener('input', ()=>{ const val=ssccInput.value.trim(); if(val.length===18){ generateQR(val); } });
clearBtn.addEventListener('click', ()=>{ ssccInput.value=''; generateQR(''); });
qtyButtons.forEach(btn=>{
  btn.addEventListener('click', ()=>{
    const qty=parseInt(btn.dataset.count);
    const sscc=ssccInput.value.trim();
    if(sscc.length!==18) return alert('SSCC non valido');
    generateQR(sscc);
    printQR(qty);
  });
});

/* -------- Dashboard -------- */
document.addEventListener('keydown', e=>{ if(e.ctrlKey && e.key==='d'){ dashboardContainer.style.display='block'; document.querySelector('.wrapper').style.display='none'; } });
loginBtn.addEventListener('click', ()=>{
  const email=document.getElementById('loginEmail').value;
  const pass=document.getElementById('loginPassword').value;
  auth.signInWithEmailAndPassword(email, pass)
  .then(()=>{ dashboardLogin.style.display='none'; dashboardContent.style.display='block'; loadStats(); })
  .catch(err=>alert("Errore login: "+err.message));
});
logoutBtn.addEventListener('click', ()=>{
  auth.signOut().then(()=>{ dashboardLogin.style.display='block'; dashboardContent.style.display='none'; });
});

function loadStats(){
  db.collection("printLogs").orderBy("timestamp","desc").limit(50).onSnapshot(snapshot=>{
    statsDiv.innerHTML='';
    snapshot.forEach(doc=>{
      const data=doc.data();
      const div=document.createElement('div');
      div.textContent=`PC: ${data.pcId}, SSCC: ${data.sscc}, Quantità: ${data.quantity}`;
      statsDiv.appendChild(div);
    });
  });
}
</script>

</body>
</html>
