// 罔两 · 记忆  v1.2  (TauriTavern / SillyTavern 通用)
// 借「柏宝书」的结构化摘要 / 分层压缩 / 时间线锚点 / 悬念簿，
// 但不做 LLM 摘要、不做向量检索 —— 记忆来源是预设已经产出的 <scene> / <state>。
// 无状态：每次从聊天数组全量重建。打开旧聊天立即生效，天然兼容删除 / 编辑 / swipe。
//
// 注入走两条路：优先 setExtensionPrompt；拿不到就退回 manifest 里的 generate_interceptor。

const MODULE = 'wgl_memory';

const DEFAULTS = {
    enabled: true,
    depth: 2,
    recent: 6,
    maxEntries: 80,
    fallbackChars: 36,
    facts: '',
};

/* ------------------------------------------------------------ ST 接口解析 */

let ST = null;
let resolved = false;
let useInterceptor = false;

async function resolveST() {
    if (resolved) return ST;
    resolved = true;

    const pairs = [
        ['../../../extensions.js', '../../../script.js'],
        ['/scripts/extensions.js', '/script.js'],
        ['./extensions.js', './script.js'],
    ];
    for (const [a, b] of pairs) {
        try {
            const E = await import(a);
            const S2 = await import(b);
            if (E?.setExtensionPrompt && S2?.getContext) {
                ST = {
                    extension_settings: E.extension_settings,
                    setExtensionPrompt: E.setExtensionPrompt,
                    posInChat: E.extension_prompt_types?.IN_CHAT ?? 1,
                    roleSystem: E.extension_prompt_roles?.SYSTEM,
                    getContext: S2.getContext,
                    eventSource: S2.eventSource,
                    event_types: S2.event_types,
                    saveSettingsDebounced: S2.saveSettingsDebounced,
                };
                resolved = 'setExtensionPrompt';
                return ST;
            }
        } catch (err) { /* 试下一条路径 */ }
    }

    // 退路：SillyTavern 全局命名空间
    const g = globalThis.SillyTavern;
    if (g?.getContext) {
        ST = { getContext: g.getContext, setExtensionPrompt: null };
        resolved = 'global';
        return ST;
    }
    ST = null;
    return ST;
}

function ctx() {
    try { return ST?.getContext?.() ?? globalThis.SillyTavern?.getContext?.() ?? null; }
    catch { return null; }
}

function settingsStore() {
    const fromModule = ST?.extension_settings;
    if (fromModule) {
        if (!fromModule[MODULE]) fromModule[MODULE] = { ...DEFAULTS };
        return fromModule[MODULE];
    }
    const c = ctx();
    const es = c?.extensionSettings ?? c?.extension_settings ?? globalThis.extension_settings;
    if (!es) return { ...DEFAULTS };          // 只读兜底，不崩
    if (!es[MODULE]) es[MODULE] = { ...DEFAULTS };
    return es[MODULE];
}

function S() {
    const s = settingsStore();
    for (const k of Object.keys(DEFAULTS)) if (s[k] === undefined) s[k] = DEFAULTS[k];
    return s;
}

function persist() {
    try { ST?.saveSettingsDebounced?.(); } catch { /* noop */ }
}

/* ---------------------------------------------------------------- 提取 */

const RE_SCENE = /<scene>([\s\S]*?)<\/scene>/i;
const RE_STATE = /<state>([\s\S]*?)<\/state>/i;
const RE_FACT = /<fact>([\s\S]*?)<\/fact>/gi;
const RE_MEM = /<mem>([\s\S]*?)<\/mem>/gi;
const clean = (v) => String(v == null ? '' : v).replace(/\s+/g, ' ').trim();

function parseScene(raw) {
    const f = String(raw).split(/[｜|]/).map(clean).filter(Boolean);
    return { place: f[0] || '', time: f[1] || '', cast: f[2] || '', body: f[3] || '' };
}

function parseState(raw) {
    const out = { base: '', axis: '', tension: '', hook: '' };
    for (const seg of String(raw).split(/[｜|]/)) {
        const m = seg.match(/^\s*([^=：:]+)\s*[=：:]\s*(.+)$/);
        if (!m) continue;
        const k = clean(m[1]);
        const v = clean(m[2]);
        if (/钩|悬念/.test(k)) out.hook = v;
        else if (/基线/.test(k)) out.base = v;
        else if (/轴/.test(k)) out.axis = v;
        else if (/紧度/.test(k)) out.tension = v;
    }
    return out;
}

