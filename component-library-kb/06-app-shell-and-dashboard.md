# App Shell, Theme Picker & Dashboard — Reference Implementation

Purpose: the **worked, copy-paste runtime recipe** for the enterprise app frame — sidebar + workspace switcher + profile menu + topbar, the **theme dropdown (preview cards)**, KPI cards, the **AI insight card**, and **dependency-free SVG charts** (line + bar). The spec-level anatomy for these lives in [`04-navigation.md`](04-navigation.md) and [`05-layout-and-patterns.md`](05-layout-and-patterns.md); **this file is the vanilla reference you paste and then port to React/Tailwind.**

> Skin it by swapping the token block only — every component re-themes automatically.

## Token vocabulary (runtime token names)

These map onto the design-system's semantic tiers ([`../design-system-kb/01-design-tokens.md`](../design-system-kb/01-design-tokens.md)). Never use raw hex in components.

| Runtime token | Role | Semantic-tier equivalent |
|---|---|---|
| `--accent`, `--accent-600/400/300` | brand + hover/active/soft ramp | `--color-primary*` |
| `--accent-soft`, `--accent-softer` | brand tints (active nav, chips) | `--color-primary-tint` |
| `--on-accent` | text on brand | `--color-on-primary` |
| `--page`, `--surface`, `--surface-2`, `--muted` | grounds | `--color-bg` / `--color-surface*` |
| `--border`, `--border-strong` | lines | `--color-border*` |
| `--t1`, `--t2`, `--t3` | text primary / secondary / tertiary | `--color-text*` |
| `--ok/--warn/--err/--info` (+`-bg`,`-line`) | fixed semantics | `--color-success/warning/danger/info*` |
| `--ai` | `linear-gradient(135deg,#4F46E5,#7C3AED)` — **AI only** | `--gradient-ai` |
| `--c1…--c6` | categorical chart series | chart palette |

---

## 1. The theme system — full-token `[data-theme]` (the important part)

**A theme is a full token override, not just a brand swap.** The default palette lives on `:root`; each theme is a `[data-theme="id"]` block that overrides the tokens it changes (brand-only themes override ~6 vars; tinted themes also override grounds; dark themes override everything). Switching = setting one attribute on `<html>`.

```css
:root{ /* default theme = full base token set */
  --accent:#0D9488; --accent-600:#0F766E; --accent-400:#14B8A6; --accent-300:#2DD4BF;
  --accent-soft:#E9FAF7; --accent-softer:#F3FDFB; --on-accent:#fff;
  --page:#F4F8F7; --surface:#fff; --surface-2:#FAFCFC; --muted:#EDF3F2; --border:#E3EBEA; --border-strong:#D2DDDC;
  --t1:#0F1B1A; --t2:#54615F; --t3:#8A9998;
  --ok:#0E8A5F;--ok-bg:#E7F6EE;--ok-line:#34B27B; --warn:#9A6212;--warn-bg:#FDF1DA;--warn-line:#E8A93C;
  --err:#C2334A;--err-bg:#FCEBEC;--err-line:#E8607A; --info:#3C57D6;--info-bg:#EAEFFF;--info-line:#8AA0F2;
  --ai:linear-gradient(135deg,#4F46E5,#7C3AED);
}
/* brand-only light theme: override the accent ramp only */
[data-theme="indigo"]{ --accent:#4F46E5;--accent-600:#4338CA;--accent-400:#6366F1;--accent-300:#818CF8;--accent-soft:#EEF2FF;--accent-softer:#F5F6FF; }
/* tinted-ground theme: also override grounds + text */
[data-theme="sunset"]{ --accent:#E1654F;--accent-600:#CE5642;--accent-400:#EE7A64;--accent-soft:#FBE7E1;--accent-softer:#FCF1ED;
  --page:#FBF6F1;--surface:#fff;--surface-2:#FDF9F5;--muted:#F4ECE4;--border:#EEE1D6;--border-strong:#E0CDBD;--t1:#2C211B;--t2:#6E5D52;--t3:#A2917F; }
/* dark theme: full override incl. semantic -bg tints + shadows */
[data-theme="dark"]{ --accent:#2DD4BF;--accent-600:#5EEAD4;--on-accent:#042F2E;--accent-soft:#0C2E2B;
  --page:#0C0F0F;--surface:#141A19;--surface-2:#191F1E;--muted:#1E2625;--border:#273230;--border-strong:#37433F;
  --t1:#E8EFEE;--t2:#9FADAB;--t3:#657270;--ok-bg:#0F2C1E;--warn-bg:#322512;--err-bg:#33181D;--info-bg:#1A2150; }
/* accessibility themes tune semantics too */
[data-theme="cbsafe"]{ --accent:#0072B2;--ok:#00674B;--ok-line:#009E73;--err:#8F3E00;--err-line:#D55E00;--warn:#7A5B00;--warn-line:#B8860B;--info:#1E6FA0;--info-line:#56B4E9;
  --c1:#0072B2;--c2:#E69F00;--c3:#009E73;--c4:#CC79A7;--c5:#56B4E9;--c6:#D55E00; } /* Okabe–Ito */
```

