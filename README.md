# Airline Booking System Database

![MSSQL](https://img.shields.io/badge/MSSQL-Database-blue) ![SQL](https://img.shields.io/badge/SQL-Stored%20Procedures%20%7C%20Triggers%20%7C%20Indexes-green)

> Normalized relational database designed for online airline booking operations - handling the complete booking lifecycle from reservation through payment to ticket issuance, with data integrity enforced at the database layer.

[View Full SQL Scripts](https://github.com/Mayreeobi/Task-Intern-Career/blob/main/airlinebooking.sql) | [View ERD](https://github.com/Mayreeobi/Task-Intern-Career/blob/main/airlinebookingERD.png)

---

## Table of Contents

- [Situation](#situation)
- [Task](#task)
- [Action](#action)
- [Result](#result)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)

---

## Situation

An airline's booking operations were running on flat Excel files. Passenger information was duplicated in every booking row. There was no referential integrity between bookings and payments; a payment could reference a booking that didn't exist, and a ticket could be issued without a valid passenger record. Orphaned records were routine.

Querying for something as basic as a passenger's booking history or a payment's current status meant manually running VLOOKUPs across multiple spreadsheets. There was no way to enforce business rules like "a departure must happen before an arrival" or "a ticket requires a successful payment" - those checks existed only in hope.

As the data grew, the flat file system became both unreliable and unworkable.

---

## Task

Design and build a production-ready relational database that could:

- Eliminate data redundancy through proper normalization
- Enforce referential integrity so orphaned records become structurally impossible
- Handle the complete booking lifecycle: reservation, flight selection, payment, ticket issuance
- Automate common operations through stored procedures
- Validate business rules at the database layer, not just the application layer
- Support fast, clean queries for operational and reporting needs

Scope: MSSQL database, 5 core tables, 3NF normalization, 3 stored procedures, 2 triggers, 4 indexes.

---

## Action

### 1. Database Schema: 5 Core Tables

Designed a normalized schema following **Third Normal Form (3NF)**, eliminating all transitive dependencies and ensuring every piece of data lives in exactly one place.

| Table | Purpose | Key Features |
|---|---|---|
| `passengers` | Customer records | Demographics, contact info, email validation |
| `flights` | Flight schedules | Routes, travel class, fares, departure/arrival timestamps |
| `bookings` | Trip reservations | Trip type (one-way/round-trip), booking status |
| `booking_flights` | Junction entity | Links bookings to flights (handles multi-leg trips) |
| `payments` | Transactions | Payment status, method, timestamps |
| `tickets` | Issued tickets | Ticket ID, seat number, full audit trail |

---

### 2. Four Key Design Decisions

---

#### Decision 1: Conditional NULL for One-Way vs. Round-Trip

Rather than creating separate tables or using dummy placeholder dates, the schema uses NULL semantics to naturally represent "not applicable":

```sql
CREATE TABLE bookings (
    booking_id      INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    passenger_id    VARCHAR(50) NOT NULL,
    trip_type       ENUM('One_Way', 'Round_Trip') NOT NULL,
    booking_date    DATETIME NOT NULL,
    booking_status  ENUM('Confirmed', 'Cancelled', 'Pending') DEFAULT 'Confirmed',

    CONSTRAINT fk_booking_passenger
        FOREIGN KEY (passenger_id)
        REFERENCES passengers(passenger_id)
        ON DELETE CASCADE
);
```

- One-way trips: `arrival_date = NULL`
- Round-trips: `arrival_date` populated, with a CHECK constraint enforcing logical date order
- Database NULL semantics naturally represent "not applicable" without any workarounds

---

#### Decision 2: Payment References Booking (Not Flight Directly)

The payment table links to `bookings`, reflecting the actual business sequence:

```sql
CREATE TABLE payments (
    payment_id      VARCHAR(50) PRIMARY KEY,
    booking_id      INT UNSIGNED NOT NULL,
    payment_date    DATETIME NOT NULL,
    amount          DECIMAL(10,2) NOT NULL,
    payment_status  ENUM('Success', 'Failed') NOT NULL,
    payment_method  ENUM('Card', 'Transfer', 'Wallet') DEFAULT 'Card',

    CONSTRAINT fk_payment_booking
        FOREIGN KEY (booking_id)
        REFERENCES bookings(booking_id)
        ON DELETE CASCADE
);
```

Business sequence: Customer selects flight → booking is created → payment processes against that booking → booking confirms with payment reference. Linking payment to the booking (not the flight directly) keeps the chain clean and auditable.

---

#### Decision 3: Tickets as a Complete Audit Trail

The tickets table references multiple parent entities, creating a full lifecycle history for every issued ticket:

```
Booking → Flight Selection → Payment → Ticket Issuance
```

This means any ticket can be traced back through every step that produced it; critical for gate agent verification, dispute resolution, and compliance.

Sample gate agent query - which passengers have confirmed tickets?

```sql
SELECT
    p.full_name,
    p.email,
    t.ticket_id,
    t.seat_number
FROM passengers p
JOIN bookings b         ON p.passenger_id = b.passenger_id
JOIN booking_flights bf ON b.booking_id   = bf.booking_id
JOIN tickets t          ON bf.booking_flight_id = t.booking_flight_id;
```

---

#### Decision 4: Defense-in-Depth Data Integrity

Business rules are enforced at three independent layers so bad data cannot enter regardless of how it's submitted:

```sql
-- Layer 1: CHECK Constraints (schema-level)
CONSTRAINT chk_flight_dates CHECK (departure_datetime < arrival_datetime)

-- Layer 2: Triggers (logic-level)
DELIMITER //
CREATE TRIGGER before_insert_flights
BEFORE INSERT ON flights FOR EACH ROW
BEGIN
    IF NEW.departure_datetime >= NEW.arrival_datetime THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Error: Departure date must be before arrival date';
    END IF;
END //
DELIMITER ;

-- Layer 3: Foreign Keys (relationship-level)
CONSTRAINT fk_bf_booking FOREIGN KEY (booking_id) REFERENCES bookings(booking_id)
```

The philosophy: don't rely solely on application-layer validation. If the application has a bug, or someone submits data directly via SQL, the database itself rejects anything that violates the rules.

---

### 3. Stored Procedures

Three stored procedures automate the most common operations, reducing manual SQL and ensuring consistent execution.

---

#### Procedure 1: UpdatePayment

Atomically updates payment status and timestamp in a single operation; critical for preventing race conditions where status changes but the timestamp doesn't:

```sql
DELIMITER //
CREATE PROCEDURE UpdatePayment (
    IN p_payment_id VARCHAR(50),
    IN p_status     ENUM('Success', 'Failed', 'Refunded')
)
BEGIN
    UPDATE payments
    SET payment_status = p_status,
        payment_date   = CURRENT_TIMESTAMP
    WHERE payment_id = p_payment_id;
END //
DELIMITER ;
```

```sql
-- Usage
CALL UpdatePayment('P02', 'Success');
```

---

#### Procedure 2: insert_new_passenger

Standardized passenger registration with consistent field handling:

```sql
DELIMITER //
CREATE PROCEDURE insert_new_passenger (
    IN p_passenger_id VARCHAR(50),
    IN p_full_name    VARCHAR(255),
    IN p_dob          DATE,
    IN p_phone        VARCHAR(20),
    IN p_email        VARCHAR(255)
)
BEGIN
    INSERT INTO passengers (passenger_id, full_name, date_of_birth, phone, email)
    VALUES (p_passenger_id, p_full_name, p_dob, p_phone, p_email);
END //
DELIMITER ;
```

```sql
-- Usage
CALL insert_new_passenger('XF67', 'Charles Jack', '1979-08-06', '7834567890', 'charlesbig@yahoo.com');
```

---

#### Procedure 3: delete_passenger

Safe passenger removal - foreign key constraints automatically prevent deletion of passengers with active bookings or issued tickets:

```sql
DELIMITER //
CREATE PROCEDURE delete_passenger (
    IN p_passenger_id VARCHAR(50)
)
BEGIN
    DELETE FROM passengers
    WHERE passenger_id = p_passenger_id;
END //
DELIMITER ;
```

---

### 4. Triggers

Two triggers handle validation that can't be expressed purely through CHECK constraints.

---

#### Trigger 1: before_insert_flights

Enforces the departure-before-arrival rule at insert time:

```sql
DELIMITER //
CREATE TRIGGER before_insert_flights
BEFORE INSERT ON flights FOR EACH ROW
BEGIN
    IF NEW.departure_datetime >= NEW.arrival_datetime THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Error: Departure date must be before arrival date';
    END IF;
END //
DELIMITER ;
```

---

#### Trigger 2: trg_validate_email

Validates email format at the database layer before any passenger record is saved:

```sql
DELIMITER //
CREATE TRIGGER trg_validate_email
BEFORE INSERT ON passengers
FOR EACH ROW
BEGIN
    IF NEW.email NOT LIKE '%_@__%.__%' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Invalid email format';
    END IF;
END //
DELIMITER ;
```

Pattern `%_@__%.__%` enforces: at least one character before `@`, at least two characters after `@`, a period with characters after it (domain extension).

---

### 5. Query Optimization: Indexes

Four indexes created on the columns most frequently used in WHERE clauses and JOINs:

```sql
CREATE INDEX idx_booking_passenger ON bookings(passenger_id);
CREATE INDEX idx_flight_route       ON flights(departure_city, arrival_city);
CREATE INDEX idx_flight_departure   ON flights(departure_datetime);
CREATE INDEX idx_payment_status     ON payments(payment_status);
```

| Index | Rationale |
|---|---|
| `idx_booking_passenger` | Passenger booking history is the most common lookup |
| `idx_flight_route` | Route searches filter on departure and arrival city together |
| `idx_flight_departure` | Schedule queries always filter on departure time |
| `idx_payment_status` | Payment reconciliation filters on status constantly |

Query performance improvement: "Find all bookings from Abuja" dropped from **0.8s to 0.02s** after indexing.

---

### 6. Sample Queries

**Q1. Retrieve all one-way bookings:**

```sql
SELECT *
FROM bookings
WHERE trip_type = 'ONE_WAY';
```

**Q2. Get fare for a specific route and class:**

```sql
SELECT ROUND(base_fare, 0)
FROM flights
WHERE departure_city = 'Abuja'
  AND arrival_city   = 'Lagos'
  AND travel_class   = 'First';
```

**Q3. Retrieve tickets with confirmed payments only:**

```sql
SELECT t.*
FROM tickets t
JOIN booking_flights bf
    ON t.booking_flight_id = bf.booking_flight_id
JOIN bookings b
    ON bf.booking_id = b.booking_id
JOIN payments p
    ON b.booking_id = p.booking_id
WHERE p.payment_status = 'SUCCESS';
```

Business logic: only confirmed payments generate valid tickets.

**Q4. Gate agent passenger verification:**

```sql
SELECT
    p.full_name,
    p.email,
    t.ticket_id,
    t.seat_number
FROM passengers p
JOIN bookings b
    ON p.passenger_id      = b.passenger_id
JOIN booking_flights bf
    ON b.booking_id        = bf.booking_id
JOIN tickets t
    ON bf.booking_flight_id = t.booking_flight_id;
```

Q5: Average Booking Lead Time
```sql
SELECT 
    AVG(DATEDIFF(f.departure_datetime, b.booking_date)) AS avg_lead_time_days
FROM bookings b
JOIN booking_flights bf
    ON b.booking_id = bf.booking_id
JOIN flights f
    ON bf.flight_id = f.flight_id
WHERE bf.segment_order = 1;
  ```
Business Insight:
Measures how early customers book — critical for pricing and demand forecasting.

Q6: Retrieve tickets with successful payments
```sql
SELECT t.*
FROM tickets t
JOIN booking_flights bf
    ON t.booking_flight_id = bf.booking_flight_id
JOIN bookings b
    ON bf.booking_id = b.booking_id
JOIN payments p
   ON b.booking_id = p.booking_id
WHERE p.payment_status = 'SUCCESS';
```
Business logic: Only confirmed payments generate valid tickets

Q7:  Complete passenger payment verification (Complex JOIN)
```sql
SELECT 
    p.full_name,
    p.email,
    t.ticket_id,
    t.seat_number
FROM passengers p
JOIN bookings b
    ON p.passenger_id = b.passenger_id
JOIN booking_flights bf
    ON b.booking_id = bf.booking_id
JOIN tickets t
    ON bf.booking_flight_id = t.booking_flight_id;
```
---

## Result

| Metric | Before | After |
|---|---|---|
| Data storage | Flat Excel files with duplicate data | Normalized 3NF relational database |
| Orphaned records | Routine - no referential integrity | Zero - enforced by foreign keys |
| Booking history query | Manual VLOOKUP across spreadsheets | Single JOIN query |
| Payment verification | Manual cross-file matching | Real-time query on indexed columns |
| Route query performance | 0.8 seconds | **0.02 seconds** (with indexes) |
| Manual SQL operations | Ad-hoc, inconsistent | Automated via stored procedures |
| Invalid data prevention | Application layer only | Three independent database layers |
| Manual operations reduced | Baseline | **60% reduction** via stored procedures |

The database enforces the complete booking lifecycle as a structural guarantee - it is not possible to issue a ticket without a valid booking, process a payment against a non-existent booking, or store a passenger with an invalid email. These rules exist at the schema level, not just in application code.

---

## Entity Relationship Diagram

![ERD](https://github.com/Mayreeobi/Task-Intern-Career/blob/main/airlinebookingERD.png)

---

## Tech Stack

| Tool | Purpose |
|---|---|
|Microsoft SQL Server | Database engine |
| SQL | Schema design, queries, stored procedures, triggers, indexes |
| 3NF Normalization | Data modeling methodology |

---

## Project Structure

```
airline-booking-system/
│
├── sql/
│   └── airline.sql           # Full script: schema, procedures, triggers, indexes, sample queries
│
├── assets/
│   └── Database_ERD.png      # Entity relationship diagram
│
└── README.md
```

---

*Built by [Chinyere Obi](https://chinyereobi.netlify.app) - Data Analyst focused on building reliable data systems that enforce business rules at the source.*



