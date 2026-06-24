
/* ===================== 무한순환 전략 시뮬레이터 ===================== */
const KEY = 'infinite_cycle_qqq_sim_v1';

const CFG = {
  total: 1000,        // 총 계좌 (USDT)
  lev: 3,             // 레버리지 (실질 3배)
  entryUp: 1,         // 상승일 진입 % (운용자산 대비)
  entryDn: 3,         // 하락일 진입 %
  storeUp: 10,        // 보관 발동: 평단가 대비 +10%
  storePct: 10,       // 보관 시 트레이딩의 10% → 펀딩
  injDn: 15,          // 투입 발동: 직전 보관/고점 대비 -15%
  injPct: 50,         // 투입 시 펀딩의 50% → 트레이딩
  startPx: 400,       // QQQ 시작가
  maxDeploy: 0.04     // 한 종목 일일 최대 투입 비중 안전장치
};

let S = null;

/* ---------- 상태 초기화 ---------- */
function fresh(){
  return {
    day: 0,
    px: CFG.startPx,
    prevPx: CFG.startPx,
    seed: Math.round(Math.random()*1e9), // 가격 난수 시드
    funding: CFG.total,   // 펀딩(현금) 계좌
    margin: 0,            // 트레이딩 투입 증거금
    qty: 0,              // 보유 수량(레버리지 노셔널 단위)
    avg: 0,              // 평단가
    base: CFG.startPx,    // 보관 기준선(평단가 기준 갱신)
    high: CFG.startPx,    // 직전 고점(투입 기준)
    realized: 0,         // 확정 손익
    lastStorePx: null,    // 마지막 보관 시점 QQQ 가격
    lastInjPx: null,      // 마지막 투입 시점 QQQ 가격
    storeCnt: 0, injCnt: 0,
    started: false,
    log: []              // 채팅 로그(재시작 복구용)
  };
}

/* ---------- 저장 / 불러오기 ---------- */
function save(){ localStorage.setItem(KEY, JSON.stringify(S)); }
function load(){
  try{ const r = localStorage.getItem(KEY); return r? JSON.parse(r): null; }catch(e){ return null; }
}

/* ---------- 결정적 난수(시드) : 같은 시드면 같은 경로 ---------- */
function rng(n){
  // mulberry32
  let t = (S.seed + n*0x6D2B79F5) >>> 0;
  t = Math.imul(t ^ (t>>>15), t|1);
  t ^= t + Math.imul(t ^ (t>>>7), t|61);
  return ((t ^ (t>>>14))>>>0)/4294967296;
}

/* ===================== 포지션 계산 ===================== */
function posValue(){ return S.qty * S.px; }              // 현재 노셔널 평가액
function costBasis(){ return S.qty * S.avg; }
function uPnL(){ return S.qty * (S.px - S.avg); }         // 미실현 손익
function tradingEquity(){ return S.margin + uPnL(); }    // 트레이딩 계좌 평가액
function totalAssets(){ return S.funding + tradingEquity(); }

/* ---------- 매수(분할/투입 공용) : marginAmt 만큼 증거금 투입 ---------- */
function buy(marginAmt){
  if(marginAmt <= 0 || S.funding < 0.01) return 0;
  marginAmt = Math.min(marginAmt, S.funding);
  const notional = marginAmt * CFG.lev;
  const addQty = notional / S.px;
  const newQty = S.qty + addQty;
  S.avg = newQty>0 ? (costBasis() + notional) / newQty : S.px;
  S.qty = newQty;
  S.funding -= marginAmt;
  S.margin  += marginAmt;
  return marginAmt;
}

/* ---------- 보관 : 트레이딩 평가액의 storePct% 를 펀딩으로 (부분 익절) ---------- */
function store(){
  if(S.qty <= 0) return 0;
  const f = CFG.storePct/100;
  const movedEquity = tradingEquity()*f;
  const realizedPart = uPnL()*f;
  S.qty    *= (1-f);
  S.margin *= (1-f);
  S.funding += movedEquity;
  S.realized += realizedPart;
  S.storeCnt++;
  S.lastStorePx = S.px;
  S.base = S.px;          // 기준선 갱신
  if(S.px > S.high) S.high = S.px;
  return movedEquity;
}

/* ---------- 투입 : 펀딩의 injPct% 를 트레이딩으로 매수 ---------- */
function inject(){
  const amt = S.funding * CFG.injPct/100;
  const used = buy(amt);
  if(used>0){ S.injCnt++; S.lastInjPx = S.px; S.base = S.px; }
  return used;
}