function readMessage(text) {
    const t = String(text || '');
    const ms = t.match(RE_SCENE);
    const st = t.match(RE_STATE);
    const sc = ms ? parseScene(ms[1]) : null;
    const sa = st ? parseState(st[1]) : null;

    if (sc || sa) {
        return {
            place: sc?.place || '', time: sc?.time || '', cast: sc?.cast || '', body: sc?.body || '',
            base: sa?.base || '', axis: sa?.axis || '', tension: sa?.tension || '', hook: sa?.hook || '',
        };
    }
    const flat = clean(t.replace(/<[^>]+>/g, ' '));
    return {
        place: '', time: '', cast: '', body: flat.slice(0, S().fallbackChars),
        base: '', axis: '', tension: '', hook: '',
    };
}

function messageText(msg) {
    // ST 把每一版回复都存在 swipes[]，mes 只是当前显示的那一版。
    // 重新生成 / 左右滑动时，swipes 才是真相。
    if (Array.isArray(msg.swipes) && msg.swipes.length) {
        let i = msg.swipe_id;
        if (!Number.isInteger(i) || i < 0 || i >= msg.swipes.length) i = msg.swipes.length - 1;
        const v = msg.swipes[i];
        if (typeof v === 'string' && v.trim()) return v;
    }
    return msg.mes || '';
}

function collect() {
    const chat = ctx()?.chat ?? [];
    const out = [];
    let turn = 0;
    for (const msg of chat) {
        if (!msg || msg.is_user || msg.is_system) continue;
        turn += 1;
        const r = readMessage(messageText(msg));
        if (!r.body && !r.place && !r.time) continue;
        out.push({ turn, ...r });
    }
    return out.slice(-Math.max(4, S().maxEntries));
}

/* ------------------------------- 记忆块 <mem>：模型顺手产出的结构化分析 */

function parseMem(raw) {
    const out = { what: '', add: '', people: [], hooks: '' };
    const segs = String(raw).replace(/\r/g, '').split(/\n|[｜|](?=\s*(?:事|增|人|钩)\s*[:：])/);
    for (const line of segs) {
        const m = line.match(/^\s*(事|增|人|钩)\s*[:：]\s*([\s\S]*)$/);
        if (!m) continue;
        const k = m[1];
        const v = clean(m[2]);
        if (k === '事') out.what = v;
        else if (k === '增') out.add = v;
        else if (k === '人') out.people = v.split(/[；;]/).map(clean).filter(Boolean);
        else if (k === '钩') out.hooks = v;
    }
    return out;
}

function collectMems() {
    const chat = ctx()?.chat ?? [];
    const out = [];
    let turn = 0;
    for (const msg of chat) {
        if (!msg || msg.is_user || msg.is_system) continue;
        turn += 1;
        RE_MEM.lastIndex = 0;
        let m; let buf = '';
        const txt = messageText(msg);
        while ((m = RE_MEM.exec(txt)) !== null) buf += m[1] + '\n';
        if (!buf.trim()) continue;
        out.push({ turn, ...parseMem(buf) });
    }
    return out;
}

function mergeHooks(mems) {
    const list = [];
    for (const m of mems) {
        if (!m.hooks || /^无$/.test(m.hooks)) continue;
        for (const seg of m.hooks.split(/[｜|]/)) {
            const t = clean(seg);
            if (!t) continue;
            const sign = t[0];
            const body = clean(t.slice(1));
            if (!body) continue;
            if (sign === '+') { if (!list.includes(body)) list.push(body); }
            else if (sign === '-') { const i = list.indexOf(body); if (i >= 0) list.splice(i, 1); }
        }
    }
    return list;
}

function mergePeople(mems) {
    const map = new Map();
    for (const m of mems) {
        for (const p of m.people) {
            const i = p.search(/——|—|：|:/);
            const name = i > 0 ? clean(p.slice(0, i)) : p;
            map.set(name, p);
        }
    }
    return [...map.values()].slice(0, 5);
}

/* ------------------------------------------- 事实：只采集，不改写（append-only） */

const norm = (t) => t.replace(/[，。、；：""''！？·\s\-—()（）]/g, '');

