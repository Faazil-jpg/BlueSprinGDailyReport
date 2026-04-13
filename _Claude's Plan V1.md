# BlueSpring Daily Report — 3 Modifications Plan

## Context
The app is working and deployed. Three improvements are needed:
1. Remove the "Reconfigure Firebase" link from the login page (confuses staff)
2. Allow staff to schedule tasks for future dates, with overdue highlighting when the date passes and the task is incomplete
3. Allow admins to assign tasks to specific staff members from the admin panel

---

## Critical File
**`c:\Users\FairoozMF\OneDrive - University of Twente\Documents\Programs\BlueSprinGDailyReport\daily.html`**

---

## Modification 1 — Remove "Reconfigure Firebase" link

**Location:** Line 173
```html
<a href="#" id="reconfigure-link" ...>Reconfigure Firebase</a>
```
**Fix:** Delete this `<a>` element entirely.
The click handler at lines 428-432 also references `reconfigure-link` — delete that event listener block too.

---

## Modification 2 — Future task scheduling + overdue highlighting

### 2a. Allow future dates in the report form
**Location:** `buildSubmitForm()` — line 1049
```
max="'+t+'"
```
Remove the `max` attribute so staff can pick any future date. Keep `min` unset (no restriction on past either — staff may want to backfill). The one-report-per-date logic already handles uniqueness per date.

### 2b. Overdue detection helper
Add a utility function after `todayStr()`:
```javascript
function isOverdue(dateStr, status) {
    return dateStr < todayStr() && status !== 'completed';
}
```

### 2c. Calendar cell overdue styling
In `makeCalCell()` (lines 847-861): when rendering task labels, check if the task's parent report date < today and task status !== 'completed'. Apply a red pill style instead of the normal status colour.

Also add a small red "⚠ Overdue" badge on the calendar cell itself if any task in that day is overdue.

### 2d. Overdue alert on Staff dashboard / history
In `buildStaffHistory()` and the admin's All Reports view (`buildAdminReports()`): add a red "OVERDUE" badge next to any task row where date is past and status is not completed.

### 2e. Admin dashboard warning
In `buildAdminDash()`: alongside the "Missing submissions" alert, add an "Overdue tasks" alert listing staff members who have past-dated incomplete tasks.

---

## Modification 3 — Admin assigns tasks to staff

### 3a. New Firestore field: `assignedBy`
Assigned reports will use the **existing** report document structure with one extra field:
```javascript
assignedBy: adminUser.id   // present only on admin-created reports
assignedByName: adminUser.name
```
No new Firestore collection needed — assigned tasks live alongside normal reports.

### 3b. New admin nav item — "Assign Tasks"
Add a 5th admin sidebar item: **Assign Tasks** (icon: `fa-tasks`), view id `admin-assign`.

### 3c. `buildAdminAssign()` function
Renders a form with:
- **Staff member** — dropdown populated from `FS.getUsers()` (role === 'staff')
- **Date** — date picker (defaults to tomorrow, no max)
- **Tasks section** — same dynamic task rows as the staff submit form (name, description, status, priority, category, timeSpent)
- **Notes** field
- **Assign button** — saves a report document with `assignedBy` set

On save: call `FS.saveReport(report)` with `assignedBy: currentUser.id`. If a report for that staff/date already exists, merge tasks into it (append, not overwrite).

### 3d. Staff sees assigned tasks
When staff opens **Submit Report** and picks a date:
- `FS.getReportsByUser(userId)` already loads their reports
- If the selected date has a report with `assignedBy` set, pre-populate those tasks in the form with a blue "Assigned by admin" badge on each task row
- Staff can change status/add notes but cannot delete admin-assigned tasks (hide delete button on those rows)

### 3e. Visual distinction in admin views
In `buildAdminReports()` and calendar: show a small admin-assigned icon (`fa-user-tie`) on reports that have `assignedBy` set.

---

## Implementation Order
1. Remove reconfigure link (trivial, 2 lines)
2. Remove `max` from date picker (1 line)
3. Add `isOverdue()` helper
4. Update calendar cell rendering for overdue styling
5. Add overdue badges to history/reports views
6. Add overdue alert to admin dashboard
7. Add "Assign Tasks" nav item
8. Write `buildAdminAssign()` function
9. Update staff submit form to detect and render admin-assigned tasks

---

## Verification
- Log in as staff → Submit Report → confirm date picker allows future dates
- Create a report for a past date with status "not-started" → check calendar shows red overdue indicator
- Log in as admin → Assign Tasks → assign a task to a staff member for tomorrow
- Log in as that staff member → open Submit Report for tomorrow → confirm pre-populated task appears with "Assigned by admin" badge
- Check admin dashboard shows overdue tasks alert
- Confirm "Reconfigure Firebase" link is gone from login page
