# Smart Parking Management & Allocation System

**Database Management Systems (ETCCMS304) – Capstone Project 6 – B.Tech CSE / Cyber Security / Data Science, Semester III**

**Submitted by (Section D)**

| Name | Roll No. | Section |
| --- | --- | --- |
| Anish Kumar Jha | 2501010273 | D |
| Aryan Kumar | 2501010043 | D |
| Bhowmik Vij | 2501010195 | D |

**Live demo:** [smart-parking-dbms-capstone-production.up.railway.app](https://smart-parking-dbms-capstone-production.up.railway.app) · **Code:** [github.com/Svn8efx/smart-parking-dbms-capstone](https://github.com/Svn8efx/smart-parking-dbms-capstone)

Stack: MySQL 8.0.31+ (core OLTP), MongoDB (event log), a Python Flask web application (frontend and backend) hosted on Railway, star schema + ETL (analytics). Source files are in the `smart_parking/` folder delivered with this report.

## 1. Problem, scope and objectives

Universities, business parks and residential complexes have a fixed number of parking slots. Static allocation leaves some zones empty while others overflow, and spreadsheet or paper booking produces double allocations, disputes and unpaid penalties. This project builds a database that manages users, vehicles, zones, slots, reservations, dynamic allocation, gate entry/exit, visitor passes, violations and penalty payments, and that reports on capacity and peak demand.

**Objectives:** (1) maintain users and vehicles; (2) reserve and dynamically allocate slots; (3) make duplicate allocation impossible, even under concurrency; (4) record entry and exit and compute fees; (5) manage visitor passes; (6) detect, fine and collect violations; (7) analyse utilisation, peak hours and repeat violators.

**Stakeholders:** students/employees/residents (book and cancel), visitors (pass holders), security staff (gate events, violations), parking administrators (full control), facility management (analytics).

## 2. Requirements and business rules

| ID | Business rule | Enforced by |
| --- | --- | --- |
| R1 | A slot cannot have two overlapping bookings (reservation or visitor pass). Intervals overlap when start1 < end2 and end1 > start2, so 10:00–12:00 and 12:00–14:00 are allowed, 10:00–12:00 and 11:00–13:00 are not. | `fn_is_slot_free`, slot row lock in procedures, triggers `trg_reservation_bi/bu`, `trg_pass_bi` |
| R2 | Only the vehicle owner books or cancels; vehicle type must fit the slot type; suspended users cannot book; a vehicle cannot hold two overlapping bookings. | `sp_reserve_slot`, `sp_cancel_reservation` |
| R3 | Allocation, cancellation, entry, exit and payment are atomic. | Transactions with exit handler + ROLLBACK |
| R4 | Entry is allowed only inside the reservation window (15 minutes early grace) or with a valid pass. | `sp_record_entry` |
| R5 | Leaving more than 15 minutes after the reservation end creates an OVERSTAY violation automatically. | `trg_exit_ai` |
| R6 | Fee = ceil(hours) × zone rate; first 15 minutes free. | `fn_parking_fee`, `trg_exit_bi` |
| R7 | A violation becomes PAID once successful payments cover the fine; overpayment and double payment are rejected. | `sp_pay_violation`, `trg_payment_ai` |
| R8 | Important changes leave an audit record. | audit triggers, `audit_log` |
| R9 | A slot, vehicle or pass cannot have two open entries (inside without exit). | `trg_entry_bi` |

Non-functional: ACID correctness for bookings and payments, least-privilege access, response time for the availability and overlap queries (indexes), auditability.

## 3. Conceptual design (ER / EER)

The full diagram is in `er_diagram.mermaid` (renders as a Mermaid ER diagram). Cardinality and participation:

| Relationship | Cardinality | Participation |
| --- | --- | --- |
| DEPARTMENT – APP\_USER | 1 : N | user side partial (residents have no department) |
| APP\_USER – VEHICLE (owns) | 1 : N | vehicle total, user partial |
| VEHICLE\_TYPE – VEHICLE / PARKING\_SLOT | 1 : N | both child sides total |
| PARKING\_ZONE – PARKING\_SLOT | 1 : N | slot total (a slot cannot exist without a zone) |
| VEHICLE – RESERVATION – PARKING\_SLOT | M : N resolved by RESERVATION (with attributes start, end, status) | reservation total on both |
| APP\_USER – VISITOR – VISITOR\_PASS – PARKING\_SLOT | 1 : N, 1 : N, N : 1 | pass total on visitor and slot |
| ENTRY\_LOG – EXIT\_LOG | 1 : 1 | exit total (but an open entry has no exit yet, so entry side partial) |
| RESERVATION – ENTRY\_LOG | 1 : N (0..N) | entry side partial (walk-ins have no reservation) |
| VEHICLE – VIOLATION – VIOLATION\_TYPE | 1 : N, 1 : N | violation total |
| VIOLATION – PAYMENT | 1 : N (part payments) | payment total |
| SECURITY\_STAFF – VIOLATION | 1 : N | partial (auto-generated violations have no staff) |

**EER notes.** (a) *Specialisation:* APP\_USER is a generalisation of STUDENT / EMPLOYEE / RESIDENT; because the sub-types share all attributes, it is implemented as a single table with the discriminator `user_type` (total, disjoint specialisation). (b) *Hierarchy:* ZONE → SLOT models the parking hierarchy. (c) *Aggregation:* an entry event belongs either to a RESERVATION-backed vehicle, a walk-in vehicle or a VISITOR\_PASS; the `CHECK ((vehicle_id IS NULL)+(pass_id IS NULL)=1)` encodes the exclusive-or.

## 4. Relational schema (16 tables)

| Table | Primary key | Foreign keys | Other keys / key constraints |
| --- | --- | --- | --- |
| department | dept\_id | – | dept\_name UNIQUE |
| app\_user | user\_id | dept\_id → department | email UNIQUE, phone CHECK 10 digits |
| vehicle\_type | vtype\_id | – | type\_name UNIQUE, wheels CHECK |
| vehicle | vehicle\_id | owner\_id → app\_user (RESTRICT), vtype\_id | reg\_no UNIQUE |
| parking\_zone | zone\_id | – | zone\_name UNIQUE, rate ≥ 0 |
| parking\_slot | slot\_id | zone\_id (CASCADE), vtype\_id | UNIQUE(zone\_id, slot\_code) |
| security\_staff | staff\_id | – | – |
| reservation | res\_id | vehicle\_id, slot\_id | CHECK end > start, CHECK ≤ 12 h |
| visitor | visitor\_id | host\_user\_id → app\_user | – |
| visitor\_pass | pass\_id | visitor\_id (CASCADE), slot\_id | pass\_code UNIQUE, CHECK valid\_to > valid\_from |
| entry\_log | entry\_id | vehicle\_id, pass\_id, reservation\_id, slot\_id | CHECK exactly one of vehicle/pass |
| exit\_log | exit\_id | entry\_id → entry\_log | entry\_id UNIQUE (1:1) |
| violation\_type | vt\_id | – | type\_code UNIQUE, fine > 0 |
| violation | violation\_id | vehicle\_id, slot\_id, vt\_id, recorded\_by | – |
| payment | payment\_id | violation\_id | amount > 0 |
| audit\_log | audit\_id | – | – |

Seed data (`03_seed_data.sql`): 60 users, 70 vehicles, 60 slots in 5 zones, 136 reservations, 15 visitors and passes, 106 entries, 100 exits, 60+ violations (many generated by the overstay trigger), payments and several hundred audit rows – well above the 200-record minimum.

## 5. Functional dependencies and normalisation

**Starting point (unnormalised log sheet).** PARKING\_LOG(res\_id, reg\_no, owner\_id, owner\_name, owner\_phone, user\_type, dept\_name, slot\_code, zone\_name, zone\_category, hourly\_rate, start\_time, end\_time, entry\_time, exit\_time, fee, violation\_code, fine\_amount).

**Functional dependencies derived from the rules**

- res\_id → reg\_no, slot\_id, start\_time, end\_time, status
- reg\_no → vehicle\_id, owner\_id, vtype\_id; vehicle\_id → reg\_no (both are candidate keys of vehicle)
- owner\_id → full\_name, email, phone, user\_type, dept\_id; email → owner\_id
- dept\_id → dept\_name
- slot\_id → zone\_id, slot\_code, vtype\_id, status; (zone\_id, slot\_code) → slot\_id
- zone\_id → zone\_name, zone\_category, hourly\_rate
- vt\_id → type\_code, description, fine\_amount
- entry\_id → vehicle\_id / pass\_id, reservation\_id, slot\_id, entry\_time; entry\_id → exit\_time, fee (exit is 1:1)
- violation\_id → vehicle\_id, vt\_id, violation\_time, status

**1NF.** Every attribute is atomic: no repeating groups such as several vehicles per user row or several payments in one violation row (they became separate tables). A primary key exists for each relation.

**2NF.** In relations with composite keys, partial dependencies were removed. Example: with (res\_id, violation\_code) as a key, fine\_amount depends only on violation\_code, so VIOLATION\_TYPE was split off; payments depend on the whole of (violation, payment) so they have their own table.

**3NF.** Transitive dependencies were removed: res\_id → reg\_no → owner\_id → owner\_name would repeat owner data in every reservation, so RESERVATION keeps only vehicle\_id and the owner is reached through VEHICLE → APP\_USER. Likewise slot → zone → hourly\_rate lives in PARKING\_ZONE and owner → department → dept\_name in DEPARTMENT. For every FD X → A in the final schema, X is a superkey or A is prime.

**BCNF discussion.** All determinants listed above are candidate keys of their relations (for parking\_slot both slot\_id and (zone\_id, slot\_code); for app\_user both user\_id and email; for vehicle both vehicle\_id and reg\_no), so each of these relations is in BCNF. RESERVATION is in BCNF with key res\_id. (slot\_id, start\_time) looks like a second key but is not one: cancelled bookings may share it, so only the overlap *rule* (a dynamic constraint) is enforced procedurally, not a key.

**Deliberate, documented design decisions**

1. `reservation.user_id` was *not* stored: the booker is always the vehicle owner, so it would be a transitive dependency.
2. `violation` does not copy `fine_amount`; the payment trigger and views join `violation_type`. (Trade-off: changing a fine later changes the amount due for open violations, which is the desired policy.)
3. `exit_log.fee_charged` is stored: it is a point-in-time snapshot (the zone rate may change later), so it is history, not a redundant derivable value.
4. `entry_log.slot_id` is kept because walk-ins have no reservation; when a reservation is given, the trigger checks that the slots match (a conditional dependency a plain FD cannot express).
5. The star schema in section 10 is intentionally denormalised; analytical convenience justifies it there but not in the OLTP core.

## 6. SQL implementation (CO2)

| File | Content |
| --- | --- |
| `01_schema.sql` | DDL, constraints, referential actions |
| `02_functions_procedures_triggers.sql` | 2 functions, 8 procedures (1 cursor), 9 triggers |
| `03_seed_data.sql` | DML, realistic data |
| `04_views_indexes.sql` | 5 views, 6 indexes, EXPLAIN before/after |
| `05_queries.sql` | 21 complex queries |
| `06_transactions_concurrency.sql` | transaction scenarios and two-session scripts |
| `07_security_rbac.sql` | roles, grants (DCL), SQL-injection demonstration |
| `08_star_schema_etl.sql` | star schema and ETL procedure |
| `09_test_cases.sql` | 31 tests + 4 invariants |

**Required queries and where they are:** available slots by zone (Q1, view), average parking duration (Q3), rank zones by utilisation with RANK (Q4, CTE), peak hours with DENSE\_RANK (Q5), repeated violators with a correlated subquery (Q7) and a view (Q8), users who reserve but do not enter with EXCEPT (Q10) and an anti-join (Q11), hourly demand with a recursive CTE (Q6), current-availability view (`v_current_availability`). Also included: INTERSECT (Q12), UNION (Q13), running totals with LAG (Q14), window over aggregate (Q15), ROW\_NUMBER with PARTITION BY (Q16), self join (Q17), cross join (Q18), outer joins (Q11, Q19).

**DCL/TCL:** CREATE ROLE/USER, GRANT, SET DEFAULT ROLE (file 07); START TRANSACTION, COMMIT, ROLLBACK, SAVEPOINT, ROLLBACK TO SAVEPOINT (files 02, 06).

## 7. Advanced DBMS features – each tied to a rule (CO3)

**Functions.** `fn_is_slot_free` (R1: single definition of conflict used by procedures and triggers), `fn_parking_fee` (R6).

**Procedures.** `sp_reserve_slot` (specific slot), `sp_allocate_slot` (dynamic allocation with `FOR UPDATE SKIP LOCKED`), `sp_cancel_reservation`, `sp_issue_visitor_pass`, `sp_record_entry`, `sp_record_exit`, `sp_pay_violation` (SAVEPOINT after the audited attempt), `sp_mark_no_shows` (explicit cursor loop for a batch job).

**Triggers.** `trg_reservation_bi` and `trg_reservation_bu` (last line of defence against double booking, also for direct INSERT/UPDATE that bypasses the procedures), `trg_reservation_ai/au` (audit), `trg_pass_bi` (visitor zone only, no overlap), `trg_entry_bi` (no two open entries), `trg_exit_bi` (validates exit and computes fee), `trg_exit_ai` (completes the reservation and creates the OVERSTAY violation), `trg_payment_ai` (marks the violation PAID, audits).

**Views.** `v_current_availability` (AVAILABLE / RESERVED / OCCUPIED / MAINTENANCE per slot), `v_zone_occupancy_now`, `v_open_violations`, `v_repeat_violators`, `v_reservation_details` (self-service surface for normal users).

**Indexes and justification** (verified with EXPLAIN in file 04). InnoDB already indexes foreign keys, so only indexes that match real predicates were added:

| Index | Query it serves |
| --- | --- |
| `idx_res_slot_time(slot_id, status, start_time, end_time)` | the overlap test run on every booking; equality columns first, range columns last |
| `idx_res_start(start_time)` | time-range reports and hourly demand; EXPLAIN changes from full scan to range scan |
| `idx_entry_slot_time(slot_id, entry_time)` | who is inside now, per-slot history |
| `idx_viol_vehicle_time(vehicle_id, violation_time)` | repeat-violator lookups |
| `idx_viol_status(status)` | open-violation dashboard |
| `idx_pass_slot_time(slot_id, status, valid_from, valid_to)` | visitor overlap test |

Cost: each index slows INSERT/UPDATE slightly and uses space, which is why no index was added "just in case".

## 8. Transactions and concurrency (CO3)

**ACID in practice.** *Atomicity:* each procedure runs `START TRANSACTION`; an `EXIT HANDLER FOR SQLEXCEPTION` issues `ROLLBACK` and returns a readable error through an OUT parameter. *Consistency:* constraints, triggers and the invariant queries INV1–INV4 in file 09. *Isolation:* described below. *Durability:* InnoDB redo log on COMMIT.

**The problem.** Two sessions both run "is the slot free? → insert". Without protection both see "free" and both insert, giving a double allocation (a write-skew / lost-update anomaly). REPEATABLE READ alone does not stop it because the two sessions read, but neither writes the row the other read.

**The solution – lock the contended resource first.** Every write procedure begins with `SELECT … FOR UPDATE` on the `parking_slot` row (or the `violation` / `reservation` row). The second session waits on that lock; when the first commits, the second runs its overlap check at READ COMMITTED, sees the committed booking and is rejected with "Slot already booked for an overlapping interval". Only one slot row is locked, so unrelated slots proceed in parallel. For dynamic allocation, `FOR UPDATE SKIP LOCKED` hands the next free slot to the second user instead of making them wait.

| Scenario (file 06 Part B) | Expected result |
| --- | --- |
| 1. Slot 18, A books 10–12 holding the lock; B asks 11–13 | B waits, then FAILED overlap; B asks 12–14 → SUCCESS |
| 2. Dynamic allocation, zone A, A and B simultaneously | A gets slot 1, B gets slot 2; no duplicates (verification query returns 0 rows) |
| 3. Zone full / wrong type | FAILED with message; ROLLBACK leaves no row |
| 4. Two payments for one fine | second waits, then FAILED already settled; exactly one payment row |
| 5. Deadlock (opposite lock order) | MySQL error 1213, one victim rolled back automatically |
| 6. SERIALIZABLE comparison | safe but locks more rows than the lock-first design |

**Deadlocks.** A deadlock needs two sessions holding one lock each and waiting for the other's. Prevention rules used: lock a single resource first; if several slots are needed, lock in ascending `slot_id` order; keep transactions short; and retry on error 1213/1205 (implemented in `with_retry` in the Python app). The three-step transaction demonstrations (COMMIT, ROLLBACK, SAVEPOINT) are in file 06 Part A, T1–T5, including the headline case 10:00–12:00 booked, 11:00–13:00 rejected, 12:00–14:00 accepted.

## 9. Security (CO3)

| Role | May do | May not |
| --- | --- | --- |
| parking\_user | SELECT availability and own-reservation views; EXECUTE reserve / allocate / cancel | read app\_user, payment; DELETE anything; pay fines |
| parking\_security | record entry/exit, issue passes, record violations, take payments, read open-violation views | modify zones or rates, DELETE violations |
| parking\_analyst | SELECT occupancy and repeat-violator views | read personal tables |
| parking\_admin | everything in the schema | – |

Procedures and triggers run with definer rights, so a user with only EXECUTE can create a reservation through the procedure but cannot touch the table directly. Negative tests are listed in file 07 (each must fail with error 1142 or 1370).

**SQL-injection prevention.** File 07 shows a concatenated query `… WHERE reg_no = 'DL01CA1007' OR '1'='1'` returning every vehicle, and the same input through a prepared statement (`?` placeholder) returning no rows because the whole string is bound as data. The Python app uses `%s` placeholders and `CALL` with typed parameters; the account it uses cannot read personal tables, so even a successful injection has a small blast radius (defence in depth).

## 10. CO4 extensions

**MongoDB event log** (`mongodb/gate_events.js`). Gate cameras, ANPR units, barriers and slot sensors emit events with different fields (confidence, image URL, battery level, error code). Documents store them naturally; the script demonstrates insertMany, find with nested fields, updateOne/updateMany with `$set`/`$push`, deleteMany, indexes (including a TTL index that purges events after 90 days), three aggregation pipelines (events per hour and type, dwell time per zone, most active vehicles) and `explain`.

| Criterion | MySQL (core) | MongoDB (events) |
| --- | --- | --- |
| Data | reservations, payments, violations – fixed structure, relationships | sensor/gate events – variable structure, high volume |
| Guarantees needed | ACID, foreign keys, row locks | append-mostly, eventual analysis |
| Schema change | costly ALTER | add a field to new documents |
| Join | native | application-level via `vehicle_reg` (see `history` in the app) |

**Star schema and ETL** (`08_star_schema_etl.sql`). Fact table `fact_parking_session` (grain: one completed session) with dimensions date, hour, zone and user type. `sp_etl_load_star` extracts from the OLTP tables, transforms (user-type key, VISITOR for pass holders, overstay flag, day part) and loads inside one transaction. OLAP queries use `GROUP BY … WITH ROLLUP` for zone/day-part subtotals, overstay rate by user type and weekday vs weekend duration.

**Application integration** (`app/parking_app.py`). Parameterised calls to the procedures, availability from the view, a threaded concurrency test that asserts no double allocation, the injection demo, and an SQL + MongoDB application-level join. **Cloud option:** the same scripts run unchanged on a managed MySQL instance (e.g. Amazon RDS, Cloud SQL) and the Mongo script on MongoDB Atlas by changing the connection string.

## 10A. Web application and live deployment

The database is also served as a working website at the live link on the first page, so a user can book, auto-allocate, cancel and view slots from a browser.

| Layer | Technology | Role |
| --- | --- | --- |
| Frontend | HTML, CSS, JavaScript (`webapp/templates/index.html`) | booking form, reservations table, live slot map grouped by zone |
| Backend | Python Flask with gunicorn (`webapp/app.py`) | REST API: `/api/availability`, `/api/reservations`, `/api/reserve`, `/api/allocate`, `/api/cancel` |
| Database | MySQL 8.4 on Railway | all tables, views, procedures and triggers |
| Hosting | Railway: web service and MySQL in one project | public HTTPS link |

**Design rule.** The backend holds no booking logic. Each endpoint calls a stored procedure (`sp_reserve_slot`, `sp_allocate_slot`, `sp_cancel_reservation`) or reads a view, so the rules of sections 7 and 8 apply to every request and cannot be bypassed from the page. All queries use bound parameters.

**Deployment steps.** (1) Create a MySQL service on Railway and enable public access. (2) Load files 01 to 04 with MySQL Workbench. (3) Deploy the `webapp` folder from GitHub as a second service with root directory `webapp` and start command `gunicorn app:app --bind 0.0.0.0:$PORT`. (4) Set `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` and `DB_NAME` as variables that reference the MySQL service. (5) Generate a public domain.

**Verified on the live site.** A booking returns `SUCCESS`; the same vehicle booking an overlapping time returns `FAILED: Vehicle already has an overlapping reservation`; the availability map shows all 60 slots read from the cloud database.

**Limits.** The cloud copy connects as the database owner, so the role-based security of section 9 is demonstrated on the local database. There is no login screen; a user ID is typed in. Hosting is a free trial, so the link may be taken down after evaluation.

## 11. Testing

`09_test_cases.sql` prints each test id with its expected outcome.

| Category | Tests | What is proved |
| --- | --- | --- |
| Positive | TC01–TC04, TC13, TC15 | normal booking, back-to-back booking, dynamic allocation, walk-in entry/exit, cancellation, visitor pass |
| Negative / business rule | TC05–TC12, TC14, TC25 | overlap, past or inverted time, wrong owner, wrong vehicle type, maintenance slot, suspended user, foreign cancellation, pass outside visitor zone, occupied slot |
| Integrity | TC16–TC24 | UNIQUE, CHECK, FOREIGN KEY, RESTRICT, trigger overlap on direct INSERT, 1:1 exit, XOR check on entry |
| Transaction | TC26–TC31 | overpayment rejected, exact payment flips status via trigger, double payment rejected, rollback leaves no row, audit rows, cursor batch |
| Invariants | INV1–INV4 | no overlapping confirmed bookings anywhere; every late exit has an OVERSTAY violation; PAID violations fully covered; at most one open entry per slot |
| Concurrency | file 06 Part B scenarios 1–5; `parking_app.py --concurrency` | no double allocation, blocking and ordering, single payment, deadlock handling |
| Performance | file 04 EXPLAIN before/after | index use confirmed |

## 12. How to run

1. Install MySQL **8.0.31 or later** (needed for INTERSECT/EXCEPT; `FOR UPDATE SKIP LOCKED` needs 8.0.1+). Optional: MongoDB, Python 3 with `mysql-connector-python` and `pymongo`.
2. `./run_all.sh root` executes files 01–05, 07–09 in the correct order (02 before 03 because the seed relies on the triggers).
3. Open two terminals and follow file 06 Part B for the live two-session demo, or run `python app/parking_app.py --concurrency`.
4. `mongosh mongodb/gate_events.js` for the NoSQL part.

**Website.** From the `webapp` folder run `pip install -r requirements.txt`, then `python app.py`, and open http://127.0.0.1:5000. The deployed version is at the live link on the first page.

## 13. Limitations and future work

- Reservation overlap is enforced by locking plus triggers because MySQL has no exclusion constraint (PostgreSQL could use `EXCLUDE USING gist` on a range type).
- Fees are simple hourly pricing; surge pricing or monthly passes would need a rate table with validity dates.
- ANPR images are not stored in the database, only their metadata in MongoDB.
- The demo delay parameters (`p_demo_delay`) exist only to make locking visible and must be 0 in production.

## 14. Viva preparation

1. *Why lock the slot row instead of using SERIALIZABLE?* Same safety, but only one row is locked; SERIALIZABLE takes next-key locks over ranges and reduces concurrency.
2. *Why both a procedure check and a trigger?* The procedure gives readable errors and takes the lock; the trigger protects against any writer that bypasses the procedure.
3. *Why is `user_id` missing from `reservation`?* It would be a transitive dependency through the vehicle owner.
4. *Why MongoDB for events but not for reservations?* Reservations need ACID and relationships; events are flexible, high-volume and append-mostly.
5. *How can a deadlock occur here and how is it handled?* Opposite lock order across two slots; prevented by ordered locking, short transactions and retry on error 1213.
6. *What does SKIP LOCKED change?* A busy row is skipped instead of waited for, so concurrent allocators receive different slots without blocking.