function collectFacts() {
    const chat = ctx()?.chat ?? [];
    const out = [];
    const seen = [];
    let turn = 0;
    for (const msg of chat) {
        if (!msg || msg.is_user || msg.is_system) continue;
        turn += 1;
        const text = messageText(msg);
        RE_FACT.lastIndex = 0;
        let m;
        while ((m = RE_FACT.exec(text)) !== null) {
            const f = clean(m[1]);
            if (!f) continue;
            const key = norm(f).slice(0, 14);
            if (!key || seen.includes(key)) continue;
            seen.push(key);
            out.push({ turn, text: f });
        }
    }
    return out;
}

/* ------------------------------- 名册与地点：纯机械统计，零模型依赖 */

function splitNames(v) {
    return String(v).split(/[、,，和与\/／]/).map(clean).filter(Boolean);
}

function roster(entries) {
    const m = new Map();
    for (const e of entries) for (const n of splitNames(e.cast)) m.set(n, (m.get(n) || 0) + 1);
    return [...m.entries()].sort((a, b) => b[1] - a[1]);
}

function places(entries) {
    const m = new Map();
    for (const e of entries) { const v = clean(e.place); if (v) m.set(v, (m.get(v) || 0) + 1); }
    return [...m.entries()].sort((a, b) => b[1] - a[1]).slice(0, 6);
}

/* ------------------------------------------ 远史：机械分层压缩（无 LLM） */

function compressFar(entries, keepRecent) {
    const old = entries.slice(0, Math.max(0, entries.length - keepRecent));
    if (!old.length) return [];
    const groups = [];
    let cur = null;
    for (const e of old) {
        const key = e.place || '（未记地点）';
        if (cur && cur.place === key) cur.items.push(e);
        else { cur = { place: key, items: [e] }; groups.push(cur); }
    }
    return groups.slice(-8).map((g) => {
        const a = g.items[0];
        const b = g.items[g.items.length - 1];
        const span = (a.time && b.time && a.time !== b.time) ? `${a.time} → ${b.time}` : (a.time || b.time || '');
        const head = [`第${a.turn}–${b.turn}轮`, g.place, span].filter(Boolean).join('｜');
        return `${head}｜结束于：${b.body || '—'}`;
    });
}

/* ------------------------------------------------------------ 组装注入 */

function buildBlock(entries) {
    const s = S();
    const parts = [];

    if (clean(s.facts)) parts.push('【设定·手动】\n' + String(s.facts).trim());

    const facts = collectFacts().slice(0, 16);
    if (facts.length) {
        parts.push('【设定·已确立】（按轮次，同一件事以最早的为准）\n'
            + facts.map((f) => `第${f.turn}轮｜${f.text}`).join('\n'));
    }

    const mems = collectMems();

    if (mems.length) {
        const adds = [];
        for (const m of mems) {
            if (!m.add || /^无$/.test(m.add)) continue;
            const k = norm(m.add).slice(0, 14);
            if (k && !adds.some((x) => norm(x.text).slice(0, 14) === k)) adds.push({ turn: m.turn, text: m.add });
        }
        if (adds.length) {
            parts.push('【设定·已确立】（按轮次，同一件事以最早的为准）\n'
                + adds.slice(0, 12).map((a) => `第${a.turn}轮｜${a.text}`).join('\n'));
        }

        const ppl = mergePeople(mems);
        if (ppl.length) parts.push('【人物·此刻】\n' + ppl.join('\n'));

        const events = mems.filter((m) => m.what).slice(-8);
        if (events.length) {
            parts.push('【事件日志】\n' + events.map((e) => `第${e.turn}轮｜${e.what}`).join('\n'));
        }

        const hooks2 = mergeHooks(mems);
        if (hooks2.length) parts.push('【未了的事】\n' + hooks2.slice(-6).map((h) => `· ${h}`).join('\n'));
    }

    if (entries.length) {
        if (!mems.length) {
            const names = roster(entries);
            if (names.length) {
                parts.push('【在场名册】（数字＝出现轮次；只出现一次的算过场，不要再让他开口）\n'
                    + names.map(([n, c]) => (c > 1 ? `${n}（${c}）` : `${n}（${c}·过场）`)).join('、'));
            }
            const ps = places(entries);
            if (ps.length) parts.push('【地点】（按出现次数）\n' + ps.map(([p, c]) => `${p}（${c}）`).join('、'));
        }

        const recent = entries.slice(-Math.max(1, s.recent));
        parts.push(`【当前·最近 ${recent.length} 轮】\n` + recent.map((e) => {
            const head = [`第${e.turn}轮`, e.place, e.time, e.cast].filter(Boolean).join('｜');
            const tail = [e.body, e.base && `基线=${e.base}`, e.tension && `紧度=${e.tension}`]
                .filter(Boolean).join('；');
            return tail ? `${head}｜${tail}` : head;
        }).join('\n'));

        const far = compressFar(entries, Math.max(1, s.recent));
        if (far.length) parts.push('【远史·按地点合并】\n' + far.join('\n'));

        if (!mems.length) {
            const hooks = [];
            for (const e of entries.slice(-Math.max(4, s.recent * 2))) {
                const h = clean(e.hook);
                if (h && h !== '无' && !hooks.includes(h)) hooks.push(h);
            }
            if (hooks.length) parts.push('【未办的事】\n' + hooks.slice(-5).map((h) => `· ${h}`).join('\n'));
        }

        const newest = entries[entries.length - 1];
        if (newest.time) parts.push(`【当前故事时间】${newest.time}`);
    }

    if (!parts.length) return '';

    parts.push(
        '以上是已经发生过的事，续写必须与之一致：\n'
        + '· 设定里同一件事出现多条时，以轮次最早的那一条为准，不许改写它。\n'
        + '· 时间、地点、在场的人不许推翻；走了的人不许凭空回来。\n'
        + '· 已经发生过的步骤不再重演，从上一条的落点往下接。\n'
        + '· 不许引入上面没出现过的人，不许另起一条线。',
    );
    return parts.join('\n\n');
}

