
// ==UserScript==
// @name         PNE4 - Clock to First Activity v9.1
// @namespace    http://tampermonkey.net/
// @version      9.1
// @description  Date fix for timeDetails. Full color tabs. Night shift fix. Panel persists.
// @match        https://fclm-portal.amazon.com/reports/functionRollup*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    var PROC_MAP = { '1002999': 'Pallet Stow', '1002967': 'Case Replen', '1002997': 'Pallet Replen' };
    var PROC_TABS = [
        { id: '1002997', label: 'Pallet Replen', color: '#fff', bg: '#dc2626', bgInactive: 'rgba(248,113,113,0.15)', border: '#ef4444', textInactive: '#f87171' },
        { id: '1002967', label: 'Case Replen', color: '#fff', bg: '#2563eb', bgInactive: 'rgba(96,165,250,0.15)', border: '#3b82f6', textInactive: '#60a5fa' },
        { id: '1002999', label: 'Pallet Stow', color: '#1a1a2e', bg: '#eab308', bgInactive: 'rgba(250,204,21,0.15)', border: '#eab308', textInactive: '#facc15' }
    ];
    var SHIFTS = {
        day: { label: '☀️ Day Shift', presets: [
            { label: '🌅 Shift Start', start: '06:45', end: '07:30', type: 'clockin' },
            { label: '☕ 1st Break', start: '10:15', end: '11:00', type: 'clockin' },
            { label: '🕳️ 2nd Break Gap', start: '13:00', end: '14:30', type: 'gap' }
        ]},
        night: { label: '🌙 Night Shift', presets: [
            { label: '🌅 Shift Start', start: '17:40', end: '18:30', type: 'clockin' },
            { label: '☕ 2nd Break', start: '21:00', end: '22:00', type: 'clockin' },
            { label: '🕳️ 2nd Break Gap', start: '00:00', end: '03:00', type: 'gap' }
        ]}
    };
    var CLOCKIN_BUFFER = 15;
    var TH_CLOCKIN = [
        { max: 17, color: '#4ade80', bg: 'rgba(74,222,128,0.12)', icon: '🟢' },
        { max: 20, color: '#facc15', bg: 'rgba(250,204,21,0.12)', icon: '🟡' },
        { max: 25, color: '#fb923c', bg: 'rgba(251,146,60,0.12)', icon: '🟠' },
        { max: Infinity, color: '#f87171', bg: 'rgba(248,113,113,0.12)', icon: '🔴' }
    ];
    var TH_GAP = [
        { max: 35, color: '#4ade80', bg: 'rgba(74,222,128,0.12)', icon: '🟢', tag: 'Good' },
        { max: 45, color: '#facc15', bg: 'rgba(250,204,21,0.12)', icon: '🟡', tag: 'Moderate' },
        { max: 55, color: '#fb923c', bg: 'rgba(251,146,60,0.12)', icon: '🟠', tag: 'Poor' },
        { max: Infinity, color: '#f87171', bg: 'rgba(248,113,113,0.12)', icon: '🔴', tag: 'Very Poor' }
    ];
    var GAP_MIN_THRESHOLD = 25;
    var HIGHLIGHT = { top: { color: '#60a5fa', bg: 'rgba(96,165,250,0.15)', border: '#3b82f6' }, bottom: { color: '#f87171', bg: 'rgba(248,113,113,0.15)', border: '#ef4444' } };
    var D = { bg: '#1a1a2e', bgCard: '#16213e', bgSection: '#0f3460', border: '#1f4068', text: '#e8e8e8', textMuted: '#a0aec0', accent: '#ff9900', gold: '#f5a623' };

    function getThreshold(m, isGap) { var arr = isGap ? TH_GAP : TH_CLOCKIN; for (var i = 0; i < arr.length; i++) { if (m <= arr[i].max) return arr[i]; } return arr[arr.length - 1]; }
    function getParams() { var p = new URLSearchParams(window.location.search); return { wh: p.get('warehouseId') || 'PNE4', pid: p.get('processId') || '', sH: parseInt(p.get('startHourIntraday')) || 7, sM: parseInt(p.get('startMinuteIntraday')) || 0, eH: parseInt(p.get('endHourIntraday')) || 10, eM: parseInt(p.get('endMinuteIntraday')) || 30, fmt: p.get('reportFormat') || 'HTML', maxD: p.get('maxIntradayDays') || '1' }; }
    function getProcName() { return PROC_MAP[getParams().pid] || 'Process'; }
    function fmtDate(d) { return d.getFullYear() + '/' + String(d.getMonth() + 1).padStart(2, '0') + '/' + String(d.getDate()).padStart(2, '0'); }
    function fmtDateInput(d) { return d.getFullYear() + '-' + String(d.getMonth() + 1).padStart(2, '0') + '-' + String(d.getDate()).padStart(2, '0'); }
    function getDateFromURL() { var p = new URLSearchParams(window.location.search); var ds = p.get('startDateDay') || p.get('startDateIntraday'); if (ds) { var x = ds.split(/[\/\-]/); return new Date(x[0], x[1] - 1, x[2]); } return new Date(); }

    function buildURLFull(date, processId, sH, sM, eH, eM) {
        var p = getParams(); var ds = fmtDate(date);
        return 'https://fclm-portal.amazon.com/reports/functionRollup?reportFormat=' + p.fmt + '&warehouseId=' + p.wh + '&processId=' + processId + '&startDateDay=' + encodeURIComponent(ds) + '&maxIntradayDays=' + p.maxD + '&spanType=Intraday&startDateIntraday=' + encodeURIComponent(ds) + '&startHourIntraday=' + sH + '&startMinuteIntraday=' + sM + '&endDateIntraday=' + encodeURIComponent(ds) + '&endHourIntraday=' + eH + '&endMinuteIntraday=' + eM;
    }
    function buildURL(date) { var p = getParams(); return buildURLFull(date, p.pid, p.sH, p.sM, p.eH, p.eM); }
    function buildURLWithTime(date, st, et) { var p = getParams(); var s = st.split(':'); var e = et.split(':'); return buildURLFull(date, p.pid, parseInt(s[0]), parseInt(s[1]) || 0, parseInt(e[0]), parseInt(e[1]) || 0); }
    function saveState(sh, md) { sessionStorage.setItem('c2a_panel_open', 'true'); sessionStorage.setItem('c2a_shift', sh); sessionStorage.setItem('c2a_mode', md); }
    function goToDate(d, sh, md) { saveState(sh, md); window.location.href = buildURL(d); }

    function makeDrag(el, handle) {
        var ox, oy, drag = false; handle.style.cursor = 'move';
        handle.addEventListener('mousedown', function (e) { var t = e.target.tagName.toLowerCase(); if (t === 'select' || t === 'input' || t === 'button' || t === 'option') return; drag = true; ox = e.clientX - el.getBoundingClientRect().left; oy = e.clientY - el.getBoundingClientRect().top; e.preventDefault(); });
        document.addEventListener('mousemove', function (e) { if (!drag) return; el.style.left = (e.clientX - ox) + 'px'; el.style.top = (e.clientY - oy) + 'px'; el.style.right = 'auto'; });
        document.addEventListener('mouseup', function () { drag = false; });
    }

    function t2m(t) { if (!t) return null; var p = t.split(':').map(Number); return p[0] * 60 + (p[1] || 0) + ((p[2] || 0) / 60); }
    function parseT(t) { var p = t.split(':').map(Number); return { h: p[0], m: p[1] || 0, s: p[2] || 0 }; }
    function addMins(t, add) { var p = parseT(t); var tot = p.h * 60 + p.m + add; return { h: Math.floor(tot / 60) % 24, m: tot % 60 }; }
    function inWindow(tM, sM, eM) { return sM <= eM ? (tM >= sM && tM <= eM) : (tM >= sM || tM <= eM); }
    function sleep(ms) { return new Promise(function (r) { setTimeout(r, ms); }); }

    function parseEmps() {
        var emps = [], seen = new Set(), rows = document.querySelectorAll('tr');
        for (var i = 0; i < rows.length; i++) {
            var rH = rows[i].innerHTML || '', idM = rH.match(/employeeId=(\d+)/);
            if (!idM) continue; var empId = idM[1]; if (seen.has(empId)) continue; seen.add(empId);
            var login = '', links = rows[i].querySelectorAll('a');
            for (var k = 0; k < links.length; k++) { var lt = (links[k].innerText || '').trim(); if (/^[a-z][a-z0-9]{4,14}$/.test(lt) && lt !== 'total' && lt !== 'hours' && lt !== 'units') { login = lt; break; } }
            if (!login) { var cells = rows[i].querySelectorAll('td'); for (var j = 0; j < cells.length; j++) { var ct = (cells[j].innerText || '').trim(); if (/^[a-z][a-z0-9]{4,14}$/.test(ct) && ct !== 'total' && ct !== 'hours' && ct !== 'units') { login = ct; break; } } }
            if (!login) { for (var l = 0; l < links.length; l++) { var href = links[l].getAttribute('href') || ''; var lm = href.match(/employeeLogin=([a-z][a-z0-9]{4,14})/); if (lm) { login = lm[1]; break; } } }
            if (!login) { var words = (rows[i].innerText || '').split(/\s+/); for (var w = 0; w < words.length; w++) { var wd = words[w].trim(); if (/^[a-z][a-z0-9]{4,14}$/.test(wd) && wd !== 'total' && wd !== 'hours' && wd !== 'units' && wd !== 'small' && wd !== 'medium' && wd !== 'large') { login = wd; break; } } }
            emps.push({ id: empId, login: login || empId });
        }
        return emps;
    }

    async function fetchClockIn(empId, wh, sM, eM, dateStr) {
        try {
            var dateForURL = dateStr.replace(/-/g, '/');
            var url = 'https://fclm-portal.amazon.com/employee/timeDetails?employeeId=' + empId + '&warehouseId=' + wh + '&timekeepingDate=' + encodeURIComponent(dateForURL);
            var r = await fetch(url, { credentials: 'same-origin' });
            if (!r.ok) { r = await fetch('https://fclm-portal.amazon.com/employee/timeDetails?employeeId=' + empId + '&warehouseId=' + wh, { credentials: 'same-origin' }); if (!r.ok) return null; }
            var html = await r.text();
            var clockIns = [];
            var idx = html.indexOf('OnClock');
            while (idx >= 0) { var chunk = html.substring(idx, idx + 500); if (chunk.indexOf('Paid') >= 0) { var m = chunk.match(/\d{2}\/\d{2}-(\d{2}:\d{2}:\d{2})/); if (m) clockIns.push(m[1]); } idx = html.indexOf('OnClock', idx + 1); }
            if (clockIns.length === 0) { var doc = new DOMParser().parseFromString(html, 'text/html'); var trs = doc.querySelectorAll('tr'); for (var i = 0; i < trs.length; i++) { var rt = trs[i].innerText || ''; if (rt.indexOf('OnClock') >= 0 && rt.indexOf('Paid') >= 0) { var ms = rt.match(/\d{2}\/\d{2}-(\d{2}:\d{2}:\d{2})/g); if (ms) { for (var k = 0; k < ms.length; k++) { var x = ms[k].match(/(\d{2}:\d{2}:\d{2})/); if (x) clockIns.push(x[1]); } } } } }
            if (clockIns.length === 0) { var doc2 = new DOMParser().parseFromString(html, 'text/html'); var trs2 = doc2.querySelectorAll('tr'); for (var ii = 0; ii < trs2.length; ii++) { var rt2 = trs2[ii].innerText || ''; if (rt2.indexOf('Paid') >= 0) { var tms = rt2.match(/(\d{2}:\d{2}:\d{2})/g); if (tms) { for (var tt = 0; tt < tms.length; tt++) clockIns.push(tms[tt]); } } } }
            if (clockIns.length === 0) { var am = html.match(/\d{2}\/\d{2}-(\d{2}:\d{2}:\d{2})/g); if (am) { for (var a = 0; a < am.length; a++) { var ax = am[a].match(/(\d{2}:\d{2}:\d{2})/); if (ax) clockIns.push(ax[1]); } } }
            var buffStart = sM - CLOCKIN_BUFFER; if (buffStart < 0) buffStart += 1440;
            var best = null, bestD = Infinity;
            for (var c = 0; c < clockIns.length; c++) { var cm = t2m(clockIns[c]); if (inWindow(cm, buffStart, eM)) { var dist = Math.abs(cm - sM); if (dist < bestD) { bestD = dist; best = clockIns[c]; } } }
            return best;
        } catch (e) { return null; }
    }

    async function fetchFirstAct(empId, wh, dateStr, clockIn) {
        var cp = parseT(clockIn); var end = addMins(clockIn, 45);
        var st = dateStr + 'T' + String(cp.h).padStart(2, '0') + '%3a' + String(cp.m).padStart(2, '0') + '%3a00-0400';
        var et = dateStr + 'T' + String(end.h).padStart(2, '0') + '%3a' + String(end.m).padStart(2, '0') + '%3a00-0400';
        var url = 'https://fclm-portal.amazon.com/employee/activityDetails?employeeId=' + empId + '&warehouseId=' + wh + '&startTime=' + st + '&endTime=' + et + '&reportFormat=HTML';
        try {
            var r = await fetch(url, { credentials: 'same-origin' }); if (!r.ok) return null;
            var html = await r.text(); var doc = new DOMParser().parseFromString(html, 'text/html');
            var allTimes = [], fullText = doc.body ? (doc.body.innerText || '') : html;
            var p1 = fullText.match(/\d{4}\/\d{2}\/\d{2}\s+(\d{2}:\d{2}:\d{2})/g);
            if (p1) { for (var i = 0; i < p1.length; i++) { var m = p1[i].match(/(\d{2}:\d{2}:\d{2})/); if (m) allTimes.push(m[1]); } }
            if (!allTimes.length) { var cells = doc.querySelectorAll('td'); for (var j = 0; j < cells.length; j++) { var ct = (cells[j].innerText || '').trim(); if (/^\d{2}:\d{2}:\d{2}$/.test(ct)) allTimes.push(ct); } }
            if (!allTimes.length) { var p3 = fullText.match(/\d{2}\/\d{2}-(\d{2}:\d{2}:\d{2})/g); if (p3) { for (var k = 0; k < p3.length; k++) { var m3 = p3[k].match(/(\d{2}:\d{2}:\d{2})/); if (m3) allTimes.push(m3[1]); } } }
            if (!allTimes.length) { var raw = html.match(/(\d{2}:\d{2}:\d{2})/g); if (raw) allTimes = raw; }
            var ciM = t2m(clockIn), bestAct = null, bestDiff = Infinity;
            for (var a = 0; a < allTimes.length; a++) { var tM = t2m(allTimes[a]); var diff = tM - ciM; if (diff < 0) diff += 1440; if (diff > 0 && diff < 60 && diff < bestDiff) { bestDiff = diff; bestAct = allTimes[a]; } }
            return bestAct;
        } catch (e) { return null; }
    }

    async function fetchGap(empId, wh, dateStr, wStart, wEnd) {
        var sp = parseT(wStart + ':00'), ep = parseT(wEnd + ':00');
        var st = dateStr + 'T' + String(sp.h).padStart(2, '0') + '%3a' + String(sp.m).padStart(2, '0') + '%3a00-0400';
        var et = dateStr + 'T' + String(ep.h).padStart(2, '0') + '%3a' + String(ep.m).padStart(2, '0') + '%3a00-0400';
        var url = 'https://fclm-portal.amazon.com/employee/activityDetails?employeeId=' + empId + '&warehouseId=' + wh + '&startTime=' + st + '&endTime=' + et + '&reportFormat=HTML';
        try {
            var r = await fetch(url, { credentials: 'same-origin' }); if (!r.ok) return null;
            var html = await r.text(); var doc = new DOMParser().parseFromString(html, 'text/html');
            var fullText = doc.body ? (doc.body.innerText || '') : html;
            var wsM = t2m(wStart + ':00'), weM = t2m(wEnd + ':00'), stamps = [];
            var p1 = fullText.match(/\d{4}\/\d{2}\/\d{2}\s+(\d{2}:\d{2}:\d{2})/g);
            if (p1) { for (var i = 0; i < p1.length; i++) { var m = p1[i].match(/(\d{2}:\d{2}:\d{2})/); if (m) { var tM = t2m(m[1]); if (inWindow(tM, wsM, weM)) stamps.push({ t: m[1], m: tM }); } } }
            if (!stamps.length) { var p2 = fullText.match(/\d{2}\/\d{2}-(\d{2}:\d{2}:\d{2})/g); if (p2) { for (var j = 0; j < p2.length; j++) { var m2 = p2[j].match(/(\d{2}:\d{2}:\d{2})/); if (m2) { var tM2 = t2m(m2[1]); if (inWindow(tM2, wsM, weM)) stamps.push({ t: m2[1], m: tM2 }); } } } }
            if (!stamps.length) { var raw = fullText.match(/(\d{2}:\d{2}:\d{2})/g); if (raw) { for (var rt = 0; rt < raw.length; rt++) { var rtM = t2m(raw[rt]); if (inWindow(rtM, wsM, weM)) stamps.push({ t: raw[rt], m: rtM }); } } }
            if (stamps.length < 2) return null;
            stamps.sort(function (a, b) { return a.m - b.m; });
            var unique = [stamps[0]]; for (var u = 1; u < stamps.length; u++) { if (stamps[u].m !== stamps[u - 1].m) unique.push(stamps[u]); }
            if (unique.length < 2) return null;
            var maxGap = 0, gs = '', ge = '';
            for (var k = 1; k < unique.length; k++) { var gap = unique[k].m - unique[k - 1].m; if (gap > maxGap) { maxGap = gap; gs = unique[k - 1].t; ge = unique[k].t; } }
            return maxGap > 0 ? { mins: maxGap, start: gs, end: ge } : null;
        } catch (e) { return null; }
    }

    async function fetchLoginFromTime(empId, wh, dateStr) {
        try {
            var dateForURL = dateStr.replace(/-/g, '/');
            var r = await fetch('https://fclm-portal.amazon.com/employee/timeDetails?employeeId=' + empId + '&warehouseId=' + wh + '&timekeepingDate=' + encodeURIComponent(dateForURL), { credentials: 'same-origin' });
            if (!r.ok) { r = await fetch('https://fclm-portal.amazon.com/employee/timeDetails?employeeId=' + empId + '&warehouseId=' + wh, { credentials: 'same-origin' }); if (!r.ok) return null; }
            var html = await r.text();
            var lm = html.match(/employee[Ll]ogin[=:]\s*["']?([a-z][a-z0-9]{4,14})/); if (lm) return lm[1];
            var tm = html.match(/<title[^>]*>([^<]*)<\/title>/i); if (tm) { var tl = tm[1].match(/([a-z][a-z0-9]{4,14})/); if (tl && tl[1] !== 'employee' && tl[1] !== 'details') return tl[1]; }
            return null;
        } catch (e) { return null; }
    }

    function copyVertical(assocs) {
        var lines = []; for (var i = 0; i < assocs.length; i++) lines.push(assocs[i].login + '\t' + assocs[i].diff.toFixed(1));
        var text = lines.join('\n');
        var done = function () { var b = document.getElementById('c2a-copy-btn'); if (b) { b.textContent = '✅ Copied!'; b.style.background = '#4ade80'; b.style.color = '#1a1a2e'; setTimeout(function () { b.textContent = '📋 Copy for Excel'; b.style.background = '#6366f1'; b.style.color = '#fff'; }, 2000); } };
        navigator.clipboard.writeText(text).then(done).catch(function () { var ta = document.createElement('textarea'); ta.value = text; ta.style.position = 'fixed'; ta.style.left = '-9999px'; document.body.appendChild(ta); ta.select(); document.execCommand('copy'); document.body.removeChild(ta); done(); });
    }

    function build() {
        var old = document.getElementById('c2a-panel'); if (old) old.remove();
        var oldB = document.getElementById('c2a-btn'); if (oldB) oldB.remove();
        var curDate = getDateFromURL(), par = getParams(), proc = getProcName(), dateStr = fmtDateInput(curDate), emps = parseEmps();
        var now = new Date(), shift = now.getHours() >= 6 && now.getHours() < 18 ? 'day' : 'night', mode = 'clockin';
        var shouldOpen = sessionStorage.getItem('c2a_panel_open') === 'true';
        var savedShift = sessionStorage.getItem('c2a_shift'), savedMode = sessionStorage.getItem('c2a_mode');
        if (savedShift) shift = savedShift; if (savedMode) mode = savedMode;
        sessionStorage.removeItem('c2a_panel_open'); sessionStorage.removeItem('c2a_shift'); sessionStorage.removeItem('c2a_mode');

        // Mini button
        var btn = document.createElement('div'); btn.id = 'c2a-btn';
        btn.style.cssText = 'position:fixed;top:10px;right:10px;background:#0073bb;color:#fff;padding:10px 18px;border-radius:8px;cursor:pointer;z-index:99999;font-family:Amazon Ember,Arial,sans-serif;font-size:14px;font-weight:bold;box-shadow:0 4px 12px rgba(0,0,0,0.5);display:' + (shouldOpen ? 'none' : 'block') + ';';
        btn.textContent = '⏱️ ' + par.wh; document.body.appendChild(btn);

        // Panel
        var panel = document.createElement('div'); panel.id = 'c2a-panel';
        panel.style.cssText = 'position:fixed;top:10px;right:10px;width:460px;max-height:85vh;overflow-y:auto;background:' + D.bg + ';border:1px solid ' + D.border + ';border-radius:12px;padding:16px;z-index:99999;font-family:Amazon Ember,Arial,sans-serif;font-size:13px;box-shadow:0 8px 32px rgba(0,0,0,0.6);color:' + D.text + ';display:' + (shouldOpen ? 'block' : 'none') + ';';

        // Header
        var hdr = document.createElement('div'); hdr.id = 'c2a-drag-handle';
        hdr.style.cssText = 'display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;border-bottom:2px solid ' + D.accent + ';padding-bottom:8px;cursor:move;';
        hdr.innerHTML = '<div><strong style="font-size:15px;color:' + D.accent + '">⏱️ Clock to First Activity</strong><div style="font-size:12px;margin-top:5px;"><span style="color:#60a5fa;font-weight:bold;">' + par.wh + '</span> | <span style="color:' + D.accent + ';font-weight:700;">' + proc + '</span></div></div><button id="c2a-close" style="background:' + D.accent + ';color:#1a1a2e;border:none;border-radius:4px;padding:4px 10px;cursor:pointer;font-weight:bold;">—</button>';
        panel.appendChild(hdr);

        // Process Tabs
        var procBar = document.createElement('div'); procBar.style.cssText = 'display:flex;gap:6px;margin-bottom:10px;';
        PROC_TABS.forEach(function (pt) {
            var isActive = par.pid === pt.id;
            var tb = document.createElement('button'); tb.textContent = pt.label;
            tb.style.cssText = 'flex:1;padding:10px 4px;border-radius:8px;border:2px solid ' + pt.border + ';cursor:pointer;font-weight:800;font-size:11px;transition:all 0.2s;background:' + (isActive ? pt.bg : pt.bgInactive) + ';color:' + (isActive ? pt.color : pt.textInactive) + ';' + (isActive ? 'box-shadow:0 4px 12px ' + pt.border + '60;transform:scale(1.02);' : 'opacity:0.7;');
            tb.addEventListener('mouseenter', function () { if (!isActive) { this.style.opacity = '1'; this.style.background = pt.bg; this.style.color = pt.color; } });
            tb.addEventListener('mouseleave', function () { if (!isActive) { this.style.opacity = '0.7'; this.style.background = pt.bgInactive; this.style.color = pt.textInactive; } });
            tb.addEventListener('click', function () { if (pt.id === par.pid) return; saveState(shift, mode); var sd = document.getElementById('c2a-date').value.split('-'); window.location.href = buildURLFull(new Date(sd[0], sd[1] - 1, sd[2]), pt.id, par.sH, par.sM, par.eH, par.eM); });
            procBar.appendChild(tb);
        });
        panel.appendChild(procBar);

        // Day/Night
        var tog = document.createElement('div'); tog.style.cssText = 'display:flex;margin-bottom:10px;border-radius:6px;overflow:hidden;border:1px solid ' + D.border + ';';
        tog.innerHTML = '<button id="c2a-day" style="flex:1;padding:8px;border:none;cursor:pointer;font-weight:700;font-size:12px;background:' + (shift === 'day' ? D.accent : D.bgCard) + ';color:' + (shift === 'day' ? '#1a1a2e' : D.textMuted) + ';">☀️ DAY</button><button id="c2a-night" style="flex:1;padding:8px;border:none;cursor:pointer;font-weight:700;font-size:12px;background:' + (shift === 'night' ? '#6366f1' : D.bgCard) + ';color:' + (shift === 'night' ? '#fff' : D.textMuted) + ';">🌙 NIGHT</button>';
        panel.appendChild(tog);

        // Date Nav
        var prev = new Date(curDate); prev.setDate(prev.getDate() - 1);
        var next = new Date(curDate); next.setDate(next.getDate() + 1);
        var today = new Date(); today.setHours(0, 0, 0, 0);
        var isToday = curDate.toDateString() === today.toDateString();
        var dn = document.createElement('div'); dn.style.cssText = 'display:flex;align-items:center;justify-content:center;gap:8px;margin-bottom:8px;padding:8px;background:' + D.bgCard + ';border-radius:8px;border:1px solid ' + D.border + ';';
        dn.innerHTML = '<button id="c2a-prev" style="padding:5px 10px;border-radius:4px;border:1px solid ' + D.accent + ';background:transparent;color:' + D.accent + ';cursor:pointer;font-weight:bold;">◀</button><input type="date" id="c2a-date" value="' + fmtDateInput(curDate) + '" max="' + fmtDateInput(today) + '" style="padding:5px 10px;border-radius:4px;border:1px solid ' + D.border + ';background:' + D.bgSection + ';color:' + D.text + ';font-size:13px;"/><button id="c2a-next" ' + (isToday ? 'disabled' : '') + ' style="padding:5px 10px;border-radius:4px;border:1px solid ' + (isToday ? D.border : D.accent) + ';background:transparent;color:' + (isToday ? D.textMuted : D.accent) + ';cursor:' + (isToday ? 'not-allowed' : 'pointer') + ';font-weight:bold;">▶</button><button id="c2a-today" ' + (isToday ? 'disabled' : '') + ' style="padding:5px 10px;border-radius:4px;border:1px solid ' + (isToday ? D.border : D.gold) + ';background:' + (isToday ? 'transparent' : D.gold) + ';color:' + (isToday ? D.textMuted : '#1a1a2e') + ';cursor:' + (isToday ? 'not-allowed' : 'pointer') + ';font-weight:600;font-size:12px;">Today</button>';
        panel.appendChild(dn);

        // Date Label
        var days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
        var mons = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
        var dl = document.createElement('div'); dl.style.cssText = 'text-align:center;margin-bottom:10px;font-size:12px;color:' + D.textMuted + ';';
        dl.innerHTML = '<strong style="font-size:14px;color:' + D.text + ';">' + days[curDate.getDay()] + ', ' + mons[curDate.getMonth()] + ' ' + curDate.getDate() + ', ' + curDate.getFullYear() + '</strong>' + (isToday ? ' <span style="background:#4ade80;color:#1a1a2e;padding:2px 6px;border-radius:3px;font-size:10px;font-weight:bold;">TODAY</span>' : '');
        panel.appendChild(dl);

        // Time Filter + Results
        var tf = document.createElement('div'); tf.id = 'c2a-tf'; tf.style.cssText = 'margin-bottom:12px;padding:12px;background:' + D.bgCard + ';border-radius:8px;border:1px solid ' + D.border + ';';
        panel.appendChild(tf);
        var res = document.createElement('div'); res.id = 'c2a-res'; panel.appendChild(res);
        document.body.appendChild(panel);

        function renderTF() {
            var s = SHIFTS[shift]; mode = savedMode || s.presets[0].type;
            var presHTML = s.presets.map(function (p) { return '<button class="c2a-pre" data-s="' + p.start + '" data-e="' + p.end + '" data-t="' + p.type + '" style="padding:4px 10px;border-radius:4px;border:1px solid ' + (p.type === 'gap' ? '#ef4444' : D.border) + ';background:' + (p.type === 'gap' ? 'rgba(239,68,68,0.1)' : D.bgSection) + ';color:' + (p.type === 'gap' ? '#f87171' : D.text) + ';cursor:pointer;font-size:10px;font-weight:600;">' + p.label + '</button>'; }).join('');
            var dS = String(par.sH).padStart(2, '0') + ':' + String(par.sM).padStart(2, '0');
            var dE = String(par.eH).padStart(2, '0') + ':' + String(par.eM).padStart(2, '0');
            tf.innerHTML = '<div style="display:flex;justify-content:space-between;margin-bottom:8px;"><span style="font-size:11px;text-transform:uppercase;letter-spacing:1px;color:' + D.accent + ';font-weight:700;">🔍 Search Window</span><span style="font-size:10px;color:' + D.textMuted + ';">' + s.label + '</span></div><div style="display:flex;align-items:center;gap:8px;margin-bottom:8px;"><span style="font-size:11px;color:' + D.textMuted + ';">From</span><input type="time" id="c2a-ws" value="' + dS + '" style="padding:4px 8px;border-radius:4px;border:1px solid ' + D.border + ';background:' + D.bgSection + ';color:' + D.text + ';font-size:13px;font-weight:600;"/><span style="font-size:11px;color:' + D.textMuted + ';">To</span><input type="time" id="c2a-we" value="' + dE + '" style="padding:4px 8px;border-radius:4px;border:1px solid ' + D.border + ';background:' + D.bgSection + ';color:' + D.text + ';font-size:13px;font-weight:600;"/><button id="c2a-run" style="padding:6px 14px;border-radius:4px;border:none;background:' + D.accent + ';color:#1a1a2e;cursor:pointer;font-weight:700;font-size:12px;">▶ Run</button></div><div style="display:flex;gap:6px;flex-wrap:wrap;">' + presHTML + '<button id="c2a-reload" style="padding:4px 10px;border-radius:4px;border:1px solid #22d3ee;background:rgba(34,211,238,0.1);color:#22d3ee;cursor:pointer;font-size:10px;font-weight:600;">🔄 Reload</button></div><div style="margin-top:8px;font-size:10px;padding:4px 8px;border-radius:4px;background:' + (mode === 'gap' ? 'rgba(239,68,68,0.1)' : 'rgba(96,165,250,0.1)') + ';color:' + (mode === 'gap' ? '#f87171' : '#60a5fa') + ';display:inline-block;">' + (mode === 'gap' ? '🕳️ Mode: 2nd Break Gap (≥25m only)' : '⏱️ Mode: Clock-In → First Scan') + '</div>';
            tf.querySelectorAll('.c2a-pre').forEach(function (b) { b.addEventListener('click', function () { saveState(shift, this.getAttribute('data-t')); var sd = document.getElementById('c2a-date').value.split('-'); window.location.href = buildURLWithTime(new Date(sd[0], sd[1] - 1, sd[2]), this.getAttribute('data-s'), this.getAttribute('data-e')); }); });
            document.getElementById('c2a-reload').addEventListener('click', function () { saveState(shift, mode); var sd = document.getElementById('c2a-date').value.split('-'); window.location.href = buildURLWithTime(new Date(sd[0], sd[1] - 1, sd[2]), document.getElementById('c2a-ws').value, document.getElementById('c2a-we').value); });
            document.getElementById('c2a-run').addEventListener('click', run);
        }

        // Events
        document.getElementById('c2a-close').addEventListener('click', function () { panel.style.display = 'none'; btn.style.display = 'block'; });
        btn.addEventListener('click', function () { btn.style.display = 'none'; panel.style.display = 'block'; });
        document.getElementById('c2a-prev').addEventListener('click', function () { goToDate(prev, shift, mode); });
        document.getElementById('c2a-next').addEventListener('click', function () { if (!isToday) goToDate(next, shift, mode); });
        document.getElementById('c2a-today').addEventListener('click', function () { if (!isToday) goToDate(today, shift, mode); });
        document.getElementById('c2a-date').addEventListener('change', function () { saveState(shift, mode); var p = this.value.split('-'); window.location.href = buildURL(new Date(p[0], p[1] - 1, p[2])); });
        makeDrag(panel, document.getElementById('c2a-drag-handle'));
        document.getElementById('c2a-day').addEventListener('click', function () { shift = 'day'; this.style.background = D.accent; this.style.color = '#1a1a2e'; document.getElementById('c2a-night').style.background = D.bgCard; document.getElementById('c2a-night').style.color = D.textMuted; savedMode = null; renderTF(); });
        document.getElementById('c2a-night').addEventListener('click', function () { shift = 'night'; this.style.background = '#6366f1'; this.style.color = '#fff'; document.getElementById('c2a-day').style.background = D.bgCard; document.getElementById('c2a-day').style.color = D.textMuted; savedMode = null; renderTF(); });

        // RUN
        async function run() {
            var ws = document.getElementById('c2a-ws').value, we = document.getElementById('c2a-we').value;
            var wsM = t2m(ws + ':00'), weM = t2m(we + ':00'), isGap = mode === 'gap';
            res.innerHTML = '<div style="text-align:center;padding:20px;color:' + D.textMuted + ';"><div style="font-size:24px;margin-bottom:8px;">' + (isGap ? '🕳️' : '⏳') + '</div><div style="font-weight:600;color:' + D.text + ';">' + (isGap ? 'Scanning gaps ≥25min' : 'Looking for clock-ins') + ' ' + ws + ' – ' + we + '</div><div style="font-size:12px;margin-top:4px;">' + emps.length + ' AAs (' + proc + ') | ' + dateStr + '</div><div id="c2a-prog" style="margin-top:8px;font-size:12px;">0 / ' + emps.length + '</div><div style="margin-top:10px;background:' + D.bgSection + ';border-radius:4px;height:6px;overflow:hidden;"><div id="c2a-bar" style="width:0%;height:100%;background:' + D.accent + ';transition:width 0.3s;"></div></div></div>';
            if (!emps.length) { res.innerHTML = '<p style="color:#f87171;font-weight:bold;padding:12px;">⚠️ No employees found on page.</p>'; return; }

            var assocs = [], notInPath = 0, done = 0;
            for (var i = 0; i < emps.length; i += 2) {
                await Promise.all(emps.slice(i, i + 2).map(async function (emp) {
                    var login = emp.login;
                    if (/^\d+$/.test(login)) { var fl = await fetchLoginFromTime(emp.id, par.wh, dateStr); if (fl) login = fl; }
                    try {
                        if (isGap) {
                            var g = await fetchGap(emp.id, par.wh, dateStr, ws, we);
                            if (g && g.mins >= GAP_MIN_THRESHOLD) assocs.push({ login: login, clockIn: g.start, firstAct: g.end, diff: g.mins }); else notInPath++;
                        } else {
                            var ci = await fetchClockIn(emp.id, par.wh, wsM, weM, dateStr);
                            if (ci) { var fa = await fetchFirstAct(emp.id, par.wh, dateStr, ci); if (fa) { var d = t2m(fa) - t2m(ci); if (d < 0) d += 1440; if (d > 0 && d < 120) assocs.push({ login: login, clockIn: ci, firstAct: fa, diff: d }); else notInPath++; } else notInPath++; } else notInPath++;
                        }
                    } catch (e) { notInPath++; }
                    done++;
                    var pe = document.getElementById('c2a-prog'); if (pe) pe.textContent = done + ' / ' + emps.length;
                    var pb = document.getElementById('c2a-bar'); if (pb) pb.style.width = (done / emps.length * 100) + '%';
                }));
                await sleep(150);
            }

            if (!assocs.length) {
                res.innerHTML = '<div style="padding:15px;background:' + D.bgCard + ';border:1px solid ' + D.border + ';border-radius:6px;text-align:center;"><p style="color:' + (isGap ? '#4ade80' : '#facc15') + ';font-weight:bold;font-size:14px;">' + (isGap ? '✅ No break abuse detected!' : '⚠️ No in-path data in ' + ws + ' – ' + we) + '</p><p style="font-size:11px;color:' + D.textMuted + ';margin-top:4px;">' + (isGap ? 'All breaks under 25 min' : notInPath + ' AAs not in path. Try adjusting window.') + '</p></div>';
                return;
            }

            var sorted = assocs.slice().sort(function (a, b) { return a.diff - b.diff; });
            var top3 = sorted.slice(0, Math.min(3, sorted.length));
            var t3set = new Set(top3.map(function (a) { return a.login; }));
            var rest = sorted.filter(function (a) { return !t3set.has(a.login); });
            var bot4 = rest.slice(-Math.min(4, rest.length));
            var b4set = new Set(bot4.map(function (a) { return a.login; }));
            var mid = sorted.filter(function (a) { return !t3set.has(a.login) && !b4set.has(a.login); });
            var avg = assocs.reduce(function (s, a) { return s + a.diff; }, 0) / assocs.length;
            var avgT = getThreshold(avg, isGap);

            var h = '<div style="margin-bottom:14px;padding:18px;background:linear-gradient(135deg,' + D.bgCard + ',' + D.bgSection + ');border-radius:12px;text-align:center;border:1px solid ' + D.border + ';"><div style="font-size:11px;text-transform:uppercase;letter-spacing:1.5px;color:' + D.textMuted + ';margin-bottom:6px;font-weight:600;">' + (isGap ? '🕳️ Avg Break Duration' : '⚡ Fast Start Average') + '</div><div style="font-size:40px;font-weight:900;color:' + avgT.color + ';">' + avg.toFixed(1) + ' <span style="font-size:16px;">min</span></div><div style="margin-top:6px;font-size:12px;color:' + D.textMuted + ';">' + assocs.length + ' AAs ' + avgT.icon + (avgT.tag ? ' ' + avgT.tag : '') + '</div><div style="margin-top:4px;font-size:11px;color:' + D.textMuted + ';">' + proc + ' | ' + ws + ' – ' + we + '</div></div>';
            h += '<div style="text-align:center;margin-bottom:12px;"><button id="c2a-copy-btn" style="padding:8px 20px;border-radius:6px;border:none;background:#6366f1;color:#fff;cursor:pointer;font-weight:700;font-size:12px;">📋 Copy for Excel</button></div>';

            function row(a, sec) {
                var t = getThreshold(a.diff, isGap), isT = sec === 'top', isB = sec === 'bottom';
                var bg = t.bg, bl = 'border-left:4px solid ' + t.color + ';';
                if (isT) { bg = HIGHLIGHT.top.bg; bl = 'border-left:5px solid ' + HIGHLIGHT.top.border + ';'; }
                if (isB) { bg = HIGHLIGHT.bottom.bg; bl = 'border-left:5px solid ' + HIGHLIGHT.bottom.border + ';'; }
                var dc = isT ? HIGHLIGHT.top.color : (isB ? HIGHLIGHT.bottom.color : t.color);
                var lc = isT ? HIGHLIGHT.top.color : (isB ? HIGHLIGHT.bottom.color : D.text);
                return '<tr style="background:' + bg + ';' + bl + '"><td style="padding:7px 10px;font-weight:700;font-size:12px;color:' + lc + ';">' + a.login + '</td><td style="padding:7px 8px;text-align:center;font-weight:900;font-size:13px;color:' + dc + ';">' + a.diff.toFixed(1) + '</td><td style="padding:7px 8px;text-align:center;font-size:11px;color:' + D.textMuted + ';">' + a.clockIn.substring(0, 5) + '</td><td style="padding:7px 8px;text-align:center;font-size:11px;color:' + D.textMuted + ';">' + a.firstAct.substring(0, 5) + '</td></tr>';
            }

            var col3 = isGap ? 'Last Scan' : 'Clock In', col4 = isGap ? 'Next Scan' : '1st Scan';
            h += '<table style="width:100%;border-collapse:collapse;font-size:12px;background:' + D.bgCard + ';border-radius:8px;overflow:hidden;"><tr style="background:' + D.bgSection + ';"><th style="padding:6px 10px;text-align:left;font-size:10px;color:' + D.textMuted + ';">Login</th><th style="padding:6px 8px;text-align:center;font-size:10px;color:' + D.textMuted + ';">Δ min</th><th style="padding:6px 8px;text-align:center;font-size:10px;color:' + D.textMuted + ';">' + col3 + '</th><th style="padding:6px 8px;text-align:center;font-size:10px;color:' + D.textMuted + ';">' + col4 + '</th></tr>';
            h += '<tr><td colspan="4" style="padding:10px;font-weight:700;text-align:center;color:' + HIGHLIGHT.top.color + ';background:' + HIGHLIGHT.top.bg + ';border-bottom:2px solid ' + HIGHLIGHT.top.border + ';">' + (isGap ? '⭐ TOP 3 SHORTEST' : '⭐ TOP 3 FASTEST') + '</td></tr>';
            top3.forEach(function (a) { h += row(a, 'top'); });
            if (mid.length) { h += '<tr><td colspan="4" style="padding:10px;font-weight:600;text-align:center;color:' + D.textMuted + ';border-top:1px solid ' + D.border + ';border-bottom:1px solid ' + D.border + ';">📋 ALL OTHERS (' + mid.length + ')</td></tr>'; mid.forEach(function (a) { h += row(a, 'middle'); }); }
            if (bot4.length) { h += '<tr><td colspan="4" style="padding:10px;font-weight:700;text-align:center;color:' + HIGHLIGHT.bottom.color + ';background:' + HIGHLIGHT.bottom.bg + ';border-top:2px solid ' + HIGHLIGHT.bottom.border + ';">🐢 BOTTOM ' + bot4.length + ' SLOWEST</td></tr>'; bot4.forEach(function (a) { h += row(a, 'bottom'); }); }
            h += '</table>';
            h += '<div style="margin-top:10px;padding:8px;text-align:center;font-size:11px;color:' + D.textMuted + ';background:' + D.bgSection + ';border-radius:6px;">📊 In Path: ' + assocs.length + ' | 🚫 Not in Path: ' + notInPath + '</div>';
            res.innerHTML = h;
            var cb = document.getElementById('c2a-copy-btn'); if (cb) cb.addEventListener('click', function () { copyVertical(sorted); });
        }

        renderTF();
    }

    if (document.readyState === 'complete') { setTimeout(build, 2000); }
    else { window.addEventListener('load', function () { setTimeout(build, 2000); }); }
})();


