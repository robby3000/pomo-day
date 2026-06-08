# Pomo/Day — Development Roadmap

## Immediate Fix: Notification API Error

The `Notification is not defined` error occurs because the Web Notifications API is not available in all contexts (file:// protocol, some mobile browsers, certain webviews). Wrap every Notification reference in a guard:

```js
function notificationsSupported() {
  return typeof Notification !== 'undefined';
}

function notify(title, body) {
  // ... existing toast UI code ...

  if (notificationsSupported()) {
    if (Notification.permission === 'granted') {
      new Notification(title, { body });
    } else if (Notification.permission !== 'denied') {
      Notification.requestPermission();
    }
  }
}
```

Also guard the init call at the bottom:

```js
if (notificationsSupported() && Notification.permission === 'default') {
  Notification.requestPermission();
}
```

---

## Local Persistence with localStorage

Store the full app state to localStorage on every mutation, and rehydrate it on load. This preserves tasks, session counts, and focus time across browser sessions.

### Saving state

Call this after every state-mutating function (addTask, toggleDone, deleteTask, onSessionEnd):

```js
function saveState() {
  const toSave = {
    tasks: state.tasks,
    sessionsCompleted: state.sessionsCompleted,
    totalFocusMin: state.totalFocusMin,
    sessionInCycle: state.sessionInCycle,
    log: state.log.slice(0, 50) // cap log length
  };
  localStorage.setItem('pomoday_state', JSON.stringify(toSave));
}
```

### Restoring state on load

Replace the init block with:

```js
function loadState() {
  try {
    const raw = localStorage.getItem('pomoday_state');
    if (!raw) return;
    const saved = JSON.parse(raw);
    state.tasks = saved.tasks || [];
    state.sessionsCompleted = saved.sessionsCompleted || 0;
    state.totalFocusMin = saved.totalFocusMin || 0;
    state.sessionInCycle = saved.sessionInCycle || 0;
    state.log = saved.log || [];
  } catch (e) {
    console.warn('Could not restore state:', e);
  }
}

loadState();
applyColor(COLORS.work);
renderClock();
renderDots();
renderTasks();
renderLog();
renderStats();
```

### Notes on localStorage

- Storage is per-origin. When running as a local file (`file://`), each directory path is its own origin — keep the file in a stable location or the data will appear missing.
- localStorage is synchronous and has a ~5MB limit per origin, which is more than sufficient for task lists.
- Add a "Clear all data" button in a settings panel so the user can reset intentionally.

---

## Suggested Functionality Improvements

### High priority

**Fix the responsive viewport dimensions** - currently the right edge of the Task area is cut off when in desktop view.

**Task ordering** — drag-and-drop reordering of tasks. Use the native HTML5 draggable API or a small library like SortableJS. The current sort-by-active is automatic; manual ordering gives the user more control for priority stacking.

**Daily reset** — store the date of the last session alongside state. On load, if the date has changed, archive completed tasks and reset session counters. Show a "Yesterday you completed X tasks" banner before clearing.

**Task editing** — tap a task name to edit it inline. Currently tasks are immutable after creation.

**Estimated vs actual pomodoros** — the current mini-dots show estimated count. Track actual time-on-task separately and display a comparison (e.g. `2/3 est → 4 actual`) to help calibrate future estimates.

### Medium priority

**Sound cues** — play a short tone when a session ends using the Web Audio API (no external files needed). A simple oscillator beep is enough. Make it optional via a toggle.

**Task notes** — expandable notes field per task. Tap a chevron to reveal a textarea for context, links, or subtask bullet points. Store alongside the task object.

**Pomodoro history chart** — a simple bar chart (pure canvas or CSS bars, no library needed) showing daily session counts for the past 7 days. Store a `history: { [dateString]: sessionCount }` object in state.

**Break suggestions** — during break mode, cycle through a short list of suggested activities ("stretch", "water", "eyes closed 20s") shown in the current-task block.

### Lower priority / longer term

**Project/tag grouping** — assign tasks to named projects (colour-coded). Filter the task list by project. Store projects as a separate array with id, name, and colour.

**Keyboard shortcuts** — spacebar to start/pause, R to reset, S to skip, N to focus the add-task input. Add a small keyboard legend toggled by `?`.

**Export** — a "Copy summary" button that formats completed tasks and session count as plain text or markdown for pasting into a daily note or journal.

**PWA manifest** — add a `manifest.json` and service worker so the app can be installed to the home screen and works fully offline. Given the existing single-file architecture, the service worker can cache just the one HTML file.

**Sync across devices** — longer term. Options in order of complexity: manual JSON export/import, sync via a shared Cloudflare KV key (fits naturally with the existing YM worker architecture), or ATproto record storage for federation with the broader Yesmate stack.