/* ----------------------------------------------------------------- 注入 */

let lastBlock = null;

async function refresh() {
    const st = await resolveST();
    const s = S();
    const entries = s.enabled ? collect() : [];
    const block = s.enabled ? buildBlock(entries) : '';
    lastBlock = block;   // 只用于状态显示，不再短路

    if (!st?.setExtensionPrompt) { useInterceptor = true; return; }

    try {
        st.setExtensionPrompt(
            MODULE, block, st.posInChat, Math.max(0, Number(s.depth) || 0), false, st.roleSystem,
        );
        useInterceptor = false;
    } catch (err) {
        console.warn('[罔两·记忆] setExtensionPrompt 失败，改用 interceptor', err);
        useInterceptor = true;
    }
}

function setStatus() {
    const el = document.getElementById('wgl-status');
    if (!el) return;
    const mode = useInterceptor ? 'interceptor' : (ST?.setExtensionPrompt ? 'setExtensionPrompt' : '未连接');
    const n = (lastBlock || '').length;
    el.textContent = `状态：${mode} · 注入 ${n} 字 · 第 ${collect().length} 轮已收`;
    el.style.color = n ? '' : '#c66';
}

function bust() { lastBlock = null; refresh().then(setStatus); }

/* manifest 里声明的 generate_interceptor（拿不到 setExtensionPrompt 时兜底） */
globalThis.wgl_generateInterceptor = function (chat) {
    try {
        if (!useInterceptor || !S().enabled) return;
        const block = buildBlock(collect());
        if (!block) return;
        const idx = Math.max(0, chat.length - Math.max(0, Number(S().depth) || 0));
        chat.splice(idx, 0, {
            is_user: false, is_system: true, mes: block, name: '记忆', send_date: Date.now(), extra: {},
        });
    } catch (err) {
        console.error('[罔两·记忆] interceptor 失败', err);
    }
};

/* ----------------------------------------------------------------- 面板 */

const PANEL_ID = 'wgl-mem-panel';

