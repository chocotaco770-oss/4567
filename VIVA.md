# VIVA / OOP EXPLANATION GUIDE

**HOSPITAL MANAGEMENT SYSTEM USING JAVA** — how the OOP concepts from the SmartCare proposal are actually implemented, with concrete code references. Use this to prepare for the viva.

---

## 1. Class and Object

Every real-world concept in the system is a class: `Patient`, `Doctor`, `Admin`, `Appointment`, `Prescription`, `MedicalRecord`, `Bill`, `Payment`, `Room`, `Ambulance`, `Hospital`, `User`, and the abstract base `Person` (package `com.hospitalmanagement.model`).

Objects are created with constructors and exchanged between layers:

```java
Patient patient = new Patient("Karim Uddin", 41, Person.Gender.MALE, "01720000002",
        "B+", "Diabetes", "Dhaka", "01720000888", "karim@hospital.com");
int id = patientService.registerPatient(patient, "karim", "pass1234");
```

## 2. Encapsulation

- All model fields are `private` with getters/setters.
- Constructors route through setters so validation happens at creation time (`Prescription.setMedicineList`, `Room.setRoomNumber`, `Ambulance.setCharge`).
- State-changing methods are **guarded** — an occupied room cannot be allocated twice:

```java
public void allocate(int patientId) {
    if (!isAvailable()) {
        throw new IllegalStateException("Room " + roomNumber + " is not available.");
    }
    this.availability = Availability.OCCUPIED;
    this.patientId = patientId;
}
```

- `Bill` keeps the total *derived*: `computeTotal()`/`recalculateTotal()` are the single source of truth for bill math — nobody can store an inconsistent total.

## 3. Abstraction

`Person` is `abstract` and declares abstract behavior that every subclass must define:

```java
public abstract String displayInformation();
public abstract void updateProfile(...);
```

Abstraction also appears at the service level: controllers depend on `PaymentService` (the interface) and `ReportGenerator` (the interface), never on concrete internals.

## 4. Inheritance

The `Person` hierarchy is the proposal's core inheritance chain:

```
Person (abstract)
├── Patient   (patientId, bloodGroup, disease, emergencyContact, email)
├── Doctor    (doctorId, specialization, experience, consultationFee, licenseNumber, availability)
└── Admin     (adminId, username, passwordHash)
```

Shared attributes (name, age, gender, phone) and behavior live once in `Person`; each subclass adds its own fields and overrides behavior.

## 5. Polymorphism

Two real (non-forced) uses:

1. **Runtime polymorphism over the Person hierarchy** — a `List<Person>` holds patients, doctors and admins; calling `displayInformation()` dispatches to the right implementation:
```java
List<Person> people = List.of(patient, doctor, admin);
for (Person p : people) System.out.println(p.displayInformation());
```

2. **Interface polymorphism** — the same interface reference calls different implementations:
```java
PaymentService paymentService = new PaymentServiceImpl(db);   // abstraction + polymorphism
paymentService.processPayment(new Payment(billId, patientId, 500, Payment.Method.CASH));

ReportGenerator report = reportService.getReport(ReportType.PATIENT); // → PatientReport
report = reportService.getReport(ReportType.BILLING);                 // → BillingReport
String text = report.generateReport();                                // same call, different output
```

## 6. Interface

The proposal requires `PaymentService`; it is implemented as an interface with a concrete implementation:

```
PaymentService (interface)  →  processPayment(...), getAllPayments(), getPaidAmount(...)
       ↓ implements
PaymentServiceImpl
```

Application code (controllers, tests) references `PaymentService`, so swapping the implementation never touches callers. `ReportGenerator` is a second interface used the same way.

## 7. Association

Meaningful associations between entities, modeled with foreign keys in SQLite:

```
Patient 1 ──── * Appointment * ──── 1 Doctor
Doctor 1 ──── * Prescription
Patient 1 ──── * Prescription
Patient 1 ──── * MedicalRecord
Patient 1 ──── * Bill, Bill 1 ──── * Payment
```

`AppointmentService` coordinates the Patient–Appointment–Doctor triangle: it loads the patient, loads the doctor, checks **both** sides for conflicts, then books.

## 8. Aggregation

`Hospital`/`HospitalService` logically contain collections of patients, doctors, rooms and ambulances, but those entities can exist independently (a doctor exists even if the hospital object is destroyed). In the database this is the *has-many* relationship via foreign keys; the `HospitalService` aggregates several repositories to produce dashboard statistics.

## 9. Composition

The strongest ownership relationship in the design: **`MedicalRecord` owns `Prescription`**. A prescription without its record has no meaning, and deleting the record deletes its prescriptions — enforced by `ON DELETE CASCADE` in the schema and verified by a test:

```java
medicalRecordService.deleteRecord(recordId);   // linked prescriptions disappear
```

## 10. Collections

`List`/`ArrayList`/`Map`/`ObservableList` are used for query results, name-lookup maps (`Map<Integer,String>` for patient/doctor names in tables), and JavaFX `TableView` data. SQLite remains the *primary* storage — collections are never used to bypass the database.

## 11. Method Overloading

```java
// PatientService
public List<Patient> searchPatient(String name)
public List<Patient> searchPatient(String name, String phone)
public List<Patient> searchPatientByPhone(String phone)

// DoctorService
public List<Doctor> searchDoctor(String name)
public List<Doctor> searchDoctor(String name, String specialization)
```

## 12. Method Overriding

`Patient`, `Doctor` and `Admin` each `@Override` the abstract `updateProfile(...)` and `displayInformation()` from `Person`, adding role-specific fields. Overriding is also used in the report classes through the interface's default method.

---

## Architecture (explain in viva)

```
FXML (view)  →  Controller (input, alerts, navigation)  →  Service (business rules)
     →  Repository (JDBC, PreparedStatement)  →  DatabaseManager  →  SQLite
```

- **Controllers never contain SQL or business logic.** Booking an appointment = `appointmentService.bookAppointment(...)`; the conflict checks, existence checks and transaction live in `AppointmentService`.
- **Repositories** own all SQL with `PreparedStatement` (no string concatenation → no SQL injection), map rows to models, and throw `DatabaseException` instead of leaking `SQLException` to the UI.
- **Services** enforce rules and use `DatabaseManager.beginTransaction()/commit()/rollback()` for multi-step writes (e.g., patient registration = patient row + login user row in one transaction).

## Business Rules (know these)

1. **Appointments**: doctor and patient cannot have two active (Booked) appointments at the same date/time; past dates are rejected; cancelling frees the slot for re-booking. The DB enforces this with a *partial unique index on active bookings* as a second line of defense.
2. **Rooms**: an occupied room can never be allocated again; an available room can never be released.
3. **Ambulances**: a booked ambulance can never be booked again; an available one can never be released.
4. **Payments**: amount must be > 0 and cannot exceed the outstanding balance; the bill status auto-transitions UNPAID → PARTIALLY_PAID → PAID.
5. **Billing**: total = consultation + room + medicine + ambulance + other, always computed by `Bill.computeTotal()`.
6. **Duplicates**: patient phone/email, doctor license/email, and login usernames are unique.

## Security

- Passwords hashed with **PBKDF2-HMAC-SHA256** (random salt, 65 536 iterations) — `PasswordUtil`.
- Role checks via `SessionManager.isAdmin()/isDoctor()/isPatient()` gate every screen's actions.
- `PreparedStatement` everywhere; `PRAGMA foreign_keys = ON`; transactions for atomicity.

## Database Design

12 tables: `users`, `patients`, `doctors`, `admins`, `appointments`, `prescriptions`, `medical_records`, `bills`, `payments`, `rooms`, `ambulances`, `hospital_info` — primary keys, foreign keys, UNIQUE/NOT NULL/CHECK constraints, and a partial unique index on active appointments.

---

## Likely Viva Questions

**Q: Why is `Person` abstract?**
It captures common attributes/behavior of all staff and patients, cannot meaningfully exist on its own, and forces every subclass to implement `displayInformation()` — that is abstraction.

**Q: Where is polymorphism used?**
Two places: (1) treating patients/doctors/admins as `Person` and calling the overridden `displayInformation()`, and (2) calling `processPayment()` / `generateReport()` through the `PaymentService` / `ReportGenerator` interfaces so one call works for every implementation.

**Q: Why use an interface for payments instead of a normal class?**
So the UI depends on a contract, not an implementation — `PaymentServiceImpl` can be swapped (e.g., a real gateway later) without changing any caller. This is programming to an interface.

**Q: How do you prevent double-booking?**
`AppointmentService` queries the repository for an active appointment with the same doctor (and separately the same patient) at the same date/time and throws `AppointmentConflictException`. The database additionally enforces it with a partial unique index so it cannot happen even if the service is bypassed.

**Q: Difference between aggregation and composition here?**
Hospital–doctor is aggregation (doctors exist without the hospital). MedicalRecord–prescription is composition: a prescription belongs to its record and is deleted with it (ON DELETE CASCADE).

**Q: How do you calculate a bill?**
The controller passes five charge components to `BillingService.generateBill(...)`, which validates them and lets `Bill.recalculateTotal()` compute the total — the model is the single source of truth for the math.

**Q: What happens if the database operation fails mid-way?**
Multi-step operations run inside `beginTransaction()/commit()/rollback()`; a failure rolls back everything and surfaces as a friendly `DatabaseException` dialog — never a raw stack trace.
