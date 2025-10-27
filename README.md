# グッズリスト作成用手順書
以下の手順で Google フォーム、スプレッドシート側設定を行ってください。

# 目次
① Google フォーム作成<br>
② フォルダの共有権限を変更<br>
③ フォームに回答を投稿しフォーム記録用のスプレッドシートを作成。<br>
④ フォーム記録用のスプレッドシートに Apps script を登録。<br>
⑥ デプロイ<br>

【以下Apps script用コード】
```
// ===========================
// ▼ 管理用設定（ここだけ変更すればOK）
// ===========================
const CONFIG = {
  SHEET_NAME: 'シート名',                  // スプレッドシートのシート名
};
// ===========================

function doGet() {
  SpreadsheetApp.flush(); // 🔸 これを追加！最新のシート状態を反映（削除済み行も確実に除外）

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(CONFIG.SHEET_NAME);
  if (!sheet) return HtmlService.createHtmlOutput('シートが見つかりません。');

  const data = sheet.getDataRange().getValues();
  const headers = data.shift();

  function toDisplayUrl(url) {
    if (!url) return '';
    const match = url.match(/id=([a-zA-Z0-9_-]+)/);
    // Drive URL → 直接表示用URL
    return match ? `https://lh3.googleusercontent.com/d/${match[1]}` : url;
  }

  // スプレッドシート行データ → images配列に変換
