/**
 * public/app.js
 *
 * Client-side logic for Million Checkboxes.
 *
 * ── Key Concepts ─────────────────────────────────────────────────────────
 *
 * 1. STATE BUFFER
 *    All 1,000,000 checkbox states are stored in a Uint8Array(125_000).
 *    1 bit per checkbox. getBit/setBit use Redis-compatible big-endian ordering.
 *
 * 2. VIRTUAL SCROLLING
 *    Rendering 1M DOM nodes is impossible. We only render rows that are
 *    currently visible in the viewport + a small buffer above/below.
 *    Rows are absolutely positioned inside a full-height container.
 *    On scroll, old rows outside the visible range are removed and new
 *    rows entering the range are created.
 *
 * 3. WEBSOCKET
 *    Toggles are sent as { type:"toggle", index:N }.
 *    Updates arrive as { type:"update", index:N, state:0|1, updatedBy:"…" }.
 *    We apply optimistic updates immediately (feels instant), then reconcile
 *    when the server broadcasts the ground-truth state.
 *
 * 4. RECONNECTION
 *    Exponential back-off reconnects the WebSocket if the connection drops.
 *    On reconnect, the full state is re-fetched so nothing is missed.
 */

"use strict";

// ─── Constants ───────────────────────────────────────────────────────────────
const TOTAL      = 1_000_000;
const CELL_SIZE  = parseInt(getComputedStyle(document.documentElement).getPropertyValue('--cell-size')) || 16;
const CELL_GAP   = parseInt(getComputedStyle(document.documentElement).getPropertyValue('--cell-gap'))  || 2;
const CELL_STEP  = CELL_SIZE + CELL_GAP;   // px per cell including gap
const ROW_H      = CELL_STEP;              // row height in px
const BUFFER     = 8;                      // extra rows to render above/below viewport

// ─── DOM refs ─────────────────────────────────────────────────────────────────
const gridContainer  = document.getElementById('grid-container');
const gridContent    = document.getElementById('grid-content');
const loadingOverlay = document.getElementById('loading-overlay');
const loadingDetail  = document.getElementById('loading-detail');
const authBanner     = document.getElementById('auth-banner');
const authSection    = document.getElementById('auth-section');
const wsStatusEl     = document.getElementById('ws-status');
const checkedCountEl = document.getElementById('checked-count');
const userCountEl    = document.getElementById('user-count');
const viewportInfo   = document.getElementById('viewport-info');
const lastUpdateEl   = document.getElementById('last-update');
const toastContainer = document.getElementById('toast-container');
const resetBtn       = document.getElementById('reset-btn');

// ─── State ────────────────────────────────────────────────────────────────────
let checkboxState  = new Uint8Array(Math.ceil(TOTAL / 8)); // 125_000 bytes
let COLS           = 100;   // computed from viewport width
let ROWS           = Math.ceil(TOTAL / COLS);
let currentUser    = null;  // { sub, name, email, avatar } or null
let checkedCount   = 0;     // running total of checked boxes
let ws             = null;
let wsReconnectMs  = 500;
let wsReconnectTimer = null;

// ─── Bit helpers (big-endian, Redis-compatible) ───────────────────────────────
function getBit(index) {
  return (checkboxState[index >> 3] >> (7 - (index & 7))) & 1;
}

function setBit(index, value) {
  const byteIdx = index >> 3;
  const bitIdx  = 7 - (index & 7);
  if (value) {
    checkboxState[byteIdx] |=  (1 << bitIdx);
  } else {
    checkboxState[byteIdx] &= ~(1 << bitIdx);
  }
}

// ─── Grid layout ─────────────────────────────────────────────────────────────
function computeLayout() {
  const containerWidth = gridContainer.clientWidth;
  // How many full cells fit in the container width?
  COLS = Math.max(10, Math.floor((containerWidth - CELL_GAP) / CELL_STEP));
  ROWS = Math.ceil(TOTAL / COLS);

  // Set total virtual height so the scrollbar is accurate
  gridContent.style.width  = `${COLS * CELL_STEP + CELL_GAP}px`;
  gridContent.style.height = `${ROWS * ROW_H + CELL_GAP}px`;
}

// ─── Virtual scroll state ─────────────────────────────────────────────────────
const renderedRows = new Map(); // rowIndex → <div class="grid-row">

function getVisibleRange() {
  const scrollTop   = gridContainer.scrollTop;
  const viewHeight  = gridContainer.clientHeight;
  const firstRow = Math.max(0,        Math.floor(scrollTop / ROW_H) - BUFFER);
  const lastRow  = Math.min(ROWS - 1, Math.ceil((scrollTop + viewHeight) / ROW_H) + BUFFER);
  return { firstRow, lastRow };
}

function renderRow(rowIndex) {
  const row = document.createElement('div');
  row.className = 'grid-row';
  row.style.top = `${rowIndex * ROW_H + CELL_GAP}px`;

  const startIndex = rowIndex * COLS;
  const endIndex   = Math.min(startIndex + COLS, TOTAL);

  for (let i = startIndex; i < endIndex; i++) {
    const cell = document.createElement('div');
    cell.className = 'cb-cell' + (getBit(i) ? ' checked' : '');
    cell.dataset.i = i;
    row.appendChild(cell);
  }

  gridContent.appendChild(row);
  renderedRows.set(rowIndex, row);
}

