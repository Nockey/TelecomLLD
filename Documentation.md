# Telecom Service Project: Low-Level Design Documentation

This document provides a comprehensive Low-Level Design (LLD) for the proposed Telecom Service Project, outlining the system's entities, module-specific logic, algorithmic considerations, and implementation details within a local MySQL environment, aligned with the provided Entity-Relationship (ER) diagram. The design prioritizes data integrity, system scalability, and maintainability.

---

## 1. System Entities

The foundational data structures (entities) within this telecom service system are defined below, with attributes and brief descriptions, directly reflecting the ER diagram.

### Customer

- **Attributes:**  
    - `Customer_ID` (Unique, Int)  
    - `Customer_Name` (String)  
    - `Customer_Phone` (Unique, Int)  
    - `Customer_Email` (String)  
    - `Customer_Address` (String)  
    - `Customer_Status` (Enum)  
    - `Customer_RegdSim` (Collection)
- **Description:** Core subscriber profile, centralizing personal and contact information. `Customer_RegdSim` represents the SIMs registered to the customer.

### Sim

- **Attributes:**  
    - `Sim_Id` (Unique, Int)  
    - `Sim_Customer_ID` (Unique, Int)  
    - `Plan_Id` (Int)  
    - `Sim_Status` (bool)
- **Description:** Manages SIM card details and associations with a customer and a plan. `Sim_Status` indicates if the SIM is active or inactive.

### Plan

- **Attributes:**  
    - `Plan_Id` (Unique, Int)  
    - `Plan_Name` (Unique, String)  
    - `Plan_Type` (String)
- **Description:** Base entity for all service plans, serving as the superclass for prepaid and postpaid plans. `Plan_Type` acts as a discriminator.

### PrepaidPlan

- **Attributes:**  
    - `Pr_PlanId` (Int)  
    - `Pr_PlanName` (String)  
    - `Pr_Duration` (Int)  
    - `Pr_StartDate` (Date)  
    - `Pr_EndDate` (Date)  
    - `Pr_DataAllowedGB` (Float)  
    - `Pr_CallsAllowed` (Int)  
    - `Pr_smsAllowed` (Int)
- **Description:** Attributes specific to prepaid plans, linked to the Plan entity via `Pr_PlanId`.

### PostpaidPlan

- **Attributes:**  
    - `Po_PlanId` (Int)  
    - `Po_PlanName` (String)  
    - `Po_Duration` (Int)  
    - `Po_StartDate` (Date)  
    - `Po_EndDate` (Date)
- **Description:** Attributes specific to postpaid plans, linked to the Plan entity via `Po_PlanId`.

### Usage

- **Attributes:**  
    - `Usage_ID` (Unique, Int)  
    - `Usage_SimId` (Unique, Int)  
    - `Usage_Month` (Integer)  
    - `Usage_CallMinutesUsed` (Integer)  
    - `Usage_DataUsedGB` (Float)  
    - `Usage_Amount` (Float)
- **Description:** Captures monthly service consumption for a specific SIM. `Usage_ID` is included as a primary key for unique identification, with `Usage_SimId` and `Usage_Month` forming a unique constraint.

### Bill

- **Attributes:**  
    - `Bill_ID` (Unique, Int)  
    - `Bill_Customer_ID` (Int)  
    - `Bill_Month` (String)  
    - `Bill_TotalAmount` (Float)  
    - `Bill_DueDate` (Date)  
    - `Bill_Status` (Enum)
- **Description:** Official invoice for services rendered to a customer for a specific month.

### Payment

- **Attributes:**  
    - `Payment_Id` (Unique, Integer)  
    - `Payment_Bill_ID` (Integer)  
    - `Payment_Date` (Date)  
    - `Payment_Amount` (Integer)  
    - `Payment_Mode` (String)
- **Description:** Records customer payments against a specific bill.

---

## 2. Module Implementations

### 2.1 Customer Management

**Add New Customer**
- Verify existence via `Customer_Phone` or `Customer_Email`.
- If exists, authenticate and allow SIM addition (linking `Sim_Customer_ID`).
- If new, collect details, generate `Customer_ID`, provision SIM, persist data, set `Customer_Status` to 'Active'.
- Use indexed queries, secure authentication, MySQL `AUTO_INCREMENT`, `INSERT` operations.

**Edit Customer Details**
- Authenticate, present editable fields, validate, commit changes to Customer table.
- Algorithms: Authentication, regex validation, `UPDATE` operations, transactions.

