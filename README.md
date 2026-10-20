# IREKA HOMES — Task & Performance Management

React (Vite) + Firebase Authentication, Cloud Firestore, Firebase Storage and Firebase Hosting.

Three roles: **Admin**, **Senior** (department senior) and **Employee**. Seniors create tasks with a manual deadline, employees work, self-recheck and submit, and the task is only **completed** when the senior approves it. Every task carries a deadline colour: **GREEN – ON TIME**, **YELLOW – LATE WORK** (up to 2 calendar days overdue), **RED – DISASTER** (more than 2 days overdue).

---

## Project structure

```
ireka-homes-task-management/
├── public/irekahomes-logo.webp      the uploaded logo (login, sidebar, reports, PDFs)
├── scripts/seed.mjs                 creates the first admin + demo data (Admin SDK)
├── src/
│   ├── components/                  Sidebar, Navbar, TaskCard, TaskStatus, NotificationPanel,
│   │                                EmployeeCard, Charts, TaskForm, SelfRecheckModal, ReviewPanel,
│   │                                TaskTable, TaskFilters, UserForm, Timeline, Modal, Toast, Icon…
│   ├── context/                     AuthContext (session + live profile), DataContext (role-scoped data)
│   ├── firebase/                    firebaseConfig, auth, firestore (all DB operations), storage
│   ├── pages/                       Login, Admin/Senior/EmployeeDashboard, Employees, Departments,
│   │                                Tasks, TaskDetails, Approvals, Activities, Reports, Performance,
│   │                                KraKpi, Settings, Profile
│   ├── utils/                       deadlineStatus (getTaskDeadlineStatus), permissions,
│   │                                reportGenerator, exporters (Excel/PDF/print), format, selectors
│   ├── App.jsx  main.jsx  styles.css
├── firestore.rules                  role-based security (the real enforcement)
├── firestore.indexes.json           composite indexes for logs and notifications
├── storage.rules                    attachment access + type/size limits
├── firebase.json  .firebaserc.example  .env.example
├── index.html  package.json  vite.config.js
```

---

## 1. Install dependencies

Requirements: Node.js 18 or newer, npm, and the Firebase CLI.

```bash
cd ireka-homes-task-management
npm install
npm install -g firebase-tools
firebase login
```

## 2. Create the Firebase project

1. Open <https://console.firebase.google.com> and click **Add project**. Name it (e.g. `ireka-homes-tasks`).
2. In **Project settings > General > Your apps**, click the **Web** icon (`</>`), register an app called `IREKA HOMES Web`, and keep the config values shown. You will need them in step 6.
3. Storage and cross-service rules need the **Blaze (pay as you go)** plan on newer projects. Usage for a team of this size normally stays within the free allowance.

## 3. Enable Firebase Authentication

**Build > Authentication > Get started > Sign-in method > Email/Password > Enable > Save.**

Password reset emails work automatically. Customise the template under **Authentication > Templates** if you like.

## 4. Create the Firestore database

**Build > Firestore Database > Create database.** Choose a location close to your users (e.g. `asia-south1` for India) and start in **production mode**. The rules from this project are deployed in step 7.

## 5. Create Firebase Storage

**Build > Storage > Get started**, production mode, same location as Firestore.

Storage rules read Firestore to check who may access a task's files. The first time you deploy `storage.rules`, the CLI asks to grant Storage permission to read Firestore: answer **yes**.

## 6. Add the Firebase configuration

```bash
cp .env.example .env.local
cp .firebaserc.example .firebaserc
```

Fill `.env.local` with the values from step 2, and put your project ID in `.firebaserc`. `.env.local` is git-ignored, so no keys end up in source control. (Firebase web keys identify the project; security comes from the rules, not from hiding the key.)

## 7. Deploy Security Rules and indexes

```bash
firebase deploy --only firestore:rules,firestore:indexes,storage
```

Indexes take a few minutes to build. If a page shows an error mentioning an index, wait and reload; Firestore also prints a direct link to create any missing index.

What the rules enforce:

| | Admin | Senior | Employee |
|---|---|---|---|
| Users | all | self + own team | self + own senior |
| Tasks | all | only tasks where they are the senior | only tasks assigned to them |
| Create task | yes | only for employees reporting to them | no |
| Approve / reject / revise | yes | only their own team's tasks | never |
| Change `seniorId`, `departmentId`, `role`, `approvalStatus`, `approvedBy`, `approvedAt`, deadline | yes | own tasks (not hierarchy) | never |
| Submit for review | — | — | only with `selfChecked == true` |
| Approval history, activity log | append-only, never edited or deleted |