/* ===================== 하루 진행 ===================== */
function step(){
  S.day++;
  S.prevPx = S.px;
  // 가격: 결정적 랜덤워크(약한 상승 드리프트)
  const r = rng(S.day);
  const drift = 0.0012, vol = 0.022;
  let chg = drift + (r-0.5)*2*vol;
  S.px = Math.max(40, +(S.px*(1+chg)).toFixed(2));

  const isUp = S.px >= S.prevPx;
  const msgs = [];

  // 1) 매일 분할매수 (운용자산 대비 %)
  const pct = (isUp? CFG.entryUp: CFG.entryDn)/100;
  const baseAmt = totalAssets()*pct;
  const used = buy(baseAmt);
  if(used>0.01){
    msgs.push({cls:'buy', icon:'🟦',
      txt:`[QQQ] ${isUp?'상승일':'하락일'} 분할매수 · <b>${fmt(used)} USDT</b> 투입(증거금)\n현재가 <b>$${S.px.toFixed(2)}</b> · 평단가 <b>$${S.avg.toFixed(2)}</b> · 노셔널 ${fmt(posValue())}`});
  }

  // 2) 보관 체크: 평단가 대비 +storeUp%
  if(S.avg>0 && S.px >= S.avg*(1+CFG.storeUp/100) && S.px >= S.base*(1+CFG.storeUp/100)){
    const moved = store();
    msgs.push({cls:'store', icon:'🟧',
      txt:`<span class="up">▲ 보관 실행 (평단가 +${CFG.storeUp}% 도달)</span>\n현재가 <b>$${S.px.toFixed(2)}</b> · 트레이딩의 ${CFG.storePct}% = <b>${fmt(moved)} USDT</b> → 펀딩(현금)\n확정익절 누적 <span class="pos">+${fmt(S.realized)}</span> · 기준가 갱신 $${S.px.toFixed(2)}`});
  }

  // 3) 투입 체크: 고점 대비 -injDn%
  if(S.px > S.high) S.high = S.px;
  if(S.high>0 && S.px <= S.high*(1-CFG.injDn/100) && S.funding > 1){
    const used2 = inject();
    if(used2>0){
      msgs.push({cls:'inj', icon:'🟩',
        txt:`<span class="dn">▼ 투입 실행 (고점 −${CFG.injDn}% 하락)</span>\n현재가 <b>$${S.px.toFixed(2)}</b> · 펀딩의 ${CFG.injPct}% = <b>${fmt(used2)} USDT</b> → 트레이딩\n평단가 <b>$${S.avg.toFixed(2)}</b>로 낮춤 · 기준가 $${S.px.toFixed(2)}`});
      S.high = S.px;
    }
  }

  // 출력
  msgs.forEach(m=> botMsg(m.txt, m.cls));
  // 7일마다 자동 현황
  if(S.day % 7 === 0) statusMsg('일일');
  updTicker();
  save();
}

/* ===================== 출력 헬퍼 ===================== */
function fmt(n){ return (Math.round(n*10)/10).toFixed(1); }
function nowTime(){ const d=new Date(); return ('0'+d.getHours()).slice(-2)+':'+('0'+d.getMinutes()).slice(-2); }
function dayTag(){ return 'Day '+S.day; }

function pushLog(html, cls){ S.log.push({html, cls}); if(S.log.length>400) S.log.shift(); }

function render(html, cls){
  const chat = document.getElementById('chat');
  const el = document.createElement('div');
  el.className = 'b '+cls;
  el.innerHTML = html;
  chat.appendChild(el);
  chat.scrollTop = chat.scrollHeight;
}
function botMsg(txt, extra){
  const html = txt + `<div class="tm">${dayTag()} · ${nowTime()}</div>`;
  const cls = 'bot '+(extra||'');
  pushLog(html, cls); render(html, cls); save();
}
function usrMsg(txt){ const html=`<span class="mono">${txt}</span>`; pushLog(html,'usr'); render(html,'usr'); }
function sysMsg(txt){ pushLog(txt,'sys'); render(txt,'sys'); }
function divMsg(txt){ pushLog(txt,'div'); render(txt,'div'); }

/* ---------- 현황 카드 ---------- */
function statusCard(){
  const pnl = uPnL();
  const pnlPct = S.avg>0 ? (S.px/S.avg-1)*100 : 0;
  const held = S.qty>0;
  return `<div class="totbox"><b class="big">총자산 ${fmt(totalAssets())} USDT</b>
<span class="lab">(초기 ${CFG.total.toFixed(1)} · 확정손익 <span class="${S.realized>=0?'pos':'neg'}">${S.realized>=0?'+':''}${fmt(S.realized)}</span> · 평가손익 <span class="${pnl>=0?'pos':'neg'}">${pnl>=0?'+':''}${fmt(pnl)}</span>)</span></div>
<div class="acct">
  <div class="acc t"><div class="l">트레이딩 계좌</div><div class="v">${fmt(tradingEquity())}</div></div>
  <div class="acc f"><div class="l">펀딩(현금) 계좌</div><div class="v">${fmt(S.funding)}</div></div>
</div>
<div class="grid2">
  <div class="k">포지션</div><div>${held?`보유 · 노셔널 ${fmt(posValue())}`:'없음(대기)'}</div>
  <div class="k">평단가</div><div>${held?'$'+S.avg.toFixed(2):'—'}</div>
  <div class="k">현재가</div><div>$${S.px.toFixed(2)}</div>
  <div class="k">포지션 PnL</div><div class="${pnl>=0?'pos':'neg'}">${held?`${pnl>=0?'+':''}${fmt(pnl)} USDT (${pnlPct>=0?'+':''}${pnlPct.toFixed(1)}%)`:'—'}</div>
  <div class="k">확정 익손실</div><div class="${S.realized>=0?'pos':'neg'}">${S.realized>=0?'+':''}${fmt(S.realized)} USDT</div>
  <div class="k">마지막 보관가</div><div>${S.lastStorePx?'$'+S.lastStorePx.toFixed(2)+` <span class="flag">${S.storeCnt}회</span>`:'—'}</div>
  <div class="k">마지막 투입가</div><div>${S.lastInjPx?'$'+S.lastInjPx.toFixed(2)+` <span class="flag">${S.injCnt}회</span>`:'—'}</div>
</div>`;
}
function statusMsg(kind){
  const head = kind==='재시작'
    ? `<span class="hd">🔄 재시작 현황 · QQQ</span>선택: QQQ(나스닥100) · 3배 · 상승${CFG.entryUp}%/하락${CFG.entryDn}% · 보관+${CFG.storeUp}%→${CFG.storePct}% · 투입−${CFG.injDn}%→${CFG.injPct}%`
    : `<span class="hd">📅 ${kind} 현황 · ${dayTag()}</span>`;
  botMsg(head + statusCard());
}