function removeRow(rowIndex) {
  const row = renderedRows.get(rowIndex);
  if (row) {
    row.remove();
    renderedRows.delete(rowIndex);
  }
}

function updateVirtualScroll() {
  const { firstRow, lastRow } = getVisibleRange();

  // Remove rows that scrolled out of range
  for (const [ri] of renderedRows) {
    if (ri < firstRow || ri > lastRow) removeRow(ri);
  }

  // Render newly visible rows
  for (let r = firstRow; r <= lastRow; r++) {
    if (!renderedRows.has(r)) renderRow(r);
  }

  // Update footer info
  viewportInfo.textContent = `Rows ${firstRow + 1}–${lastRow + 1} / ${ROWS.toLocaleString()}`;
}

// ─── Click handler (event delegation on grid content) ────────────────────────
gridContent.addEventListener('click', (e) => {
  const cell = e.target.closest('.cb-cell');
  if (!cell) return;

  if (!currentUser) {
    showToast('Sign in to toggle checkboxes.', 'error');
    return;
  }

  const index = parseInt(cell.dataset.i);
  if (isNaN(index)) return;

  // Optimistic update: flip locally immediately
  const newState = getBit(index) ? 0 : 1;
  setBit(index, newState);
  cell.classList.toggle('checked', newState === 1);
  updateCheckedCount(newState === 1 ? 1 : -1);

  // Send to server via WebSocket
  if (ws?.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'toggle', index }));
  } else {
    // Revert optimistic update if not connected
    setBit(index, newState ? 0 : 1);
    cell.classList.toggle('checked', !newState);
    updateCheckedCount(newState === 1 ? -1 : 1);
    showToast('Not connected — please wait for reconnect.', 'error');
  }
});

// ─── Apply a server-authoritative update to a single checkbox ────────────────
function applyUpdate(index, state, updatedBy) {
  const oldState = getBit(index);
  if (oldState === state) return; // already correct (from optimistic update)

  setBit(index, state);
  updateCheckedCount(state === 1 ? 1 : -1);

  // Update DOM if the cell is currently rendered
  const rowIndex = Math.floor(index / COLS);
  const colIndex = index % COLS;
  const row = renderedRows.get(rowIndex);
  if (row) {
    const cell = row.children[colIndex];
    if (cell) {
      cell.classList.toggle('checked', state === 1);
      // Briefly highlight the changed cell
      cell.style.outline = `2px solid var(--accent-2)`;
      setTimeout(() => { cell.style.outline = ''; }, 400);
    }
  }

  if (updatedBy && updatedBy !== currentUser?.name) {
    lastUpdateEl.textContent = `#${index.toLocaleString()} ${state ? '✔' : '✕'} by ${updatedBy}`;
  }
}

// ─── Checked count helper ─────────────────────────────────────────────────────
function updateCheckedCount(delta) {
  checkedCount = Math.max(0, Math.min(TOTAL, checkedCount + delta));
  checkedCountEl.textContent = checkedCount.toLocaleString();
}

function recomputeCheckedCount() {
  let count = 0;
  for (let i = 0; i < checkboxState.length; i++) {
    let b = checkboxState[i];
    while (b) { count += b & 1; b >>= 1; }
  }
  checkedCount = count;
  checkedCountEl.textContent = count.toLocaleString();
}

// ─── Load full checkbox state from server ─────────────────────────────────────
async function loadState() {
  loadingDetail.textContent = 'Fetching checkbox state…';
  try {
    const res  = await fetch('/api/checkboxes/state');
    const json = await res.json();

    // Decode base64 → Uint8Array
    const b64    = json.data;
    const binary = atob(b64);
    const bytes  = new Uint8Array(binary.length);
    for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);

    checkboxState = bytes;
    loadingDetail.textContent = 'Counting checked boxes…';
    recomputeCheckedCount();
    return true;
  } catch (err) {
    console.error('[State] Load failed:', err);
    loadingDetail.textContent = 'Failed to load — using empty state.';
    return false;
  }
}

