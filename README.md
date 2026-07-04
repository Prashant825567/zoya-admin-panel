<!DOCTYPE html>

<html>
<head>
  <title>Zoya Admin Panel</title>

  <style>
    body {
      margin: 0;
      font-family: 'Segoe UI', sans-serif;
      background: linear-gradient(135deg, #0f172a, #020617);
      color: white;
      text-align: center;
    }

    h1 {
      margin-top: 20px;
      color: #38bdf8;
    }

    .container {
      max-width: 650px;
      margin: auto;
      padding: 20px;
    }

    input {
      padding: 10px;
      width: 85%;
      border-radius: 10px;
      border: none;
      background: #1e293b;
      color: white;
      margin: 5px;
    }

    button {
      padding: 10px 15px;
      border: none;
      border-radius: 10px;
      background: #3b82f6;
      color: white;
      margin: 5px;
      cursor: pointer;
    }

    button:disabled {
      opacity: 0.4;
      cursor: not-allowed;
    }

    .hidden { display: none; }

    .card {
      background: rgba(255,255,255,0.05);
      padding: 12px;
      margin-top: 12px;
      border-radius: 12px;
      text-align: left;
    }

    .status {
      font-size: 13px;
      padding: 4px 8px;
      border-radius: 6px;
      margin-left: 5px;
    }

    .on { background: #22c55e; }
    .off { background: #ef4444; }

  </style>

</head>

<body>

<h1>Zoya Admin Panel</h1>

<!-- LOGIN -->

<div id="loginBox" class="container">
  <input id="password" placeholder="Enter Password" type="password" />
  <br>
  <button onclick="login()">Login</button>
</div>

<!-- MAIN PANEL -->

<div id="mainPanel" class="container hidden">

  <h3 id="roleText"></h3>
  <h4 id="limitText"></h4>

  <input id="username" placeholder="Enter User Name" />
  <br>

<button onclick="generateKey()">Generate Key</button>

  <h2 id="keyDisplay"></h2>
  <button onclick="saveKey()">Save Key</button>

  <hr>

  <h2>All Keys</h2>
  <div id="keysList"></div>

</div>

<script type="module">

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import {
  getFirestore,
  doc,
  setDoc,
  getDoc,
  collection,
  getDocs,
  updateDoc,
  deleteDoc
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyDdXDEqjuYNNFyapUE6cvru-9KsFkj9bh0",
  authDomain: "max-6f064.firebaseapp.com",
  projectId: "max-6f064"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

let role = "";
let currentKey = "";
let veerCount = 0;
const LIMIT = 10;

// 🔐 LOGIN
window.login = function () {
  const pass = document.getElementById("password").value;

  if (pass === "noida123") {
    role = "super";
  } else if (pass === "Veer") {
    role = "veer";
  } else {
    alert("Wrong Password");
    return;
  }

  document.getElementById("loginBox").classList.add("hidden");
  document.getElementById("mainPanel").classList.remove("hidden");

  document.getElementById("roleText").innerText =
    role === "super" ? "Super Admin" : "Veer Admin";

  loadKeys();
};

// 🔑 GENERATE
window.generateKey = function () {
  if (role === "veer" && veerCount >= LIMIT) {
    alert("Limit reached (10 max)");
    return;
  }

  const p1 = Math.floor(100 + Math.random() * 900);
  let p2 = "";

  for (let i = 0; i < 10; i++) {
    p2 += Math.floor(Math.random() * 10);
  }

  currentKey = `ZOYA-${p1}-${p2}`;
  document.getElementById("keyDisplay").innerText = currentKey;
};

// 💾 SAVE
window.saveKey = async function () {

  const name = document.getElementById("username").value.trim();

  if (!name || !currentKey) {
    alert("Fill name & generate key");
    return;
  }

  const ref = doc(db, "accessKeys", currentKey);
  const snap = await getDoc(ref);

  if (snap.exists()) {
    alert("Key already exists");
    return;
  }

  await setDoc(ref, {
    name,
    deviceId: "",
    enabled: true,
    createdBy: role
  });

  if (role === "veer") veerCount++;

  currentKey = "";
  document.getElementById("keyDisplay").innerText = "";
  document.getElementById("username").value = "";

  loadKeys();
};

// 📥 LOAD
async function loadKeys() {

  const list = document.getElementById("keysList");
  list.innerHTML = "";

  const snapshot = await getDocs(collection(db, "accessKeys"));

  veerCount = 0;

  snapshot.forEach((docSnap) => {

    const data = docSnap.data();
    const key = docSnap.id;

    if (data.createdBy === "veer") veerCount++;

    const isOwner = data.createdBy === role;
    const isSuper = role === "super";
    const canControl = isSuper || isOwner;

    const loggedIn = data.deviceId && data.deviceId !== "";

    const card = document.createElement("div");
    card.className = "card";

    card.innerHTML = `
      <b>${key}</b> (${data.createdBy})
      <br>

      <input value="${data.name}"
        ${!canControl ? "disabled" : ""}
        onchange="updateName('${key}', this.value)" />

      <br>

      Status:
      <span class="status ${data.enabled ? 'on' : 'off'}">
        ${data.enabled ? "Enabled" : "Disabled"}
      </span>

      <br>

      Login:
      ${loggedIn ? "Logged (" + data.deviceId + ")" : "Not Logged"}

      <br><br>

      <button ${!canControl ? "disabled" : ""}
        onclick="toggleKey('${key}', ${data.enabled})">
        Toggle
      </button>

      <button ${!canControl || !loggedIn ? "disabled" : ""}
        onclick="resetDevice('${key}')">
        Reset
      </button>

      <button ${!canControl ? "disabled" : ""}
        onclick="deleteKey('${key}')">
        Delete
      </button>
    `;

    list.appendChild(card);

  });

  if (role === "veer") {
    document.getElementById("limitText").innerText =
      `Used ${veerCount} / 10`;
  }
}

// 🔄 TOGGLE
window.toggleKey = async function (key, current) {

  const ref = doc(db, "accessKeys", key);
  const snap = await getDoc(ref);
  const data = snap.data();

  if (role !== "super" && data.createdBy !== "veer") {
    alert("Not allowed");
    return;
  }

  await updateDoc(ref, { enabled: !current });
  loadKeys();
};

// ✏️ UPDATE
window.updateName = async function (key, name) {

  const ref = doc(db, "accessKeys", key);
  const snap = await getDoc(ref);
  const data = snap.data();

  if (role !== "super" && data.createdBy !== "veer") {
    alert("Not allowed");
    return;
  }

  await updateDoc(ref, { name });
};

// 📱 RESET
window.resetDevice = async function (key) {

  const ref = doc(db, "accessKeys", key);
  const snap = await getDoc(ref);
  const data = snap.data();

  if (role !== "super" && data.createdBy !== "veer") {
    alert("Not allowed");
    return;
  }

  if (!confirm("Reset device?")) return;

  await updateDoc(ref, { deviceId: "" });
  loadKeys();
};

// ❌ DELETE
window.deleteKey = async function (key) {

  const ref = doc(db, "accessKeys", key);
  const snap = await getDoc(ref);
  const data = snap.data();

  if (role !== "super" && data.createdBy !== "veer") {
    alert("Not allowed");
    return;
  }

  if (!confirm("Delete this key?")) return;

  await deleteDoc(ref);
  loadKeys();
};

</script>

</body>
</html>

