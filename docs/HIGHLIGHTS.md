# Engineering highlights — code excerpts

Real excerpts from NetDash's private codebase (lightly trimmed), picked to show how the hard parts are solved.

## 1. Heartbeat bars that never lie (web app)

Every bucket gets exactly one state: **Up / Down / Unstable** only from real measurements, **No data** when a
monitor ran but the check is missing, **Monitor off** when nothing was checking. A single lost ping on Wi-Fi is
not "unstable", and late uploads keep their original minute.

```js
/** Samples in a laptop minute taken while the laptop itself had no network — they say nothing about the targets.
 *  v10 rows record it (lap_off). Older rows: anything that answered (router, internet, a mesh node) proves it was online. */
function lapOffline(r){
  if(r.lap_off!=null)return Math.min(r.samples||0,+r.lap_off||0);
  const nodeMax=Math.max(0,...Object.values(r.nodes||{}).map(v=>+v?.[0]||0));
  return Math.max(0,(r.samples||0)-Math.max(r.gw_ok||0,r.internet_ok||0,r.blocked||0,nodeMax));
}
function beatsFor(data,{key,n,mac,isRoot,senId,win}){
  const{start,step}=win;
  const b=Array.from({length:n},(_,i)=>({t0:start+i*step,sok:0,stot:0,lok:0,ltot:0,ok:0,tot:0,ms:[],lapOff:0,lapRows:0,rok:0,rtot:0,
    meas:new Set(),late:0,replayed:false}));
  for(const r of data.rows){const i=Math.floor((r._t-start)/step);if(i<0||i>=n)continue;const x=b[i];
    const late=r.received_at?Date.parse(r.received_at)-(r._t+60e3):0;
    if(r.collector===senId){if(isRoot&&r.samples){x.sok+=r.internet_ok||0;x.stot+=r.samples;x.meas.add(r._t);}continue;}
    x.lapRows++;
    const off=lapOffline(r);
    if(r.samples&&off>=r.samples){x.lapOff++;continue;}               // laptop off the network all minute: monitor offline
    // laptop's view of the router: it answered a ping, OR the internet worked (every packet goes through the router —
    // so it's up even while it ignores the laptop's pings). Nothing answered at all → counted as down.
    if(isRoot){const t2=Math.max(0,(r.samples||0)-off);if(!t2)continue;if(r.gw_ms!=null)x.ms.push(+r.gw_ms);
      x.lok+=Math.min(t2,Math.max(r.gw_ok||0,r.internet_ok||0));x.ltot+=t2;x.lmeas=(x.lmeas||new Set()).add(r._t);}
    else{const v=r.nodes?.[mac];if(!v||!v[1])continue;
      // v10 rows already leave offline samples out of the node counts; older rows need them taken off
      const t2=r.lap_off!=null?+v[1]:Math.max(0,v[1]-off);if(!t2)continue;
      x.ok+=Math.min(+v[0],t2);x.tot+=t2;if(v[2]!=null)x.ms.push(+v[2]);x.meas.add(r._t);}
    if(late>x.late)x.late=late;if(r.replayed)x.replayed=true;}
  // stand-in for mesh nodes until per-minute pings are stored: the router's own reads of the mesh
  const useRouter=!isRoot;                                    // per bucket, only where no pings were recorded
  if(useRouter){const nm=normMac(mac);
    for(const u of data.uh||[]){const i=Math.floor((u._t-start)/step);if(i<0||i>=n)continue;
      const v=u.n[nm];if(v===undefined)continue;b[i].rtot++;if(v)b[i].rok++;}
    // a router read covers the next few minutes (reads come every 1–5 min)
    if(step===60e3)for(let i=1;i<n;i++)if(!b[i].rtot)for(let j=1;j<=4&&i-j>=0;j++)if(b[i-j].rtot&&!b[i-j].carried){b[i].rok=b[i-j].rok?1:0;b[i].rtot=1;b[i].carried=true;break;}}
  const now=Date.now();
  return b.map(x=>{let ok=0,tot=0,src=null,meas=x.meas;
    if(isRoot){if(x.stot){ok=x.sok;tot=x.stot;src="cloud monitor";}else if(x.ltot){ok=x.lok;tot=x.ltot;meas=x.lmeas||meas;src="laptop — the cloud monitor reports ~5 min late";}}
    else if(x.tot){ok=x.ok;tot=x.tot;src="laptop pings";}
    else if(useRouter&&x.rtot){ok=x.rok;tot=x.rtot;src="router's mesh list";}
    const ms=x.ms.length?Math.round(x.ms.reduce((a,c)=>a+c,0)/x.ms.length):null;
    // minutes this bar should cover: the whole bar, except the newest one stops at the newest reported minute
    const expect=Math.max(1,Math.round((Math.min(x.t0+step,win.last||x.t0+step)-x.t0)/60e3));
    const measured=src==="router's mesh list"?expect:Math.min(expect,meas.size);
    // UP / DOWN / UNSTABLE only from real measurements. A single lost ping is normal on Wi-Fi — not "unstable".
    const missed=tot-ok,slow=ms!=null&&ms>200;
    const st=!tot?(x.lapRows>x.lapOff||x.stot?"none":"offline")
      :ok===0?"down":(missed<=1||ok/tot>=.98)&&!slow?"up":"partial";
    const why=tot?(st==="partial"?(slow&&missed<=1?`slow — ${ms} ms average`:`${missed} of ${tot} checks failed`):null)
      :st==="offline"?(x.lapOff?"the laptop couldn't reach the home network (offline, away, or the router/hotspot link blocked it) — nothing was checked":x.t0>now-6*60e3?"not reported yet":"no monitor was running (laptop asleep or off)")
      :x.t0+step>now-3*60e3?"not reported yet":"monitor was running but no check was recorded for this";
    return{t0:x.t0,t1:x.t0+step,ok,tot,ms,st,src,why,expect,measured,late:x.late,replayed:x.replayed};});
}
```

