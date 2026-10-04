<!DOCTYPE html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Subject Tracker</title>
<style>
:root{--bg:#f6f6f4;--card:#fff;--tx:#1a1a1a;--mu:#777;--ac:#2563eb;--ok:#16a34a;--bad:#dc2626;--bd:#ddd;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#121212;--card:#1e1e1e;--tx:#eee;--mu:#999;--ac:#60a5fa;--ok:#4ade80;--bad:#f87171;--bd:#333}}
:root[data-theme="dark"]{--bg:#121212;--card:#1e1e1e;--tx:#eee;--mu:#999;--ac:#60a5fa;--ok:#4ade80;--bad:#f87171;--bd:#333}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--tx);font:16px system-ui,-apple-system,Segoe UI,sans-serif}
.w{max-width:640px;margin:0 auto;padding:12px}
.tabs{display:flex;gap:6px;overflow-x:auto;padding-bottom:6px}
.tab{flex:0 0 auto;padding:8px 12px;border:1px solid var(--bd);border-radius:20px;background:var(--card);color:var(--tx);cursor:pointer;font-size:14px}
.tab.on{background:var(--ac);color:#fff;border-color:var(--ac)}
.card{background:var(--card);border:1px solid var(--bd);border-radius:12px;padding:12px;margin-top:10px}
input{font:inherit;padding:8px;border:1px solid var(--bd);border-radius:8px;background:var(--bg);color:var(--tx);min-width:0}
button{font:inherit;cursor:pointer}
.row{display:flex;gap:6px;flex-wrap:wrap}
.row input[type=text]{flex:1 1 160px}
.add{background:var(--ac);color:#fff;border:0;border-radius:8px;padding:8px 14px}
.bar{height:8px;background:var(--bd);border-radius:4px;overflow:hidden;margin:8px 0 2px}
.bar i{display:block;height:100%;background:var(--ok)}
.t{display:flex;align-items:center;gap:10px;padding:8px 0;border-top:1px solid var(--bd)}
.t input{width:20px;height:20px;flex:0 0 auto}
.t .n{flex:1;min-width:0;word-break:break-word}
.t.d .n{text-decoration:line-through;color:var(--mu)}
.dt{font-size:13px;color:var(--mu);white-space:nowrap}
.dt.ov{color:var(--bad);font-weight:600}
.x{background:none;border:0;color:var(--mu);font-size:18px}
h3{margin:14px 0 0;font-size:13px;color:var(--mu);text-transform:uppercase;letter-spacing:.05em}
.sb{font-size:12px;color:var(--ac);border:1px solid var(--bd);border-radius:10px;padding:1px 8px;white-space:nowrap;max-width:30%;overflow:hidden;text-overflow:ellipsis}
.nm{font-weight:600;font-size:18px;width:100%;border:0;background:transparent;padding:0}
</style></head><body><div class="w">
<div class="tabs" id="tabs"></div>
<div class="card"><div class="nm" id="ttl" style="display:none">All subjects, by due date</div><input class="nm" id="nm" aria-label="Subject name">
<div class="bar"><i id="pb"></i></div><div class="dt" id="ps"></div></div>
<div class="card" id="addc"><div class="row"><input type="text" id="tt" placeholder="New task"><input type="date" id="td"><button class="add" id="ad">Add</button></div></div>
<div class="card"><h3 style="margin-top:0">To do</h3><div id="todo"></div><h3>Done (kept for revision)</h3><div id="done"></div></div>
</div>
<script>
var K="trk1",S,cur=-1;
function load(){try{S=JSON.parse(localStorage.getItem(K))}catch(e){}
if(!S||!S.length){S=[];for(var i=1;i<=7;i++)S.push({n:"Subject "+i,t:[]})}}
var ref=null,last="",first=true;
function save(){var j=JSON.stringify(S);try{localStorage.setItem(K,j)}catch(e){}
if(ref){last=j;ref.set({j:j}).catch(function(){})}}
function sync(){if(!window.claude||!claude.use)return;
(async function(){try{var db=await claude.use("db"),u=await claude.use("user");if(!db||!u)return;
var id=await u.id();if(!id)return;
ref=db.collection("data/users/"+id).doc("tracker");
ref.onSnapshot(function(sn){
if(sn.exists){var j=(sn.data()||{}).j;if(j&&j!==last){last=j;try{S=JSON.parse(j);try{localStorage.setItem(K,j)}catch(e){}draw()}catch(e){}}}
else if(first){save()}
first=false},function(){})}catch(e){}})()}
function today(){var d=new Date();return d.getFullYear()+"-"+String(d.getMonth()+1).padStart(2,"0")+"-"+String(d.getDate()).padStart(2,"0")}
function row(t,si,i){var ov=!t.d&&t.due&&t.due<today();
var r=document.createElement("div");r.className="t"+(t.d?" d":"");
var c=document.createElement("input");c.type="checkbox";c.checked=t.d;c.onchange=function(){t.d=c.checked;save();draw()};
var n=document.createElement("span");n.className="n";n.textContent=t.x;
r.append(c,n);
if(cur<0){var sb=document.createElement("span");sb.className="sb";sb.textContent=S[si].n;r.append(sb)}
var d=document.createElement("span");d.className="dt"+(ov?" ov":"");d.textContent=t.due||"";
var x=document.createElement("button");x.className="x";x.textContent="\u00d7";x.setAttribute("aria-label","Delete");
x.onclick=function(){S[si].t.splice(i,1);save();draw()};
r.append(d,x);return r}
function byDue(x,y){var a=x.t.due||"9",b=y.t.due||"9";return a<b?-1:a>b?1:0}
function draw(){var all=cur<0,tb=document.getElementById("tabs");tb.innerHTML="";
var tot=0,td=0;S.forEach(function(s){tot+=s.t.length;td+=s.t.filter(function(t){return t.d}).length});
var ab=document.createElement("button");ab.className="tab"+(all?" on":"");ab.textContent="All "+td+"/"+tot;
ab.onclick=function(){cur=-1;draw()};tb.appendChild(ab);
S.forEach(function(s,i){var b=document.createElement("button");b.className="tab"+(i==cur?" on":"");
var dn=s.t.filter(function(t){return t.d}).length;b.textContent=s.n+" "+dn+"/"+s.t.length;
b.onclick=function(){cur=i;draw()};tb.appendChild(b)});
var nm=document.getElementById("nm");nm.style.display=all?"none":"";
document.getElementById("ttl").style.display=all?"":"none";
document.getElementById("addc").style.display=all?"none":"";
var items=[];
(all?S:[S[cur]]).forEach(function(s){var si=S.indexOf(s);s.t.forEach(function(t,i){items.push({si:si,i:i,t:t})})});
if(!all)nm.value=S[cur].n;
var dn=items.filter(function(o){return o.t.d}).length,p=items.length?Math.round(dn/items.length*100):0;
document.getElementById("pb").style.width=p+"%";
document.getElementById("ps").textContent=dn+" of "+items.length+" done ("+p+"%)";
var a=document.getElementById("todo"),b2=document.getElementById("done");a.innerHTML="";b2.innerHTML="";
items.filter(function(o){return !o.t.d}).sort(byDue).forEach(function(o){a.appendChild(row(o.t,o.si,o.i))});
items.filter(function(o){return o.t.d}).sort(byDue).forEach(function(o){b2.appendChild(row(o.t,o.si,o.i))})}
document.getElementById("nm").oninput=function(e){S[cur].n=e.target.value||"Subject";save()};
document.getElementById("nm").onchange=draw;
document.getElementById("ad").onclick=function(){var v=document.getElementById("tt").value.trim();if(!v)return;
S[cur].t.push({x:v,due:document.getElementById("td").value,d:false});
document.getElementById("tt").value="";save();draw()};
document.getElementById("tt").onkeydown=function(e){if(e.key=="Enter")document.getElementById("ad").click()};
load();draw();sync();
</script></body></html>