// ─── WebSocket ────────────────────────────────────────────────────────────────
function connectWebSocket() {
  if (ws && ws.readyState !== WebSocket.CLOSED) return;

  const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:';
  const url      = `${protocol}//${location.host}/ws`;

  setWsStatus('connecting');
  ws = new WebSocket(url);

  ws.addEventListener('open', () => {
    setWsStatus('connected');
    wsReconnectMs = 500; // reset back-off
    console.log('[WS] Connected');
  });

  ws.addEventListener('message', (e) => {
    let msg;
    try { msg = JSON.parse(e.data); } catch { return; }

    switch (msg.type) {
      case 'update':
        applyUpdate(msg.index, msg.state, msg.updatedBy);
        break;

      case 'stats':
        userCountEl.textContent = msg.connectedUsers.toLocaleString();
        break;

      case 'error':
        console.warn('[WS] Server error:', msg.message);
        if (msg.code === 'AUTH_REQUIRED') {
          showToast('Sign in to toggle checkboxes.', 'error');
        } else if (msg.code === 'RATE_LIMITED') {
          showToast(`Slow down! Max toggles reached.`, 'error');
        } else {
          showToast(msg.message, 'error');
        }
        break;

      case 'pong':
        break; // heartbeat response

      default:
        console.log('[WS] Unknown message:', msg);
    }
  });

  ws.addEventListener('close', () => {
    setWsStatus('disconnected');
    console.warn(`[WS] Disconnected. Reconnecting in ${wsReconnectMs}ms…`);
    wsReconnectTimer = setTimeout(() => {
      wsReconnectMs = Math.min(wsReconnectMs * 2, 30_000); // cap at 30s
      connectWebSocket();
    }, wsReconnectMs);
  });

  ws.addEventListener('error', (err) => {
    console.error('[WS] Error:', err);
  });
}

function setWsStatus(status) {
  wsStatusEl.className = `ws-badge ws-${status}`;
  const label = wsStatusEl.querySelector('.ws-label');
  label.textContent = { connecting: 'Connecting…', connected: 'Live', disconnected: 'Offline' }[status];
}

// ─── Auth UI ──────────────────────────────────────────────────────────────────
async function loadUser() {
  try {
    const res = await fetch('/auth/me', { credentials: 'include' });
    if (res.ok) {
      currentUser = await res.json();
      renderAuthUI(currentUser);
    } else {
      currentUser = null;
      renderAuthUI(null);
    }
  } catch {
    currentUser = null;
    renderAuthUI(null);
  }
}

function renderAuthUI(user) {
  if (user) {
    authSection.innerHTML = `
      <div class="user-chip">
        <div class="user-avatar">${user.avatar || user.name[0].toUpperCase()}</div>
        <span class="user-name">${user.name}</span>
      </div>
      <a href="/auth/logout" class="btn-logout">Sign Out</a>
    `;
    gridContainer.classList.remove('readonly');
    authBanner.style.display = 'none';
    resetBtn.style.display = 'block';
  } else {
    authSection.innerHTML = `<a href="/auth/login" class="btn-login">Sign In</a>`;
    gridContainer.classList.add('readonly');
    authBanner.style.display = 'flex';
    gridContainer.classList.add('has-banner');
    resetBtn.style.display = 'none';
  }
}

// ─── Reset handler ────────────────────────────────────────────────────────────
async function resetAll() {
  if (!currentUser) return;
  if (!confirm('Reset ALL 1,000,000 checkboxes to unchecked?')) return;
  try {
    const res = await fetch('/api/checkboxes/reset', {
      method: 'POST',
      credentials: 'include',
    });
    if (res.ok) {
      checkboxState.fill(0);
      checkedCount = 0;
      checkedCountEl.textContent = '0';
      // Re-render all visible rows
      for (const [ri] of [...renderedRows]) {
        removeRow(ri);
      }
      updateVirtualScroll();
      showToast('All checkboxes reset!', 'success');
    }
  } catch (err) {
    showToast('Reset failed.', 'error');
  }
}

// ─── Toast notifications ──────────────────────────────────────────────────────
function showToast(message, type = '') {
  const toast = document.createElement('div');
  toast.className = `toast ${type ? 'toast-' + type : ''}`;
  toast.textContent = message;
  toastContainer.appendChild(toast);
  setTimeout(() => {
    toast.style.opacity = '0';
    toast.style.transition = 'opacity 0.3s';
    setTimeout(() => toast.remove(), 300);
  }, 3000);
}

// ─── Handle window resize ─────────────────────────────────────────────────────
let resizeTimer;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    // Clear all rendered rows and recompute layout
    for (const [ri] of [...renderedRows]) removeRow(ri);
    computeLayout();
    updateVirtualScroll();
  }, 200);
});

// ─── Scroll handler (throttled) ───────────────────────────────────────────────
let scrollRaf = null;
gridContainer.addEventListener('scroll', () => {
  if (scrollRaf) return;
  scrollRaf = requestAnimationFrame(() => {
    updateVirtualScroll();
    scrollRaf = null;
  });
});

// ─── Boot sequence ────────────────────────────────────────────────────────────
async function boot() {
  // 1. Load current user
  await loadUser();

  // 2. Compute grid layout based on viewport
  computeLayout();

  // 3. Fetch full checkbox state
  await loadState();

  // 4. Connect WebSocket
  connectWebSocket();

  // 5. Render initial visible rows
  updateVirtualScroll();

  // 6. Hide loading overlay
  loadingOverlay.classList.add('hidden');
  setTimeout(() => { loadingOverlay.style.display = 'none'; }, 500);

  // 7. Periodic stats refresh (backup in case WS stats message missed)
  setInterval(async () => {
    try {
      const res = await fetch('/api/checkboxes/stats');
      const j   = await res.json();
      if (j.connectedUsers !== undefined) userCountEl.textContent = j.connectedUsers.toLocaleString();
    } catch { /* ignore */ }
  }, 10_000);
}

boot();