const images = data.map(row => {
  const obj = {};
  headers.forEach((h,i)=> obj[h]=row[i]);

  // 入手日をYYYY年MM月DD日に変換
  let dateStr = '';
  if(obj['入手日']){
    const date = new Date(obj['入手日']);
    if(!isNaN(date.getTime())){
      const y = date.getFullYear();
      const m = ('0'+(date.getMonth()+1)).slice(-2);
      const d = ('0'+date.getDate()).slice(-2);
      dateStr = `${y}年${m}月${d}日`;
    }
  }

  return {
    src: toDisplayUrl(obj['画像URL']),
    caption: obj['アイテムタイトル'] || '(キャプションなし)',
    date: dateStr,
    place: obj['入手箇所'] || '',
    free: obj['無料/有料'] || '',
    comment: obj['コメント'] || ''
  };
}).filter(img=>img.src);

  const html = `
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- ヘッダー -->
<header id="gallery-title">
  EXPO 2025 関西大阪万博グッズリスト
</header>

<!-- フィルター -->
<div id="filter-container">
  <input type="date" id="filter-date">
  <select id="filter-place"><option value="">入手箇所</option></select>
  <select id="filter-free">
    <option value="">無料/有料</option>
    <option value="無料">無料</option>
    <option value="有料">有料</option>
  </select>
  <button id="filter-btn">絞り込み</button>
  <button id="filter-reset">リセット</button>
</div>

<!-- ギャラリー -->
<div id="gallery-container">
  <div id="gallery"></div>
</div>

<!-- Lightbox Stage1 -->
<div id="lightbox-stage1" style="display:none; ...">
  <div id="lb1-image-box">
    <button id="lb1-prev">&#10094;</button>
    <img id="lb1-img">
    <button id="lb1-next">&#10095;</button>
  </div>
  <div class="lightbox-info">
    <p><strong>アイテムタイトル:</strong> <span id="lb1-caption"></span></p>
    <p><strong>入手日:</strong> <span id="lb1-date"></span></p>
    <p><strong>入手箇所:</strong> <span id="lb1-place"></span></p>
    <p><strong>無料/有料:</strong> <span id="lb1-free"></span></p>

    <!-- コメントだけスクロール -->
    <div id="lb1-comment-box">
    <p><strong>コメント:</strong> <span id="lb1-comment"></span></p>
  </div>
</div>
</div>

<!-- Lightbox Stage2 -->
<div id="lightbox-stage2">
  <img id="lb2-img">
  <button id="lb2-close-btn">×</button>
</div>

<style>
/* 全体 */
body {
  font-family: 'Arial', sans-serif;
  background: #fff8f0;
  margin: 0; padding: 0;
}

/* フィルター */
#filter-container {
  position: sticky;       /* スクロールしても固定 */
  top: 0;                 /* 上部に固定 */
  z-index: 1000;          /* ギャラリーより前面に */
  margin: 0 auto;         /* 中央寄せ */
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: center;
  background: #ffeedd;
  padding: 12px;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);
}
#filter-container input, #filter-container select {
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid #ffddcc;
  background: #fff5f0;
}
#filter-container button {
  padding: 6px 12px;
  border-radius: 8px;
  border: none;
  background: #ffbfa5;
  color: white;
  font-weight: bold;
  cursor: pointer;
  transition: background 0.2s;
}
#filter-container button:hover { background: #ffa07a; }

/* ギャラリー */
#gallery-title {
  font-family: 'Comic Sans MS', 'ヒラギノ角ゴ Pro', 'Arial', sans-serif; /* ポップ系フォント */
  font-size: 2rem;       /* 大きめサイズ */
  text-align: center;
  color: #ff4500;        /* 目立つオレンジ色 */
  margin: 20px 0 10px 0;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.2); /* 少し立体感 */
}
#gallery-container {
  height: calc(100vh - 100px); /* 元々の高さ計算 */
  overflow: auto;
  display: flex;
  justify-content: center;
  padding: 10px;
  padding-top: 80px; /* フィルター高さ分を確保 */
}
#gallery { display:flex; flex-wrap:wrap; justify-content:flex-start; gap:8px; align-content:flex-start; }
.gallery-item {
  border-radius: 10px;
  box-shadow:0 4px 8px rgba(0,0,0,0.1);
  overflow:hidden;
  display:flex;
  justify-content:center;
  align-items:center;
  background:#fff0e5;
  transition: transform 0.2s;
}
.gallery-item:hover { transform: scale(1.05); }
@media(min-width:769px) { .gallery-item { width:150px; height:150px; flex:0 0 auto; } }
@media(max-width:768px) { .gallery-item { width: calc((100% - 12px)/3); aspect-ratio:1/1; flex:0 0 auto; } }
.gallery-item img { width:100%; height:100%; object-fit:cover; border-radius:10px; cursor:pointer; transition:transform 0.2s; }
.gallery-item img:hover { transform:scale(1.05); }

/* Lightbox Stage1 */
#lightbox-stage1 {
  display:none; position:fixed; inset:0; background:rgba(0,0,0,0.85);
  justify-content:center; align-items:center; z-index:9998; display:flex; padding:20px;
}
#lb1-image-box {
  max-width:60%; max-height:80%; display:flex; justify-content:center; align-items:center; position:relative;
  background: rgba(255,240,230,0.3); border-radius:16px; box-shadow:0 8px 20px rgba(0,0,0,0.4); padding:10px;
}
#lb1-img {
  max-width:100%; max-height:100%; object-fit:contain; cursor:pointer;
  border-radius:10px; border:3px solid rgba(255,200,170,0.6); box-shadow:0 4px 10px rgba(0,0,0,0.3);
  transition: transform 0.2s, box-shadow 0.2s;
}
#lb1-comment-box p {
  white-space: pre-wrap;  /* 改行を反映 */
  margin: 0;
}
#lb1-img:hover { transform:scale(1.02); box-shadow:0 6px 16px rgba(0,0,0,0.35); }

.lightbox-info {
  margin-left: 20px;
  max-width: 35%;
  padding: 50px 20px 20px 20px;
  background: rgba(255,190,150,0.85);
  border-radius: 16px;
  color: white;
  box-shadow: 0 8px 20px rgba(0,0,0,0.4);
  line-height: 1.2;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

#lb1-comment-box {
  max-height: 150px;        /* コメント欄の最大高さ */
  overflow-y: auto;          /* 縦スクロール */
  margin-top: 8px;
  padding-right: 4px;        /* スクロールバー余白 */
}

/* スクロールバーを丸く可愛く */
#lb1-comment-box::-webkit-scrollbar {
  width: 8px;
  border-radius: 10px;
  background: #ffdeda;
}

#lb1-comment-box::-webkit-scrollbar-thumb {
  background-color: #ff8c42;
  border-radius: 10px;
  border: 2px solid #ffdeda;
}

#lb1-comment-box::-webkit-scrollbar-track {
  background: #ffdeda;
  border-radius: 10px;
}
.lightbox-info:hover { transform:scale(1.02); box-shadow:0 10px 24px rgba(0,0,0,0.45); }

/* Lightbox Stage1ボタン */
#lb1-prev, #lb1-next {
  font-size:28px; background:#ffbfa5; border:none; border-radius:50%; color:white; cursor:pointer;
  width:40px; height:40px; display:flex; align-items:center; justify-content:center; transition:background 0.2s, transform 0.2s;
}
#lb1-prev:hover, #lb1-next:hover { background:#ffa07a; transform:scale(1.1); }

/* Lightbox Stage2 */
#lightbox-stage2 { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.95);
  justify-content:center; align-items:center; z-index:9999; overflow:auto;
}
#lb2-img { cursor:grab; max-width:90%; max-height:90%; object-fit:contain;
  border-radius:12px; border:3px solid rgba(255,200,170,0.6); box-shadow:0 8px 20px rgba(0,0,0,0.4);
  transition: transform 0.2s, box-shadow 0.2s;
}
#lb2-close-btn { position:absolute; top:20px; right:20px; font-size:28px; background:#ffbfa5; border:none; border-radius:50%; color:white; cursor:pointer;
  width:40px; height:40px; display:flex; align-items:center; justify-content:center; transition: background 0.2s, transform 0.2s;
}
#lb2-close-btn:hover { background:#ffa07a; transform:scale(1.1); }
</style>

<script>
const images = ${JSON.stringify(images)};
const gallery = document.getElementById('gallery');
let currentIndex=0, lb2Scale=1, lb2OriginX=0, lb2OriginY=0, isDragging=false, startX=0, startY=0;

function createItem(i,imgData){
  const div=document.createElement('div'); div.className='gallery-item';
  const img=document.createElement('img'); img.dataset.index=i; img.onclick=()=>openStage1(i);
  div.appendChild(img); gallery.appendChild(div);
  const observer=new IntersectionObserver(entries=>{ entries.forEach(entry=>{ if(entry.isIntersecting){ if(!entry.target.src) entry.target.src=imgData.src; observer.unobserve(entry.target); } }); },{rootMargin:'300px'});
  observer.observe(img);
}

function renderItems(list){ gallery.innerHTML=''; list.forEach((img,i)=>createItem(i,img)); }
renderItems(images);

function populatePlaceFilter(){
  const placeSet=new Set(images.map(img=>img.place).filter(p=>p));
  const placeSelect=document.getElementById('filter-place');
  placeSet.forEach(place=>{ const opt=document.createElement('option'); opt.value=place; opt.textContent=place; placeSelect.appendChild(opt); });
}
populatePlaceFilter();

function applyFilter(){
  const dateVal=document.getElementById('filter-date').value;
  const placeVal=document.getElementById('filter-place').value;
  const freeVal=document.getElementById('filter-free').value;
  const filtered=images.filter(img=>(!dateVal||img.date===dateVal)&&(!placeVal||img.place===placeVal)&&(!freeVal||img.free===freeVal));
  renderItems(filtered);
}
document.getElementById('filter-btn').onclick=applyFilter;
document.getElementById('filter-reset').onclick=()=>{ document.getElementById('filter-date').value=''; document.getElementById('filter-place').value=''; document.getElementById('filter-free').value=''; renderItems(images); };

// Lightbox Stage1
const lb1=document.getElementById('lightbox-stage1'); const lb1Img=document.getElementById('lb1-img');
function openStage1(i){ currentIndex=i; const imgData=images[i]; lb1.style.display='flex'; lb1Img.src=imgData.src;
document.getElementById('lb1-caption').textContent=imgData.caption;
document.getElementById('lb1-date').textContent=imgData.date;
document.getElementById('lb1-place').textContent=imgData.place;
document.getElementById('lb1-free').textContent=imgData.free;
document.getElementById('lb1-comment').textContent=imgData.comment; }
lb1Img.onclick=()=>{ openStage2(currentIndex); };
document.getElementById('lb1-prev').onclick=e=>{ e.stopPropagation(); currentIndex=(currentIndex-1+images.length)%images.length; openStage1(currentIndex);}
document.getElementById('lb1-next').onclick=e=>{ e.stopPropagation(); currentIndex=(currentIndex+1)%images.length; openStage1(currentIndex);}
lb1.onclick=()=>{ lb1.style.display='none'; };

// Lightbox Stage2
const lb2=document.getElementById('lightbox-stage2'); const lb2Img=document.getElementById('lb2-img'); const lb2Close=document.getElementById('lb2-close-btn');
function openStage2(i){ const imgData=images[i]; lb2.style.display='flex'; lb2Img.src=imgData.src;
const scaleX=window.innerWidth*0.8/lb2Img.naturalWidth; const scaleY=window.innerHeight*0.8/lb2Img.naturalHeight;
lb2Scale=Math.min(scaleX,scaleY,1); lb2OriginX=0; lb2OriginY=0; lb2Img.style.transform=\`scale(\${lb2Scale}) translate(0px,0px)\`; enableZoomPan(); }
lb2Close.onclick=e=>{ e.stopPropagation(); lb2.style.display='none'; };

function enableZoomPan(){
  lb2Img.style.cursor='grab';
  lb2Img.onwheel=e=>{ e.preventDefault(); const rect=lb2Img.getBoundingClientRect(); const offsetX=e.clientX-rect.left; const offsetY=e.clientY-rect.top; const delta=e.deltaY>0?-0.1:0.1; lb2Scale=Math.min(Math.max(0.1,lb2Scale+delta),10); lb2Img.style.transformOrigin=offsetX+'px '+offsetY+'px'; lb2Img.style.transform=\`scale(\${lb2Scale}) translate(\${lb2OriginX}px,\${lb2OriginY}px)\`; };
  lb2Img.onmousedown=e=>{ e.preventDefault(); isDragging=true; startX=e.clientX-lb2OriginX; startY=e.clientY-lb2OriginY; lb2Img.style.cursor='grabbing'; };
  window.onmousemove=e=>{ if(!isDragging)return; lb2OriginX=e.clientX-startX; lb2OriginY=e.clientY-startY; lb2Img.style.transform=\`scale(\${lb2Scale}) translate(\${lb2OriginX}px,\${lb2OriginY}px)\`; };
  window.onmouseup=()=>{ isDragging=false; lb2Img.style.cursor='grab'; };
}

document.addEventListener('keydown', e=>{
  if(e.key==='Escape'){ if(lb2.style.display==='flex'){ lb2.style.display='none'; } else if(lb1.style.display==='flex'){ lb1.style.display='none'; } }
  if(lb1.style.display==='flex'){ if(e.key==='ArrowLeft'){ currentIndex=(currentIndex-1+images.length)%images.length; openStage1(currentIndex); } else if(e.key==='ArrowRight'){ currentIndex=(currentIndex+1)%images.length; openStage1(currentIndex); } }
});

</script>
`;

  return HtmlService.createHtmlOutput(html)
                    .setTitle('グッズギャラリー')
                    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}
```