**Rules that hold across the catalog** (from [`../design-system-kb/02-theming-and-modes.md`](../design-system-kb/02-theming-and-modes.md)): semantic colors keep their meaning in every theme (only the two accessibility themes retune them for contrast); the `--ai` gradient is reserved for AI features; ship ≥1 **High Contrast (AAA)** and ≥1 **Colorblind-Safe** theme; support **System** (follow OS).

## 2. The theme picker (dropdown of preview cards)

The picker is a dropdown menu of **preview cards** — each card shows a mini swatch (surface + two brand bars + a CTA dot), name, and description. This is the pattern to use — **not a bare row of swatches.**

```html
<button class="ic-btn" onclick="toggleTheme(event)" title="Theme">🎨</button>
<div class="menu theme-menu" id="themeMenu">
  <div class="tm-h"><b>Theme</b><span id="tmCount"></span></div>
  <div class="tm-grid" id="tmGrid"></div>
  <div class="tm-foot">Theme changes brand colours &amp; surfaces across every screen. Semantic colours stay consistent for accessibility. The gradient is reserved for AI.</div>
</div>
```
```js
// each theme: id + name + description + icon + preview [accent, soft-bg, surface, accent2]
const THEMES=[
  {id:'',n:'Teal',d:'Teal on white · default',ic:'🟢',p:['#0D9488','#E9FAF7','#fff','#2DD4BF']},
  {id:'indigo',n:'Indigo',d:'Indigo · classic',ic:'🔷',p:['#4F46E5','#EEF2FF','#fff','#818CF8']},
  {id:'dark',n:'Dark',d:'Teal on slate',ic:'🌙',p:['#2DD4BF','#0C2E2B','#141A19','#5EEAD4']},
  {id:'system',n:'System',d:'Follow OS preference',ic:'🖥️',p:['#0D9488','#0C2E2B','#141A19','#2DD4BF']},
  {id:'contrast',n:'High Contrast',d:'AAA · max legibility',ic:'◐',p:['#1D4ED8','#fff','#fff','#000']},
  {id:'cbsafe',n:'Colorblind-Safe',d:'Okabe–Ito',ic:'👁️',p:['#0072B2','#EAF4FB','#fff','#E69F00']},
  /* …the rest of the catalog… */
];
let THEME='';
function applyTheme(id){
  THEME=id; let real=id;
  if(id==='system') real = matchMedia('(prefers-color-scheme: dark)').matches?'dark':'';
  document.documentElement.setAttribute('data-theme', real);        // ← the whole switch
  document.querySelectorAll('.tm-card').forEach(c=>c.classList.toggle('sel',c.dataset.t===id));
  localStorage.setItem('theme', id);                                // persist per user
}
function renderThemeMenu(){
  document.getElementById('tmCount').textContent=THEMES.length+' to choose from';
  document.getElementById('tmGrid').innerHTML=THEMES.map(t=>`
    <div class="tm-card ${t.id===THEME?'sel':''}" data-t="${t.id}" onclick="applyTheme('${t.id}')">
      <div class="tm-prev" style="background:${t.p[2]}"><span class="l1" style="background:${t.p[0]}"></span><span class="l2" style="background:${t.p[3]}"></span><span class="cta" style="background:${t.p[0]}"></span></div>
      <div class="tn">${t.ic} ${t.n}</div><div class="td">${t.d}</div></div>`).join('');
}
function toggleTheme(e){e&&e.stopPropagation();renderThemeMenu();document.getElementById('themeMenu').classList.toggle('on');}
document.addEventListener('click',e=>{ if(!e.target.closest('.menu')&&!e.target.closest('[onclick*="toggleTheme"]')) document.getElementById('themeMenu').classList.remove('on'); });
applyTheme(localStorage.getItem('theme')||'');   // restore on load; no flash if set pre-paint
```
```css
.menu{position:fixed;top:58px;background:var(--surface);border:1px solid var(--border);border-radius:12px;box-shadow:var(--sh-pop);z-index:60;display:none}
.menu.on{display:block}
.theme-menu{right:60px;width:436px;max-width:calc(100vw - 32px);border-radius:16px;padding:16px}
.tm-h{display:flex;justify-content:space-between;align-items:baseline;margin-bottom:12px}
.tm-h b{font-size:11px;text-transform:uppercase;letter-spacing:.06em;color:var(--t3)} .tm-h span{font-size:11.5px;color:var(--t3)}
.tm-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;max-height:400px;overflow-y:auto}
.tm-card{border:1.5px solid var(--border);border-radius:12px;padding:9px;cursor:pointer;transition:.13s}
.tm-card:hover{border-color:var(--accent-300)} .tm-card.sel{border-color:var(--accent);box-shadow:var(--ring)}
.tm-prev{height:40px;border-radius:8px;margin-bottom:8px;position:relative;display:flex;align-items:center;padding:7px;gap:5px;border:1px solid rgba(0,0,0,.06)}
.tm-prev .l1{width:24px;height:5px;border-radius:3px} .tm-prev .l2{width:14px;height:5px;border-radius:3px} .tm-prev .cta{position:absolute;right:6px;top:7px;width:16px;height:8px;border-radius:3px}
.tm-card .tn{font-size:12px;font-weight:700} .tm-card .td{font-size:10px;color:var(--t3);margin-top:1px}
```

