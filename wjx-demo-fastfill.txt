// ==UserScript==
// @name         问卷星 Demo 快速填充
// @namespace    https://local.demo/wjx-demo-fastfill
// @version      0.27.0
// @description  仅用于自有或已授权的问卷星 demo 测试：按本机保存的固定配置填写（含校区、本科生/研究生），支持自定义题目规则自动作答单选/多选/下拉/填空，未匹配题目可在面板现场勾选并记住；配置没有的字段留空、不随机；倒计时与页面同步，归零后 30ms 自动点击开始；逐字段随机延时；不绕过安全校验、验证码、登录或风控。
// @homepageURL  https://github.com/Guzifeng0613/Auto-fill
// @supportURL   https://github.com/Guzifeng0613/Auto-fill/issues
// @downloadURL  https://raw.githubusercontent.com/Guzifeng0613/Auto-fill/main/wjx-demo-fastfill.user.js
// @updateURL    https://raw.githubusercontent.com/Guzifeng0613/Auto-fill/main/wjx-demo-fastfill.user.js
// @match        *://*.wjx.cn/*
// @match        *://*.wjx.top/*
// @match        *://*.wjx.com/*
// @match        *://*.sojump.com/*
// @run-at       document-idle
// @grant        GM_getValue
// @grant        GM_setValue
// @license      MIT
// ==/UserScript==

