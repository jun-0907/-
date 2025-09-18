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
    input[type="number"], input[type="text"], input[type="password"], select {
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

    <!-- 参加者登録 -->
    <h2>参加者登録</h2>
    <input type="text" id="participantName" placeholder="名前">
    <button onclick="addParticipant()">追加</button>

    <h3>登録済み参加者</h3>
    <ul id="participantList"></ul>

    <hr>

    <!-- ポイント割り振り -->
    <h2>ポイント割り振り</h2>
    <select id="selectParticipant"></select>
    <div style="margin-top:10px;">
      <label>1: <input type="number" id="bet1" value="0" min="0"></label>
      <label>2: <input type="number" id="bet2" value="0" min="0"></label>
      <label>3: <input type="number" id="bet3" value="0" min="0"></label>
      <label>4: <input type="number" id="bet4" value="0" min="0"></label>
      <label>5: <input type="number" id="bet5" value="0" min="0"></label>
      <label>6: <input type="number" id="bet6" value="0" min="0"></label>
      <button onclick="submitBets()">ポイント使用</button>
    </div>

    <h3>自分の持ちポイント</h3>
    <p id="myPoints">0</p>

    <hr>

    <!-- 管理者ログイン -->
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
            <th>1</th><th>2</th><th>3</th>
            <th>4</th><th>5</th><th>6</th>
          </tr>
        </thead>
        <tbody></tbody>
      </table>

      <!-- 🔹 ポイント増減フォーム -->
      <h3>ポイント編集</h3>
      <select id="editTarget"></select>
      <input type="number" id="editAmount" value="0" step="1" min="1">
      <button onclick="adjustPoints(true)">追加</button>
      <button onclick="adjustPoints(false)">減算</button>

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
    // Firebase 設定
    const firebaseConfig = {
      apiKey: "AIzaSyDJMPJFhvKtiTbst0JlCqCGbgK2tLsJjf0",
      authDomain: "ningenkeiba-6350f.firebaseapp.com",
      projectId: "ningenkeiba-6350f",
      storageBucket: "ningenkeiba-6350f.appspot.com",
      messagingSenderId: "655803286740",
      appId: "1:655803286740:web:487d467f504e4f6a5e2741",
      measurementId: "G-G21ZHE4BKL",
      databaseURL: "https://ningenkeiba-6350f-default-rtdb.firebaseio.com/"
    };
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();
    const participantsRef = db.ref("participants");

    let participants = {};
    let myName = localStorage.getItem("myName") || null;
    const ADMIN_PASSWORD = "sugawara";

    // Firebase → ローカル同期
    participantsRef.on("value", snapshot => {
      participants = snapshot.val() || {};
      updateParticipantList();
      updateParticipantSelect();
      updateAdminTable();
      updateAdminSelect();
    });

    // 参加者追加
    function addParticipant() {
      const name = document.getElementById("participantName").value.trim();
      if (!name) return alert("名前を入力してください");
      if (participants[name]) return alert("すでに存在します");

      participantsRef.child(name).set({
        points: 100,
        bets: {1:0,2:0,3:0,4:0,5:0,6:0}
      });

      // 自分の名前を記録
      localStorage.setItem("myName", name);
      myName = name;

      document.getElementById("participantName").value = "";
      alert("登録完了！ あなたのアカウントは「" + name + "」です。");
    }

    // 登録済み参加者リスト
    function updateParticipantList() {
      const ul = document.getElementById("participantList");
      ul.innerHTML = '';
      Object.keys(participants).forEach(name => {
        const li = document.createElement("li");
        li.textContent = name + "（" + participants[name].points + "ポイント）";
        ul.appendChild(li);
      });
    }

    // セレクト更新
    function updateParticipantSelect() {
      const select = document.getElementById("selectParticipant");
      select.innerHTML = '';

      if (myName && participants[myName]) {
        // 自分の名前だけ選択可能
        const option = document.createElement("option");
        option.value = myName;
        option.text = myName;
        select.appendChild(option);
        select.disabled = true;
      } else {
        Object.keys(participants).forEach(name=>{
          const option = document.createElement("option");
          option.value = name;
          option.text = name;
          select.appendChild(option);
        });
      }
      if (select.options.length > 0 && !select.value) select.value = select.options[0].value;
      updateMyPoints();
    }

    // 自分のポイント表示
    function updateMyPoints() {
      const select = document.getElementById("selectParticipant");
      if(!select.value) {
        document.getElementById("myPoints").innerText = 0;
        return;
      }
      const name = select.value;
      document.getElementById("myPoints").innerText = participants[name]?.points || 0;
    }

    // ベット処理（累積）
    function submitBets() {
      const name = document.getElementById("selectParticipant").value;
      if (!myName || name !== myName) {
        return alert("自分のアカウントでのみ操作できます");
      }

      const p = participants[name];
      if (!p) return alert("参加者が見つかりません");

      const bets = {...p.bets};
      let newBetSum = 0;

      for (let i = 1; i <= 6; i++) {
        let val = document.getElementById("bet" + i).value;
        if (val === "") val = "0";
        let bet = parseInt(val, 10);

        if (isNaN(bet) || bet < 0) {
          return alert("ポイントは 0 以上の整数で入力してください");
        }

        bets[i] += bet;
        newBetSum += bet;
      }

      if (newBetSum > p.points) {
        alert("ポイントが足りません！");
        return;
      }

      participantsRef.child(name).update({
        points: p.points - newBetSum,
        bets: bets
      });

      for (let i = 1; i <= 6; i++) {
        document.getElementById("bet" + i).value = "0";
      }
    }

    // 管理者テーブル
    function updateAdminTable() {
      const tbody = document.getElementById("adminTable").querySelector("tbody");
      tbody.innerHTML = '';
      Object.entries(participants).forEach(([name,data])=>{
        const tr = document.createElement("tr");
        tr.innerHTML = `<td>${name}</td><td>${data.points}</td>` +
          Array.from({length:6},(_,i)=>`<td>${data.bets[i+1]}</td>`).join('');
        tbody.appendChild(tr);
      });

      const totalBets = [0,0,0,0,0,0];
      Object.values(participants).forEach(p=>{
        for(let i=1;i<=6;i++) totalBets[i-1] += p.bets[i] || 0;
      });
      const trTotal = document.createElement("tr");
      trTotal.style.fontWeight = "bold";
      trTotal.style.backgroundColor = "#d0ffd0";
      trTotal.innerHTML = `<td>合計</td><td>-</td>` +
        totalBets.map(v=>`<td>${v}</td>`).join('');
      tbody.appendChild(trTotal);
    }

    // 管理者セレクト更新
    function updateAdminSelect() {
      const select = document.getElementById("editTarget");
      if (!select) return;
      select.innerHTML = '';
      Object.keys(participants).forEach(name => {
        const option = document.createElement("option");
        option.value = name;
        option.text = name;
        select.appendChild(option);
      });
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
      alert("参加者情報をリセットしました。");
    }

    // ポイント増減処理
    function adjustPoints(isAdd) {
      const name = document.getElementById("editTarget").value;
      const amount = parseInt(document.getElementById("editAmount").value, 10);

      if (!name || isNaN(amount) || amount <= 0) {
        return alert("正しい数値を入力してください");
      }

      const p = participants[name];
      if (!p) return alert("参加者が見つかりません");

      let newPoints = p.points + (isAdd ? amount : -amount);
      if (newPoints < 0) newPoints = 0;

      participantsRef.child(name).update({
        points: newPoints
      });

      alert(`${name} のポイントを ${isAdd ? "追加" : "減算"}しました`);
    }
  </script>
</body>
</html>
