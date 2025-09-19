# 人間競馬
芝生祭の人間競馬で使うサイトです。

<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>SIT ポイント判定システム</title>
  <style>
    body { font-family: sans-serif; margin: 0; padding: 0; background-color: #dfffd6; color: #333; }
    header { background-color: #8fd694; color: white; text-align: center; font-size: 36px; padding: 20px 0; font-weight: bold; }
    .container { padding: 20px; max-width: 900px; margin: 0 auto; }
    h2, h3 { color: #fff; background-color: #8fd694; padding: 5px 10px; border-radius: 5px; }
    table { border-collapse: collapse; width: 100%; margin-bottom: 20px; background-color: white; }
    th, td { border: 1px solid #aaa; padding: 8px 12px; text-align: center; }
    input[type="number"], input[type="text"], input[type="password"], select { width: 120px; padding: 3px; border: 1px solid #aaa; border-radius: 3px; }
    button { background-color: white; color: #8fd694; border: 2px solid #8fd694; padding: 5px 12px; border-radius: 5px; cursor: pointer; font-weight: bold; margin-left: 5px; }
    button:hover { background-color: #8fd694; color: white; }
    #adminSection { display: none; margin-top: 20px; border-top: 2px solid #8fd694; padding-top: 10px; }
  </style>
</head>
<body>
<header>SIT</header>
<div class="container">
  <h2>参加者登録 / ログイン</h2>
  <input type="text" id="participantName" placeholder="名前">
  <input type="password" id="participantPass" placeholder="パスワード">
  <button onclick="registerOrLogin()">登録/ログイン</button>
  <p id="loginStatus">未ログイン</p>

  <h3>登録済み参加者</h3>
  <ul id="participantList"></ul>
  <hr>

  <h2>ポイント割り振り</h2>
  <div style="margin-top:10px;">
    <label>1: <input type="number" id="bet1" value="0" min="0" step="1"></label>
    <label>2: <input type="number" id="bet2" value="0" min="0" step="1"></label>
    <label>3: <input type="number" id="bet3" value="0" min="0" step="1"></label>
    <label>4: <input type="number" id="bet4" value="0" min="0" step="1"></label>
    <label>5: <input type="number" id="bet5" value="0" min="0" step="1"></label>
    <label>6: <input type="number" id="bet6" value="0" min="0" step="1"></label>
    <button onclick="submitBets()">ポイント使用</button>
  </div>

  <h3>自分の持ちポイント</h3>
  <p id="myPoints">0</p>

  <hr>
  <h2>管理者ログイン</h2>
  <input type="password" id="adminPass" placeholder="パスワード">
  <button onclick="checkAdmin()">ログイン</button>

  <div id="adminSection">
    <h2>管理者画面</h2>
    <button onclick="resetParticipants()">参加者情報リセット</button>
    <table id="adminTable">
      <thead>
        <tr>
          <th>名前</th>
          <th>持ちポイント</th>
          <th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>6</th>
          <th>ポイント調整</th>
        </tr>
      </thead>
      <tbody></tbody>
    </table>
    <p>
      当たり番号: <input type="number" id="hitNumber" min="1" max="6">
      倍率: <input type="number" id="hitOdds" step="0.1" value="2">
      <button onclick="judgeAll()">判定</button>
    </p>
  </div>
</div>

<!-- Firebase SDK -->
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-database.js"></script>
<script>
const firebaseConfig = {
  apiKey: "AIzaSyDJMPJFhvKtiTbst0JlCqCGbgK2tLsJjf0",
  authDomain: "ningenkeiba-6350f.firebaseapp.com",
  projectId: "ningenkeiba-6350f",
  storageBucket: "ningenkeiba-6350f.appspot.com",
  messagingSenderId: "655803286740",
  appId: "1:655803286740:web:487d467f504e4f6a5e2741"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();
const participantsRef = db.ref("participants");

let participants = {};
let currentUser = localStorage.getItem("currentUser") || null;
const ADMIN_PASSWORD = "sugawara";

// Firebase → ローカル同期
participantsRef.on("value", snapshot => {
  participants = snapshot.val() || {};
  updateParticipantList();
  updateAdminTable();
  updateMyPoints();
  if (currentUser) {
    document.getElementById("loginStatus").innerText = "ログイン中: " + currentUser;
  }
});

// 登録/ログイン
function registerOrLogin() {
  const name = document.getElementById("participantName").value.trim();
  const pass = document.getElementById("participantPass").value.trim();
  if (!name || !pass) return alert("名前とパスワードを入力してください");

  participantsRef.child(name).get().then(snapshot => {
    if (!snapshot.exists()) {
      // 新規登録
      participantsRef.child(name).set({
        password: pass,
        points: 100,
        bets: {1:0,2:0,3:0,4:0,5:0,6:0}
      }).then(() => {
        alert("新規登録しました: " + name);
        finishLogin(name);
      });
    } else {
      // 既存 → パスワードチェック
      const data = snapshot.val();
      if (data.password === pass) {
        alert("ログインしました: " + name);
        finishLogin(name);
      } else {
        alert("パスワードが間違っています");
      }
    }
  }).catch(err => {
    console.error("Firebaseエラー:", err);
    alert("登録/ログインに失敗しました");
  });
}

function finishLogin(name) {
  currentUser = name;
  localStorage.setItem("currentUser", name);
  document.getElementById("participantName").value = "";
  document.getElementById("participantPass").value = "";
  document.getElementById("loginStatus").innerText = "ログイン中: " + currentUser;
  updateMyPoints();
}

// 参加者リスト
function updateParticipantList() {
  const ul = document.getElementById("participantList");
  ul.innerHTML = '';
  Object.keys(participants).forEach(name => {
    const li = document.createElement("li");
    li.textContent = name + "（" + participants[name].points + "ポイント）";
    ul.appendChild(li);
  });
}

// 自分のポイント表示
function updateMyPoints() {
  if (!currentUser || !participants[currentUser]) {
    document.getElementById("myPoints").innerText = 0;
    return;
  }
  document.getElementById("myPoints").innerText = participants[currentUser].points;
}

// ベット処理
function submitBets() {
  if (!currentUser) return alert("先にログインしてください");
  const p = participants[currentUser];
  const alreadyBet = Object.values(p.bets).some(v => v > 0);
  if (alreadyBet) {
    alert("すでにベット済みです。判定後までお待ちください。");
    return;
  }

  const bets = {};
  let totalBet = 0;
  for (let i=1;i<=6;i++){
    let val = parseInt(document.getElementById("bet"+i).value)||0;
    if (val < 0) val = 0;
    bets[i] = val;
    totalBet += val;
  }
  if(totalBet > p.points){
    alert("ポイントが足りません！");
    return;
  }
  participantsRef.child(currentUser).update({
    points: p.points - totalBet,
    bets: bets
  });
}

// 管理者テーブル更新
function updateAdminTable() {
  const tbody = document.getElementById("adminTable").querySelector("tbody");
  tbody.innerHTML = '';
  let totalPoints = 0;
  const totalBets = [0,0,0,0,0,0];

  Object.entries(participants).forEach(([name,data])=>{
    totalPoints += data.points || 0;
    for(let i=1;i<=6;i++) totalBets[i-1] += data.bets[i] || 0;

    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${name}</td><td>${data.points}</td>` +
      Array.from({length:6},(_,i)=>`<td>${data.bets[i+1]}</td>`).join('') +
      `<td>
        <input type="number" id="adjust_${name}" value="0" step="1">
        <button onclick="adjustPoints('${name}')">適用</button>
      </td>`;
    tbody.appendChild(tr);
  });

  // 合計行
  const trTotal = document.createElement("tr");
  trTotal.style.fontWeight = "bold";
  trTotal.style.backgroundColor = "#d0ffd0";
  trTotal.innerHTML = `<td>合計</td><td>${totalPoints}</td>` +
    totalBets.map(v=>`<td>${v}</td>`).join('') + `<td>-</td>`;
  tbody.appendChild(trTotal);
}

// ポイント調整（管理者用）
function adjustPoints(name) {
  const val = parseInt(document.getElementById("adjust_"+name).value)||0;
  const newPoints = (participants[name].points||0) + val;
  participantsRef.child(name).update({ points: Math.max(0,newPoints) });
}

// 管理者ログイン
function checkAdmin() {
  const pass = document.getElementById("adminPass").value;
  if(pass === ADMIN_PASSWORD){
    document.getElementById("adminSection").style.display = "block";
    alert("管理者ログイン成功！");
  } else {
    alert("パスワードが違います");
  }
}

// 当たり判定
function judgeAll() {
  const hit = parseInt(document.getElementById("hitNumber").value);
  const odds = parseFloat(document.getElementById("hitOdds").value);
  Object.entries(participants).forEach(([name,p])=>{
    const bet = p.bets[hit]||0;
    let newPoints = p.points;
    if(bet>0){
      newPoints += bet * odds;
    }
    participantsRef.child(name).update({
      points: newPoints,
      bets: {1:0,2:0,3:0,4:0,5:0,6:0}
    });
  });
  alert("判定完了！（全員に共有されました）");
}

// リセット
function resetParticipants() {
  if(!confirm("本当にすべての参加者情報を削除しますか？")) return;
  participantsRef.set({});
  localStorage.removeItem("currentUser");
  currentUser = null;
  alert("参加者情報をリセットしました。");
}
</script>
</body>
</html>