## 2. Offline-first telemetry with safe replay (Python collector)

When the internet or database is down, batches are queued on disk and replayed in order later. A batch the server
permanently rejects is dropped; one it crashes on moves to the back so it can't block the queue; a storage outage
(503) keeps everything. The rewrite is atomic, so a kill mid-write can't lose the queue.

```python
    def _flush_queue(self):
        if not os.path.exists(QUEUE_FILE):
            return
        with open(QUEUE_FILE, encoding="utf-8") as f:
            lines = [l for l in f if l.strip()]
        sent, back, crashed = 0, [], 0
        try:
            for l in lines[:QUEUE_FLUSH_MAX]:              # bounded: probing must keep running
                try:
                    b = json.loads(l)
                except ValueError:
                    sent += 1; continue                    # corrupt line: drop
                b["replayed"] = True
                try:
                    self._post(b)
                except requests.HTTPError as e:
                    code = e.response.status_code if e.response is not None else 0
                    if 400 <= code < 500 and code not in (401, 403, 408, 429):
                        print(f"[Monitor] Dropping a queued batch the server rejected ({code})")
                        sent += 1; continue
                    if code in (500, 502) and crashed < 3:
                        # the server is up but crashed on THIS batch: move it to the back so it can't block
                        # the rest of the queue; give up on it after 20 tries. (503 = storage down → keep all.)
                        crashed += 1; sent += 1
                        b["_tries"] = b.get("_tries", 0) + 1
                        if b["_tries"] < 20: back.append(json.dumps(b) + "\n")
                        else: print(f"[Monitor] Dropping a queued batch the server failed on 20 times ({code})")
                        continue
                    raise
                sent += 1
        finally:
            rest = lines[sent:] + back
            with open(QUEUE_FILE + ".tmp", "w", encoding="utf-8") as f:     # atomic: a kill mid-write can't wipe the queue
                f.writelines(rest)
            os.replace(QUEUE_FILE + ".tmp", QUEUE_FILE)
            if sent:
                print(f"[Monitor] Uploaded {sent} queued telemetry batch(es) from while offline")
```

## 3. Privacy for public viewers (server)

Visitors get a redacted copy of the data. Public IPs are removed even inside free text ("…with IP 10.x.x.x.")
while LAN addresses and version numbers survive.