**View Customer Profile**
- Retrieve profile from Customer table and associated SIMs/plans/usage by joining relevant tables.
- Algorithms: `SELECT` with `JOIN`s, indexing, read-only data mapping.

### 2.2 Plan Management

**Create Mobile Plans**
- Capture common attributes for Plan and type-specific attributes for PrepaidPlan or PostpaidPlan.
- Validate inputs, persist base and specialized records atomically.
- Algorithms: Validation, transactions, `INSERT`s, class table inheritance.

**Update Mobile Plans**
- Identify plan by `Plan_Id`, authorize, present details, validate, commit updates atomically.
- Algorithms: Validation, transactions, `UPDATE`s.

**Delete Mobile Plans**
- Check for active Sim dependencies. Prevent deletion or reassign if needed.
- Delete records atomically from PrepaidPlan/PostpaidPlan and then Plan tables.
- Algorithms: Dependency check, transactions, `DELETE`s.

### 2.3 Usage Tracking

**Add Call/Data Usage**
- Validate `Sim_Id` and usage data. Persist usage record.
- Algorithms: Validation, `INSERT`.

**View Usage Per Customer**
- Aggregate usage for all SIMs associated with a customer for a given period.
- Algorithms: `SELECT` with `JOIN`s, `SUM()`, `GROUP BY`, indexing.

### 2.4 Billing Module

**Monthly Bill Generation**
- Scheduled task, iterate through customers and their active SIMs.
- Aggregate usage, calculate charges, apply taxes/surcharges, create Bill record atomically.
- Algorithms: Iterative processing, arithmetic, `SELECT`s, transactions, scheduling.

**Tax and Surcharge Calculation**
- Apply configured rates to the calculated subtotal.
- Algorithms: Arithmetic.

**PDF/HTML Bill Generation (Optional)**
- Retrieve bill details, format with a template, render PDF/HTML.
- Algorithms: Data retrieval, templating, PDF conversion.

### 2.5 Payment Module

**Payment Due/Paid Status**
- Aggregate payments for a `Bill_ID`.
- Update `Bill_Status` conditionally based on payment.
- Algorithms: `SUM()`, conditional `UPDATE`, transactions.

**Payment Reminders**
- Scheduled task, identify unpaid/imminent bills, send notifications.
- Algorithms: `SELECT`, external notification API.

**Late Payment Penalty**
- Scheduled task, identify overdue bills, calculate/apply penalty, notify customer.
- Algorithms: `SELECT`, arithmetic, transactions, `UPDATE`.

### 2.6 Service Control

**Temporarily Deactivate Customer**
- Verify customer, update `Customer_Status` and `Sim_Status` atomically, notify customer.
- Algorithms: Transactions, `UPDATE`, notification.

**Reactivate Customer**
- Verify customer, revert statuses to 'Active', notify customer.
- Algorithms: Transactions, `UPDATE`.

**Permanent Disconnection**
- Pre-check for unpaid bills, update statuses, optionally archive data.
- Algorithms: Pre-check, transactions, `UPDATE`, archiving.

---

## 3. Scope of Variables, Data Types, and Inheritance in Local MySQL LLD

### 3.1 Scope of Variables

- **Local Scope:** Variables defined within functions/methods for temporary use.
- **Parameter Scope:** Data passed explicitly between layers or functions.
- **Class/Instance Scope:** Attributes within classes maintaining object state.
- **Global Scope:** Limited to configuration settings, database connection pools, or application-wide constants.

### 3.2 Data Types (MySQL Context)

- **INT, INTEGER, BIGINT:** Identifiers, counts, monetary values.
- **VARCHAR(length):** Variable-length strings (names, email, address, payment mode, bill month).
- **FLOAT:** Decimal numbers (data allowances/usage, total amounts).
- **DATE:** Date values.
- **BOOLEAN (TINYINT(1)):** True/false flags.
- **ENUM:** Fixed value sets (statuses).
- **TEXT:** Longer, unpredictable strings (if needed).

### 3.3 Inheritance (Class Table Inheritance)

- **Mechanism:**  
    - Superclass table (`Plan`) holds common attributes.
    - Subclass tables (`PrepaidPlan`, `PostpaidPlan`) hold specific attributes.
    - One-to-one relationship via foreign keys.
    - `Plan_Type` acts as a discriminator.

- **Advantages:**  
    - Normalization, flexibility, clarity, query efficiency.

---

This LLD provides a robust blueprint for developing the telecom service project in a local MySQL environment, ensuring a well-structured, maintainable, and scalable system, directly mapping to the provided ER diagram.