## 3. AppShell — sidebar + topbar + profile menu

Grid: fixed sidebar column + 60px topbar row. Sidebar = logo/collapse → **workspace switcher** → grouped nav (icon + label + optional count badge) → **user button** (opens profile menu). Collapse narrows to icon-rail via `#app.collapsed{--side:74px}` hiding labels.

```css
#app{display:grid;grid-template-columns:var(--side) 1fr;grid-template-rows:60px 1fr;grid-template-areas:"side top" "side main";min-height:100vh;--side:250px}
.sidebar{grid-area:side;background:var(--surface);border-right:1px solid var(--border);display:flex;flex-direction:column;height:100vh;position:sticky;top:0}
.ws-switch{margin:4px 12px 8px;border:1px solid var(--border);border-radius:12px;padding:9px 11px;display:flex;align-items:center;gap:10px;background:var(--surface-2);cursor:pointer;width:calc(100% - 24px)}
.nav-group{font-size:10.5px;font-weight:700;letter-spacing:.07em;text-transform:uppercase;color:var(--t3);padding:14px 10px 6px}
.nav-item{display:flex;align-items:center;gap:11px;width:100%;padding:9px 10px;border-radius:8px;color:var(--t2);font-size:13.5px;font-weight:500;text-align:left}
.nav-item .ni{width:18px;text-align:center;font-size:15px} .nav-item .nlbl{flex:1;min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.nav-item:hover{background:var(--muted);color:var(--t1)} .nav-item.active{background:var(--accent-soft);color:var(--accent);font-weight:600}
.nav-badge{font-size:11px;font-weight:700;font-family:var(--mono);background:var(--muted);color:var(--t2);border-radius:99px;padding:1px 7px}
.nav-item.active .nav-badge{background:var(--accent);color:var(--on-accent)}
.side-user{margin:0 12px 12px;border-top:1px solid var(--border);padding-top:10px;display:flex;align-items:center;gap:10px;cursor:pointer;width:calc(100% - 24px)}
#app.collapsed{--side:74px}
#app.collapsed .nlbl,#app.collapsed .nav-group,#app.collapsed .nav-badge,#app.collapsed .ws-switch .wt,#app.collapsed .side-user .uw{display:none}
#app.collapsed .nav-item,#app.collapsed .ws-switch,#app.collapsed .side-user{justify-content:center}
@media(max-width:820px){ #app{grid-template-columns:1fr;grid-template-areas:"top" "main"} .sidebar{position:fixed;transform:translateX(-104%);transition:.22s;z-index:50}.sidebar.open{transform:none} }
```
```js
// nav data: [group, [ [key,label,icon,badge?] … ] ]
const NAVGROUPS=[ ["Workspace",[["dashboard","Dashboard","🏠"]]],
  ["Records",[["search","Search","🔎"],["new","New","➕",""]]], /* … */ ];
function renderNav(){ const nav=document.getElementById('nav'); nav.innerHTML='';
  NAVGROUPS.forEach(([g,items])=>{ nav.insertAdjacentHTML('beforeend',`<div class="nav-group">${g}</div>`);
    items.forEach(([key,label,ic,bdg])=>nav.insertAdjacentHTML('beforeend',
      `<button class="nav-item" data-key="${key}" onclick="go('${key}')"><span class="ni">${ic}</span><span class="nlbl">${label}</span>${bdg?`<span class="nav-badge">${bdg}</span>`:''}</button>`)); }); }
```