(function () {
  'use strict';

  const PANEL_ID = 'wjx-demo-fastfill-panel';
  const VERSION = '0.27.0';
  const CLICK_DELAY_MS = 30;

  function cmpVersion(a, b) {
    const pa = String(a).split('.').map(Number);
    const pb = String(b).split('.').map(Number);
    for (let i = 0; i < 3; i += 1) {
      const d = (pa[i] || 0) - (pb[i] || 0);
      if (d) return d;
    }
    return 0;
  }

  if (window.__wjxDemoFastFillLoaded) {
    const prev = window.__wjxDemoFastFillVersion || null;
    if (prev && cmpVersion(prev, VERSION) >= 0) return;
    try {
      if (typeof window.__wjxDemoFastFillStop === 'function') window.__wjxDemoFastFillStop();
    } catch (e) {}
    const oldPanel = document.getElementById(PANEL_ID);
    if (oldPanel) oldPanel.remove();
    window.__wjxDemoFastFillDuplicate = (window.__wjxDemoFastFillDuplicate || 0) + 1;
  }
  window.__wjxDemoFastFillLoaded = true;
  window.__wjxDemoFastFillVersion = VERSION;
  const BLOCK_SELECTOR = '.divQuestion, .field, .question, .ui-field-contain, .form-group, tr, li';
  const WAIT_KEY = 'wjxDemoAutoWait';
  const RELOAD_KEY = 'wjxDemoReloadCount';
  const ALLOW_SUBMIT_KEY = 'wjxDemoAllowSubmit';
  const LAST_SUBMIT_KEY = 'wjxDemoLastSubmit';
  const DELAY_KEY = 'wjxDemoDelayRange';
  const FIXED_KEY = 'wjxDemoFixedAnswers';
  const RULES_KEY = 'wjxDemoCustomRules';
  const FALLBACK_KEY = 'wjxDemoFallbackText';
  const REMEMBER_KEY = 'wjxDemoRememberPicks';
  const MAX_AUTO_RELOAD = 8;
  const LOG_KEY = 'wjxDemoDebugLog';

  const touchedText = new Map();
  const touchedSelect = new Map();
  const touchedOption = new Map();
  const touchedCustom = new Map();
  let submittedThisLoad = false;
  let filledThisLoad = false;
  let pageCountdownSeen = false;
  let waitTimer = null;
  let watchIvRef = null;
  let startAttempts = 0;

  window.__wjxDemoFastFillStop = () => {
    if (waitTimer) { clearInterval(waitTimer); waitTimer = null; }
    if (watchIvRef) { clearInterval(watchIvRef); watchIvRef = null; }
    const p = document.getElementById(PANEL_ID);
    if (p) p.remove();
  };

  const randInt = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min;
  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
  const clean = (s) => String(s || '').replace(/\s+/g, ' ').trim();

  function readStore(key, def) {
    try {
      if (typeof GM_getValue === 'function') return GM_getValue(key, def);
    } catch (e) {}
    try {
      const raw = localStorage.getItem(key);
      return raw === null ? def : JSON.parse(raw);
    } catch (e) {
      return def;
    }
  }

  function writeStore(key, value) {
    try {
      if (typeof GM_setValue === 'function') {
        GM_setValue(key, value);
        return;
      }
    } catch (e) {}
    localStorage.setItem(key, JSON.stringify(value));
  }

  function delayRange() {
    const r = readStore(DELAY_KEY, null) || {};
    const rawMin = Number(r.min);
    const rawMax = Number(r.max);
    const min = Number.isFinite(rawMin) ? Math.max(0, rawMin) : 30;
    const max = Number.isFinite(rawMax) ? Math.max(min, rawMax) : Math.max(min, 300);
    return { min, max };
  }

  function humanDelay() {
    const r = delayRange();
    return sleep(randInt(r.min, r.max));
  }

  function allowSubmit() {
    return readStore(ALLOW_SUBMIT_KEY, true) !== false;
  }

  function loadFixed() {
    return readStore(FIXED_KEY, {}) || {};
  }

  function dlog(msg) {
    try {
      const arr = readStore(LOG_KEY, []) || [];
      arr.push(new Date().toLocaleTimeString('zh-CN', { hour12: false }) + ' v' + VERSION + ' ' + msg);
      while (arr.length > 40) arr.shift();
      writeStore(LOG_KEY, arr);
    } catch (e) {}
  }

  function saveFixed(obj) {
    writeStore(FIXED_KEY, obj);
  }

  const FIXED_FIELDS = [
    ['studentId', '学号'],
    ['name', '姓名'],
    ['gender', '性别'],
    ['phone', '手机号'],
    ['campus', '校区'],
    ['degree', '本研'],
    ['college', '学院'],
    ['major', '专业'],
    ['className', '班级'],
    ['dorm', '宿舍号'],
    ['dormRoom', '房间号'],
    ['idCard', '身份证号']
  ];

  const FIELD_PLACEHOLDER = {
    campus: '如 昌平校区',
    degree: '本科生 或 研究生'
  };

  function hasFixedConfig() {
    const fixed = loadFixed();
    return FIXED_FIELDS.some(([key]) => fixed[key]);
  }

  function fixedKeyFor(hint) {
    const h = String(hint || '').toLowerCase().replace(/[^一-龥a-z0-9]/g, '');
    if (!h) return null;
    if (/身份证|证件号|证件编号|idcard|idnumber|sfz|shenfenzheng/.test(h)) return 'idCard';
    if (/学号|学籍号|学生号|学工号|学生证号|学生编号|studentid|studentno|studentnumber|stuid|stuno|sno|xh|xuehao/.test(h)
      || /学(生|工|籍|证)[a-z0-9]*号/.test(h)) return 'studentId';
    if (/姓名|名字|称呼|真实姓名|学生姓名|name|xm|xingming/.test(h)) return 'name';
    if (/性别|gender|sex|xb|xingbie/.test(h)) return 'gender';
    if (!/班级|班号|专业|学院|宿舍|学号|满意|评价|建议|意见/.test(h)
      && /本科生|研究生|学历|培养层次|学生类别|学生类型|在读层次|教育层次|degree|education/.test(h)) return 'degree';
    if (!/满意|评价|建议|意见/.test(h) && /校区|campus|xiaoqu/.test(h)) return 'campus';
    if (/房间|room|fj|fangjian/.test(h)) return 'dormRoom';
    if (/宿舍|寝室|楼栋|公寓|dorm|ss|sushe/.test(h)) return 'dorm';
    if (/学院|院系|系别|分院|所在系|学校|college|faculty|school|department|xy|xueyuan/.test(h)) return 'college';
    if (/班级|班别|行政班|所在班|class|bj|banji/.test(h) || /班/.test(h)) return 'className';
    if (/专业|major|zy|zhuanye/.test(h)) return 'major';
    if (/手机|电话|联系方式|mobile|phone|tel|sj|dh|shouji/.test(h)) return 'phone';
    return null;
  }

  function inPanel(el) {
    return !!(el && el.closest && el.closest('#' + PANEL_ID));
  }

  function surveyRoot() {
    return document.querySelector('#divQuestion')
      || document.querySelector('.questionnaire')
      || document.querySelector('.survey-content')
      || document.querySelector('#form1')
      || document.querySelector('form')
      || document.body;
  }

  function isVisible(el) {
    if (!el || inPanel(el)) return false;
    const rect = el.getBoundingClientRect();
    const style = window.getComputedStyle(el);
    return rect.width > 0 && rect.height > 0 && style.visibility !== 'hidden' && style.display !== 'none';
  }

  function isEnabled(el) {
    return !!el && !el.disabled && el.getAttribute('aria-disabled') !== 'true';
  }

  function isEditableText(el) {
    return isEnabled(el) && !el.readOnly && el.getAttribute('aria-readonly') !== 'true';
  }

  function fire(el, names) {
    names.forEach((name) => {
      el.dispatchEvent(new Event(name, { bubbles: true, cancelable: true }));
    });
  }

  function setNativeValue(el, value) {
    const proto = el instanceof HTMLTextAreaElement
      ? HTMLTextAreaElement.prototype
      : el instanceof HTMLSelectElement
        ? HTMLSelectElement.prototype
        : HTMLInputElement.prototype;
    const descriptor = Object.getOwnPropertyDescriptor(proto, 'value');
    if (descriptor && descriptor.set) descriptor.set.call(el, value);
    else el.value = value;
    fire(el, ['input', 'change', 'blur']);
  }

  function questionBlock(el) {
    return el.closest(BLOCK_SELECTOR) || el.parentElement || el;
  }

  function labelFor(el) {
    if (!el.id) return null;
    const esc = window.CSS && CSS.escape ? CSS.escape(el.id) : el.id.replace(/["\\]/g, '\\$&');
    return document.querySelector('label[for="' + esc + '"]');
  }

  function hintFor(el) {
    const block = questionBlock(el);
    return clean([
      labelFor(el) && labelFor(el).innerText,
      block && block.innerText,
      el.getAttribute('aria-label'),
      el.getAttribute('placeholder'),
      el.name,
      el.id
    ].filter(Boolean).join(' ')).slice(0, 160);
  }

  function optionLabel(input) {
    const label = labelFor(input)
      || input.closest('label')
      || input.closest('.ui-radio, .ui-checkbox, [role="radio"], [role="checkbox"], .radio, .checkbox, .option');
    return clean(label ? label.innerText : input.value || input.id || '选项');
  }

  function optionTarget(input) {
    return labelFor(input)
      || input.closest('label')
      || input.closest('.ui-radio, .ui-checkbox, [role="radio"], [role="checkbox"], .radio, .checkbox, .option')
      || input.parentElement
      || input;
  }

  function snapshotOption(input) {
    if (!touchedOption.has(input)) touchedOption.set(input, input.checked);
  }

  function setChecked(input, desired) {
    snapshotOption(input);
    if (input.checked === desired) return;
    const target = optionTarget(input);
    try { target.click(); } catch (e) { try { input.click(); } catch (e2) {} }
    if (input.checked !== desired) {
      input.checked = desired;
      fire(input, ['input', 'change']);
    }
  }

  function groupKey(input) {
    if (input.name) return 'name:' + input.name;
    const block = questionBlock(input);
    return 'block:' + (block.id || block.getAttribute('data-id') || clean(block.innerText).slice(0, 60) || input.id);
  }

  function groupBy(inputs, keyFn) {
    const map = new Map();
    inputs.forEach((input) => {
      const key = keyFn(input);
      if (!map.has(key)) map.set(key, []);
      map.get(key).push(input);
    });
    return map;
  }

  function choiceVisible(input) {
    if (isVisible(input)) return true;
    const target = optionTarget(input);
    return !!target && target !== input && isVisible(target);
  }

  function isConsent(el) {
    const idName = ((el.id || '') + ' ' + (el.name || '')).toLowerCase();
    if (/checkxiexi|agree|consent/.test(idName)) return true;
    const hint = hintFor(el).toLowerCase();
    return /协议|隐私|条款|同意|checkxiexi/.test(hint);
  }

  const OPTION_ALIAS = [
    ['计科', '计算机'],
    ['信工', '信息工程'],
    ['经管', '经济管理'],
    ['外语', '英语'],
    ['机械', '机械工程'],
    ['艺术', '艺术设计']
  ];

  function normOption(s) {
    let t = clean(s).toLowerCase().replace(/[^一-龥a-z0-9]/g, '');
    t = t.replace(/(学院|学系|系|专业|班级|班|方向|性别)$/g, '');
    OPTION_ALIAS.forEach(([from, to]) => { if (!t.includes(to)) t = t.split(from).join(to); });
    return t;
  }

  function matchOption(label, want) {
    const a = normOption(label);
    const b = normOption(want);
    if (!a || !b) return false;
    if (a.includes(b) || b.includes(a)) return true;
    const pa = a.match(/^([^\d]*)(\d*)$/);
    const pb = b.match(/^([^\d]*)(\d*)$/);
    if (pa && pb && pa[2] && pb[2] && pa[2] === pb[2]) {
      const ca = pa[1];
      const cb = pb[1];
      if (ca && cb && (ca.includes(cb) || cb.includes(ca))) return true;
    }
    return false;
  }

  function loadRules() {
    const list = readStore(RULES_KEY, []);
    if (!Array.isArray(list)) return [];
    return list.filter((r) => r && clean(r.q) && clean(r.a));
  }

  function saveRules(list) {
    writeStore(RULES_KEY, list.filter((r) => r && clean(r.q) && clean(r.a)));
  }

  function fallbackText() {
    return clean(readStore(FALLBACK_KEY, '') || '');
  }

  function rememberPicks() {
    return readStore(REMEMBER_KEY, true) !== false;
  }

  function normKey(s) {
    return clean(s).toLowerCase().replace(/[^一-龥a-z0-9]/g, '');
  }

  function ruleAnswerFor(hint) {
    const h = normKey(hint);
    if (!h) return '';
    const rules = loadRules();
    let best = '';
    let bestLen = 0;
    rules.forEach((r) => {
      const q = normKey(r.q);
      if (q && h.includes(q) && q.length > bestLen) { best = clean(r.a); bestLen = q.length; }
    });
    return best;
  }

  function splitWants(value) {
    return String(value || '').split(/[|｜,，、;；\/]+/).map(clean).filter(Boolean);
  }

  function questionTitle(el) {
    const block = questionBlock(el);
    const cand = block && block.querySelector
      ? block.querySelector('.field-label, .topic, .div_title_question, .question-title, .q-title, legend')
      : null;
    let t = cand ? clean(cand.innerText) : '';
    if (!t) t = clean((block && block.innerText) || hintFor(el));
    t = t.replace(/^[*\s]*\d+\s*[.、．)）]\s*/, '').replace(/\s*[*＊]\s*$/, '');
    return t.slice(0, 40) || clean(hintFor(el)).slice(0, 40);
  }

  function wantFor(el, fixed) {
    const hint = hintFor(el);
    const key = fixedKeyFor(hint);
    const own = key ? clean(fixed[key]) : '';
    if (own) return { value: own, from: 'fixed', key };
    const rule = ruleAnswerFor(hint);
    if (rule) return { value: rule, from: 'rule', key };
    return { value: '', from: '', key };
  }

  function rememberRule(title, answer) {
    if (!rememberPicks()) return;
    const q = clean(title);
    const a = clean(answer);
    if (!q || !a) return;
    const rules = loadRules().filter((r) => normKey(r.q) !== normKey(q));
    rules.push({ q, a });
    saveRules(rules);
    dlog('remember rule q=' + q.slice(0, 20) + ' a=' + a.slice(0, 20));
  }

  async function fillNativeChoices(root, fixed) {
    let filled = 0;
    let unmapped = 0;

    const radios = Array.from(root.querySelectorAll('input[type="radio"]'))
      .filter((el) => !inPanel(el) && isEnabled(el) && choiceVisible(el) && !isConsent(el));
    for (const inputs of groupBy(radios, groupKey).values()) {
      const enabled = inputs.filter(isEnabled);
      if (!enabled.length) continue;
      if (enabled.some((el) => el.checked)) continue;
      const res = wantFor(enabled[0], fixed);
      if (!res.value) { if (!res.key) unmapped += 1; continue; }
      await humanDelay();
      enabled.forEach(snapshotOption);
      const pick = enabled.find((input) => splitWants(res.value).some((w) => matchOption(optionLabel(input), w)));
      if (!pick) { unmapped += 1; continue; }
      setChecked(pick, true);
      filled += 1;
    }

    const checkboxes = Array.from(root.querySelectorAll('input[type="checkbox"]'))
      .filter((el) => !inPanel(el) && isEnabled(el) && choiceVisible(el) && !isConsent(el));
    for (const inputs of groupBy(checkboxes, groupKey).values()) {
      const enabled = inputs.filter(isEnabled);
      if (!enabled.length) continue;
      if (enabled.some((el) => el.checked)) continue;
      const res = wantFor(enabled[0], fixed);
      if (!res.value) { unmapped += 1; continue; }
      await humanDelay();
      enabled.forEach(snapshotOption);
      let hit = 0;
      splitWants(res.value).forEach((w) => {
        const pick = enabled.find((input) => matchOption(optionLabel(input), w));
        if (pick && !pick.checked) { setChecked(pick, true); hit += 1; }
      });
      if (hit) filled += 1;
      else unmapped += 1;
    }

    return { filled, unmapped };
  }

  async function fillCustomChoices(root, fixed) {
    let filled = 0;
    let unmapped = 0;
    const selector = '[role="radio"], [role="checkbox"], .ui-radio, .ui-checkbox, .radio, .checkbox';
    const elements = Array.from(root.querySelectorAll(selector)).filter((el) => {
      if (inPanel(el) || !isVisible(el) || isConsent(el)) return false;
      if (el.querySelector('input[type="radio"], input[type="checkbox"]')) return false;
      return true;
    });

    const groups = groupBy(elements, (el) => {
      const role = el.getAttribute('role') || (/checkbox/.test(el.className) ? 'checkbox' : 'radio');
      const block = questionBlock(el);
      return role + '|' + (block.id || block.getAttribute('data-id') || clean(block.innerText).slice(0, 60));
    });

    for (const [key, items] of groups) {
      const role = key.split('|')[0];
      const enabled = items.filter((el) => el.getAttribute('aria-disabled') !== 'true');
      if (!enabled.length) continue;
      if (enabled.some((el) => el.getAttribute('aria-checked') === 'true')) continue;
      const res = wantFor(enabled[0], fixed);
      if (!res.value) { if (role === 'checkbox' || !res.key) unmapped += 1; continue; }
      await humanDelay();
      const wants = splitWants(res.value);
      const picks = role === 'checkbox'
        ? enabled.filter((el) => wants.some((w) => matchOption(clean(el.innerText), w)))
        : enabled.filter((el) => wants.some((w) => matchOption(clean(el.innerText), w))).slice(0, 1);
      if (!picks.length) { unmapped += 1; continue; }
      picks.forEach((pick) => {
        if (!touchedCustom.has(pick)) touchedCustom.set(pick, pick.getAttribute('aria-checked'));
        pick.dataset.wjxFfCustom = '1';
        try { pick.click(); } catch (e) {}
        if (pick.getAttribute('aria-checked') !== 'true') {
          pick.setAttribute('aria-checked', 'true');
          fire(pick, ['input', 'change', 'click']);
        }
      });
      filled += 1;
    }

    return { filled, unmapped };
  }

  async function fillSelects(root, fixed) {
    let filled = 0;
    let unmapped = 0;
    for (const select of root.querySelectorAll('select')) {
      if (inPanel(select) || !isVisible(select) || !isEnabled(select)) continue;
      const options = Array.from(select.options).filter((opt) => !opt.disabled && opt.value !== '');
      if (!options.length) continue;
      if (Array.from(select.selectedOptions).some((opt) => opt.value !== '')) continue;
      const res = wantFor(select, fixed);
      if (!res.value) { if (!res.key) unmapped += 1; continue; }
      await humanDelay();
      const wants = splitWants(res.value);
      if (select.multiple) {
        let hit = 0;
        options.forEach((opt) => {
          const text = clean(opt.innerText || opt.textContent || opt.value);
          if (wants.some((w) => matchOption(text, w))) { opt.selected = true; hit += 1; }
        });
        if (!hit) { unmapped += 1; continue; }
        if (!touchedSelect.has(select)) touchedSelect.set(select, select.value);
        fire(select, ['input', 'change']);
        filled += 1;
        continue;
      }
      const chosen = options.find((opt) => wants.some((w) => matchOption(clean(opt.innerText || opt.textContent || opt.value), w)));
      if (!chosen) { unmapped += 1; continue; }
      if (!touchedSelect.has(select)) touchedSelect.set(select, select.value);
      setNativeValue(select, chosen.value);
      filled += 1;
    }
    return { filled, unmapped };
  }

  async function fillTexts(root, fixed, status) {
    let filled = 0;
    let unmapped = 0;
    const selector = [
      'input[type="text"]',
      'input[type="tel"]',
      'input[type="email"]',
      'input[type="number"]',
      'input[type="date"]',
      'input[type="datetime-local"]',
      'input[type="url"]',
      'input:not([type])',
      'textarea'
    ].join(',');

    for (const el of root.querySelectorAll(selector)) {
      if (inPanel(el) || !isVisible(el) || !isEditableText(el)) continue;
      if (/^__/.test(el.name || '') || el.autocomplete === 'off' && el.name === 'search') continue;
      if (clean(el.value)) continue;
      const res = wantFor(el, fixed);
      let want = res.value;
      if (!want) want = fallbackText();
      if (!want) { if (!res.key) unmapped += 1; continue; }
      await humanDelay();
      let value = String(want);
      const maxLength = Number(el.getAttribute('maxlength') || 0);
      if (maxLength > 0) value = value.slice(0, maxLength);
      if (!touchedText.has(el)) touchedText.set(el, el.value);
      setNativeValue(el, value);
      filled += 1;
      if (status) status.textContent = '填写文本 ' + filled + '…';
    }
    return { filled, unmapped };
  }

  function collectAnswers() {
    const root = surveyRoot();
    const answers = [];

    groupBy(Array.from(root.querySelectorAll('input[type="radio"]')).filter((el) => !inPanel(el)), groupKey)
      .forEach((inputs) => {
        const checked = inputs.find((el) => el.checked);
        if (checked) answers.push({ type: 'radio', question: hintFor(checked), answer: optionLabel(checked) });
      });

    groupBy(Array.from(root.querySelectorAll('input[type="checkbox"]')).filter((el) => !inPanel(el)), groupKey)
      .forEach((inputs) => {
        const checked = inputs.filter((el) => el.checked);
        if (checked.length) answers.push({ type: 'checkbox', question: hintFor(checked[0]), answer: checked.map(optionLabel) });
      });

    Array.from(root.querySelectorAll('[role="radio"][aria-checked="true"], [role="checkbox"][aria-checked="true"]'))
      .filter((el) => !inPanel(el) && isVisible(el))
      .forEach((el) => answers.push({ type: 'custom-choice', question: hintFor(el), answer: clean(el.innerText) }));

    Array.from(root.querySelectorAll('select')).forEach((select) => {
      if (inPanel(select) || !isVisible(select)) return;
      const selected = Array.from(select.selectedOptions).map((opt) => clean(opt.innerText || opt.value));
      if (selected.length) answers.push({ type: 'select', question: hintFor(select), answer: selected });
    });

    Array.from(root.querySelectorAll('input[type="text"], input[type="tel"], input[type="email"], input[type="number"], input[type="date"], input[type="datetime-local"], input[type="url"], input:not([type]), textarea'))
      .forEach((el) => {
        if (inPanel(el) || !isVisible(el) || !isEditableText(el)) return;
        answers.push({ type: 'text', question: hintFor(el), answer: el.value });
      });

    return answers;
  }

  function findConsent() {
    const known = document.getElementById('checkxiexi');
    if (known) return known;
    return Array.from(document.querySelectorAll('input[type="checkbox"]'))
      .find((el) => !inPanel(el) && isConsent(el) && choiceVisible(el)) || null;
  }

  function clickConsent() {
    const cb = findConsent();
    if (!cb) return false;
    if (cb.checked) return true;
    const target = labelFor(cb) || cb.closest('label') || cb;
    try { target.click(); } catch (e) { try { cb.click(); } catch (e2) {} }
    if (!cb.checked) {
      cb.checked = true;
      fire(cb, ['input', 'change', 'click']);
    }
    return cb.checked;
  }

  function findSubmit() {
    const preferred = ['#ctlNext', '#submit_button', '.submitbtn'];
    for (const sel of preferred) {
      const el = document.querySelector(sel);
      if (el && !inPanel(el) && isVisible(el)) return el;
    }
    return Array.from(document.querySelectorAll('a, button, div, span'))
      .find((el) => !inPanel(el) && isVisible(el) && (el.innerText || '').trim() === '提交') || null;
  }

  function submitNow(status) {
    if (submittedThisLoad) {
      status.textContent = '本次已提交过。';
      return false;
    }
    const btn = findSubmit();
    if (!btn) {
      status.textContent = '未找到提交按钮。';
      return false;
    }
    submittedThisLoad = true;
    writeStore(LAST_SUBMIT_KEY, { url: location.href, at: Date.now() });
    status.textContent = '已勾选并提交。';
    try { btn.click(); } catch (e) {}
    setTimeout(() => {
      if (!document.contains(status)) return;
      const text = pageText();
      if (/安全校验|请重新提交|验证码|滑块|人机/.test(text)) {
        status.textContent = '触发安全校验，请人工完成。';
        return;
      }
      const err = Array.from(document.querySelectorAll('.field-error, .ui-li-tip, .error, .tip-error'))
        .map((el) => el.innerText || '').filter(Boolean).join(' ');
      if (err) status.textContent = '页面提示：' + err.slice(0, 80);
    }, 800);
    return true;
  }

  async function fillAll(status) {
    filledThisLoad = true;
    const root = surveyRoot();
    const fixed = loadFixed();
    status.textContent = '填写选项…';
    const choices = await fillNativeChoices(root, fixed);
    status.textContent = '填写自定义选项…';
    const custom = await fillCustomChoices(root, fixed);
    status.textContent = '填写下拉…';
    const selects = await fillSelects(root, fixed);
    const texts = await fillTexts(root, fixed, status);
    const filled = choices.filled + custom.filled + selects.filled + texts.filled;
    const unmapped = choices.unmapped + custom.unmapped + selects.unmapped + texts.unmapped;
    const pending = renderPicker(status);
    dlog('fill done filled=' + filled + ' unmapped=' + unmapped + ' pending=' + pending);
    status.textContent = '完成：填写' + filled + ' 项'
      + (unmapped ? '，未对应' + unmapped + ' 项' : '')
      + (pending ? '，待选' + pending + ' 项' : '');
    if (!allowSubmit()) return;
    if (!clickConsent()) {
      status.textContent = '未找到同意框。';
      return;
    }
    if (unmapped > 0) {
      status.textContent = '有未对应字段，已勾选不提交。';
      return;
    }
    status.textContent = '提交前随机延时…';
    await humanDelay();
    submitNow(status);
  }

  function clearAll(status) {
    touchedText.forEach((original, el) => {
      if (document.contains(el)) setNativeValue(el, original);
    });
    touchedSelect.forEach((original, el) => {
      if (document.contains(el)) setNativeValue(el, original);
    });
    touchedOption.forEach((original, el) => {
      if (document.contains(el)) setChecked(el, original);
    });
    touchedCustom.forEach((original, el) => {
      if (!document.contains(el)) return;
      const current = el.getAttribute('aria-checked');
      if (current !== original) {
        try { el.click(); } catch (e) {}
        if (original === null) el.removeAttribute('aria-checked');
        else el.setAttribute('aria-checked', original);
        fire(el, ['input', 'change']);
      }
      delete el.dataset.wjxFfCustom;
    });
    touchedText.clear();
    touchedSelect.clear();
    touchedOption.clear();
    touchedCustom.clear();
    status.textContent = '已恢复本次修改。';
  }

  async function copyResult(status) {
    const text = JSON.stringify(collectAnswers(), null, 2);
    try {
      await navigator.clipboard.writeText(text);
      status.textContent = '已复制答案 JSON。';
    } catch (e) {
      const area = document.createElement('textarea');
      area.value = text;
      area.style.position = 'fixed';
      area.style.left = '-9999px';
      document.body.appendChild(area);
      area.select();
      document.execCommand('copy');
      area.remove();
      status.textContent = '已复制答案 JSON。';
    }
  }

  function pageText() {
    const panel = document.getElementById(PANEL_ID);
    const parts = [];
    Array.from(document.body.childNodes).forEach((n) => {
      if (panel && (n === panel || (n.contains && n.contains(panel)))) return;
      parts.push(n.innerText || n.textContent || '');
    });
    return parts.join(' ').replace(/\s+/g, ' ');
  }

  function parseOpenTarget() {
    const text = pageText();
    if (!/距离开始还有|将于.*开放|尚未开始|未开始/.test(text)) return null;
    const c = text.match(/(\d+)\s*天\s*(\d+)\s*时\s*(\d+)\s*分\s*(\d+)\s*秒/);
    const m = text.match(/(\d{4})-(\d{1,2})-(\d{1,2})\s+(\d{1,2}):(\d{2})(?::(\d{2}))?/);
    const target = m ? new Date(+m[1], +m[2] - 1, +m[3], +m[4], +m[5], +(m[6] || 0)).getTime() : null;
    if (c) return target !== null ? target : Date.now() + ((+c[1] * 86400 + +c[2] * 3600 + +c[3] * 60 + +c[4]) * 1000);
    if (target !== null && target > Date.now()) return target;
    return null;
  }

  function stillClosed() {
    const live = liveCountdown();
    if (pageCountdownSeen) return !!(live && live.ms > 0);
    const t = parseOpenTarget();
    return t !== null && t > Date.now();
  }

  const CD_FAST_SELECTOR = '#divStartTimeTip, #divTip, [id*="StartTime"], [id*="countdown"], [class*="countdown"]';

  function liveCountdown() {
    let text = null;
    const fast = document.querySelectorAll(CD_FAST_SELECTOR);
    for (const node of fast) {
      if (inPanel(node)) continue;
      const t = node.textContent || '';
      if (/\d+\s*天\s*\d+\s*时\s*\d+\s*分\s*\d+\s*秒/.test(t)) { text = t; break; }
    }
    if (text === null) text = pageText();
    const c = text.match(/(\d+)\s*天\s*(\d+)\s*时\s*(\d+)\s*分\s*(\d+)\s*秒/);
    if (!c) return null;
    pageCountdownSeen = true;
    return {
      ms: (+c[1] * 86400 + +c[2] * 3600 + +c[3] * 60 + +c[4]) * 1000,
      label: c[1] + '天' + c[2] + '时' + c[3] + '分' + c[4] + '秒'
    };
  }

  function fmtRemain(ms) {
    const total = Math.max(0, Math.ceil(ms / 1000));
    const d = Math.floor(total / 86400);
    const h = Math.floor((total % 86400) / 3600);
    const m = Math.floor((total % 3600) / 60);
    const s = total % 60;
    const pad = (n) => String(n).padStart(2, '0');
    return (d > 0 ? d + '天' : '') + h + '时' + m + '分' + s + '秒';
  }

  function startTextOf(el) {
    return clean(el.innerText || el.textContent || el.value || el.getAttribute('value') || '');
  }

  function depthOf(el) {
    let d = 0;
    let p = el;
    while (p && p.parentElement) { d += 1; p = p.parentElement; }
    return d;
  }

  function deepest(list) {
    let best = null;
    let bd = -1;
    list.forEach((el) => {
      const d = depthOf(el);
      if (d > bd) { bd = d; best = el; }
    });
    return best;
  }

  function findStartButton() {
    const nodes = Array.from(document.querySelectorAll(
      'a, button, div, span, input[type="button"], input[type="submit"], [role="button"]'
    )).filter((el) => !inPanel(el));
    const exact = ['立即开始', '开始作答', '开始填答', '立即填写', '进入问卷', '开始答题', '立即进入', '开始', '进入'];
    for (const want of exact) {
      const hits = nodes.filter((el) => startTextOf(el) === want && (isVisible(el) || el.offsetWidth > 0 || el.offsetHeight > 0));
      if (hits.length) return deepest(hits);
    }
    return deepest(nodes.filter((el) => {
      const t = startTextOf(el);
      if (!t || t.length > 10) return false;
      if (!/开始|进入|填写/.test(t)) return false;
      if (/停止|取消|返回|关闭|隐私|协议/.test(t)) return false;
      return isVisible(el) || el.offsetWidth > 0 || el.offsetHeight > 0;
    })) || null;
  }

  function firstSurveyField() {
    return Array.from(document.querySelectorAll(
      'input[type="text"], input[type="tel"], input[type="number"], textarea, select, input[type="radio"]'
    )).find((el) => !inPanel(el) && isVisible(el)) || null;
  }

  function findStartNode() {
    const visible = findStartButton();
    if (visible) return visible;
    const known = document.getElementById('cgstartbutton');
    if (known && !inPanel(known)) return known;
    const exact = ['立即开始', '开始作答', '开始填答', '立即填写', '进入问卷', '开始答题', '立即进入', '开始', '进入'];
    return deepest(Array.from(document.querySelectorAll(
      'a, button, input[type="button"], input[type="submit"], [role="button"]'
    )).filter((el) => !inPanel(el) && exact.includes(startTextOf(el)))) || null;
  }

  function clickStart(el) {
    try { el.click(); } catch (e) {}
    setTimeout(() => {
      if (firstSurveyField() || !stillClosed()) return;
      try {
        el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true, view: window }));
      } catch (e) {}
      try {
        if (window.jQuery) window.jQuery(el).trigger('click');
      } catch (e) {}
    }, 150);
  }

  function stopWaiting(status, btn) {
    writeStore(WAIT_KEY, false);
    dlog('stopWaiting');
    if (waitTimer) { clearInterval(waitTimer); waitTimer = null; }
    if (btn) btn.textContent = '开启自动等待';
    if (status) status.textContent = '已停止自动等待。';
  }

  async function enterAndFill(status) {
    for (let i = 0; i < 12; i += 1) {
      if (i > 0) await sleep(250);
      const fields = firstSurveyField();
      if (fields && !stillClosed()) {
        writeStore(WAIT_KEY, false);
        writeStore(RELOAD_KEY, 0);
        dlog('entered fields=' + (fields.id || fields.name || fields.type));
        if (filledThisLoad) {
          status.textContent = '已完成填写。';
          return;
        }
        if (hasFixedConfig()) {
          status.textContent = '已进入问卷，自动填写中…';
          await fillAll(status);
        } else {
          status.textContent = '请先保存配置。';
        }
        return;
      }
      const btn = findStartNode();
      if (btn) {
        status.textContent = '等待开放，重试点击…' + (i + 1);
        dlog('retry click#' + (i + 1));
        clickStart(btn);
      }
    }
    const count = Number(readStore(RELOAD_KEY, 0)) + 1;
    if (count > MAX_AUTO_RELOAD) {
      writeStore(WAIT_KEY, false);
      dlog('giveup reload>' + MAX_AUTO_RELOAD);
      status.textContent = '未能进入，请手动刷新。';
      return;
    }
    writeStore(RELOAD_KEY, count);
    dlog('reload#' + count + ' closed=' + stillClosed() + ' fields=' + !!firstSurveyField());
    status.textContent = '自动刷新进入…';
    location.reload();
  }

  function startWaiting(status, btn, target) {
    writeStore(WAIT_KEY, true);
    dlog('wait start target=' + new Date(target).toLocaleTimeString('zh-CN', { hour12: false }));
    startAttempts = 0;
    let zeroSince = 0;
    if (btn) btn.textContent = '停止自动等待';
    if (waitTimer) { clearInterval(waitTimer); clearTimeout(waitTimer); waitTimer = null; }
    const clearTimer = () => {
      if (waitTimer) { clearTimeout(waitTimer); clearInterval(waitTimer); waitTimer = null; }
    };
    const schedule = (ms) => {
      clearTimer();
      waitTimer = setTimeout(tick, ms);
    };
    const tick = () => {
      try {
        const openNow = !parseOpenTarget();
        const live = liveCountdown();
        const remain = live ? live.ms : (pageCountdownSeen ? 0 : target - Date.now());
        const fieldsNow = firstSurveyField();
        if (openNow && !live && fieldsNow && !filledThisLoad) {
          clearTimer();
          writeStore(WAIT_KEY, false);
          writeStore(RELOAD_KEY, 0);
          status.textContent = '问卷已开放，自动填写中…';
          if (hasFixedConfig()) fillAll(status);
          else status.textContent = '请先保存配置。';
          return;
        }
        if (remain > 0) {
          zeroSince = 0;
          status.textContent = '倒计时 ' + (live ? live.label : fmtRemain(remain));
          schedule(remain <= 1200 ? 10 : (remain <= 3000 ? 100 : 1000));
          return;
        }
        if (!zeroSince) zeroSince = Date.now();
        const startBtn = findStartNode();
        if (startBtn) {
          clearTimer();
          status.textContent = '倒计时结束，点击开始…';
          waitTimer = setTimeout(() => {
            waitTimer = null;
            const btnNow = findStartNode() || startBtn;
            dlog('click#' + (btnNow.id || btnNow.tagName) + ':' + startTextOf(btnNow) + ' delay=' + (Date.now() - zeroSince));
            clickStart(btnNow);
            enterAndFill(status);
          }, CLICK_DELAY_MS);
          return;
        }
        startAttempts += 1;
        if (Date.now() - zeroSince < 3000) {
          status.textContent = '等待开始按钮…';
          schedule(100);
          return;
        }
        const count = Number(readStore(RELOAD_KEY, 0)) + 1;
        if (count > MAX_AUTO_RELOAD) {
          stopWaiting(status, btn);
          status.textContent = '仍未开放，已停止。';
          return;
        }
        writeStore(RELOAD_KEY, count);
        dlog('tick reload#' + count);
        status.textContent = '倒计时结束，自动刷新进入…';
        clearTimer();
        location.reload();
      } catch (e) {
        status.textContent = '等待出错：' + (e && e.message ? e.message : e);
        schedule(500);
      }
    };
    tick();
  }

  async function startFill(status, fixedBox) {
    if (stillClosed()) {
      status.textContent = '问卷未开放。';
      return;
    }
    if (!hasFixedConfig()) {
      status.textContent = '请先保存配置。';
      fixedBox.style.display = 'block';
      fixedBox.loadValues();
      return;
    }
    await fillAll(status);
  }

  function scanUnanswered() {
    const root = surveyRoot();
    const items = [];
    const usable = (el) => !inPanel(el) && isEnabled(el) && choiceVisible(el) && !isConsent(el);

    groupBy(Array.from(root.querySelectorAll('input[type="radio"]')).filter(usable), groupKey)
      .forEach((inputs) => {
        if (inputs.some((el) => el.checked)) return;
        items.push({
          kind: 'radio',
          title: questionTitle(inputs[0]),
          options: inputs.map((el) => ({ label: optionLabel(el), el }))
        });
      });

    groupBy(Array.from(root.querySelectorAll('input[type="checkbox"]')).filter(usable), groupKey)
      .forEach((inputs) => {
        if (inputs.some((el) => el.checked)) return;
        items.push({
          kind: 'checkbox',
          title: questionTitle(inputs[0]),
          options: inputs.map((el) => ({ label: optionLabel(el), el }))
        });
      });

    const customSel = '[role="radio"], [role="checkbox"], .ui-radio, .ui-checkbox, .radio, .checkbox';
    const customEls = Array.from(root.querySelectorAll(customSel)).filter((el) => {
      if (inPanel(el) || !isVisible(el) || isConsent(el)) return false;
      return !el.querySelector('input[type="radio"], input[type="checkbox"]');
    });
    groupBy(customEls, (el) => {
      const role = el.getAttribute('role') || (/checkbox/.test(el.className) ? 'checkbox' : 'radio');
      const block = questionBlock(el);
      return role + '|' + (block.id || block.getAttribute('data-id') || clean(block.innerText).slice(0, 60));
    }).forEach((els, key) => {
      if (els.some((el) => el.getAttribute('aria-checked') === 'true')) return;
      items.push({
        kind: key.split('|')[0] === 'checkbox' ? 'custom-checkbox' : 'custom-radio',
        title: questionTitle(els[0]),
        options: els.map((el) => ({ label: clean(el.innerText) || '选项', el }))
      });
    });

    Array.from(root.querySelectorAll('select')).forEach((select) => {
      if (inPanel(select) || !isVisible(select) || !isEnabled(select)) return;
      const options = Array.from(select.options).filter((opt) => !opt.disabled && opt.value !== '');
      if (!options.length) return;
      if (Array.from(select.selectedOptions).some((opt) => opt.value !== '')) return;
      items.push({
        kind: 'select',
        title: questionTitle(select),
        el: select,
        options: options.map((opt) => ({ label: clean(opt.innerText || opt.textContent || opt.value), el: opt }))
      });
    });

    const textSel = 'input[type="text"], input[type="tel"], input[type="email"], input[type="number"],'
      + 'input[type="date"], input[type="datetime-local"], input[type="url"], input:not([type]), textarea';
    Array.from(root.querySelectorAll(textSel)).forEach((el) => {
      if (inPanel(el) || !isVisible(el) || !isEditableText(el) || clean(el.value)) return;
      items.push({ kind: 'text', title: questionTitle(el), el });
    });

    return items;
  }

  function applyPick(item, option, status) {
    const kind = item.kind;
    if (kind === 'radio') {
      setChecked(option.el, true);
      rememberRule(item.title, option.label);
    } else if (kind === 'checkbox') {
      setChecked(option.el, !option.el.checked);
      const picked = item.options.filter((o) => o.el.checked).map((o) => o.label);
      if (picked.length) rememberRule(item.title, picked.join('|'));
    } else if (kind === 'select') {
      if (!touchedSelect.has(item.el)) touchedSelect.set(item.el, item.el.value);
      if (item.el.multiple) {
        option.el.selected = !option.el.selected;
        fire(item.el, ['input', 'change']);
        const picked = Array.from(item.el.selectedOptions).map((o) => clean(o.innerText || o.value)).filter(Boolean);
        if (picked.length) rememberRule(item.title, picked.join('|'));
      } else {
        setNativeValue(item.el, option.el.value);
        rememberRule(item.title, option.label);
      }
    } else if (kind === 'text') {
      const value = clean(option && option.value);
      if (!value) return false;
      if (!touchedText.has(item.el)) touchedText.set(item.el, item.el.value);
      setNativeValue(item.el, value);
      rememberRule(item.title, value);
    } else {
      const el = option.el;
      if (!touchedCustom.has(el)) touchedCustom.set(el, el.getAttribute('aria-checked'));
      el.dataset.wjxFfCustom = '1';
      try { el.click(); } catch (e) {}
      const want = kind === 'custom-checkbox' ? el.getAttribute('aria-checked') !== 'true' : true;
      if ((el.getAttribute('aria-checked') === 'true') !== want) {
        el.setAttribute('aria-checked', want ? 'true' : 'false');
        fire(el, ['input', 'change', 'click']);
      }
      const picked = item.options.filter((o) => o.el.getAttribute('aria-checked') === 'true').map((o) => o.label);
      if (picked.length) rememberRule(item.title, picked.join('|'));
    }
    if (status) status.textContent = '已选：' + clean(option && (option.label || option.value)).slice(0, 24);
    return true;
  }

  function renderPicker(status) {
    const box = document.getElementById('wjx-ff-picker');
    if (!box) return 0;
    box.innerHTML = '';
    const items = scanUnanswered();
    if (!items.length) {
      box.style.display = 'none';
      if (allowSubmit() && filledThisLoad && !submittedThisLoad) {
        box.style.display = 'block';
        const done = document.createElement('div');
        done.style.cssText = 'display:flex;align-items:center;gap:6px;margin-top:8px;border-top:1px solid #374151;padding-top:8px';
        const txt = document.createElement('span');
        txt.textContent = '全部已填';
        txt.style.cssText = 'color:#34d399;flex:1';
        const sub = document.createElement('button');
        sub.type = 'button';
        sub.textContent = '勾选并提交';
        sub.style.cssText = 'border:0;border-radius:8px;padding:5px 9px;background:#059669;color:#fff;cursor:pointer';
        sub.addEventListener('click', () => {
          if (!clickConsent()) { status.textContent = '未找到同意框。'; return; }
          submitNow(status);
          renderPicker(status);
        });
        done.appendChild(txt);
        done.appendChild(sub);
        box.appendChild(done);
      }
      return 0;
    }
    box.style.display = 'block';
    box.style.cssText += ';margin-top:8px;border-top:1px solid #374151;padding-top:8px';

    const head = document.createElement('div');
    head.style.cssText = 'display:flex;align-items:center;gap:6px;margin-bottom:6px';
    const title = document.createElement('span');
    title.textContent = '待选题目 ' + items.length;
    title.style.cssText = 'color:#fbbf24;flex:1';
    const remLabel = document.createElement('label');
    remLabel.style.cssText = 'display:flex;align-items:center;gap:4px;color:#9ca3af;cursor:pointer';
    const rem = document.createElement('input');
    rem.type = 'checkbox';
    rem.checked = rememberPicks();
    rem.addEventListener('change', () => writeStore(REMEMBER_KEY, rem.checked));
    const remText = document.createElement('span');
    remText.textContent = '记住';
    remLabel.appendChild(rem);
    remLabel.appendChild(remText);
    head.appendChild(title);
    head.appendChild(remLabel);
    box.appendChild(head);

    items.slice(0, 12).forEach((item) => {
      const row = document.createElement('div');
      row.style.cssText = 'margin-bottom:6px';
      const t = document.createElement('div');
      t.textContent = item.title.slice(0, 34);
      t.style.cssText = 'color:#d1d5db;margin-bottom:3px;line-height:1.35';
      row.appendChild(t);
      if (item.kind === 'text') {
        const line = document.createElement('div');
        line.style.cssText = 'display:flex;gap:5px';
        const input = document.createElement('input');
        input.type = 'text';
        input.placeholder = '填这题';
        input.style.cssText = 'flex:1;min-width:0;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
        const ok = document.createElement('button');
        ok.type = 'button';
        ok.textContent = '填入';
        ok.style.cssText = 'border:0;border-radius:6px;padding:4px 9px;background:#2563eb;color:#fff;cursor:pointer';
        const doFill = () => {
          if (!applyPick(item, { value: input.value }, status)) return;
          renderPicker(status);
        };
        ok.addEventListener('click', doFill);
        input.addEventListener('keydown', (e) => { if (e.key === 'Enter') doFill(); });
        line.appendChild(input);
        line.appendChild(ok);
        row.appendChild(line);
      } else {
        const wrap = document.createElement('div');
        wrap.style.cssText = 'display:flex;flex-wrap:wrap;gap:4px';
        item.options.slice(0, 12).forEach((opt) => {
          const b = document.createElement('button');
          b.type = 'button';
          b.textContent = opt.label.slice(0, 14);
          const isOn = () => (item.kind === 'checkbox' ? opt.el.checked
            : item.kind === 'custom-checkbox' ? opt.el.getAttribute('aria-checked') === 'true'
              : item.kind === 'select' && item.el.multiple ? opt.el.selected : false);
          const paint = () => {
            const on = isOn();
            b.style.cssText = 'border:1px solid ' + (on ? '#2563eb' : '#374151') + ';border-radius:6px;padding:3px 7px;'
              + 'background:' + (on ? '#2563eb' : '#1f2937') + ';color:#f9fafb;cursor:pointer;max-width:100%';
          };
          paint();
          b.addEventListener('click', () => {
            applyPick(item, opt, status);
            const multi = item.kind === 'checkbox' || item.kind === 'custom-checkbox'
              || (item.kind === 'select' && item.el.multiple);
            if (multi) {
              paint();
              const left = scanUnanswered().length;
              title.textContent = left ? '待选题目 ' + left : '待选题目 0';
              if (!left) renderPicker(status);
            } else {
              renderPicker(status);
            }
          });
          wrap.appendChild(b);
        });
        row.appendChild(wrap);
      }
      box.appendChild(row);
    });

    if (items.length > 12) {
      const more = document.createElement('div');
      more.textContent = '还有 ' + (items.length - 12) + ' 题，填完上面继续显示';
      more.style.cssText = 'color:#9ca3af';
      box.appendChild(more);
    }
    return items.length;
  }

  function buildFixedBox(status) {
    const box = document.createElement('div');
    box.style.cssText = 'margin-top:8px;border-top:1px solid #374151;padding-top:8px';

    const tip = document.createElement('div');
    tip.textContent = '留空不填·仅本机保存';
    tip.style.cssText = 'color:#9ca3af;margin-bottom:6px';
    box.appendChild(tip);

    const delayRow = document.createElement('div');
    delayRow.style.cssText = 'display:flex;align-items:center;gap:6px;margin-bottom:6px;color:#d1d5db';
    const delayLabel = document.createElement('span');
    delayLabel.textContent = '随机延时(ms)';
    const delayMin = document.createElement('input');
    delayMin.type = 'number';
    delayMin.min = '0';
    delayMin.style.cssText = 'width:64px;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
    const delayMax = document.createElement('input');
    delayMax.type = 'number';
    delayMax.min = '0';
    delayMax.style.cssText = 'width:64px;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
    const range = delayRange();
    delayMin.value = String(range.min);
    delayMax.value = String(range.max);
    const saveDelay = () => {
      const min = Math.max(0, Number(delayMin.value) || 0);
      const max = Math.max(min, Number(delayMax.value) || min);
      writeStore(DELAY_KEY, { min, max });
      status.textContent = '延时 ' + min + '–' + max + 'ms。';
    };
    delayMin.addEventListener('change', saveDelay);
    delayMax.addEventListener('change', saveDelay);
    delayRow.appendChild(delayLabel);
    delayRow.appendChild(delayMin);
    delayRow.appendChild(document.createTextNode('–'));
    delayRow.appendChild(delayMax);
    box.appendChild(delayRow);

    const inputs = {};
    FIXED_FIELDS.forEach(([key, label]) => {
      const row = document.createElement('label');
      row.style.cssText = 'display:flex;align-items:center;gap:6px;margin-bottom:4px';
      const span = document.createElement('span');
      span.textContent = label;
      span.style.cssText = 'width:56px;color:#d1d5db';
      const input = document.createElement('input');
      input.type = 'text';
      input.placeholder = FIELD_PLACEHOLDER[key] || '留空不填';
      input.style.cssText = 'flex:1;min-width:0;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
      row.appendChild(span);
      row.appendChild(input);
      box.appendChild(row);
      inputs[key] = input;
    });

    const fbRow = document.createElement('label');
    fbRow.style.cssText = 'display:flex;align-items:center;gap:6px;margin-top:6px';
    const fbSpan = document.createElement('span');
    fbSpan.textContent = '填空兜底';
    fbSpan.style.cssText = 'width:56px;color:#d1d5db';
    const fbInput = document.createElement('input');
    fbInput.type = 'text';
    fbInput.placeholder = '留空=未匹配不填';
    fbInput.style.cssText = 'flex:1;min-width:0;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
    fbInput.value = fallbackText();
    fbInput.addEventListener('change', () => writeStore(FALLBACK_KEY, clean(fbInput.value)));
    fbRow.appendChild(fbSpan);
    fbRow.appendChild(fbInput);
    box.appendChild(fbRow);

    const ruleHead = document.createElement('div');
    ruleHead.style.cssText = 'display:flex;align-items:center;gap:6px;margin-top:8px;border-top:1px solid #374151;padding-top:8px';
    const ruleTitle = document.createElement('span');
    ruleTitle.textContent = '自定义题目';
    ruleTitle.style.cssText = 'color:#d1d5db;flex:1';
    const ruleAdd = document.createElement('button');
    ruleAdd.type = 'button';
    ruleAdd.textContent = '＋ 加一题';
    ruleAdd.style.cssText = 'border:1px solid #374151;border-radius:6px;padding:3px 8px;background:#1f2937;color:#f9fafb;cursor:pointer';
    ruleHead.appendChild(ruleTitle);
    ruleHead.appendChild(ruleAdd);
    box.appendChild(ruleHead);

    const ruleTip = document.createElement('div');
    ruleTip.textContent = '题目关键词 → 答案；多选用 | 分隔';
    ruleTip.style.cssText = 'color:#9ca3af;margin:4px 0 6px';
    box.appendChild(ruleTip);

    const ruleList = document.createElement('div');
    box.appendChild(ruleList);
    const ruleRows = [];

    function addRuleRow(q, a) {
      const row = document.createElement('div');
      row.style.cssText = 'display:flex;gap:4px;margin-bottom:4px';
      const qi = document.createElement('input');
      qi.type = 'text';
      qi.placeholder = '题目关键词';
      qi.value = q || '';
      qi.style.cssText = 'flex:1;min-width:0;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
      const ai = document.createElement('input');
      ai.type = 'text';
      ai.placeholder = '答案/选项';
      ai.value = a || '';
      ai.style.cssText = 'flex:1;min-width:0;border:1px solid #374151;border-radius:6px;background:#1f2937;color:#f9fafb;padding:4px 6px';
      const del = document.createElement('button');
      del.type = 'button';
      del.textContent = '×';
      del.style.cssText = 'border:1px solid #374151;border-radius:6px;padding:2px 7px;background:#1f2937;color:#9ca3af;cursor:pointer';
      const entry = { row, qi, ai };
      del.addEventListener('click', () => {
        row.remove();
        const i = ruleRows.indexOf(entry);
        if (i >= 0) ruleRows.splice(i, 1);
        saveRules(currentRules());
      });
      row.appendChild(qi);
      row.appendChild(ai);
      row.appendChild(del);
      ruleList.appendChild(row);
      ruleRows.push(entry);
    }

    function currentRules() {
      return ruleRows.map((r) => ({ q: clean(r.qi.value), a: clean(r.ai.value) })).filter((r) => r.q && r.a);
    }

    ruleAdd.addEventListener('click', () => addRuleRow('', ''));
    loadRules().forEach((r) => addRuleRow(r.q, r.a));

    const btnRow = document.createElement('div');
    btnRow.style.cssText = 'display:flex;gap:6px;margin-top:6px';

    const save = document.createElement('button');
    save.type = 'button';
    save.textContent = '保存固定答案';
    save.style.cssText = 'border:0;border-radius:8px;padding:6px 10px;background:#059669;color:#fff;cursor:pointer';
    save.addEventListener('click', () => {
      const obj = {};
      FIXED_FIELDS.forEach(([key]) => {
        const value = inputs[key].value.trim();
        if (value) obj[key] = value;
      });
      saveFixed(obj);
      saveRules(currentRules());
      writeStore(FALLBACK_KEY, clean(fbInput.value));
      status.textContent = '已保存。';
    });
    btnRow.appendChild(save);
    box.appendChild(btnRow);

    const moreRow = document.createElement('div');
    moreRow.style.cssText = 'display:flex;gap:6px;margin-top:6px';
    const clearFill = document.createElement('button');
    clearFill.type = 'button';
    clearFill.textContent = '清空恢复';
    clearFill.style.cssText = 'border:0;border-radius:8px;padding:6px 10px;background:#4b5563;color:#fff;cursor:pointer';
    clearFill.addEventListener('click', () => clearAll(status));
    const copyAnswer = document.createElement('button');
    copyAnswer.type = 'button';
    copyAnswer.textContent = '复制答案';
    copyAnswer.style.cssText = 'border:0;border-radius:8px;padding:6px 10px;background:#4b5563;color:#fff;cursor:pointer';
    copyAnswer.addEventListener('click', () => copyResult(status));
    moreRow.appendChild(clearFill);
    moreRow.appendChild(copyAnswer);
    const copyLog = document.createElement('button');
    copyLog.type = 'button';
    copyLog.textContent = '复制诊断';
    copyLog.style.cssText = 'border:0;border-radius:8px;padding:6px 10px;background:#4b5563;color:#fff;cursor:pointer';
    copyLog.addEventListener('click', async () => {
      const text = location.href + '\n' + (readStore(LOG_KEY, []) || []).join('\n');
      try {
        await navigator.clipboard.writeText(text);
        status.textContent = '已复制诊断日志。';
      } catch (e) {
        const area = document.createElement('textarea');
        area.value = text;
        area.style.position = 'fixed';
        area.style.left = '-9999px';
        document.body.appendChild(area);
        area.select();
        document.execCommand('copy');
        area.remove();
        status.textContent = '已复制诊断日志。';
      }
    });
    moreRow.appendChild(copyLog);
    box.appendChild(moreRow);

    box.loadValues = () => {
      const fixed = loadFixed();
      FIXED_FIELDS.forEach(([key]) => { inputs[key].value = fixed[key] || ''; });
      const r = delayRange();
      delayMin.value = String(r.min);
      delayMax.value = String(r.max);
      fbInput.value = fallbackText();
      ruleList.innerHTML = '';
      ruleRows.length = 0;
      loadRules().forEach((rule) => addRuleRow(rule.q, rule.a));
    };
    return box;
  }

  function buildPanel() {
    if (document.getElementById(PANEL_ID)) return;
    if (!document.getElementById('wjx-ff-style')) {
      const style = document.createElement('style');
      style.id = 'wjx-ff-style';
      style.textContent = '#' + PANEL_ID + ' button{transition:transform .08s,filter .12s}'
        + '#' + PANEL_ID + ' button:hover{filter:brightness(1.15)}'
        + '#' + PANEL_ID + ' button:active{transform:scale(.94);filter:brightness(1.3)}'
        + '#' + PANEL_ID + ' input[type="checkbox"]{appearance:none;-webkit-appearance:none;width:13px;height:13px;margin:0;'
        + 'border:1px solid #6b7280;border-radius:3px;background:#0f1728;cursor:pointer;position:relative;flex:0 0 auto}'
        + '#' + PANEL_ID + ' input[type="checkbox"]:checked{background:#2563eb;border-color:#2563eb}'
        + '#' + PANEL_ID + ' input[type="checkbox"]:checked::after{content:"";position:absolute;left:3px;top:0;width:4px;height:8px;'
        + 'border:solid #fff;border-width:0 2px 2px 0;transform:rotate(45deg)}';
      document.head.appendChild(style);
    }
    const panel = document.createElement('div');
    panel.id = PANEL_ID;
    panel.style.cssText = [
      'position:fixed', 'right:16px', 'bottom:16px', 'z-index:2147483647',
      'width:320px', 'padding:12px', 'border-radius:14px',
      'background:#111827', 'color:#f9fafb', 'font:13px/1.5 system-ui,sans-serif',
      'border:1px solid #1f2937', 'box-shadow:0 10px 28px rgba(0,0,0,.4)'
    ].join(';');

    const head = document.createElement('div');
    head.style.cssText = 'display:flex;align-items:flex-start;gap:10px';

    const title = document.createElement('div');
    title.textContent = '问卷星填写';
    title.style.cssText = 'flex:0 0 auto;padding:6px 12px;border-radius:10px;background:#1f2937;'
      + 'border:1px solid #374151;font-weight:700;white-space:nowrap';

    const headRight = document.createElement('div');
    headRight.style.cssText = 'flex:1;min-width:0;display:flex;flex-direction:column;gap:4px;align-items:flex-end';

    const status = document.createElement('div');
    status.id = 'wjx-ff-status';
    status.textContent = '就绪';
    status.style.cssText = 'color:#d1d5db;font-size:11.5px;line-height:1.35;text-align:right;'
      + 'word-break:break-all;min-height:16px';

    const optRow = document.createElement('label');
    optRow.style.cssText = 'display:flex;align-items:center;gap:6px;color:#9ca3af;font-size:11.5px;cursor:pointer';
    const opt = document.createElement('input');
    opt.type = 'checkbox';
    opt.id = 'wjx-ff-submit-opt';
    opt.checked = allowSubmit();
    const optText = document.createElement('span');
    const syncOptText = () => { optText.textContent = opt.checked ? '勾选同意并提交' : '仅填写不提交'; };
    syncOptText();
    opt.addEventListener('change', () => {
      writeStore(ALLOW_SUBMIT_KEY, opt.checked);
      syncOptText();
      status.textContent = opt.checked ? '将勾选同意并提交。' : '仅填写，不提交。';
    });
    optRow.appendChild(opt);
    optRow.appendChild(optText);

    headRight.appendChild(status);
    headRight.appendChild(optRow);

    const minBtn = document.createElement('button');
    minBtn.type = 'button';
    minBtn.textContent = '—';
    minBtn.title = '收起/展开';
    minBtn.style.cssText = 'flex:0 0 auto;border:0;border-radius:6px;padding:2px 8px;background:#374151;color:#fff;cursor:pointer';

    head.appendChild(title);
    head.appendChild(headRight);
    head.appendChild(minBtn);

    const body = document.createElement('div');
    body.id = 'wjx-ff-body';
    body.style.cssText = 'margin-top:10px';

    const row = document.createElement('div');
    row.id = 'wjx-ff-mainrow';
    row.style.cssText = 'display:flex;gap:8px';

    function button(label, onClick, parent, variant) {
      const btn = document.createElement('button');
      btn.type = 'button';
      btn.textContent = label;
      const base = variant === 'ghost'
        ? 'border:1px solid #374151;border-radius:8px;padding:3px 8px;background:transparent;color:#9ca3af;font-size:11px;cursor:pointer'
        : 'flex:1;border:0;border-radius:10px;padding:9px 10px;color:#fff;font-size:13px;cursor:pointer;background:'
          + (variant === 'secondary' ? '#374151' : '#2563eb');
      btn.style.cssText = base;
      btn.addEventListener('click', () => {
        Promise.resolve(onClick(status)).catch(() => { status.textContent = '操作出错。'; });
      });
      (parent || row).appendChild(btn);
      return btn;
    }

    const fixedBox = buildFixedBox(status);
    fixedBox.style.display = 'none';

    button('填写配置信息', async () => {
      const show = fixedBox.style.display === 'none';
      fixedBox.style.display = show ? 'block' : 'none';
      if (show) fixedBox.loadValues();
      status.textContent = show ? '配置已展开。' : '配置已收起。';
    }, row, 'secondary');
    button('自动填写', () => startFill(status, fixedBox), row, 'primary');

    const footRow = document.createElement('div');
    footRow.style.cssText = 'display:flex;align-items:center;justify-content:space-between;margin-top:6px;min-height:20px';
    const version = document.createElement('span');
    version.id = 'wjx-ff-version';
    version.textContent = 'v' + VERSION;
    version.style.cssText = 'color:#6b7280;font-size:11px';
    const waitSlot = document.createElement('span');
    waitSlot.id = 'wjx-ff-waitslot';
    footRow.appendChild(version);
    footRow.appendChild(waitSlot);

    if (window.__wjxDemoFastFillDuplicate) {
      const warn = document.createElement('div');
      warn.textContent = '检测到旧版脚本残留：请在篡改猴中删除旧条目，仅保留 v' + VERSION + '。';
      warn.style.cssText = 'color:#fca5a5;margin-bottom:8px;font-size:11.5px';
      body.appendChild(warn);
      dlog('duplicate takeover count=' + window.__wjxDemoFastFillDuplicate);
    }
    body.appendChild(row);
    body.appendChild(footRow);
    const picker = document.createElement('div');
    picker.id = 'wjx-ff-picker';
    picker.style.display = 'none';
    body.appendChild(picker);
    body.appendChild(fixedBox);

    minBtn.addEventListener('click', () => {
      const hide = body.style.display !== 'none';
      body.style.display = hide ? 'none' : 'block';
      minBtn.textContent = hide ? '+' : '—';
      panel.style.width = hide ? 'auto' : '320px';
    });

    panel.appendChild(head);
    panel.appendChild(body);
    document.body.appendChild(panel);

    let waitBtn = null;
    const ensureWaitBtn = () => {
      if (!waitBtn) {
        waitBtn = button('停止自动等待', async () => {
          if (readStore(WAIT_KEY, false)) stopWaiting(status, waitBtn);
          else startWaiting(status, waitBtn, parseOpenTarget() || Date.now());
        }, waitSlot, 'ghost');
      }
      return waitBtn;
    };
    const openTarget = parseOpenTarget();
    dlog('boot url=' + location.pathname + ' closed=' + !!openTarget + ' wait=' + readStore(WAIT_KEY, false) + ' fields=' + !!firstSurveyField());
    if (openTarget) {
      startWaiting(status, ensureWaitBtn(), openTarget);
    } else if (readStore(WAIT_KEY, false)) {
      writeStore(WAIT_KEY, false);
      writeStore(RELOAD_KEY, 0);
      if (hasFixedConfig()) {
        status.textContent = '自动填写中…';
        setTimeout(() => { fillAll(status); }, 100);
      } else {
        status.textContent = '请先保存配置。';
      }
    }
    const surveyFieldsNow = Array.from(document.querySelectorAll(
      '#divQuestion input, #divQuestion textarea, #divQuestion select, form input, form textarea, form select'
    )).some((el) => !inPanel(el));
    if (!openTarget && !surveyFieldsNow) {
      let watchTicks = 0;
      watchIvRef = setInterval(() => {
        watchTicks += 1;
        const t = parseOpenTarget();
        if (t && !waitTimer) {
          clearInterval(watchIvRef);
          watchIvRef = null;
          startWaiting(status, ensureWaitBtn(), t);
        } else if (waitTimer || watchTicks > 120) {
          clearInterval(watchIvRef);
          watchIvRef = null;
        }
      }, 1000);
    }
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', buildPanel, { once: true });
  } else {
    buildPanel();
  }
})();