## 8. Create the first admin and demo data

Firestore rules (correctly) prevent anyone from making themselves an admin, so the first admin is created with the Admin SDK:

1. **Project settings > Service accounts > Generate new private key**. Save the file as `serviceAccountKey.json` in the project root (it is git-ignored; never share it).
2. Run:

```bash
npm run seed
# or choose the demo password:
SEED_PASSWORD="Choose-A-Strong-Pass1" npm run seed
```

This creates:

| Account | Role |
|---|---|
| admin@irekahomes.com | Admin |
| senior.sales@irekahomes.com | Senior, Sales (Amit Sharma) |
| senior.operations@irekahomes.com | Senior, Operations (to test team isolation) |
| employee1@irekahomes.com, employee2@…, employee3@… | Employees reporting to the Sales senior |

Default password: `Ireka@12345`. Departments: Sales, HR, Operations. Ten tasks cover ON TIME, LATE WORK, DISASTER, PENDING REVIEW, APPROVED, REJECTED and REVISION REQUIRED, with approval history, KRA/KPI samples and notifications. Deadlines are relative to the day you run the seed, so the colours are always correct. Re-running replaces the demo tasks.

Skip the seed for production: create only the admin with it, then delete the demo users and tasks, or create real users from **Users** in the app.

## 9. Run locally

```bash
npm run dev
```

Open <http://localhost:5173> and sign in. Each role lands on its own dashboard.

## 10. Build and deploy to Firebase Hosting

```bash
npm run build          # outputs dist/
firebase deploy        # hosting + rules + indexes + storage
# or only the site:
firebase deploy --only hosting
```

The app is then live at `https://<project-id>.web.app`. `firebase.json` rewrites every route to `index.html` so deep links such as `/tasks/abc` work.

---

## How the workflow maps to the app

1. **Admin > Departments**: create Sales, HR, Operations… and tick the seniors of each.
2. **Admin > Users > Add senior / Add employee**: choosing a department lists only that department's seniors; the employee is linked to the selected senior. Only admins can change this later.
3. **Senior > Create task**: title, description, type (daily/weekly/monthly/custom), employee (own team only), priority, start date/time, deadline date/time (always chosen manually), estimated hours, instructions, checklist, attachments, and optional repeat (daily/weekly/monthly series).
4. **Employee > task page**: Start task, Update progress (percentage, minutes spent, notes, work description), Upload files, tick checklist, Mark as complete.
5. **Self-recheck**: six checks plus "I have completed and rechecked this task." Only then is **Submit for senior review** enabled; `selfChecked`, `selfCheckedAt`, `selfCheckedBy` are stored.
6. **Senior > Approvals**: sees employee, task, description, deadline, time spent, progress, notes, files, self-recheck and submission time. **Approve**, **Reject** (reason required) or **Request revision** (instructions required). Every action is appended to `taskApprovals`; nothing is overwritten.
7. Rejected or revision tasks go back to the employee, who fixes, re-runs the self-recheck and resubmits.
8. **Performance**: monthly figures per employee, senior evaluation (quality, productivity, timeliness, accuracy, discipline, communication, 1 to 5), overall score and rating, admin override, PDF/print. **Generate month-end reports** stores a snapshot for the month.

### Tasks for seniors (assigned by the admin)

Admins can give tasks to seniors: **Tasks > Create task > Assign to: Senior**, or **Users > Assign task** on a senior's row. The senior sees them under **Dashboard > My tasks** and **Tasks > My tasks**, works on them exactly like an employee (start, progress, files, self-recheck, submit), and the admin reviews them in **Approvals**. Seniors can never approve their own tasks; the rules allow only admins to. Admin also sees seniors in **Performance** and can evaluate them.

### KRA & KPI (permanent monthly KPIs)

Each person has their own KRAs; each KRA holds KPIs with three fields only: **KPI name**, **Month** (the month it starts) and **Time period** (Daily, Weekly or Monthly). A KPI repeats every month automatically until the admin sets a stop month.

