const CFG={total:1000,lev:3,entryUp:1,entryDn:3,storeUp:10,storePct:10,injDn:15,injPct:50,startPx:400};
let S={day:0,px:400,prevPx:400,seed:12345,funding:1000,margin:0,qty:0,avg:0,base:400,high:400,realized:0,lastStorePx:null,lastInjPx:null,storeCnt:0,injCnt:0};
function rng(n){let t=(S.seed+n*0x6D2B79F5)>>>0;t=Math.imul(t^(t>>>15),t|1);t^=t+Math.imul(t^(t>>>7),t|61);return((t^(t>>>14))>>>0)/4294967296;}
const uPnL=()=>S.qty*(S.px-S.avg);
const costBasis=()=>S.qty*S.avg;
const tradingEquity=()=>S.margin+uPnL();
const totalAssets=()=>S.funding+tradingEquity();
const posValue=()=>S.qty*S.px;
function buy(a){if(a<=0||S.funding<0.01)return 0;a=Math.min(a,S.funding);const not=a*CFG.lev,aq=not/S.px,nq=S.qty+aq;S.avg=nq>0?(costBasis()+not)/nq:S.px;S.qty=nq;S.funding-=a;S.margin+=a;return a;}
function store(){if(S.qty<=0)return 0;const f=CFG.storePct/100;const me=tradingEquity()*f;const rp=uPnL()*f;S.qty*=(1-f);S.margin*=(1-f);S.funding+=me;S.realized+=rp;S.storeCnt++;S.lastStorePx=S.px;S.base=S.px;if(S.px>S.high)S.high=S.px;return me;}
function inject(){const amt=S.funding*CFG.injPct/100;const u=buy(amt);if(u>0){S.injCnt++;S.lastInjPx=S.px;S.base=S.px;}return u;}
function step(){S.day++;S.prevPx=S.px;const r=rng(S.day);let chg=0.0012+(r-0.5)*2*0.022;S.px=Math.max(40,+(S.px*(1+chg)).toFixed(2));const isUp=S.px>=S.prevPx;
 const pct=(isUp?CFG.entryUp:CFG.entryDn)/100;buy(totalAssets()*pct);
 if(S.avg>0&&S.px>=S.avg*(1+CFG.storeUp/100)&&S.px>=S.base*(1+CFG.storeUp/100))store();
 if(S.px>S.high)S.high=S.px;
 if(S.high>0&&S.px<=S.high*(1-CFG.injDn/100)&&S.funding>1){if(inject()>0)S.high=S.px;}
}
// invariant check: total assets should only change due to price PnL, never created/destroyed by transfers
let bad=0;
for(let i=0;i<400;i++){
  const before=totalAssets();
  const pxBefore=S.px;
  step();
  // recompute "before" at new price to isolate transfer effects is complex; instead check no NaN and funding>=0, margin>=0
  if(isNaN(totalAssets())||S.funding<-0.01||S.margin<-0.01||S.qty<-1e-9){bad++; if(bad<5)console.log('BAD day',S.day,JSON.stringify({f:S.funding,m:S.margin,q:S.qty,avg:S.avg}));}
}
console.log('days:',S.day);
console.log('px:',S.px.toFixed(2));
console.log('funding:',S.funding.toFixed(2),'margin:',S.margin.toFixed(2),'qty:',S.qty.toFixed(4),'avg:',S.avg.toFixed(2));
console.log('tradingEquity:',tradingEquity().toFixed(2));
console.log('uPnL:',uPnL().toFixed(2),'realized:',S.realized.toFixed(2));
console.log('TOTAL ASSETS:',totalAssets().toFixed(2));
console.log('store count:',S.storeCnt,'inject count:',S.injCnt,'lastStore:',S.lastStorePx,'lastInj:',S.lastInjPx);
console.log('bad invariants:',bad);
// Transfer conservation test: store() must not change total assets at constant price
S2check();
function S2check(){
  S={day:0,px:500,prevPx:500,seed:1,funding:300,margin:200,qty:(200*3)/450,avg:450,base:500,high:500,realized:0,lastStorePx:null,lastInjPx:null,storeCnt:0,injCnt:0};
  const t0=totalAssets();store();const t1=totalAssets();
  console.log('STORE conservation (should be ~equal):',t0.toFixed(4),'->',t1.toFixed(4),'diff',(t1-t0).toExponential(2));
  const t2=totalAssets();inject();const t3=totalAssets();
  console.log('INJECT conservation (should be ~equal):',t2.toFixed(4),'->',t3.toFixed(4),'diff',(t3-t2).toExponential(2));
}
