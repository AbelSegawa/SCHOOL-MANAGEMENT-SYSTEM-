# School SMS Uganda — Full App Description

## Overview

**School SMS Uganda** is an Android school management app built for Ugandan primary and secondary schools. It runs **offline-first** on each phone with a local **SQLite** database, and uses **Supabase** for **multi-school** staff login, notices, marks, and staff chat so data from different schools does not mix. Staff get role-based menus, parents can use a simple portal, and **Gemini** powers online AI help with an offline fallback.

---

## Target users

| User | How they use the app |
|------|----------------------|
| **School admin** | Setup school (UNEB/EMIS), users, classes, students, backup, cloud registration |
| **DOS** | Approve results, reports, notices, library, timetable, settings |
| **Teachers** | Marks, attendance, view students (per permissions) |
| **Bursar** | Finance, receipts (with admin) |
| **Parents** | Portal: child marks/fees; notices; messages (local ADM + PIN style login possible) |
| **Multiple schools** | Each school identified by **UNEB centre no** or **EMIS** → unique cloud `school_id` |

---

## Technical stack

- **Platform:** Android (Java), Material UI, navigation drawer with **card-style** menu items  
- **Local data:** SQLite (`DatabaseHelper`), single access layer **`SchoolRepository`** (UI should not call `DatabaseHelper` directly)  
- **Cloud:** Supabase REST + Auth (no OkHttp in sync layer — `HttpURLConnection` via `SupabaseHttp`)  
- **AI:** `GeminiClient` with model fallback; optional user API key in prefs; `OfflineAiHelper` uses local DB summaries  
- **Session:** `SessionManager` — user id, username, role, optional `school_id`, Supabase tokens  
- **Entry:** `SplashActivity` → `LoginActivity` or `MainActivity`; `MainActivity` guards session  

---

## Multi-school model (no data collision)

1. **Cloud school id** is derived from profile:  
   - `uneb_<code>` if UNEB is set (not placeholder `U0000`)  
   - else `emis_<code>` if EMIS is set (not placeholder `EMIS`)  
   - else default id from `AppConfig`  

2. **Staff Supabase login** uses synthetic emails:  
   `{school_id}__{username}@schoolsms.local`  
   Same username in **School A** and **School B** is fine — different emails and `school_staff.school_id`.

3. **Tables (Supabase):** `schools`, `school_staff` (plus your existing notices/marks/messages tables scoped by `school_id`).

4. **New school:** “New school” tab → register school + admin in cloud → save profile locally → sign in.

5. **Existing school / new device:** Login with **UNEB or EMIS** + staff username + password.

6. **Admin adds staff:** Local `users` row + `StaffCloudSync.registerStaffMember` → Supabase Auth + `school_staff` for **current** `school_id` only.

---

## Authentication flows

### Staff (Supabase configured)

- **Login:** School code + username + password → `SupabaseAuth.signIn` → role from `school_staff` → sync local user if missing → `SessionManager.login(id, user, role, schoolId, token, userId)`.
- **Signup (first device):** School name, UNEB/EMIS, admin user/password → `StaffCloudSync.registerSchoolAndAdmin`.
- **Offline fallback:** If Supabase not configured in `AppConfig`, verify against SQLite (`admin` / `admin123` seed, etc.).

### Parents

- Designed for **local** login (e.g. admission number + PIN); not required to use Supabase unless you extend it later.
- **Parent portal** menu: child marks/fees, messages, notices.

### Session

- Logout clears session and opens `LoginActivity`.
- `MainActivity` redirects to login if `SessionManager.isLoggedIn()` is false (`id > 0`).

---

## Roles and permissions

Roles: **ADMIN**, **DOS**, **TEACHER**, **BURSAR**, **PARENT**.

- **`RolePermissions`** + `SessionManager` helpers (`isAdmin`, `isDos`, `isTeacher`, `isBursar`, `isParent`) control menu visibility and fragments.
- Examples from your app:
  - Finance: Bursar / Admin only (`access_denied_finance`).
  - School profile: Admin edits; others may view only.
  - Students: Admin full; teachers/DOS may view per rules.
  - Marks lock in settings affects editing.

