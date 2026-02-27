## DCOMC Enrollment & Registration System

Modern web-based enrollment and registration system for **Daraga Community College (DCOMC)**, built on Laravel.  
This README documents the **current progress**, **what’s implemented**, and **what is still missing** compared to the full target scope.

---

## 1. High-Level Architecture

- **Backend**: Laravel 12, single `web` guard.
- **Frontend**: Blade + TailwindCSS, Vite dev server for assets.
- **Authentication**:
  - Three portals:
    - `/` – Student login (`auth.login-student`)
    - `/dcomc-login` – DCOMC staff login (Registrar/Staff/Dean/UniFast)
    - `/admin-login` – System Admin login
  - Custom `portal_type` field in all login forms.
  - `AuthenticatedSessionController` enforces **role vs portal**:
    - Admin portal only accepts `role = admin`.
    - DCOMC portal accepts `registrar`, `staff`, `dean`, `unifast`.
    - Student portal only accepts `role = student`.

- **Roles implemented**:
  - `admin`, `registrar`, `student`, `staff`, `unifast`, `dean`

- **Core custom modules**:
  - Admin account management (`AdminAccountController`, `admin-accounts.blade.php`)
  - Registrar enrollment form builder & deployment (`RegistrarController`, `EnrollmentForm`, `FormResponse`)
  - Academic settings (school years, semesters, year levels)
  - Student profile & self-service enrollment
  - Admin-only temporary role switch (dev switch) with session-based impersonation

---

## 2. Current Feature Coverage vs Target Scope

### 2.1 Student Enrollment Module

**Implemented**

- **Student profile & base data**
  - Detailed profile flows:
    - First-time completion: `student.profile`
    - Edits: `student.profile.edit`
  - Stored on `users` table (`User` model), including:
    - Personal info (name, gender, DOB, civil status, citizenship).
    - Academic info (college, course, year level, semester, student_status).
    - Family/financial info (`monthly_income`, `num_family_members`, `dswd_household_no`).
    - Address (purok, street, barangay, municipality, province, zip).

- **Enrollment forms**
  - Registrar builds enrollment forms using:
    - `EnrollmentForm` model with JSON `questions`, `incoming_year_level`, `incoming_semester`.
    - UIs: `manual-registration.blade.php`, `form-library.blade.php`, `form-builder.blade.php`.
  - Features:
    - Save drafts.
    - Update and delete forms.
    - Deploy forms to specific **assigned year + semester**.
  - Students:
    - Dashboard shows whether an enrollment form is available for their current year/semester.
    - `student-enrollment-form.blade.php` renders dynamic questions.
    - `student.submit-enrollment` creates `FormResponse` with `pending` status.

- **Global enrollment toggle**
  - Registrar can enable/disable enrollment globally:
    - Stored via `Cache::forever('global_enrollment_active', true|false)`.
  - Student dashboard respects this and shows an appropriate message when closed.

**Missing vs scope**

- Multi-step **guided “online enrollment form”** that matches all scope fields in one process.
- **Auto-save** while filling out forms.
- Explicit fields and UI for:
  - Student type (Freshman, Regular, Transferee, Returnee, Irregular).
  - Day/Night shift selection.
  - Color-coded status (Yellow = irregular, Blue = returnee, Green = transferee).

---

### 2.2 Registration & COR Generation

**Implemented**

- **Approval workflow**
  - Admin/Registrar can approve applications:
    - `AdminAccountController::enrollApplication`
    - `RegistrarController::approveResponse`
  - On approval, student’s `year_level` and `semester` are updated to **incoming values** from the approved `EnrollmentForm`.
  - Application `FormResponse` status set to `approved` or `rejected`, with metadata (`reviewed_by`, `reviewed_at`).

- **Student application status**
  - Student dashboard:
    - **Application Status** card using the latest `FormResponse`:
      - `Not Enrolled`, `Pending`, `Rejected`, `Enrolled`.
    - **Enrollment Access** card for form availability and submission rules.

**Missing vs scope**

- No **Certificate of Registration (COR)** model or view:
  - No subject listing, room, professor, units, or printed COR PDF.
- No distinct `Scheduled` / `Completed` states beyond approval.

---

### 2.3 Block Management System

**Currently**

- `User` contains `year_level` and `semester` but no block or section field.
- All grouping is via year+semester+forms, not by distinct blocks.

**Missing vs scope**

- `Block` entity (e.g. `block_code`, course, year level, shift (day/night), capacity).
- Rules:
  - Max **50 students per block**; new block created when full.
  - Different block codes:
    - Elementary → numeric (e.g. `BEED 1`).
    - Other programs → lettered (e.g. `BSED-ENG A`).
- **Block change requests**:
  - Student-initiated, stored and approved by Registrar.
  - Requires valid reason and a corresponding replacement.
- Shifts & gender separation:
  - Day vs Night blocks.
  - Where necessary, male/female-specific blocks.

---

### 2.4 Scheduling & Room Assignment

**Currently**

- Registrar-focused configuration:
  - School Years (`SchoolYear`).
  - Academic Semesters (`AcademicSemester`).
  - Academic Year Levels (`AcademicYearLevel`).

**Missing vs scope**

- No models for:
  - Subjects, curriculum, schedules, rooms, or faculty load.
