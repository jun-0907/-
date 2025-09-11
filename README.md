# 人間競馬
芝生祭の人間競馬で使うサイトです。

<html lang="ja">
<head>
<meta charset="UTF-8">
<title>SIT ポイント判定システム</title>
<style>
body {
  font-family: sans-serif;
  margin: 0;
  padding: 0;
  background-color: #dfffd6;
  color: #333;
}
header {
  background-color: #8fd694;
  color: white;
  text-align: center;
  font-size: 36px;
  padding: 20px 0;
  font-weight: bold;
}
.container {
  padding: 20px;
  max-width: 900px;
  margin: 0 auto;
}
h2, h3 {
  color: #fff;
  background-color: #8fd694;
  padding: 5px 10px;
  border-radius: 5px;
}
table {
  border-collapse: collapse;
  width: 100%;
  margin-bottom: 20px;
  background-color: white;
}
th, td {
  border: 1px solid #aaa;
  padding: 8px 12px;
  text-align: center;
}
input, select {
  padding: 3px;
  border: 1px solid #aaa;
  border-radius: 3px;
}
button {
  background-color: white;
  color: #8fd694;
  border: 2px solid #8fd694;
  padding: 5px 12px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  margin-left: 5px;
}
button:hover {
  background-color: #8fd694;
  color: white;
}
#adminSection {
  display: none;
  margin-top: 20px;
  border-top: 2px solid #8fd694;
  padding-top: 10px;
}
</style>
</head>
<body>
<header>SIT</header>

<div class="container">
  <h2>参加者登録</h2>
  <input type="text" id="participantName" placeholder="名前">
  <button onclick="addParticipant()">追加</button>

  <h3>登録済み参加者</h3>
  <ul id="participantList"></ul>

  <hr>

  <h2>ポイント割り振り</h2>
  <select id="selectParticipant" onchange="updateMyPoints()"></select>
  <div style="margin-top:10px;">
    <label>1: <input type="number" id="bet1" value="0"></label>
    <label>2: <input type="number" id="bet2" value="0"></label>
    <label>3: <input type="number" id="bet3" value="0"></label>
    <label>4: <input type="number" id="bet4" value="0"></label>
    <label>5: <input type="number" id="bet5" value="0"></label>
    <label>6: <input type="number" id="bet6" value="0"></label>
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
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database.js"></script>

<script>
// ✅ Firebase設定（自分のプロジェクト情報に置き換えてください）
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT.firebaseio.com",
  projectId: "YOUR_PROJECT",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "XXXXXXX",
  appId: "YOUR_APP_ID"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

const ADMIN_PASSWORD = "sugawara";

// 参加者追加
function addParticipant() {
  const name = document.getElementById("participantName").value.trim();
  if (!name) return alert("名前を入力してください");

  const ref = db.ref("participants/" + name);
  ref.get().then(snapshot=>{
    if(snapshot.exists()) {
      alert("すでに存在します");
    } else {
      ref.set({points:100, bets:{1:0,2:0,3:0,4:0,5:0,6:0}});
    }
  });
}

// ポイント割り振り
function submitBets() {
  const name = document.getElementById("selectParticipant").value;
  if(!name) return alert("参加者を選択してください");

  db.ref("participants/"+name).get().then(snapshot=>{
    if(snapshot.exists()){
      const p = snapshot.val();
      const bets = {};
      let totalBet = 0;
      for(let i=1;i<=6;i++){
        bets[i] = parseInt(document.getElementById("bet"+i).value)||0;
        totalBet += bets[i];
      }
      if(totalBet > p.points){
        alert("ポイントが足りません！");
        return;
      }
      p.points -= totalBet;
      p.bets = bets;
      db.ref("participants/"+name).set(p);
    }
  });
}

// 管理者判定
function judgeAll() {
  const hit = parseInt(document.getElementById("hitNumber").value);
  const odds = parseFloat(document.getElementById("hitOdds").value);
  db.ref("participants").once("value").then(snapshot=>{
    snapshot.forEach(child=>{
      const p = child.val();
      const bet = p.bets[hit]||0;
      if(bet>0){
        p.points += bet * odds;
      }
      p.bets = {1:0,2:0,3:0,4:0,5:0,6:0};
      db.ref("participants/"+child.key).set(p);
    });
  });
  alert("判定完了！");
}

// 管理者リセット
function resetParticipants() {
  if(!confirm("本当に全員削除しますか？")) return;
  db.ref("participants").remove();
}

// 管理者ログイン
function checkAdmin() {
  const pass = document.getElementById("adminPass").value;
  if(pass === ADMIN_PASSWORD){
    document.getElementById("adminSection").style.display = "block";
  } else {
    alert("パスワードが違います");
  }
}

// 🔄 リアルタイム更新（参加者リスト＆管理者画面）
db.ref("participants").on("value", snapshot=>{
  const participants = snapshot.val()||{};
  // 参加者リスト
  const ul = document.getElementById("participantList");
  ul.innerHTML = "";
  Object.keys(participants).forEach(name=>{
    const li = document.createElement("li");
    li.textContent = `${name}（${participants[name].points}ポイント）`;
    ul.appendChild(li);
  });

  // セレクト更新
  const select = document.getElementById("selectParticipant");
  const current = select.value;
  select.innerHTML = "";
  Object.keys(participants).forEach(name=>{
    const opt = document.createElement("option");
    opt.value = name;
    opt.text = name;
    select.appendChild(opt);
  });
  if(current && participants[current]) select.value = current;

  updateMyPoints();

  // 管理者テーブル
  const tbody = document.getElementById("adminTable").querySelector("tbody");
  tbody.innerHTML = "";
  Object.entries(participants).forEach(([name,p])=>{
    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${name}</td><td>${p.points}</td>` +
      Array.from({length:6},(_,i)=>`<td>${p.bets[i+1]}</td>`).join("");
    tbody.appendChild(tr);
  });

  // 合計行
  const totalBets = [0,0,0,0,0,0];
  Object.values(participants).forEach(p=>{
    for(let i=1;i<=6;i++) totalBets[i-1] += p.bets[i]||0;
  });
  const trTotal = document.createElement("tr");
  trTotal.style.fontWeight="bold";
  trTotal.style.backgroundColor="#d0ffd0";
  trTotal.innerHTML = `<td>合計</td><td>-</td>` + totalBets.map(v=>`<td>${v}</td>`).join("");
  tbody.appendChild(trTotal);
});

function updateMyPoints(){
  const name = document.getElementById("selectParticipant").value;
  if(!name) return document.getElementById("myPoints").innerText=0;
  db.ref("participants/"+name+"/points").get().then(s=>{
    document.getElementById("myPoints").innerText = s.exists()?s.val():0;
  });
}
</script>
</body>
</html>
