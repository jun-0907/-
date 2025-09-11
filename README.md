# 人間競馬
芝生祭の人間競馬で使うサイトです。

<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>SIT ポイント判定システム</title>
<style>
body {
  font-family: sans-serif;
  margin: 0;
  padding: 0;
  background-color: #dfffd6; /* 薄い黄緑 */
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

input[type="number"], input[type="text"], input[type="password"], select {
  width: 50px;
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
<select id="selectParticipant"></select>

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

<script>
let participants = {};
const ADMIN_PASSWORD = "sugawara";
let registeredThisSession = false;

// 参加者追加
function addParticipant() {
  if (registeredThisSession) {
    alert("このセッションではすでに参加者登録済みです。ページを再読み込みすると再登録可能です。");
    return;
  }

  const name = document.getElementById("participantName").value.trim();
  if(!name) return alert("名前を入力してください");
  if(participants[name]) return alert("すでに存在します");

  participants[name] = {points: 100, bets:{1:0,2:0,3:0,4:0,5:0,6:0}};
  registeredThisSession = true;

  updateParticipantList();
  updateParticipantSelect();
  updateAdminTable();
}

// 登録済み参加者リスト更新
function updateParticipantList() {
  const ul = document.getElementById("participantList");
  ul.innerHTML = '';
  Object.keys(participants).forEach(name => {
    const li = document.createElement("li");
    li.textContent = name + "（" + participants[name].points + "ポイント）";
    ul.appendChild(li);
  });
}

// 参加者選択セレクト更新
function updateParticipantSelect() {
  const select = document.getElementById("selectParticipant");
  select.innerHTML = '';
  Object.keys(participants).forEach(name=>{
    const option = document.createElement("option");
    option.value = name;
    option.text = name;
    select.appendChild(option);
  });

  if(select.options.length>0 && !select.value) select.value = select.options[0].value;

  updateMyPoints();
}

// 自分の持ちポイント表示
function updateMyPoints() {
  const select = document.getElementById("selectParticipant");
  if(!select.value) {
    document.getElementById("myPoints").innerText = 0;
    return;
  }
  const name = select.value;
  document.getElementById("myPoints").innerText = participants[name]?.points || 100;
}

// ポイント割り振り
function submitBets() {
  const name = document.getElementById("selectParticipant").value;
  const p = participants[name];
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
  updateMyPoints();
  updateAdminTable();
}

// 管理者用テーブル更新（個人 + 選択肢合計表示）
function updateAdminTable() {
  const tbody = document.getElementById("adminTable").querySelector("tbody");
  tbody.innerHTML = '';

  // 個人行
  Object.entries(participants).forEach(([name,data])=>{
    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${name}</td><td>${data.points}</td>` +
      Array.from({length:6},(_,i)=>`<td>${data.bets[i+1]}</td>`).join('');
    tbody.appendChild(tr);
  });

  // 合計行
  const totalBets = [0,0,0,0,0,0];
  Object.values(participants).forEach(p=>{
    for(let i=1;i<=6;i++) totalBets[i-1] += p.bets[i] || 0;
  });

  const trTotal = document.createElement("tr");
  trTotal.style.fontWeight = "bold";
  trTotal.style.backgroundColor = "#d0ffd0"; // 薄い黄緑
  trTotal.innerHTML = `<td>合計</td><td>-</td>` +
    totalBets.map(v=>`<td>${v}</td>`).join('');
  tbody.appendChild(trTotal);
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

  Object.values(participants).forEach(p=>{
    const bet = p.bets[hit]||0;
    if(bet>0){
      p.points += bet * odds;
    }
    p.bets = {1:0,2:0,3:0,4:0,5:0,6:0};
  });

  updateMyPoints();
  updateAdminTable();
  alert("判定完了！");
}

// 管理者向け参加者情報リセット
function resetParticipants() {
  if(!confirm("本当にすべての参加者情報を削除しますか？")) return;

  participants = {};
  registeredThisSession = false;
  updateParticipantList();
  updateParticipantSelect();
  updateAdminTable();
  alert("参加者情報をリセットしました。");
}
</script>
</body>
</html>