**Profile menu** — a dropdown anchored to the avatar (and the sidebar user button): header (avatar, name, role, mono email) + item list (My profile, Account settings, Appearance & theme, Help, Switch workspace, Sign out). Same open/close pattern as the theme menu; only one menu open at a time.

## 4. KPI card + AI insight card

```css
.kpis{display:grid;grid-template-columns:repeat(4,1fr);gap:14px}
.kpi{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:16px;box-shadow:var(--sh-sm);transition:.15s}
.kpi:hover{transform:translateY(-2px);box-shadow:var(--sh)}
.kpi .ki{width:34px;height:34px;border-radius:10px;display:flex;align-items:center;justify-content:center;font-size:16px;margin-bottom:12px}
.kpi .kl{font-size:11px;font-weight:700;letter-spacing:.04em;text-transform:uppercase;color:var(--t3)}
.kpi .kv{font-size:26px;font-weight:800;letter-spacing:-.02em;margin-top:5px;font-variant-numeric:tabular-nums} /* tabular-nums always */
.kpi .kd{font-size:12px;margin-top:6px;color:var(--t3)}
/* AI card — the only gradient surface */
.ai-card{background:linear-gradient(135deg,var(--accent-softer),var(--surface));border:1px solid var(--accent-soft);border-radius:16px}
.ai-card .spark{width:28px;height:28px;border-radius:8px;background:var(--ai);color:#fff;display:flex;align-items:center;justify-content:center}
.ai-tag{font-family:var(--mono);font-size:9.5px;font-weight:700;padding:1px 6px;border-radius:5px}
.tag-risk{background:var(--err-bg);color:var(--err)} .tag-ai{background:var(--accent);color:var(--on-accent)} .tag-up{background:var(--ok-bg);color:var(--ok)}
```
The `.ki` icon chip is tinted per semantic tone — success uses `background:var(--ok-bg);color:var(--ok)`, etc. **KPI numbers use `tabular-nums`.**

## 5. Dependency-free SVG charts

No chart library needed for dashboards — inline SVG driven by tokens (`stroke:var(--accent)`, gradient fill), so they **re-theme automatically**. For heavy/interactive charting wrap a lib (ECharts/Recharts) behind a component per [`../design-system-kb/`](../design-system-kb/README.md), but these cover most dashboards.

