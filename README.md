<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>체크리스트</title>
<style>
body { font-family:sans-serif; margin:0; padding:0; background:#f5f5f5; }
header { display:flex; flex-wrap:wrap; justify-content:space-between; align-items:center;
  padding:10px; background:#333; color:white; }
header button { margin:5px 5px 5px 0; padding:10px 12px; border:none; border-radius:6px; cursor:pointer;
  background:#555; color:white; font-size:16px; min-height:44px; min-width:44px; }
#centerTimeWrapper { flex:1; text-align:center; }
#currentTime { font-weight:bold; font-size:20px; }

.board {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 12px;
  padding: 12px;
  max-width: calc(4 * 280px + 3 * 12px);
  margin: 0 auto;
}

.list { background:white; border-radius:10px; padding:10px; box-shadow:0 2px 6px rgba(0,0,0,0.15);
  display:flex; flex-direction:column; }
.list-header { display:flex; justify-content:space-between; align-items:center; margin-bottom:10px; flex-wrap:wrap; }
.list-header h2 { margin:0; font-size:18px; word-break:break-word; }
.list-header .actions { display:flex; gap:5px; }
.list-header button { background:none; border:none; font-size:18px; cursor:pointer; min-width:36px; min-height:36px; }
ul { list-style:none; padding:0; margin:0; flex-grow:1; }
li { display:flex; align-items:center; justify-content:space-between;
  padding:8px; border-bottom:1px solid #eee; font-size:16px; }
li input[type="checkbox"] { transform:scale(1.4); margin-right:10px; cursor:pointer; }
li span { flex-grow:1; word-break:break-word; cursor:pointer; }
li span.done { text-decoration:line-through; color:gray; }

.item-actions {
  display:flex;
  gap:1px; /* 버튼 사이 간격 1px */
}
.item-actions button {
  font-size:16px;
  min-width:30px;
  min-height:30px;
  padding:5px 6px;
  border:none;
  background:none;
  cursor:pointer;
}

.new-item { display:flex; flex-wrap: wrap; margin-top:5px; gap:5px; }
.new-item input { flex: 1 1 0; min-width: 0; padding:10px; font-size:16px; border-radius:6px; border:1px solid #ccc; }
.new-item button { flex-shrink:0; padding:10px 12px; font-size:16px; border-radius:6px; cursor:pointer; }

/* 모달 */
.modal-overlay { position:fixed; top:0; left:0; width:100%; height:100%;
  background:rgba(0,0,0,0.5); display:flex; align-items:center; justify-content:center;
  z-index:1000; opacity:0; pointer-events:none; transition:opacity 0.3s ease; }
.modal-overlay.show { opacity:1; pointer-events:auto; }
.modal { background:white; padding:20px; border-radius:8px; width:90%; max-width:400px; text-align:left;
  box-shadow:0 4px 12px rgba(0,0,0,0.3); transform:scale(0.9); opacity:0;
  transition:all 0.3s ease; }
.modal-overlay.show .modal { transform:scale(1); opacity:1; }
.modal button { margin:5px; padding:10px 16px; border:none; border-radius:5px; cursor:pointer; font-size:16px; }
#modalDelete { background:#e74c3c; color:white; }
#modalDelete:hover { background:#c0392b; }
.modal button.cancel { background:#bdc3c7; color:black; }
.modal button.cancel:hover { background:#95a5a6; }

/* 도움말 전용 */
#helpModal .modal { text-align:left; max-width:500px; }
#helpModal h2 { margin-top:0; font-size:20px; text-align:center; }
</style>
</head>
<body>
<header>
  <div>
    <button id="addList">➕ 새 목록</button>
    <button id="helpBtn">❔ 도움말</button>
  </div>
  <div id="centerTimeWrapper">
    <span id="currentTime"></span>
  </div>
  <div>
    <button id="exportJson">💾 저장</button>
    <button id="importJson">📂 열기</button>
    <button id="exportTxt">📑 TXT 내보내기</button>
  </div>
</header>

<div class="board" id="board"></div>

<!-- 삭제 확인 모달 -->
<div class="modal-overlay" id="modal">
  <div class="modal">
    <p id="modalMessage">이 목록을 삭제하시겠습니까?</p>
    <button id="modalDelete">삭제</button>
    <button class="cancel" id="modalCancelBtn">취소</button>
  </div>
</div>

<!-- 도움말 모달 -->
<div class="modal-overlay" id="helpModal">
  <div class="modal">
    <h2>체크리스트 사용 안내</h2>
    <ul>
      <li>➕ 새 목록: 새 체크리스트 목록 추가</li>
      <li>🔄 체크 초기화: 해당 목록의 모든 항목 체크 해제</li>
      <li>❌ 목록 삭제: 선택한 목록 삭제</li>
      <li>항목 추가: 입력 후 Enter 또는 추가 버튼 클릭</li>
      <li>💾 저장 / 📂 열기: JSON 파일로 저장/불러오기</li>
      <li>📑 TXT 내보내기: 체크리스트 TXT 파일 저장</li>
      <li>날짜가 바뀌면 체크 항목이 자동 초기화됩니다</li>
      <li>항목 클릭 또는 체크박스 클릭으로 체크 토글 가능</li>
      <li>항목 화살표 버튼으로 순서 이동 가능</li>
    </ul>
    <div style="text-align:center;">
      <button id="helpCloseBtn">닫기</button>
    </div>
  </div>
</div>

<script>
document.addEventListener("DOMContentLoaded", ()=>{
let lists = JSON.parse(localStorage.getItem("lists")||"[]");
let deleteTargetIndex = null;

function save(){ localStorage.setItem("lists", JSON.stringify(lists)); }

function resetChecksIfNewDay(){
  const today = new Date();
  const todayStr = `${today.getFullYear()}-${today.getMonth()+1}-${today.getDate()}`;
  const lastDate = localStorage.getItem("lastDate");
  if(lastDate !== todayStr){
    lists.forEach(list => list.items.forEach(item => item.done = false));
    save();
    localStorage.setItem("lastDate", todayStr);
  }
}

function updateTime(){
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth()+1).padStart(2,'0');
  const day = String(now.getDate()).padStart(2,'0');
  const h = String(now.getHours()).padStart(2,'0');
  const m = String(now.getMinutes()).padStart(2,'0');
  const s = String(now.getSeconds()).padStart(2,'0');
  document.getElementById("currentTime").innerText = `${year}-${month}-${day} ${h}:${m}:${s}`;
}

function render(){
  resetChecksIfNewDay();
  updateTime();
  const board = document.getElementById("board"); 
  board.innerHTML = "";
  lists.forEach((list,listIndex)=>{
    const div = document.createElement("div"); div.className = "list";

    const header = document.createElement("div"); header.className = "list-header";
    header.innerHTML=`<h2 contenteditable="true">${list.title}</h2>
      <div class="actions">
        <button title="체크 초기화">🔄</button>
        <button title="목록 삭제">❌</button>
      </div>`;
    header.querySelector("h2").oninput=(e)=>{ list.title=e.target.innerText; save(); };
    header.querySelector("button[title='체크 초기화']").onclick = ()=>{ list.items.forEach(i=>i.done=false); save(); render(); };
    header.querySelector("button[title='목록 삭제']").onclick = ()=>{
      deleteTargetIndex = listIndex;
      document.getElementById("modalMessage").innerHTML = 
        `목록 <strong style="color:#e74c3c;">${list.title}</strong> 을(를) 삭제하시겠습니까?`;
      openModal();
    };
    div.appendChild(header);

    const ul = document.createElement("ul");

    list.items.forEach((item,itemIndex)=>{
      const li = document.createElement("li");
      li.innerHTML=`<div style="display:flex;align-items:center;flex:1;">
                      <input type="checkbox" ${item.done?'checked':''}/> 
                      <span class="${item.done?'done':''}">${item.text}</span>
                    </div>
                    <div class="item-actions">
                      <button title="위로">⬆️</button>
                      <button title="아래로">⬇️</button>
                      <button title="삭제">🗑️</button>
                    </div>`;

      const checkbox = li.querySelector("input");
      const span = li.querySelector("span");
      const upBtn = li.querySelector("button[title='위로']");
      const downBtn = li.querySelector("button[title='아래로']");
      const delBtn = li.querySelector("button[title='삭제']");

      function toggleCheck(){ item.done = !item.done; save(); render(); }
      checkbox.onchange = toggleCheck;
      span.onclick = toggleCheck;

      upBtn.onclick = ()=>{
        if(itemIndex===0) return;
        [list.items[itemIndex-1], list.items[itemIndex]] = [list.items[itemIndex], list.items[itemIndex-1]];
        save(); render();
      };
      downBtn.onclick = ()=>{
        if(itemIndex===list.items.length-1) return;
        [list.items[itemIndex+1], list.items[itemIndex]] = [list.items[itemIndex], list.items[itemIndex+1]];
        save(); render();
      };
      delBtn.onclick = ()=>{
        list.items.splice(itemIndex,1); save(); render();
      };

      ul.appendChild(li);
    });

    const newDiv = document.createElement("div"); newDiv.className = "new-item";
    newDiv.innerHTML = `<input placeholder="새 항목"/><button>추가</button>`;
    const inputField = newDiv.querySelector("input"); 
    const addBtn = newDiv.querySelector("button");
    function addNewItem(){
      const val = inputField.value.trim();
      if(val){
        list.items.push({text:val, done:false});
        save(); render();
        setTimeout(()=>{
          const lastList = document.querySelectorAll(".list")[listIndex];
          const newInput = lastList.querySelector(".new-item input");
          newInput.value=""; newInput.focus();
        });
      }
    }
    addBtn.onclick = addNewItem;
    inputField.addEventListener("keypress", e=>{ if(e.key==="Enter") addNewItem(); });

    div.appendChild(ul);
    div.appendChild(newDiv);
    board.appendChild(div);
  });
}

// 삭제 모달
const modal = document.getElementById("modal");
const modalDelete = document.getElementById("modalDelete");
const modalCancelBtn = document.getElementById("modalCancelBtn");

function openModal(){ modal.classList.add("show"); document.addEventListener("keydown", handleModalKey); }
function closeModal(){ modal.classList.remove("show"); document.removeEventListener("keydown", handleModalKey); deleteTargetIndex=null; }
function handleModalKey(e){ if(e.key==="Enter") confirmDelete(); if(e.key==="Escape") closeModal(); }
function confirmDelete(){ if(deleteTargetIndex!==null){ lists.splice(deleteTargetIndex,1); save(); render(); closeModal(); } }
modalDelete.onclick = confirmDelete;
modalCancelBtn.onclick = closeModal;

// 도움말 모달
const helpModal = document.getElementById("helpModal");
const helpBtn = document.getElementById("helpBtn");
const helpCloseBtn = document.getElementById("helpCloseBtn");

function openHelp(){ helpModal.classList.add("show"); document.addEventListener("keydown", handleHelpKey); }
function closeHelp(){ helpModal.classList.remove("show"); document.removeEventListener("keydown", handleHelpKey); }
function handleHelpKey(e){ if(e.key==="Escape") closeHelp(); }

helpBtn.addEventListener("click", openHelp);
helpCloseBtn.addEventListener("click", closeHelp);

// 상단 버튼
document.getElementById("addList").addEventListener("click", ()=>{ lists.push({title:"체크 리스트", items:[]}); save(); render(); });
document.getElementById("exportJson").addEventListener("click", ()=>{
  const blob = new Blob([JSON.stringify(lists,null,2)],{type:"application/json"});
  const a = document.createElement("a"); a.href = URL.createObjectURL(blob); a.download = "lists.json"; a.click();
});
document.getElementById("importJson").addEventListener("click", ()=>{
  const inp = document.createElement("input"); inp.type="file"; inp.accept=".json";
  inp.onchange = e=>{
    const file = e.target.files[0]; const reader = new FileReader();
    reader.onload = ()=>{ lists = JSON.parse(reader.result); save(); render(); };
    reader.readAsText(file);
  }; inp.click();
});
document.getElementById("exportTxt").addEventListener("click", ()=>{
  let txt=""; 
  lists.forEach(l=>{
    txt += `[${l.title}]\n`; 
    l.items.forEach(i=>{ txt += `- [${i.done?"x":" "}] ${i.text}\n`; });
    txt += "\n";
  });
  const blob = new Blob([txt],{type:"text/plain"});
  const a = document.createElement("a"); a.href = URL.createObjectURL(blob); a.download = "lists.txt"; a.click();
});

render();
setInterval(updateTime,1000);
});
</script>
</body>
</html>
