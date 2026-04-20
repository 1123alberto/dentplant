# Master Calendar Upgrade Guide

Use this document to carry over the optimized "Smart Calendar" from Dentplant to your whitening site or other projects.

---

## 1. Google Apps Script (Backend)
**Location**: [script.google.com](https://script.google.com)
**Action**: Replace your entire script with the code below. Remember to click **Deploy > New Version**.

```javascript
/* 
  MASTER CALENDAR BACKEND (v2.0)
  Features:
  - Monthly Busy-Day Scanning
  - [BLOCK] keyword detection
  - Patient Appointment Booking & Email Reminders
*/

// --- CONFIGURATION ---
const SERVICE_DURATION = 60; 
const CALENDAR_NAMES = ['Online Appointment', 'Dental Office', "M's calendar"]; 
const CLINIC_HOURS = {
  1: { start: 10, end: 20 }, // Mon
  2: { start: 10, end: 20 }, // Tue
  3: { start: 10, end: 20 }, // Wed
  4: { start: 10, end: 20 }, // Thu
  5: { start: 10, end: 20 }, // Fri
  6: null, // Sat
  0: null  // Sun
};
const BLOCKED_DATES = [];
const SENDER_ALIAS = '1123alberto@gmail.com'; 
const SENDER_NAME = "Οδοντιατρείο - A. Moshopoulos - Dental Clinic"; 
// ----------------------

function getCalendars() {
  const cals = [];
  CALENDAR_NAMES.forEach(name => {
    const list = CalendarApp.getCalendarsByName(name);
    if (list.length > 0) cals.push(list[0]);
  });
  if (cals.length === 0) cals.push(CalendarApp.getDefaultCalendar());
  return cals;
}

function getPrimaryCalendar() {
  const list = CalendarApp.getCalendarsByName(CALENDAR_NAMES[0]);
  return list.length > 0 ? list[0] : CalendarApp.getDefaultCalendar();
}

function doGet(e) {
  const action = e.parameter.action;
  if (action === 'getAppointment') return handleGetAppointment(e.parameter.uid);
  if (action === 'getBusyDays') return handleGetBusyDays(e);

  const dateStr = e.parameter.date; 
  if (!dateStr) return respondJSON({ error: "No date provided." });

  const [year, month, day] = dateStr.split('-').map(Number);
  const targetDate = new Date(year, month - 1, day);
  const hours = CLINIC_HOURS[targetDate.getDay()];
  if (!hours || BLOCKED_DATES.includes(dateStr)) return respondJSON({ date: dateStr, slots: [] }); 

  const allCals = getCalendars();
  const startOfDay = new Date(year, month - 1, day, 0, 0, 0);
  const endOfDay = new Date(year, month - 1, day, 23, 59, 59);
  
  let busyPeriods = [];
  allCals.forEach(cal => {
    const events = cal.getEvents(startOfDay, endOfDay);
    events.forEach(ev => {
      if (ev.isAllDayEvent()) {
        if (ev.getTitle().toUpperCase().includes("[BLOCK]")) busyPeriods.push({ start: startOfDay, end: endOfDay });
        return;
      }
      busyPeriods.push({ start: ev.getStartTime(), end: ev.getEndTime() });
    });
  });
  busyPeriods.sort((a, b) => a.start - b.start);
  
  let availableSlots = [];
  const now = new Date(); 
  const minTime = new Date(now.getTime() + 6 * 60 * 60 * 1000); 

  const allowedHours = [10, 11, 12, 13, 17, 18, 19];
  for (let h of allowedHours) {
    const slotS = new Date(year, month - 1, day, h, 0, 0);
    const slotE = new Date(slotS.getTime() + SERVICE_DURATION * 60000);
    if (slotS > minTime) {
      let isOverlapping = false;
      for (const busy of busyPeriods) {
        if (slotS < busy.end && slotE > busy.start) { isOverlapping = true; break; }
      }
      if (!isOverlapping) availableSlots.push(`${h.toString().padStart(2, '0')}:00`);
    }
  }
  return respondJSON({ date: dateStr, slots: availableSlots });
}

function handleGetBusyDays(e) {
  const year = parseInt(e.parameter.year);
  const month = parseInt(e.parameter.month);
  if (!year || !month) return respondJSON({ error: "Year/Month required." });

  const startOfMonth = new Date(year, month - 1, 1, 0, 0, 0);
  const endOfMonth = new Date(year, month, 0, 23, 59, 59);
  const allEvents = [];
  getCalendars().forEach(cal => { allEvents.push(...cal.getEvents(startOfMonth, endOfMonth)); });

  let busyDays = [];
  for (let d = 1; d <= endOfMonth.getDate(); d++) {
    const dS = new Date(year, month - 1, d, 0, 0, 0);
    const dE = new Date(year, month - 1, d, 23, 59, 59);
    const dateStr = Utilities.formatDate(dS, Session.getScriptTimeZone(), "yyyy-MM-dd");
    const dayEv = allEvents.filter(ev => ev.getStartTime() < dE && ev.getEndTime() > dS);

    if (dayEv.some(ev => ev.isAllDayEvent() && ev.getTitle().toUpperCase().includes("[BLOCK]"))) {
      busyDays.push(dateStr); continue;
    }

    const allowed = [10, 11, 12, 13, 17, 18, 19];
    let busyCount = 0;
    allowed.forEach(h => {
      const sS = new Date(year, month - 1, d, h, 0, 0);
      const sE = new Date(sS.getTime() + SERVICE_DURATION * 60000);
      if (dayEv.some(ev => !ev.isAllDayEvent() && sS < ev.getEndTime() && sE > ev.getStartTime())) busyCount++;
    });
    if (busyCount >= allowed.length) busyDays.push(dateStr);
  }
  return respondJSON({ busyDays: busyDays });
}

function doPost(e) {
  try {
    const postData = JSON.parse(e.postData.contents);
    if (postData.action === 'book') return handleBook(postData);
  } catch (error) { return respondJSON({ error: error.toString() }); }
}

function handleBook(data) {
  const { date, time, name, email, phone } = data;
  const uid = Math.random().toString(36).substring(2, 10).toUpperCase();
  const [y, m, d] = date.split('-').map(Number);
  const [hr, min] = time.split(':').map(Number);
  const start = new Date(y, m - 1, d, hr, min);
  const end = new Date(start.getTime() + SERVICE_DURATION * 60000);
  getPrimaryCalendar().createEvent(name, start, end, { description: `UID: ${uid}\nPhone: ${phone}` });
  return respondJSON({ success: true, uid: uid });
}

function respondJSON(data) { return ContentService.createTextOutput(JSON.stringify(data)).setMimeType(ContentService.MimeType.JSON); }
```

---

## 2. HTML Structure
**Action**: Use this grid container in your booking form.

```html
<div class="calendar-header flex justify-between items-center mb-5 px-4">
    <button id="prev-week" class="calendar-nav-btn w-8 h-8 rounded-full border border-gray-200">&larr;</button>
    <h3 id="calendar-month" class="font-bold text-lg">Απρίλιος 2026</h3>
    <button id="next-week" class="calendar-nav-btn w-8 h-8 rounded-full border border-gray-200">&rarr;</button>
</div>
<div class="calendar-grid grid grid-cols-7 gap-y-2 gap-x-1 text-center" id="calendar-grid">
    <!-- JS Populates -->
</div>
<div id="calendar-loader" style="display: none;" class="absolute inset-0 bg-white/60 flex items-center justify-center z-50">
    <!-- Spinner -->
    <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
</div>
```

---

## 3. CSS Styling
**Action**: Add these styles to your main CSS file.

```css
.calendar-grid {
    display: grid;
    grid-template-columns: repeat(7, 1fr);
}
.date-btn {
    width: 36px;
    height: 36px;
    border-radius: 8px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 0.875rem;
    cursor: pointer;
    transition: all 0.2s;
}
.date-btn.disabled {
    color: #cbd5e1;
    cursor: not-allowed;
    opacity: 0.4;
}
.today-date {
    border: 2px solid #0284c7 !important;
    color: #0284c7 !important;
    font-weight: 700;
}
```

---

## 4. JavaScript Logic
**Action**: This logic handles navigation, caching, and dynamic blocking.

```javascript
let currentViewDate = new Date();
currentViewDate.setDate(1); 
let monthlyBusyData = {}; 

async function fetchMonthlyBusyDays(year, month) {
    const key = `${year}-${month}`;
    const loader = document.getElementById('calendar-loader');
    if (loader) loader.style.display = 'flex';

    try {
        const response = await fetch(`${GOOGLE_SCRIPT_URL}?action=getBusyDays&year=${year}&month=${month}`);
        const data = await response.json();
        monthlyBusyData[key] = data.busyDays || [];
        renderCalendar();
    } finally {
        if (loader) loader.style.display = 'none';
    }
}

function renderCalendar() {
    const grid = document.getElementById('calendar-grid');
    const monthLabel = document.getElementById('calendar-month');
    grid.innerHTML = '';

    const year = currentViewDate.getFullYear();
    const month = currentViewDate.getMonth() + 1;
    const key = `${year}-${month}`;

    if (!monthlyBusyData.hasOwnProperty(key)) fetchMonthlyBusyDays(year, month);
    const busyDays = monthlyBusyData[key] || [];

    const gridStart = new Date(currentViewDate);
    const dow = gridStart.getDay();
    const padding = dow === 0 ? 6 : dow - 1;
    gridStart.setDate(gridStart.getDate() - padding);

    monthLabel.textContent = currentViewDate.toLocaleString('el-GR', { month: 'long', year: 'numeric' });

    for (let i = 0; i < 35; i++) { 
        const date = new Date(gridStart);
        date.setDate(date.getDate() + i);
        const btn = document.createElement('button');
        btn.textContent = date.getDate();
        
        const dateStr = date.toISOString().split('T')[0];
        const isPast = date < new Date().setHours(0,0,0,0);
        const isCurrentM = date.getMonth() === currentViewDate.getMonth();
        const isBusy = busyDays.includes(dateStr);

        if (isPast || isBusy || !isCurrentM || date.getDay() === 0) {
            btn.classList.add('disabled');
            btn.disabled = true;
        } else {
            btn.onclick = () => selectDate(date);
        }
        grid.appendChild(btn);
    }
}
```