```js
// line chart: area fill + line + endpoint dots + x labels
function lineChart(pts,labels){
  const w=560,h=180,pad=30,max=Math.max(...pts)*1.1,min=Math.min(...pts)*.9;
  const X=i=>pad+i*(w-pad*2)/(pts.length-1), Y=v=>h-pad-((v-min)/(max-min))*(h-pad*2);
  const line=pts.map((p,i)=>`${i?'L':'M'}${X(i).toFixed(1)} ${Y(p).toFixed(1)}`).join(' ');
  const area=`M${pad} ${h-pad} `+pts.map((p,i)=>`L${X(i).toFixed(1)} ${Y(p).toFixed(1)}`).join(' ')+` L${w-pad} ${h-pad} Z`;
  return `<svg viewBox="0 0 ${w} ${h}" style="width:100%;height:auto">
    <defs><linearGradient id="g1" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="var(--accent)" stop-opacity=".22"/><stop offset="1" stop-color="var(--accent)" stop-opacity="0"/></linearGradient></defs>
    <path d="${area}" fill="url(#g1)"/><path d="${line}" fill="none" stroke="var(--accent)" stroke-width="2.5" stroke-linecap="round"/>
    ${pts.map((p,i)=>`<circle cx="${X(i).toFixed(1)}" cy="${Y(p).toFixed(1)}" r="3.5" fill="var(--surface)" stroke="var(--accent)" stroke-width="2"/>`).join('')}
    ${labels.map((l,i)=>`<text x="${X(i).toFixed(1)}" y="${h-8}" font-size="10" fill="var(--t3)" text-anchor="middle">${l}</text>`).join('')}</svg>`;
}
// bar chart: [label, value, displayValue, color]
function barChart(data){ const max=Math.max(...data.map(d=>d[1]));
  return `<div class="bars">${data.map(d=>`<div class="bcol"><span class="bv">${d[2]}</span>
    <div class="bar" style="height:${Math.round(d[1]/max*100)}%;background:${d[3]||'var(--accent)'}"></div><span class="bl">${d[0]}</span></div>`).join('')}</div>`; }
// control chart (e.g. mean ±1/2/3 SD guide lines, points flagged past ±2 SD): same SVG approach with horizontal guide lines.
```

## 6. Dashboard composition

Compose a role dashboard as: **greet header** (title + context + live pill + actions) → **KPI row** (`.kpis`, 4-up) → **`.row2`** split (a `1.5fr` chart card + a `1fr` AI insight card) → another `.row2` (two charts) → a full-width table card. All cards are token-driven and reflow to one column under ~940px.

```css
.row2{display:grid;grid-template-columns:1.5fr 1fr;gap:16px;align-items:start}
@media(max-width:940px){.row2,.kpis{grid-template-columns:1fr}}
```

## Acceptance checklist
- [ ] Themes are **full `[data-theme]` token overrides**; switching sets one attribute on `<html>`; choice persisted (localStorage / user prefs) and restored before paint (no flash).
- [ ] Theme picker is a **dropdown of preview cards** (swatch + name + description + selected state), shows the count, closes on outside click; includes **System**, **High Contrast (AAA)**, **Colorblind-Safe**.
- [ ] Only one dropdown (theme / profile / notifications) open at a time.
- [ ] Sidebar: grouped nav with icon + label + count badge, active state, **collapsible to icon rail**, workspace switcher, user button → profile menu; off-canvas under 820px with scrim.
- [ ] KPI numbers use `tabular-nums`; semantic tone drives the icon-chip tint (not the whole card).
- [ ] **Only the AI card/badge uses the gradient**; nothing else.
- [ ] Charts are token-driven SVG (`var(--accent)`, gradient fill) and re-theme automatically; heavy charts wrap a lib behind a component.
- [ ] Everything wired through tokens — **no raw hex in components**; both light and dark verified; WCAG 2.2 AA.

## Related
- Spec anatomy → [`04-navigation.md`](04-navigation.md) (Sidebar, DropdownMenu, CommandPalette), [`05-layout-and-patterns.md`](05-layout-and-patterns.md) (AppShell, Dashboard grid)
- Tokens & catalog → [`../design-system-kb/01-design-tokens.md`](../design-system-kb/01-design-tokens.md), [`02-theming-and-modes.md`](../design-system-kb/02-theming-and-modes.md)
- Enterprise theming rules → [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md)
