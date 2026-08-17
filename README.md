# HOSPITAL MANAGEMENT SYSTEM USING JAVA

A complete, university-level **Hospital Management System desktop application** built with **Java + JavaFX + FXML + CSS + SQLite + JDBC + Maven**.

This is the **enhanced implementation** of the *SmartCare: An Object-Oriented Hospital Management System Using Java* proposal (OOP Sessional). The proposal's business requirements and OOP design are preserved exactly; only the delivery technology is upgraded from a console + file-handling app to a professional layered desktop application with a real database.

| Aspect | Original proposal | This implementation |
|---|---|---|
| UI | Console-based menu | JavaFX desktop app (FXML + CSS) |
| Storage | Java File Handling | SQLite database via JDBC |
| Build | Manual compilation | Maven |
| Architecture | Model + Service + FileManager | Controller → Service → Repository → Database |
| File handling role | Primary storage | Database backup/restore + report export |

---

## Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Language | Java (LTS) | 21 |
| UI | JavaFX (Controls + FXML) | 21.0.4 |
| Styling | JavaFX CSS | — |
| Database | SQLite (org.xerial sqlite-jdbc) | 3.46.1.3 |
| Database access | JDBC (`PreparedStatement`, transactions) | — |
| Build | Maven | 3.9+ |
| Tests | JUnit 5 (Jupiter) | 5.10.2 |

---

## Features

**Three roles, one login, role-based dashboards:**

- **Administrator** — patient/doctor/appointment CRUD + search, prescription & medical-record oversight, billing & payments, room allocation/release, ambulance booking/release, polymorphic reports with export, hospital info, password change, **SQLite backup & restore**.
- **Doctor** — own appointments (reschedule/cancel/complete), create/edit/delete prescriptions, add/edit/delete medical records, availability management.
- **Patient** — self-registration, find doctors, book/reschedule/cancel own appointments, view prescriptions & medical records, view bills, **make simulated payments**, book an ambulance, view rooms.

**Business rules enforced in the service layer:** appointment double-booking prevention (doctor *and* patient), past-date rejection, occupied-room / booked-ambulance guards, overpayment and negative-payment rejection, automatic bill totals, duplicate phone/email/license/username detection, password hashing (PBKDF2), role-based authorization.

---

## Getting Started

### Prerequisites
- **JDK 21** (Temurin/OpenJDK 21 LTS recommended)
- **Maven 3.9+**
- (Optional) IntelliJ IDEA — the project is IntelliJ-ready: open the folder, let Maven import, run `com.hospitalmanagement.Main`.

### Run from the command line

```bash
mvn clean javafx:run
```

### Run the test suite (50 tests — models, repositories, services, database, UI)

```bash
mvn test
```

The tests use disposable temporary SQLite files and **never touch `data/hospital.db`**. The UI test also renders every screen off-screen into `visual-pass.html` (a screenshot contact sheet for visual review).

### IntelliJ IDEA
1. `File → Open` → select the project folder (Maven project).
2. Wait for Maven import; ensure the project SDK is **JDK 21**.
3. Run `com.hospitalmanagement.Main` (or `mvn javafx:run` from the Maven panel).

### Eclipse (if needed)
1. `File → Import → Maven → Existing Maven Projects`.
2. JDK 21 + JavaFX 21 (Eclipse 2024-03+ / with the JavaFX support) — JavaFX must be on the module path or classpath per Eclipse's JavaFX setup.

---

## Building a Distributable Windows App (double-clickable .exe)

The project packages into a **standalone Windows app** with a bundled Java runtime — no JDK, Maven or JavaFX install needed on the target machine. The result is a real `HospitalManagementSystem.exe` that can be double-clicked or shared as a zip.

**One command:**

```bat
package-windows.cmd
```

What it does (steps 1–4):
1. `mvn package -DskipTests` — builds the application jar.
2. Stages `hospital-management-system-1.0.0.jar` + `sqlite-jdbc-3.46.1.3.jar`.
3. Runs JDK `jpackage --type app-image` with the JavaFX jmods on the module path (`--add-modules javafx.controls,javafx.fxml,java.sql,java.logging`), embedding the custom app icon (`src/main/resources/icons/app.ico`) and version metadata (name, version 1.0.0, vendor, description, copyright).
4. Zips the app folder.

The same icon (`src/main/resources/icons/app.png`) is also used as the **window/taskbar icon** at runtime — set in `Main.java` via `primaryStage.getIcons()`.

**Outputs:**