```js
const isLanIp = (ip) => /^192\.168\./.test(ip) || ip === "127.0.0.1" || ip === "0.0.0.0";
const IP4 = /(?<![\d.])(?:\d{1,3}\.){3}\d{1,3}(?!\d|\.\d)/g;   // a full stop after it (end of sentence) is still an IP
const IP6 = /(?<![0-9A-Fa-f:])(?:[23][0-9A-Fa-f]{3}|fe80|f[cd][0-9A-Fa-f]{2})(?::[0-9A-Fa-f]{0,4}){2,7}(?![0-9A-Fa-f:])/gi;
const secretIp = (ip) => !isLanIp(ip) && !KEEP_IPS.has(ip);
const onlyIp = (v) => typeof v === "string" && ((/^(?:\d{1,3}\.){3}\d{1,3}$/.test(v) && secretIp(v)) || /^(?:[23][0-9A-Fa-f]{3}|fe80|f[cd][0-9A-Fa-f]{2})(?::[0-9A-Fa-f]{0,4}){2,7}$/i.test(v));
const ENUM_WORDS = new Set(["home", "isp", "external", "unknown", "monitoring", "up", "down", "wifi", "mesh", "internet", "router"]);
const reEsc = (x) => x.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
// "ISP gateway (38.1.2.3) reachable" → "ISP gateway reachable": no placeholder, nothing that looks removed
const MAC_IN_TEXT = /(?:\bMAC(?: address)?\s*[:=]?\s*)?(?<![0-9A-Fa-f:-])[0-9A-Fa-f]{2}([:-])(?:[0-9A-Fa-f]{2}\1){4}[0-9A-Fa-f]{2}(?![0-9A-Fa-f:-])(?:\s*[·,;|]\s*)?/gi;
const IS_ID = /^(?:[0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2}$|^[0-9A-Fa-f]{12}$/;
// MACs written inside sentences ("MAC: e2:59:… · Android 11") disappear; a value that IS a MAC stays (it's an id → stand-in)
const scrubMacs = (t) => (IS_ID.test(t) ? t : t.replace(MAC_IN_TEXT, "").replace(/\s*·\s*$/, "").replace(/^\s*·\s*/, ""));
const scrubText = (t) => scrubMacs(t).replace(/\s*[(\[]\s*((?:\d{1,3}\.){3}\d{1,3})\s*[)\]]/g, (m, ip) => (secretIp(ip) ? "" : m))
  .replace(/\s*(?:at |via |from |to |with IP |IP:? )?(?<![\d.])((?:\d{1,3}\.){3}\d{1,3})(?!\d|\.\d)/g, (m, ip) => (secretIp(ip) ? "" : m))
  .replace(IP6, "").replace(/ {2,}/g, " ").replace(/\s+([,.;:)])/g, "$1");
```

## 4. Second opinion from the cloud (server)

The laptop can't tell "the router died" from "my own Wi-Fi dropped". The cloud sentinel hears the router's own
syslog, so a laptop "router down" is relabelled as a monitoring issue when the sentinel heard the router through
the whole window — and a claimed router *restart* is only believed if the sentinel missed it at least once.

```js
/** true = sentinel heard the router in ≥ minFrac of the window, false = it didn't, null = not enough evidence yet. */
export async function sentinelHeardRouter(sb, startIso, endIso, minFrac = 0.9) {
  const { data: sen } = await sb.from("collectors").select("id").eq("platform", "sentinel").eq("simulated", false).limit(1);
  const sid = sen?.[0]?.id;
  if (!sid) return null;
  const { data } = await sb.from("net_samples").select("samples,internet_ok")
    .eq("collector", sid).gte("minute", startIso).lte("minute", endIso).limit(2000);
  const minutes = (Date.parse(endIso) - Date.parse(startIso)) / 60e3;
  const need = minFrac >= 1 ? Math.max(3, Math.floor(minutes) - 1) : Math.max(3, minutes * 0.6);   // strict: every minute present
  if (!data || data.length < need) return null;                        // sentinel minutes finalize ~5 min late
  const samples = data.reduce((a, r) => a + (r.samples || 0), 0);
  const heard = data.reduce((a, r) => a + (r.internet_ok || 0), 0);
  return samples ? heard / samples >= minFrac : null;
}
```
