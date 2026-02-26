// ==UserScript==
// @name         Bounce Breaker
// @version      1.0
// @description  Auto-detect transfers via API when entering a case — no need to click Case Details
// @updateURL    https://raw.githubusercontent.com/vregla/Bounce-Breaker/main/Bounce-Breaker.user.js
// @downloadURL  https://raw.githubusercontent.com/vregla/Bounce-Breaker/main/Bounce-Breaker.user.js
// @match        https://optimus-internal-eu.amazon.com/wims/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const SOP_LINK = 'https://policy.a2z.com/docs/694472/publication';
    const PS_BLURB = 'This case has been transferred between queues more than 3 times without resolution. Transferring to Problem Solver as last resort for proper handling and resolution.';

    // ← CHANGED: renamed FL, added R
    const problemSolverLabels = ['DM FL Problem Solver', 'DM AFP Problem Solver', 'Rail Problem Solver', 'DM R Problem Solver'];

    const targetEmails = [
        'eu-roc-ob-unloading-delays@amazon.com',
        'eu-roc-ob-rec-monitoring@amazon.com',
        'fleet-preventive-maintenance@amazon.de',
        'eu-roc-ob-add-truck@amazon.com',
        'eu-roc-ob-sourcing@amazon.com',
        'eu-roc-loss-prevention@amazon.de',
        'eu-roc-ob-support@amazon.com',
        'eu-roc-dm-frontline-problemsolver@amazon.com',
        'eu-roc-safety@amazon.com',
        'eu-roc-ob-our-trailer@amazon.com',
        'eu-roc-dm-afp-problemsolver@amazon.com',
        'eu-fleet-support@amazon.com',
        'eu-roc-ob-scheduling@amazon.com',
        'eu-roc-ob-equipement@amazon.com',
        'roc-tio-missing-trailers@amazon.com',
        'roc-intermodal@amazon.com',
        'eu-roc-ob-late-truck@amazon.com',
        'roc-intermodal-tours@amazon.com',
        'eu-roc-rail-problemsolver@amazon.com',
        'eu-roc-dm-recovery-problemsolver@amazon.com'  // ← NEW
    ];

    const emailLabels = {
        'eu-roc-ob-unloading-delays@amazon.com': 'Unloading Delays',
        'eu-roc-ob-rec-monitoring@amazon.com': 'Receiving Monitoring',
        'fleet-preventive-maintenance@amazon.de': 'Fleet Maintenance',
        'eu-roc-ob-add-truck@amazon.com': 'Add Truck',
        'eu-roc-ob-sourcing@amazon.com': 'Sourcing',
        'eu-roc-loss-prevention@amazon.de': 'Loss Prevention',
        'eu-roc-ob-support@amazon.com': 'OB Support',
        'eu-roc-dm-frontline-problemsolver@amazon.com': 'DM FL Problem Solver',       // ← CHANGED
        'eu-roc-safety@amazon.com': 'Safety',
        'eu-roc-ob-our-trailer@amazon.com': 'OUR Trailer',
        'eu-roc-dm-afp-problemsolver@amazon.com': 'DM AFP Problem Solver',
        'eu-fleet-support@amazon.com': 'Fleet Support',
        'eu-roc-ob-scheduling@amazon.com': 'OB Scheduling',
        'eu-roc-ob-equipement@amazon.com': 'OB Equipment',
        'roc-tio-missing-trailers@amazon.com': 'TIO Missing Trailers',
        'roc-intermodal@amazon.com': 'Intermodal',
        'eu-roc-ob-late-truck@amazon.com': 'OB Late Truck',
        'roc-intermodal-tours@amazon.com': 'Intermodal Tours',
        'eu-roc-rail-problemsolver@amazon.com': 'Rail Problem Solver',
        'eu-roc-dm-recovery-problemsolver@amazon.com': 'DM R Problem Solver'          // ← NEW
    };

    let isMinimized = false;
    let lastScannedTaskId = null;
    let lastTransferData = null;

    // ════════════════════════════════════════════════════
    //  API FUNCTIONS
    // ════════════════════════════════════════════════════
    function getTaskIdFromUrl() {
        const match = window.location.href.match(/\/tasks\/([a-f0-9-]{36})/i);
        return match ? match[1] : null;
    }

    async function fetchCaseId(taskId) {
        try {
            const response = await fetch('/wims/task/' + taskId);
            if (!response.ok) throw new Error('HTTP ' + response.status);
            const data = await response.json();
            const task = data.task || data;
            return task.context?.entities?.case?.id || null;
        } catch (e) {
            console.error('Transfer Route: Failed to fetch case ID', e);
            return null;
        }
    }

    async function fetchMessages(caseId) {
        try {
            const response = await fetch('/wims/case/' + caseId + '/messages');
            if (!response.ok) throw new Error('HTTP ' + response.status);
            const data = await response.json();
            return data.correspondences || [];
        } catch (e) {
            console.error('Transfer Route: Failed to fetch messages', e);
            return [];
        }
    }

    function countTransfers(correspondences) {
        const sorted = [...correspondences].sort((a, b) => a.createdAt - b.createdAt);
        const relevant = sorted.filter(c => targetEmails.includes(c.fromAddress));

        const flow = [];
        relevant.forEach(c => {
            if (flow.length === 0 || flow[flow.length - 1].email !== c.fromAddress) {
                flow.push({
                    email: c.fromAddress,
                    label: emailLabels[c.fromAddress] || c.fromAddress
                });
            }
        });

        return {
            count: Math.max(0, flow.length - 1),
            flow: flow
        };
    }

    // ════════════════════════════════════════════════════
    //  UI CONTAINER
    // ════════════════════════════════════════════════════
    const container = document.createElement('div');
    container.style.cssText = `
        position: fixed;
        bottom: 55px;
        right: 10px;
        z-index: 10000;
        width: 280px;
        font-family: Arial, sans-serif;
        user-select: none;
        transition: box-shadow 0.2s;
        box-shadow: 0 2px 8px rgba(0,0,0,0.3);
        border-radius: 8px;
        overflow: hidden;
        background: #232f3e;
    `;

    const headerBar = document.createElement('div');
    headerBar.style.cssText = `
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 8px 12px;
        background: #232f3e;
        color: white;
        cursor: move;
        border-bottom: 1px solid rgba(255,255,255,0.1);
    `;

    const headerTitle = document.createElement('span');
    headerTitle.textContent = '📋 Transfer Route';
    headerTitle.style.cssText = 'font-weight: bold; font-size: 13px; pointer-events: none;';

    const headerBadge = document.createElement('span');
    headerBadge.id = 'tc-header-badge';
    headerBadge.style.cssText = `
        padding: 2px 8px;
        border-radius: 10px;
        font-size: 11px;
        font-weight: bold;
        color: white;
        background: #555;
        margin-left: 6px;
        pointer-events: none;
        display: none;
    `;

    const headerLeft = document.createElement('div');
    headerLeft.style.cssText = 'display: flex; align-items: center;';
    headerLeft.appendChild(headerTitle);
    headerLeft.appendChild(headerBadge);

    const headerButtons = document.createElement('div');
    headerButtons.style.cssText = 'display: flex; gap: 4px;';

    const minimizeBtn = document.createElement('button');
    minimizeBtn.textContent = '—';
    minimizeBtn.title = 'Minimize';
    minimizeBtn.style.cssText = `
        background: rgba(255,255,255,0.15);
        color: white;
        border: none;
        border-radius: 4px;
        width: 24px;
        height: 24px;
        font-size: 14px;
        cursor: pointer;
        display: flex;
        align-items: center;
        justify-content: center;
        line-height: 1;
    `;

    headerButtons.appendChild(minimizeBtn);
    headerBar.appendChild(headerLeft);
    headerBar.appendChild(headerButtons);

    const bodyDiv = document.createElement('div');
    bodyDiv.style.cssText = 'background: #2d3a4a; padding: 8px;';

    const statusDiv = document.createElement('div');
    statusDiv.style.cssText = `
        color: #aaa;
        font-size: 12px;
        padding: 6px;
        text-align: center;
    `;
    statusDiv.textContent = 'Loading...';

    const rescanButton = document.createElement('button');
    rescanButton.textContent = '🔄 Rescan';
    rescanButton.style.cssText = `
        background: #0073bb;
        color: white;
        padding: 7px 16px;
        border: none;
        border-radius: 5px;
        font-weight: bold;
        cursor: pointer;
        display: block;
        width: 100%;
        font-size: 12px;
        margin-top: 6px;
    `;

    const resultDiv = document.createElement('div');
    resultDiv.style.cssText = `
        background: white;
        color: #333;
        padding: 12px;
        border-radius: 5px;
        margin-top: 6px;
        display: none;
        max-height: 400px;
        overflow-y: auto;
        font-size: 12px;
    `;

    bodyDiv.appendChild(statusDiv);
    bodyDiv.appendChild(resultDiv);
    bodyDiv.appendChild(rescanButton);

    container.appendChild(headerBar);
    container.appendChild(bodyDiv);

    minimizeBtn.addEventListener('click', (e) => {
        e.stopPropagation();
        isMinimized = !isMinimized;
        if (isMinimized) {
            bodyDiv.style.display = 'none';
            minimizeBtn.textContent = '□';
            minimizeBtn.title = 'Expand';
            container.style.width = 'auto';
        } else {
            bodyDiv.style.display = 'block';
            minimizeBtn.textContent = '—';
            minimizeBtn.title = 'Minimize';
            container.style.width = '280px';
        }
    });

    (function makeDraggable(el, handle) {
        let isDragging = false, startX, startY, startLeft, startTop, hasMoved;

        handle.addEventListener('mousedown', (e) => {
            if (e.target.tagName === 'BUTTON') return;
            isDragging = true;
            hasMoved = false;
            const rect = el.getBoundingClientRect();
            startX = e.clientX;
            startY = e.clientY;
            startLeft = rect.left;
            startTop = rect.top;
            el.style.left = rect.left + 'px';
            el.style.top = rect.top + 'px';
            el.style.right = 'auto';
            el.style.bottom = 'auto';
            el.style.transition = 'none';
            el.style.boxShadow = '0 6px 20px rgba(0,0,0,0.4)';
            e.preventDefault();
        });

        document.addEventListener('mousemove', (e) => {
            if (!isDragging) return;
            const dx = e.clientX - startX;
            const dy = e.clientY - startY;
            if (Math.abs(dx) > 3 || Math.abs(dy) > 3) hasMoved = true;
            if (!hasMoved) return;
            const maxX = window.innerWidth - el.offsetWidth;
            const maxY = window.innerHeight - el.offsetHeight;
            el.style.left = Math.max(0, Math.min(maxX, startLeft + dx)) + 'px';
            el.style.top = Math.max(0, Math.min(maxY, startTop + dy)) + 'px';
        });

        document.addEventListener('mouseup', () => {
            if (isDragging && hasMoved) {
                el.addEventListener('click', (e) => {
                    e.stopPropagation();
                    e.preventDefault();
                }, { capture: true, once: true });
            }
            isDragging = false;
            el.style.transition = 'box-shadow 0.2s';
            el.style.boxShadow = '0 2px 8px rgba(0,0,0,0.3)';
        });
    })(container, headerBar);

    // ════════════════════════════════════════════════════
    //  QUEUE HISTORY PILLS
    // ════════════════════════════════════════════════════
    function buildQueueHistoryHTML(flow) {
        if (flow.length === 0) return '';

        const currentQueue = flow[flow.length - 1].label;

        const visits = {};
        flow.forEach(item => {
            visits[item.label] = (visits[item.label] || 0) + 1;
        });

        const pills = Object.entries(visits)
            .sort((a, b) => b[1] - a[1])
            .map(([queue, count]) => {
                const isCurrent = (queue === currentQueue);
                if (isCurrent) {
                    return `<span style="display:inline-block;padding:3px 8px;margin:2px;border-radius:10px;font-size:11px;font-weight:bold;background:#dbeafe;border:1px solid #3b82f6;">📍 ${queue} (CURRENT — ${count}×)</span>`;
                } else {
                    return `<span style="display:inline-block;padding:3px 8px;margin:2px;border-radius:10px;font-size:11px;background:#fef2f2;border:1px solid #fca5a5;">⛔ ${queue} — ${count}× </span>`;
                }
            }).join('');

        return `
            <div style="margin: 10px 0; padding: 10px; background: #f8f9fa; border-radius: 6px; border: 1px solid #dee2e6;">
                <div style="font-weight: bold; font-size: 12px; color: #333; margin-bottom: 6px;">📊 Queue Visit History — avoid re-transferring to these queues:</div>
                <div style="display: flex; flex-wrap: wrap; gap: 2px;">${pills}</div>
            </div>
        `;
    }

    // ════════════════════════════════════════════════════
    //  TIERED WARNING MODAL
    // ════════════════════════════════════════════════════
    function showTransferWarning(transferCount, flow) {
        const overlay = document.createElement('div');
        overlay.style.cssText = `
            position: fixed;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0, 0, 0, 0.7);
            z-index: 20000;
            display: flex;
            justify-content: center;
            align-items: center;
        `;

        const modal = document.createElement('div');
        modal.style.cssText = `
            background: white;
            padding: 15px 30px 30px;
            border-radius: 12px;
            box-shadow: 0 5px 20px rgba(0,0,0,0.3);
            max-width: 560px;
            width: 90%;
            animation: tcSlideIn 0.3s ease;
            max-height: 85vh;
            overflow-y: auto;
        `;

        const currentQueue = flow.length > 0 ? flow[flow.length - 1].label : 'Unknown';
        const isInProblemSolver = problemSolverLabels.includes(currentQueue);

        let icon, titleText, titleColor, actionHTML;

        if (transferCount > 5) {
            icon = '🚨';
            titleText = 'CRITICAL — Escalate Now';
            titleColor = '#e53e3e';
            actionHTML = `
                <div style="padding: 15px; background: #fde8e8; border-left: 4px solid #e53e3e; border-radius: 6px; margin-top: 12px;">
                    <div style="font-weight: bold; color: #e53e3e; margin-bottom: 8px; font-size: 15px;">
                        🚨 Do NOT transfer — Escalate to Leadership
                    </div>
                    <ul style="margin: 8px 0 0 18px; padding: 0; font-size: 13px; line-height: 1.8; color: #333;">
                        <li>Click <strong>"Escalate"</strong> in WIMS.</li>
                        <li>Include why it's still not resolved.</li>
                    </ul>
                        <div style="margin-top: 10px; padding: 8px; background: #f8d7da; border-radius: 4px; font-size: 11px; color: #721c24; text-align: center;">
                            ⚠️ Wrong transfers are audited by iTrace and <strong>may impact your performance</strong>.
                        </div>
                </div>
            `;
        } else if (transferCount > 3) {
            icon = '⚠️';
            titleColor = '#d97706';

            if (isInProblemSolver) {
                titleText = 'HIGH — Own the Case';
                actionHTML = `
                    <div style="padding: 15px; background: #fffbeb; border-left: 4px solid #f59e0b; border-radius: 6px; margin-top: 12px;">
                        <div style="font-weight: bold; color: #d97706; margin-bottom: 8px; font-size: 15px;">
                            ⚠️ Do NOT transfer — Resolve it here
                        </div>
                        <ul style="margin: 8px 0 0 18px; padding: 0; font-size: 13px; line-height: 1.8; color: #333;">
                            <li>Read the full case history.</li>
                            <li>Check the <a href="${SOP_LINK}" target="_blank" style="color: #2563eb; font-weight: bold; text-decoration: underline;">Transfer Guidelines SOP</a>.</li>
                            <li><strong>You're a Problem Solver — trust your judgment.</strong> Take ownership and resolve.</li>
                            <li>If you need guidance, reaching out to leadership is always an option.</li>
                        </ul>
                        <div style="margin-top: 10px; padding: 8px; background: #f8d7da; border-radius: 4px; font-size: 11px; color: #721c24; text-align: center;">
                            ⚠️ Wrong transfers are audited by iTrace and <strong>may impact your performance</strong>.
                        </div>
                    </div>
                `;
            } else {
                // ← CHANGED: determine which PS based on ULD or not
                titleText = 'HIGH — Verify Before Transferring';
                const isInULD = currentQueue === 'Unloading Delays';
                const psLabel = isInULD
                    ? '<strong style="color:#2563eb;">DM <span style="background:#2563eb;color:white;padding:1px 5px;border-radius:3px;">FL</span> Problem Solver</strong>'
                    : '<strong style="color:#7c3aed;">DM <span style="background:#7c3aed;color:white;padding:1px 5px;border-radius:3px;">R</span> Problem Solver</strong>';

                actionHTML = `
                    <div style="padding: 15px; background: #fffbeb; border-left: 4px solid #f59e0b; border-radius: 6px; margin-top: 12px;">
                        <div style="font-weight: bold; color: #d97706; margin-bottom: 8px; font-size: 15px;">
                            ⚠️ Check SOP — Don't guess the next queue
                        </div>
                        <ul style="margin: 8px 0 0 18px; padding: 0; font-size: 13px; line-height: 1.8; color: #333;">
                            <li>Check the <a href="${SOP_LINK}" target="_blank" style="color: #2563eb; font-weight: bold; text-decoration: underline;">Transfer Guidelines SOP</a> → identify the correct owner.</li>
                            <li><strong>Is this the correct queue?</strong> → Try to resolve it.</li>
                            <li><strong>Another queue is clearly the owner?</strong> → Transfer with a clear problem statement: what the issue is and why it belongs to that queue.</li>
                            <li><strong>Not sure?</strong> → Transfer to ${psLabel}. Don't guess.</li>
                        </ul>
                        <div style="margin-top: 10px; padding: 8px; background: #f8d7da; border-radius: 4px; font-size: 11px; color: #721c24; text-align: center;">
                            ⚠️ Wrong transfers are audited by iTrace and <strong>may impact your performance</strong>.
                        </div>
                    </div>
                `;
            }
        } else if (transferCount > 2) {
            icon = 'ℹ️';
            titleText = 'MODERATE — Review Before Acting';
            titleColor = '#2563eb';
            actionHTML = `
                <div style="padding: 15px; background: #eff6ff; border-left: 4px solid #3b82f6; border-radius: 6px; margin-top: 12px;">
                    <div style="font-weight: bold; color: #2563eb; margin-bottom: 8px; font-size: 15px;">
                        ℹ️ Read carefully before acting
                    </div>
                    <ul style="margin: 8px 0 0 18px; padding: 0; font-size: 13px; line-height: 1.8; color: #333;">
                        <li>Read the full case — don't skim.</li>
                        <li>Check the <a href="${SOP_LINK}" target="_blank" style="color: #2563eb; font-weight: bold; text-decoration: underline;">Transfer Guidelines SOP</a> before replying.</li>
                        <li>Make your next action <strong>precise</strong>.</li>
                    </ul>
                        <div style="margin-top: 10px; padding: 8px; background: #f8d7da; border-radius: 4px; font-size: 11px; color: #721c24; text-align: center;">
                            ⚠️ Wrong transfers are audited by iTrace and <strong>may impact your performance</strong>.
                        </div>
                </div>
            `;
        }

        modal.innerHTML = `
            <div style="font-size: 22px; font-weight: bold; color: ${titleColor}; margin-bottom: 8px; text-align: center;">
                ${titleText}
            </div>
            <div style="font-size: 15px; color: #333; margin-bottom: 5px; text-align: center;">
                This case has been transferred <strong>${transferCount} times</strong>.
            </div>
            <div style="font-size: 13px; color: #555; margin-bottom: 12px; text-align: center;">
                📍 Currently in: <strong style="color: #0073bb;">${currentQueue}</strong>
            </div>
            ${actionHTML}
            <div style="margin: 10px 0; padding: 8px; background: #f8f9fa; border-radius: 6px; border: 1px solid #dee2e6; font-size: 12px; color: #555; text-align: center;">
                📊 Full transfer history available in the <strong>Transfer Route panel</strong> (bottom-right corner).
            </div>
            <button class="close-warning-btn" style="
                display: block; width: 100%; margin-top: 18px;
                background: ${titleColor}; color: white;
                padding: 12px 30px; border: none; border-radius: 6px;
                font-weight: bold; cursor: pointer; font-size: 15px;
                box-shadow: 0 2px 5px rgba(0,0,0,0.2);
            ">Got It</button>
        `;

        overlay.appendChild(modal);
        document.body.appendChild(overlay);

        if (!document.getElementById('tc-anim-style')) {
            const style = document.createElement('style');
            style.id = 'tc-anim-style';
            style.textContent = `
                @keyframes tcSlideIn {
                    from { transform: translateY(-50px); opacity: 0; }
                    to { transform: translateY(0); opacity: 1; }
                }
            `;
            document.head.appendChild(style);
        }

        modal.querySelector('.close-warning-btn').addEventListener('click', () => overlay.remove());

        const blurbEl = modal.querySelector('.ps-blurb-copyable');
        if (blurbEl) {
            blurbEl.addEventListener('click', () => {
                navigator.clipboard.writeText(PS_BLURB).then(() => {
                    blurbEl.style.background = '#d4edda';
                    blurbEl.textContent = '✅ Copied to clipboard!';
                    setTimeout(() => {
                        blurbEl.style.background = '#f7f7f7';
                        blurbEl.textContent = PS_BLURB;
                    }, 2000);
                });
            });
        }
    }

    // ════════════════════════════════════════════════════
    //  UPDATE HEADER BADGE
    // ════════════════════════════════════════════════════
    function updateHeaderBadge(transferCount) {
        headerBadge.style.display = 'inline-block';

        if (transferCount > 5) {
            headerBadge.textContent = `🚨 ${transferCount}T`;
            headerBadge.style.background = '#e53e3e';
        } else if (transferCount > 3) {
            headerBadge.textContent = `⚠️ ${transferCount}T`;
            headerBadge.style.background = '#d97706';
        } else if (transferCount > 2) {
            headerBadge.textContent = `ℹ️ ${transferCount}T`;
            headerBadge.style.background = '#3b82f6';
        } else if (transferCount >= 1) {
            headerBadge.textContent = `${transferCount}T`;
            headerBadge.style.background = '#0073bb';
        } else {
            headerBadge.textContent = '0T';
            headerBadge.style.background = '#28a745';
        }
    }

    // ════════════════════════════════════════════════════
    //  DISPLAY RESULTS IN PANEL
    // ════════════════════════════════════════════════════
    function displayResults(transferCount, flow) {
        lastTransferData = { transferCount, flow };
        updateHeaderBadge(transferCount);

        let transferBg = '#0073bb';
        let transferLabel = '';
        if (transferCount > 5) { transferBg = '#e53e3e'; transferLabel = ' 🚨 CRITICAL'; }
        else if (transferCount > 3) { transferBg = '#d97706'; transferLabel = ' ⚠️ HIGH'; }
        else if (transferCount > 2) { transferBg = '#3b82f6'; transferLabel = ' ℹ️ MODERATE'; }

        let html = `
            <div style="text-align: center; margin-bottom: 10px; padding: 8px; background: ${transferBg}; color: white; border-radius: 4px; font-weight: bold;">
                Total Transfers: ${transferCount}${transferLabel}
            </div>
        `;

        if (flow.length > 0) {
            const currentQueue = flow[flow.length - 1].label;
            html += `
                <div style="text-align: center; margin-bottom: 10px; padding: 6px 8px; background: #dbeafe; border: 1px solid #3b82f6; border-radius: 4px; font-size: 12px;">
                    📍 Currently in: <strong>${currentQueue}</strong>
                </div>
            `;
        }

        if (transferCount > 2) {
            html += buildQueueHistoryHTML(flow);
        }

        if (flow.length > 0) {
            html += `
                <div style="padding: 10px; background: #e8f4f8; border-radius: 4px; border: 2px solid #0073bb;">
                    <div style="font-weight: bold; margin-bottom: 8px; color: #0073bb; font-size: 13px;">
                        📋 Transfer Flow (Oldest → Newest)
                    </div>
            `;

            flow.forEach((item, index) => {
                const isLast = index === flow.length - 1;
                html += `
                    <div style="margin: 8px 0;">
                        <div style="background: ${isLast ? '#dbeafe' : 'white'}; padding: 8px; border-radius: 3px; border-left: 3px solid ${isLast ? '#3b82f6' : '#0073bb'};">
                            <div style="font-size: 11px; color: #666; font-weight: bold;">
                                #${index + 1} ${isLast ? '📍 (Current)' : ''}
                            </div>
                            <div style="font-weight: bold; font-size: 12px; color: #333; margin-top: 2px;">
                                ${item.label}
                            </div>
                            <div style="font-size: 10px; color: #666; margin-top: 2px;">
                                ${item.email}
                            </div>
                        </div>
                        ${!isLast ? '<div style="text-align: center; color: #0073bb; font-size: 16px; margin: 4px 0;">↓</div>' : ''}
                    </div>
                `;
            });

            html += '</div>';
        } else {
            html += '<div style="text-align: center; color: #666; padding: 10px;">No transfer emails found</div>';
        }

        resultDiv.innerHTML = html;
        resultDiv.style.display = 'block';

        console.log('=== Transfer Route Results ===');
        console.log(`Total Transfers: ${transferCount}`);
        if (transferCount > 5) {
            console.log('🚨 CRITICAL: Transfer count > 5. Escalate to Leadership in WIMS!');
        } else if (transferCount > 3) {
            console.log('⚠️ HIGH: Transfer count > 3. Check SOP or transfer to Problem Solver.');
        } else if (transferCount > 2) {
            console.log('ℹ️ MODERATE: Transfer count > 2. Review SOP before next action.');
        }
        flow.forEach((item, index) => {
            console.log(`${index + 1}. ${item.label} (${item.email})`);
        });
    }

    // ════════════════════════════════════════════════════
    //  MAIN SCAN
    // ════════════════════════════════════════════════════
    async function performScan() {
        const taskId = getTaskIdFromUrl();
        if (!taskId) {
            statusDiv.textContent = 'No task ID found in URL';
            resultDiv.style.display = 'none';
            headerBadge.style.display = 'none';
            return;
        }

        if (taskId === lastScannedTaskId && lastTransferData) {
            return;
        }

        statusDiv.textContent = '⏳ Fetching case info...';
        statusDiv.style.color = '#aaa';
        resultDiv.style.display = 'none';
        rescanButton.disabled = true;
        rescanButton.textContent = '⏳ Scanning...';

        try {
            statusDiv.textContent = '⏳ Getting case ID...';
            const caseId = await fetchCaseId(taskId);

            if (!caseId) {
                statusDiv.textContent = '⚠️ No case ID found (signal task?)';
                statusDiv.style.color = '#d97706';
                headerBadge.textContent = '—';
                headerBadge.style.background = '#6c757d';
                headerBadge.style.display = 'inline-block';
                rescanButton.disabled = false;
                rescanButton.textContent = '🔄 Rescan';
                lastScannedTaskId = taskId;
                return;
            }

            statusDiv.textContent = '⏳ Reading messages...';
            const correspondences = await fetchMessages(caseId);

            const { count, flow } = countTransfers(correspondences);

            statusDiv.textContent = `✅ Case ${caseId} — ${count} transfers`;
            statusDiv.style.color = count > 3 ? '#e53e3e' : count > 2 ? '#d97706' : '#28a745';

            displayResults(count, flow);
            lastScannedTaskId = taskId;

            if (count > 2) {
                setTimeout(() => {
                    showTransferWarning(count, flow);
                }, 800);
            }

        } catch (e) {
            console.error('Transfer Route: Scan failed', e);
            statusDiv.textContent = '❌ Scan failed — click Rescan';
            statusDiv.style.color = '#e53e3e';
        } finally {
            rescanButton.disabled = false;
            rescanButton.textContent = '🔄 Rescan';
        }
    }

    rescanButton.addEventListener('click', () => {
        lastScannedTaskId = null;
        lastTransferData = null;
        performScan();
    });

    // ════════════════════════════════════════════════════
    //  VISIBILITY
    // ════════════════════════════════════════════════════
    function isInsideCase() {
        return /\/tasks\/[a-f0-9-]{36}/i.test(window.location.href);
    }

    function updateVisibility() {
        if (isInsideCase()) {
            if (!container.parentElement) {
                document.body.appendChild(container);
            }
            container.style.display = '';
            setTimeout(performScan, 1000);
        } else {
            container.style.display = 'none';
            lastScannedTaskId = null;
            lastTransferData = null;
        }
    }

    updateVisibility();

    let lastUrl = window.location.href;
    const urlObserver = new MutationObserver(() => {
        if (window.location.href !== lastUrl) {
            lastUrl = window.location.href;
            updateVisibility();
        }
    });
    urlObserver.observe(document.body, { childList: true, subtree: true });

    window.addEventListener('popstate', updateVisibility);

})();