```
target\dist\HospitalManagementSystem\HospitalManagementSystem.exe   ← double-clickable app (~107 MB, self-contained)
target\dist\HospitalManagementSystem.zip                             ← shareable archive (~44 MB)
target\dist-msi\HospitalManagementSystem-1.0.0.msi                   ← installer (Start Menu + desktop shortcut)
```

Requirements to *build* (not to run): JDK 21 (`JAVA_HOME` or `tools\jdk-21.0.12+8`), Maven 3.9+ (`PATH` or `tools\apache-maven-3.9.9`), JavaFX 21.0.4 jmods (`tools\javafx-jmods-21.0.4` or the `JAVA_FX_JMODS` env var), and — only for the MSI step — WiX 3.x (`tools\wix314` or the `WIX` env var). The script auto-detects the portable toolchain in `tools\` and skips the MSI gracefully if WiX is absent.

The **MSI installer** installs to `Program Files\HospitalManagementSystem` (machine-wide), adds a **Start Menu** entry under *Hospital Management System* and a **desktop shortcut**, and registers the app for add/remove programs. Double-click it and accept the UAC prompt.

The portable exe/zip needs no installation at all: on first launch the app creates `data\hospital.db` next to the exe and seeds the demo credentials — copy the folder anywhere and run.

---

## Demo Credentials (seeded on first run)

| Role | Username | Password |
|---|---|---|
| Admin | `admin` | `admin123` |
| Doctor | `doctor@hospital.com` | `doctor123` |
| Patient | `patient@hospital.com` | `patient123` |

Passwords are stored as **PBKDF2-HMAC-SHA256 hashes** (random 16-byte salt, 65 536 iterations) — never in plaintext. Demo data also includes 4 doctors, 4 patients, 6 rooms, 3 ambulances, appointments, a medical record with a linked prescription, and an unpaid + a paid bill with payment, so every screen is alive on first launch.

---

## Project Structure

```
src/main/java/com/hospitalmanagement/
├── Main.java                      ← JavaFX entry point
├── model/      (13)               ← Person (abstract) → Patient/Doctor/Admin, User,
│                                    Appointment, Prescription, MedicalRecord, Bill,
│                                    Payment, Room, Ambulance, Hospital
├── repository/ (12)               ← JDBC DAOs — PreparedStatements, row mapping, search
├── service/    (13)               ← business rules + transactions
│                                    (incl. PaymentService interface + PaymentServiceImpl)
├── report/     (10)               ← ReportGenerator interface + 8 polymorphic reports
├── database/                       ← DatabaseManager + SeedData
├── controller/ (17)               ← thin JavaFX controllers (input → service → UI)
├── exception/  (10)               ← custom exceptions
└── util/       (6)                ← Validator, PasswordUtil, SessionManager,
                                     AlertUtil, NavigationUtil, TimeSlotUtil

src/main/resources/
├── fxml/      (17 screens)        ← Login, PatientRegistration, ChangePassword,
│                                    3 dashboards, 13 management screens
├── css/application.css            ← full design system
└── database/schema.sql            ← 12 tables (PK/FK/UNIQUE/CHECK, FK enforcement)

data/hospital.db                   ← created + seeded automatically on first run
```

**Layering rule enforced throughout:** `FXML → Controller → Service → Repository → SQLite`. Controllers contain no SQL, no business rules, and no calculations.

---

## Documentation

| Document | Contents |
|---|---|
| [REQUIREMENT_ANALYSIS.md](REQUIREMENT_ANALYSIS.md) | Step 1 analysis of the SmartCare proposal |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Step 2 final architecture — classes, layers, navigation, DB design |
| [REQUIREMENT_CHECKLIST.md](REQUIREMENT_CHECKLIST.md) | Final requirement checklist (proposal → implementation) |
| [VIVA.md](VIVA.md) | OOP concept explanations and viva guide |

---

## Security & Data Safety

- PBKDF2 password hashing; credentials never shown in UI or logs.
- `PreparedStatement` everywhere — no SQL injection; user input never concatenated into SQL.
- `PRAGMA foreign_keys = ON` + `busy_timeout`; transactions for multi-step operations.
- Backup uses SQLite `VACUUM INTO` (safe snapshot); restore requires confirmation, closes the connection, copies the file, and re-initializes safely.
- Friendly error dialogs — raw stack traces never reach the UI.

*Simulated payments only (Cash / Card / Mobile Banking) — this is a university project, no real gateway is integrated.*