Drawer shows **role** and **cloud school id** when logged in via Supabase.

---

## Main features (by module)

### Dashboard
- Stats: students, teachers, attendance today, fees collected, etc.
- Entry point after login.

### School setup
- Name, UNEB, EMIS, address, motto, phone.
- Drives **cloud school id** for all sync.

### Students, teachers, classes, subjects
- CRUD (admin); soft-delete students.
- UNEB flag on students for export.

### Marks
- CAT / EOT (and related assessments), grades via **`Grading`**.
- Local save; **push/pull cloud** (`MarkCloudSync`) scoped by school.
- DOS **approve results** for class/term.
- Optional **marks locked** setting.

### Attendance
- Daily status per student; counts for dashboard and AI.
- Automation: optional SMS when marked absent.

### Finance (Bursar)
- Class fees per term, payments, receipts text.
- Fee debtors for reports and AI.

### Reports
- **UNEB CSV** export for registered candidates.

### Report cards & receipts
- Printable/shareable text built from DB.

### SMS
- Bulk SMS to parents (device `SEND_SMS` permission).

### Terms
- Academic terms; one **current** term for marks/fees.

### Staff users
- Admin adds staff: **local + Supabase** registration.
- List: `username · role`.

### Notices
- Audience: ALL, PARENTS, TEACHERS, STAFF.
- Local DB + **`NoticeCloudSync`** (pull, realtime listener) so all school phones see the same notices.

### Messages
- Staff/parent channels; cached in **`SchoolRepository`** (SharedPreferences), synced via **`MessageCloudSync`**.

### Library
- Books and loans (basic).

### Timetable
- Periods by class/teacher, conflicts (room/teacher), copy class, export text.

### Promotions
- Move students between classes (end of year).

### Backup
- Email/Drive backup and restore of SQLite DB.

### Settings
- Theme (light/dark/system), change password, delete student, automation toggles, lock marks.

### AI
- **Floating FAB** + bottom sheet “Quick AI”.
- **Full-screen AI Assistant** with school context (`aiContextBlock`: attendance, debtors, counts, fees).
- Online: Gemini; offline: rule-based answers from repository.

### Share app
- Install link helper for spreading APK to other school phones.

---

## UI / UX

- Green-tinted backgrounds (`#E8F5E9`), **Material cards** for sections (`CardUi`).
- **Premium list** pattern: FAB add, RecyclerView lists (students, users, etc.).
- Drawer: **animated nav cards** (emoji, title, subtitle, colors).
- Splash → login or main.

---

## Data architecture

```
UI (Fragments / Activities)
        ↓
SchoolRepository  ← messages cache, AI context, staff helpers, cloud id
        ↓
DatabaseHelper (SQLite)
        +
SupabaseHttp / SupabaseAuth / StaffCloudSync / NoticeCloudSync / MarkCloudSync / MessageCloudSync
        ↓
Supabase (per school_id)
```

- **Messages** intentionally not in SQLite — stored in repo prefs with dedup by `cloud_id`.
- **Audit log** in SQLite for admin actions.

---

## Configuration

- **`AppConfig`:** `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `DEFAULT_SCHOOL_CLOUD_ID`, `isSupabaseConfigured()`.
- **Gemini:** user-saved key in `AiPrefs` (unless you later hard-code built-in key).
- **Internet** permission required for cloud and AI.

---

## Security notes (for operators)

- Anon key in the APK is normal for Supabase client apps; tighten **RLS** in production beyond “open” dev policies.
- Staff passwords: hashed locally (`PasswordUtil`); Supabase holds auth passwords for cloud login.
- Built-in API keys in APK can be extracted — rotate keys if exposed.

---

## Typical deployment story

1. Admin installs app on first phone → **New school** → real UNEB/EMIS → admin account.
2. Same school on more phones → **Login** with same school code + staff users created under **Users**.
3. Another school → different UNEB/EMIS → separate cloud partition.
4. Notices/marks/messages sync when online; day-to-day teaching works offline.

---