function ensurePanel() {
    if (document.getElementById(PANEL_ID)) return;
    const host = document.getElementById('extensions_settings2') || document.getElementById('extensions_settings');
    if (!host) { setTimeout(ensurePanel, 700); return; }

    const s = S();
    const wrap = document.createElement('div');
    wrap.id = PANEL_ID;
    wrap.innerHTML = `
<div class="wgl-mem">
  <div class="inline-drawer">
    <div class="inline-drawer-toggle inline-drawer-header">
      <b>罔两 · 记忆</b>
      <div class="inline-drawer-icon fa-solid fa-circle-chevron-down down"></div>
    </div>
    <div class="inline-drawer-content">
      <label class="checkbox_label"><input type="checkbox" id="wgl-enabled" ${s.enabled ? 'checked' : ''}><span>启用</span></label>
      <label>注入深度 <input type="number" id="wgl-depth" class="text_pole" min="0" max="20" value="${s.depth}"></label>
      <label>逐条保留轮数 <input type="number" id="wgl-recent" class="text_pole" min="1" max="30" value="${s.recent}"></label>
      <label>扫描轮数上限 <input type="number" id="wgl-max" class="text_pole" min="8" max="400" value="${s.maxEntries}"></label>
      <label>固定设定（每轮注入，权重最高）
        <textarea id="wgl-facts" class="text_pole textarea_compact" rows="5" placeholder="林晚，27，旧书店店主。左手小指小时候骨折过，是弯的。目标：把店盘下来。">${s.facts}</textarea>
      </label>
      <div class="wgl-mem-row">
        <div class="menu_button" id="wgl-preview-btn">预览</div>
        <div class="menu_button" id="wgl-export-btn">导出</div>
        <div class="menu_button" id="wgl-import-btn">导入</div>
      </div>
      <input type="file" id="wgl-import-file" accept="application/json" style="display:none">
      <div class="wgl-mem-row">
        <div class="menu_button" id="wgl-reinject-btn">立即重注入</div>
      </div>
      <div id="wgl-status" class="wgl-status">状态：未注入</div>
      <pre id="wgl-preview" class="wgl-preview">（点「预览」查看当前会注入什么）</pre>
    </div>
  </div>
</div>`;
    host.appendChild(wrap);

    const on = (id, ev, fn) => document.getElementById(id)?.addEventListener(ev, fn);
    const bind = () => {
        on('wgl-enabled', 'change', (e) => { s.enabled = e.target.checked; persist(); bust(); });
        on('wgl-depth', 'input', (e) => { s.depth = Number(e.target.value) || 0; persist(); bust(); });
        on('wgl-recent', 'input', (e) => { s.recent = Math.max(1, Number(e.target.value) || 6); persist(); bust(); });
        on('wgl-max', 'input', (e) => { s.maxEntries = Math.max(8, Number(e.target.value) || 80); persist(); bust(); });
        on('wgl-facts', 'input', (e) => { s.facts = e.target.value; persist(); bust(); });
        on('wgl-reinject-btn', 'click', async () => { await refresh(); setStatus(); });
        on('wgl-preview-btn', 'click', () => {
            const box = document.getElementById('wgl-preview');
            const b = buildBlock(collect());
            if (box) box.textContent = b || '（暂无记忆：确认预设输出里带 <scene> / <state>）';
        });
        on('wgl-export-btn', 'click', () => {
            const blob = new Blob(
                [JSON.stringify({ facts: s.facts, entries: collect() }, null, 2)],
                { type: 'application/json' },
            );
            const a = document.createElement('a');
            a.href = URL.createObjectURL(blob);
            a.download = `wgl-memory-${Date.now()}.json`;
            a.click();
            URL.revokeObjectURL(a.href);
        });
        on('wgl-import-btn', 'click', () => document.getElementById('wgl-import-file')?.click());
        on('wgl-import-file', 'change', async (e) => {
            const file = e.target.files?.[0];
            if (!file) return;
            try {
                const data = JSON.parse(await file.text());
                if (typeof data.facts === 'string') {
                    s.facts = data.facts;
                    const ta = document.getElementById('wgl-facts');
                    if (ta) ta.value = data.facts;
                }
                persist(); bust();
                globalThis.toastr?.success?.('记忆已导入');
            } catch (err) {
                globalThis.toastr?.error?.('导入失败：' + err.message);
            }
        });
    };
    bind();
}

/* ----------------------------------------------------------------- 事件 */

async function boot() {
    await resolveST();
    S();
    ensurePanel();

    const es = ST?.eventSource;
    const et = ST?.event_types;
    const names = ['CHAT_CHANGED', 'MESSAGE_RECEIVED', 'MESSAGE_SENT', 'MESSAGE_DELETED',
                   'MESSAGE_EDITED', 'MESSAGE_UPDATED', 'MESSAGE_SWIPED', 'MESSAGE_SWIPE',
                   'GENERATION_STARTED', 'GENERATION_AFTER_COMMANDS', 'GENERATE_BEFORE_COMBINE_PROMPTS'];
    if (es && et) {
        for (const n of names) if (et[n] !== undefined) es.on(et[n], bust);
    }
    bust();
    setInterval(setStatus, 3000);
}

if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', () => setTimeout(boot, 300));
} else {
    setTimeout(boot, 300);
}

globalThis.WGLMemory = { collect, buildBlock, refresh, settings: S, ctx };