- No UI for Deans to:
  - Assign subjects to blocks.
  - Assign professors and time slots.
  - See a timetable grid.
- No automatic conflict detection for:
  - Room occupancy.
  - Professor double-booking.
  - Student overlaps across blocks.
- No enforcement of faculty type rules:
  - Permanent: Weekdays 8–5, max 24 units.
  - COS: Weekdays + weekends.
  - Part-time: Weekends only.

---

### 2.5 Faculty & Staff Management

**Currently**

- All staff-like roles are entries in `users` with a `role` string:
  - `admin`, `registrar`, `staff`, `dean`, `unifast`.
- Simple dashboards for:
  - Staff (`staff.blade.php`).
  - Dean (`dean.blade.php`).
  - UniFast (`unifast.blade.php`).

**Missing vs scope**

- No distinct **Accounting Officer** role and no specific accounting UI.
- No dedicated **Faculty** model with:
  - Type (Permanent/COS/Part-time).
  - Maximum units and schedule view.
- No faculty workload or availability pages.

---

### 2.6 Assessment & Financial Monitoring

**Currently**

- Student profile includes financial-related fields:
  - `monthly_income`, `num_family_members`, `dswd_household_no`.
  - Enough to support future family-income classification and UniFast eligibility logic.

**Missing vs scope**

- No `Assessment` or `Billing` table.
- No:
  - Accounting dashboard.
  - UniFast summary of eligible students.
  - Status flags for assessment/approval/UniFast tagging.

---

### 2.7 Role-Based Dashboards & Admin Dev Switch

**Implemented**

- **Role middleware**:
  - Alias `role` configured in `bootstrap/app.php`.
  - `RoleMiddleware` uses `User::effectiveRole()` for access checks:
    - Supports admin impersonation while still respecting route permissions.

- **Dashboards**
  - Admin:
    - `admin.blade.php` – landing.
    - `admin-accounts.blade.php` – user management.
    - `admin-student-status.blade.php` – application overview and status actions.
  - Registrar:
    - `registrar.blade.php` – landing.
    - Registration & builder pages: manual registration, form library, form builder, responses, response folder.
    - Academic settings pages (school years, semesters, year levels).
    - Student data listing: `registrar-students.blade.php`.
  - Student:
    - `student.blade.php`, `student-enrollment-form.blade.php`, `student-profile*.blade.php`.
  - Staff, Dean, UniFast:
    - Basic dashboards scaffolded, ready to be expanded.

- **Admin role switch (dev switch)**
  - Implemented via:
    - `AdminRoleSwitchController`.
    - `User::effectiveRole()`, `User::isAdmin()`.
    - Session key `role_switch` with:
      - `active`, `as_role`, `original_role`, `started_at`.
  - UI:
    - Switch dropdown in the top-right of admin header.
    - Warning banners on impersonated dashboards (Student, Registrar, Staff, Dean, UniFast).
  - Logout protection:
    - Admin cannot logout while role switch is active; must “Switch Back” first.

**Missing vs scope**

- No advanced **analytics dashboard** yet (no charts, trends, geo breakdown).

---

## 3. Progress Summary

**Done**

- Solid **authentication** with strict per-portal role checks.
- Foundational **enrollment** workflow:
  - Global on/off.
  - Form builder + deployment per year/semester.
  - Student application submission with duplicate prevention.
  - Admin/Registrar approvals that promote students to target year/semester.
- Robust **registrar tools** for forms and student listings.
- **Admin tools** for account management and application review.
- Thorough **student profile** structure consistent with real registrar data.
- **Dev switch** for admin to mirror any non-admin role without role mutation.

**Partially done / conceptual**

- Block management, scheduling, room assignment, faculty load, financial modules, and analytics are conceptually aligned with your notes but still need full models, controllers, and UIs.

---

## 4. What’s Missing (Gap List)

- **Block Management**
  - `blocks` table and model.
  - Auto-assignment algorithm (max 50 per block, new block on overflow).
  - Block coding (numeric vs lettered).
  - Shift-aware and gender-aware blocks.
  - Block change request / approval flow.

- **Scheduling & Rooms**
  - Subject catalog.
  - Dean-facing schedule builder.
  - Room inventory & automatic availability filtering.
  - Full conflict detection.

- **Faculty Load**
  - Faculty type & rules.
  - Load tracking with unit counts and overload detection.

- **Assessment & Financial**
  - Assessment records.
  - Accounting and UniFast coordinator dashboards.

- **Analytics**
  - Admin charts for:
    - Enrollment trends.
    - Enrolled vs not approved.
    - Per-program, gender, locality breakdowns.

---

## 5. Roadmap (High-Level)

1. **Stabilize login across all portals and browsers**
   - Continue refining session handling and cookie domains.
2. **Introduce Block Management**
   - New `Block` model and related migrations.
   - Admin/Registrar UI for viewing and editing blocks.
3. **Implement Scheduling & Room Assignment**
   - Curriculum + schedule + room models and UI.
   - Conflict and overload checks.
4. **Add Faculty & Assessment Modules**
   - Faculty types, workloads, and assessment data structures.
   - Accounting and UniFast dashboards.
5. **Build Analytics Dashboard**
   - Charts, filters, and reports per requirements.

All new functionality will be added **without breaking existing flows**; we will extend, not rewrite, the current behavior.
