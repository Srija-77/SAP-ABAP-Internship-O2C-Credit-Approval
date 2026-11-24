# SAP ABAP Developer Internship — Order-to-Cash (O2C) Credit & High-Value Order Approval System

## 📌 Project Overview
This repository contains the deliverables for my **SAP ABAP Developer Internship** (SAP S/4HANA / ECC 6.0). The project addresses operational bottlenecks in the **Order-to-Cash (O2C)** cycle by automating real-time credit limit validation and high-value sales order approvals (orders exceeding **₹10,00,000**).

### Business Problem
- Manual credit checks and off-system email approvals delay outbound deliveries (`VL01N`).
- Sales representatives lack visibility into why high-value orders are blocked.
- Absence of an auditable approval log inside SAP increases financial risk.

### Proposed SAP ABAP Solution (RICEFW)
- **Data Dictionary (`SE11`):** Custom tables `ZCREDIT_RULES` (configurable thresholds & approvers) and `ZORDER_AUDIT` (complete approval audit trail).
- **Enhancement (`BADI_SD_SALES` / `MV45AFZZ`):** Intercepts Sales Order creation (`VA01`) before saving, checks order net value and live FI-AR credit exposure, applies delivery block `Z1`, and triggers email alerts via `CL_BCS`.
- **OO-ALV Cockpit (`ZCRED_APP` / `SE38`):** Interactive `CL_SALV_TABLE` report enabling Finance Managers to review blocked orders and approve/reject in one click via `BAPI_SALESORDER_CHANGE`.

---

## 🔄 End-to-End Business Process Flow

```mermaid
flowchart TD
    A([Start: Sales Rep Creates Order in VA01]) --> B[Save Event Triggered]
    B --> C[BAdI / User Exit MV45AFZZ Intercepts Order]
    C --> D[Fetch Threshold from ZCREDIT_RULES & FI-AR Credit Exposure]
    D --> E{Order > ₹10L OR Credit Exceeded?}
    
    E -- No (Within Limits) --> F[Save Sales Order Normally]
    F --> G[Proceed to Outbound Delivery VL01N & Billing VF01]
    G --> Z([End Process])
    
    E -- Yes (Risk Triggered) --> H[Assign Delivery Block 'Z1' to VBAK-LIFSK]
    H --> I[Log Pending Record in ZORDER_AUDIT]
    I --> J[Send Automated Email/Alert to Finance Manager]
    J --> K[Finance Manager Reviews in ALV Cockpit ZCRED_APP]
    K --> L{Manager Decision?}
    
    L -- Approve --> M[Call BAPI_SALESORDER_CHANGE to Remove Block 'Z1']
    M --> N[Update ZORDER_AUDIT Status = 'APPROVED' & Notify Sales]
    N --> G
    
    L -- Reject --> O[Set Rejection Reason ABGRU on Order Items]
    O --> P[Update ZORDER_AUDIT Status = 'REJECTED' & Notify Sales]
    P --> Z

## Three-Tier SAP S/4HANA Technical Architecture

flowchart TB
    subgraph Presentation["1. Presentation Layer (UI)"]
        UI1["SAP GUI / Fiori: VA01 (Sales Order Creation)"]
        UI2["SAP GUI: ZCRED_APP (OO-ALV Approval Cockpit)"]
        UI3["SM30 (Maintenance View for Credit Rules)"]
    end

    subgraph Application["2. Application Layer (ABAP Runtime)"]
        EXIT["Enhancement: BADI_SD_SALES / MV45AFZZ"]
        CLS["Helper Class: ZCL_SD_CREDIT_VALIDATOR"]
        ALV["Report: ZSD_CREDIT_APPROVAL_COCKPIT (CL_SALV_TABLE)"]
        BAPI["Standard BAPI: BAPI_SALESORDER_CHANGE"]
        MAIL["Notification Service: CL_BCS (Email Trigger)"]
    end

    subgraph Database["3. Database Layer (SAP HANA)"]
        STD["Standard SAP Tables: VBAK, VBAP, KNKK, ACDOCA/BSID"]
        ZTBL1["Custom Table: ZCREDIT_RULES (Thresholds & Approvers)"]
        ZTBL2["Custom Table: ZORDER_AUDIT (Approval Audit Logs)"]
    end

    UI1 --> EXIT
    EXIT --> CLS
    CLS --> MAIL
    UI2 --> ALV
    ALV --> BAPI
    UI3 --> ZTBL1

    CLS <--> STD
    CLS <--> ZTBL1
    CLS --> ZTBL2
    ALV <--> ZTBL2
    BAPI --> STD