/* ---------- 티커 ---------- */
function updTicker(){
  document.getElementById('pxLbl').textContent = '$'+S.px.toFixed(2);
  document.getElementById('dayLbl').textContent = 'Day '+S.day;
  const d = S.px - S.prevPx, p = S.prevPx? d/S.prevPx*100:0;
  const c = document.getElementById('chgLbl');
  c.textContent = (d>=0?'▲ +':'▼ ')+d.toFixed(2)+' ('+(p>=0?'+':'')+p.toFixed(2)+'%)';
  c.className = 'chg '+(d>=0?'up':'dn');
}

/* ===================== 초기 구동 ===================== */
function bootSetup(){
  divMsg('① 설정');
  usrMsg('/start');
  botMsg(`<span class="hd">∞ 무한순환 봇 설정</span>총 계좌 <b>1,000 USDT</b> · 단일 종목 <b>QQQ(나스닥100 ETF)</b> · 중복투자 없음.`);
  botMsg(`<span class="hd">📋 설정 요약</span>· 종목 <b>QQQ</b> · 레버리지 <b>${CFG.lev}배</b>(실질)
· 분할매수 상승일 <b>${CFG.entryUp}%</b> / 하락일 <b>${CFG.entryDn}%</b> (운용자산 대비)
· 보관 평단가 <b>+${CFG.storeUp}%</b>마다 트레이딩 <b>${CFG.storePct}%</b> → 펀딩
· 투입 고점 <b>−${CFG.injDn}%</b>마다 펀딩 <b>${CFG.injPct}%</b> → 트레이딩`);
  divMsg('② 구동 시작 — "다음날"을 눌러 진행');
  S.started = true;
  statusMsg('초기');
  updTicker();
  save();
}

/* ---------- 재시작 복구 ---------- */
function replayLog(){
  const chat = document.getElementById('chat');
  chat.innerHTML = '';
  S.log.forEach(m=> render(m.html, m.cls));
}

/* ===================== 컨트롤 ===================== */
let auto = null;
function stopAuto(){ if(auto){ clearInterval(auto); auto=null; document.getElementById('autoBtn').textContent='⏩ 자동재생'; } }

document.getElementById('nextBtn').onclick = ()=>{ stopAuto(); step(); };
document.getElementById('skipBtn').onclick = ()=>{ stopAuto(); for(let i=0;i<5;i++) step(); };
document.getElementById('statusBtn').onclick = ()=>{ usrMsg('/현황'); statusMsg('요청'); };
document.getElementById('autoBtn').onclick = ()=>{
  if(auto){ stopAuto(); return; }
  document.getElementById('autoBtn').textContent='⏸ 정지';
  auto = setInterval(step, 700);
};
document.getElementById('restartBtn').onclick = ()=>{
  stopAuto();
  const saved = load();
  if(!saved){ alert('저장된 데이터가 없습니다.'); return; }
  S = saved;
  replayLog();
  sysMsg('☁️ Render 재시작 → 저장 상태 로드 → 아래 현황 발송 후 이어서 구동');
  statusMsg('재시작');
  sysMsg('✓ 잔고·평단가·기준가·확정손익 그대로 복구 → 끊김 없이 재개');
  updTicker();
  save();
};
document.getElementById('resetBtn').onclick = ()=>{
  if(!confirm('모든 데이터를 지우고 처음부터 시작할까요?')) return;
  stopAuto();
  localStorage.removeItem(KEY);
  S = fresh();
  document.getElementById('chat').innerHTML='';
  bootSetup();
};

/* ===================== 시작 ===================== */
(function init(){
  const saved = load();
  if(saved && saved.started){
    S = saved;
    replayLog();
    divMsg('— 저장된 세션 복구됨 —');
    statusMsg('재시작');
    sysMsg('✓ 이전 상태에서 이어서 진행합니다. "다음날"을 누르세요.');
    updTicker();
  }else{
    S = fresh();
    bootSetup();
  }
})();