- Daily KPIs have one box per working day (Sundays off); weekly KPIs have W1 to W5 (1–7, 8–14, 15–21, 22–28, 29 to month end); monthly KPIs have one box.
- The person clicks a box to mark it done. Only the **admin** approves or marks it not approved (with a note), or approves directly. **Approve all waiting** approves a whole month at once.
- KPI score = approved ÷ periods due so far. It is shown on KRA & KPI and in Performance.
- **Apply template** adds a ready-made set built from the role sheets: Accountant Executive, Assistant Accountant, E-commerce Executive, Warehouse Operations, Graphic Designer, E-commerce Marketplace Manager, HR & Admin Executive (`src/utils/kraTemplates.js`).
- Only the admin can change the month being viewed; employees and seniors see the current month.

### Deadline logic (`src/utils/deadlineStatus.js`)

```
APPROVED / CANCELLED / REJECTED / REVISION REQUIRED  -> that label
pending senior review                                -> colour frozen at submission time
now <= deadline                                      -> GREEN  ON TIME
deadline passed, overdue <= 2 calendar days          -> YELLOW LATE WORK
overdue > 2 calendar days                            -> RED    DISASTER
```

Deadline 10 Oct 5 PM: 10 Oct 3 PM green; 11 Oct and 12 Oct yellow; 13 Oct red. The 2-day window is adjustable in **Settings**. Colours refresh every minute without reloading.

### Dashboards and KPI rules

- **Admin dashboard** (any month): company KPI score, people with KPIs, waiting for approval (with Review buttons), needs attention (score under 60% or missed), daily trend, score by week, status split, score by time period, department and people rankings.
- **Senior dashboard**: their own KPIs running now (mark done from the dashboard) and score, then the team's KPI table and charts. A small strip keeps task shortcuts.
- **Employee dashboard**: "My KPIs right now" with Mark done / Undo, monthly KPI score, KRA-wise chart, daily trend.
- **Settings > KPI rules** (admin): weekly off days (default Sunday), late marking window in days (default 2), KPI weight in the overall score (default 70%), and rating limits. Stored in `settings/kpiRules`.

### Performance and reports (based on KRA & KPI, not tasks)

- **Reports**: daily (that day's daily KPIs), weekly (daily KPIs of that KPI week plus the weekly KPIs), monthly (all KPIs). Columns: KPIs, due, approved, waiting, not approved, missed, KPI score; monthly also shows Daily %, Weekly % and Monthly %. Export summary or KPI-wise Excel, PDF, print.
- **Performance** (monthly): KPI score = approved ÷ due so far. Overall = 70% KPI score + 30% evaluation (KPI score alone until an evaluation is saved). Includes a KRA-wise breakdown, evaluation (senior for their team, admin for everyone), admin override, PDF and month-end snapshots.
- Rating: 90+ Excellent, 75+ Very good, 60+ Good, 40+ Average, below 40 Needs improvement.
- Dashboards still show task statistics; tasks no longer affect performance scores.

### Notifications

Created in-app (bell icon) for: new task, deadline approaching, late, disaster, rejected, revision required, approved, comments (employees); new submission, resubmission, team task overdue (seniors); disaster tasks and high overdue volume (admins, via a shared admin inbox).

Deadline alerts are produced by a monitor that runs in the browser of any signed-in user who can see the task (every 5 minutes, de-duplicated with a transaction). If nobody is signed in, alerts fire at the next sign-in. For alerts at exact times with no one online, move `runDeadlineMonitor` into a scheduled Cloud Function (Blaze plan).

### Exports

Excel (SheetJS), PDF (jsPDF with the IREKA HOMES logo) and browser print on Tasks, Reports, Performance, Users, KRA/KPI and Activity log. Each role exports only the data its rules allow it to read.

---

## Notes and limits

- **Deleting users**: the app removes the Firestore profile, which blocks sign-in immediately. The Auth account itself can be removed in **Firebase Console > Authentication** (the client SDK cannot delete other users' Auth accounts without a Cloud Function).
- **Cross-department access** for seniors is not granted from the UI; seniors see only their own team, as the rules require.
- **File types**: PDF, Word, Excel, CSV and images up to 20 MB, enforced both in the app and in `storage.rules`.
- **Changing rules**: edit `firestore.rules` and redeploy with `firebase deploy --only firestore:rules`. Test changes with the Firebase Emulator Suite (`firebase emulators:start`) before deploying.
