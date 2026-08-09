# SAP BTP ABAP Environment → Job Scheduler Integration via Destination Service

[![SAP BTP](https://img.shields.io/badge/SAP%20BTP-ABAP%20Environment-0FA5E9?logo=sap)](https://cloudplatform.sap.com)
[![ABAP Cloud](https://img.shields.io/badge/ABAP-Cloud%20Steampunk-5B21B6)](https://community.sap.com/topics/abap)
[![OAuth2](https://img.shields.io/badge/Auth-OAuth2%20Client%20Credentials-16A34A)](https://oauth.net/2/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

End-to-end architecture and implementation guide for calling the **SAP Job Scheduler REST API** from the **SAP BTP ABAP Environment (Steampunk)** using the standard **Destination Service** path.

> [!NOTE]  
> **Key Architectural Insight**: OAuth2 token retrieval and authorization header injection are handled automatically by the BTP Destination Service framework (`cl_http_destination_provider=>create_by_cloud_destination`). The ABAP code never touches client credentials, secrets, or manual token logic.

---

## 📋 Table of Contents
- [System Specifications](#-system-specifications)
- [BTP Cockpit Destination Configuration](#-btp-cockpit-destination-configuration)
- [End-to-End Architecture Diagram](#-end-to-end-architecture-diagram)
- [Step-by-Step Implementation Guide](#-step-by-step-implementation-guide)
  - [Step 1 · Create Service Consumption Model (EDMX)](#step-1--create-service-consumption-model-edmx)
  - [Step 2 · Create Outbound Service](#step-2--create-outbound-service)
  - [Step 3 · Create Communication Scenario](#step-3--create-communication-scenario)
  - [Step 4 · Create Communication System](#step-4--create-communication-system)
  - [Step 5 · Create Communication Arrangement](#step-5--create-communication-arrangement)
  - [Step 6 · Configure BTP Cockpit Destination](#step-6--configure-btp-cockpit-destination)
  - [Step 7 · Write ABAP Code](#step-7--write-abap-code)
  - [Step 8 · Response Payload](#step-8--response-payload)
- [Object-by-Object Detailed Breakdown](#-object-by-object-detailed-breakdown)
  - [Package Structure — ZJSS (5 Objects)](#package-structure--zjss-5-objects)
  - [Object Flow Diagram](#object-flow-diagram)
  - [Detailed Breakdown & Analogies](#detailed-breakdown--analogies)
- [ABAP Source Code](#-abap-source-code)
- [Artifacts Summary](#-artifacts-summary)

---

## ⚙️ System Specifications

| Parameter | Value / Details |
| :--- | :--- |
| **BTP Environment** | SAP BTP ABAP Environment (Trial - Shared US10) |
| **ABAP System ID** | `TRL_EN` (TRL, Client 100, User CB9980005587, EN) |
| **System Web Host** | `https://e9233ee0-105b-4c71-893f-d4b6f0ddd36a.abap-web.us10.hana.ondemand.com` |
| **Development IDE** | Eclipse ADT (ABAP Development Tools) |
| **Package** | `ZJSS` |

---

## 🔑 BTP Cockpit Destination Configuration

A destination named **`JSS`** must be configured in BTP Cockpit under your subaccount connectivity:

| Field | Configuration Value |
| :--- | :--- |
| **Name** | `JSS` |
| **URL** | `https://jobscheduler-rest.cfapps.us10.hana.ondemand.com/scheduler` |
| **Authentication** | `OAuth2ClientCredentials` |
| **Client ID** | `sb-228520c9-2109-4c48-a1d0-0f9eaaf64bfc|sap-jobscheduler!b1` |
| **Token Service URL** | `https://7e221c01trial.authentication.us10.hana.ondemand.com/oauth/token` |

---

## 🏗️ End-to-End Architecture Diagram

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'curve': 'linear' }}}%%
flowchart TD
    subgraph L1 ["🎨 LAYER 1 · DESIGN TIME (Eclipse ADT)"]
        direction TB
        EDMX["📄 EDMX Metadata File\nJob Scheduler OData V4 Spec"]
        SCM["📦 ZJSSSERVICECONSUMPTION\nService Consumption Model"]
        OS["🔌 ZOUTBOUNDSERVICE_REST\nHTTP Outbound Service Declaration"]
        PROXY["🧩 ZJSSSERVICECONSUMPTION\nGenerated ABAP Proxy Class (Types)"]
        CS["📜 ZJSS_COMM_SCENARIO\nCommunication Scenario Template"]

        EDMX -->|"Import EDMX Wizard"| SCM
        SCM -->|"Generates"| OS
        SCM -->|"Generates"| PROXY
        OS -->|"Referenced in Outbound Tab"| CS
    end

    subgraph L2 ["🔐 LAYER 2 · ADMIN CONFIGURATION (Fiori Launchpad)"]
        direction TB
        CSYS["💻 Communication System\nSAP_COM_0276 (Destination Service)"]
        CARR["📋 Communication Arrangement\nSAP_COM_0276 (Activates Destination Lookup)"]

        CSYS -->|"Binds System to Scenario"| CARR
    end

    subgraph L3 ["⚡ LAYER 3 · ABAP RUNTIME (BTP ABAP Steampunk)"]
        direction TB
        CLASSRUN["🚀 ZCL_JSS_SERVICE_CALL\nif_oo_adt_classrun~main"]
        DEST_PROV["🔍 cl_http_destination_provider\ncreate_by_cloud_destination('JSS')"]
        HTTP_MGR["🌐 cl_web_http_client_manager\ncreate_by_http_destination(lo_dest)"]
        REQ["SET URI PATH\nset_uri_path('/scheduler/jobs')"]
        EXEC["EXECUTE REQUEST\nexecute( if_web_http_client=>get )"]

        CLASSRUN -->|"Step 1: Resolve Destination"| DEST_PROV
        DEST_PROV -->|"Step 2: Create Client"| HTTP_MGR
        HTTP_MGR -->|"Step 3: Build Request"| REQ
        REQ -->|"Step 4: Execute Call"| EXEC
    end

    subgraph L4 ["☁️ LAYER 4 · BTP PLATFORM (Connectivity & Destination Service)"]
        direction TB
        DEST["🔑 BTP Destination 'JSS'\nURL: .../scheduler\nAuth: OAuth2ClientCredentials\nClient ID | Secret | Token URL"]
    end

    subgraph L5 ["🌐 LAYER 5 · EXTERNAL CLOUD FOUNDRY SERVICES"]
        direction TB
        XSUAA["🔒 XSUAA Authentication Server\nOAuth2 Token Endpoint (/oauth/token)"]
        JOBSCHED["⏱️ SAP BTP Job Scheduler REST API\nEndpoint: /scheduler/jobs"]
        RESP["✅ 200 OK Response Payload\nJSON List of Scheduled Jobs"]

        JOBSCHED -->|"Returns Data"| RESP
    end

    %% Cross-Layer Interconnections
    CS -->|"Published to Admin"| CARR
    DEST_PROV -.->|"Enables Lookup via"| CARR
    DEST_PROV ==>|"Looks up Destination"| DEST
    DEST ==>|"1. Request Token"| XSUAA
    XSUAA -.->|"2. Bearer Access Token"| DEST
    EXEC ==>|"3. HTTP GET + Bearer Token"| JOBSCHED
    RESP -.->|"4. Response to Console"| CLASSRUN
    PROXY -.->|"Provides Types (tys_job)"| CLASSRUN

    %% Custom Styling Tokens
    style L1 fill:#f0f9ff,stroke:#0284c7,stroke-width:2px,color:#0369a1
    style L2 fill:#f0fdf4,stroke:#16a34a,stroke-width:2px,color:#15803d
    style L3 fill:#faf5ff,stroke:#9333ea,stroke-width:2px,color:#7e22ce
    style L4 fill:#fff7ed,stroke:#ea580c,stroke-width:2px,color:#c2410c
    style L5 fill:#eef2ff,stroke:#4f46e5,stroke-width:2px,color:#3730a3

    style SCM fill:#bae6fd,stroke:#0284c7,stroke-width:1.5px,color:#0f172a
    style OS fill:#bae6fd,stroke:#0284c7,stroke-width:1.5px,color:#0f172a
    style PROXY fill:#bae6fd,stroke:#0284c7,stroke-width:1.5px,color:#0f172a
    style CS fill:#7dd3fc,stroke:#0369a1,stroke-width:1.5px,color:#0f172a
    style CSYS fill:#bbf7d0,stroke:#16a34a,stroke-width:1.5px,color:#0f172a
    style CARR fill:#86efac,stroke:#15803d,stroke-width:1.5px,color:#0f172a
    style CLASSRUN fill:#e9d5ff,stroke:#9333ea,stroke-width:1.5px,color:#0f172a
    style DEST_PROV fill:#d8b4fe,stroke:#7e22ce,stroke-width:1.5px,color:#0f172a
    style HTTP_MGR fill:#d8b4fe,stroke:#7e22ce,stroke-width:1.5px,color:#0f172a
    style REQ fill:#c084fc,stroke:#6b21a8,stroke-width:1.5px,color:#0f172a
    style EXEC fill:#a855f7,stroke:#581c87,stroke-width:1.5px,color:#fff
    style DEST fill:#fed7aa,stroke:#ea580c,stroke-width:2px,color:#0f172a
    style XSUAA fill:#fde68a,stroke:#d97706,stroke-width:1.5px,color:#0f172a
    style JOBSCHED fill:#c7d2fe,stroke:#4338ca,stroke-width:1.5px,color:#0f172a
    style RESP fill:#4ade80,stroke:#15803d,stroke-width:2px,color:#0f172a
```

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1 · Create Service Consumption Model (EDMX)
- **Object**: `ZJSSSERVICECONSUMPTION`
- **Tool**: Eclipse ADT
- **Input**: EDMX metadata file downloaded from the Job Scheduler OData V4 service.
- **What it does**: Imports external service metadata (entity types, function imports) and generates ABAP proxy types like `tys_job`, `tyt_job`, and constants for entity sets (`JOBS`, `RUN_LOGS`, `SCHEDULES`).

```text
Eclipse ADT Procedure:
Right-click Package ZJSS → New → Other ABAP Repository Object → Business Services → Service Consumption Model → Remote Consumption Mode = OData → Upload EDMX → Finish.
```

### Step 2 · Create Outbound Service
- **Object**: `ZOUTBOUNDSERVICE_REST`
- **Type**: HTTP
- **What it does**: Pure metadata object declaring that your ABAP system needs to make outbound HTTP calls.

```text
Eclipse ADT Procedure:
Right-click Package ZJSS → New → Other ABAP Repository Object → Cloud Communication Management → Outbound Service → Service Type = HTTP.
```

### Step 3 · Create Communication Scenario
- **Object**: `ZJSS_COMM_SCENARIO`
- **What it does**: Bundles the outbound service (`ZOUTBOUNDSERVICE_REST`) into a scenario template for Fiori launchpad consumption.

```text
Eclipse ADT Procedure:
Right-click Package ZJSS → New → Other ABAP Repository Object → Cloud Communication Management → Communication Scenario → Add ZOUTBOUNDSERVICE_REST in Outbound tab → Publish Locally.
```

### Step 4 · Create Communication System
- **Tool**: Fiori Launchpad App (`F1762` - Communication Systems)
- **Scenario**: `SAP_COM_0276`
- **What it does**: Registers the BTP Destination Service as a trusted external communication system.

### Step 5 · Create Communication Arrangement
- **Tool**: Fiori Launchpad App (`F1763` - Communication Arrangements)
- **Scenario**: `SAP_COM_0276`
- **What it does**: Binds the Communication Scenario to the Communication System and activates `cl_http_destination_provider=>create_by_cloud_destination('JSS')`.

### Step 6 · Configure BTP Cockpit Destination
- **Tool**: BTP Cockpit $\rightarrow$ Connectivity $\rightarrow$ Destinations
- **Name**: `JSS`
- **Auth**: `OAuth2ClientCredentials` (URL, Client ID, Secret, Token Service URL).

### Step 7 · Write ABAP Code (`ZCL_JSS_SERVICE_CALL`)
- **Object**: `ZCL_JSS_SERVICE_CALL`
- **Interface**: `IF_OO_ADT_CLASSRUN` (Executable via F9 in Eclipse ADT).

### Step 8 · Response Payload
Sample JSON response output returned by GET `/scheduler/jobs`:

```json
{
  "total": 1,
  "results": [
    {
      "jobId": 7675697,
      "name": "TEST",
      "action": "HTTPS://TEST.COM",
      "active": true,
      "httpMethod": "GET",
      "ACTIVECOUNT": 0,
      "INACTIVECOUNT": 5
    }
  ]
}
```

---

## 📖 Object-by-Object Detailed Breakdown

### Package Structure — `ZJSS` (5 Objects)

```text
TRL_EN [TRL, 100, CB9980005587, EN]
└── Favorite Packages
    └── ZJSS (5 objects)
        ├── Business Services
        │   └── Service Consumption Models
        │       └── ZJSSSERVICECONSUMPTION          Job Scheduler Service Consumption
        ├── Cloud Communication Management
        │   ├── Communication Scenarios
        │   │   └── ZJSS_COMM_SCENARIO             Communication Scenario
        │   │       └── Outbound Service
        │   │           └── ZOUTBOUNDSERVICE_REST      (referenced)
        │   └── Outbound Services
        │       └── ZOUTBOUNDSERVICE_REST          HTTP Outbound Service
        └── Source Code Library
            └── Classes
                ├── ZCL_JSS_SERVICE_CALL           Call BTP Job Scheduler API
                │   ├── IF_OO_ADT_CLASSRUN~MAIN    (method)
                │   └── ZCL_JSS_SERVICE_CALL       (text elements)
                └── ZJSSSERVICECONSUMPTION          Consumption Model (generated proxy class)
```

---

### Object Flow Diagram

```mermaid
%%{init: {'themeVariables': { 'curve': 'linear' }}}%%
flowchart TD
    subgraph Pkg ["Package ZJSS (5 Objects)"]
        EDMX["EDMX Metadata File"] -->|Import| SCM["ZJSSSERVICECONSUMPTION\nService Consumption Model"]
        SCM -->|Generates| OUT["ZOUTBOUNDSERVICE_REST\nOutbound Service (HTTP)"]
        SCM -->|Generates| PROXY["ZJSSSERVICECONSUMPTION\nGenerated Proxy Class"]
        OUT -->|Referenced by| SCEN["ZJSS_COMM_SCENARIO\nCommunication Scenario"]
    end

    subgraph Admin ["Admin Configuration"]
        SCEN -->|Consumed by Admin| CA["Communication Arrangement\nSAP_COM_0276"]
    end

    subgraph Runtime ["Runtime Execution"]
        CA -->|Enables| CLS["ZCL_JSS_SERVICE_CALL\nABAP Class Runner"]
        PROXY -.->|Provides Data Types| CLS
    end

    style SCM fill:#6dbfb0,stroke:#0f766e,stroke-width:2px,color:#000
    style OUT fill:#6dbfb0,stroke:#0f766e,stroke-width:2px,color:#000
    style PROXY fill:#6dbfb0,stroke:#0f766e,stroke-width:2px,color:#000
    style SCEN fill:#6dbfb0,stroke:#0f766e,stroke-width:2px,color:#000
    style CA fill:#86efac,stroke:#15803d,stroke-width:2px,color:#000
    style CLS fill:#c084fc,stroke:#6b21a8,stroke-width:2px,color:#000
```

---

### Detailed Breakdown & Analogies

#### 1. `ZJSSSERVICECONSUMPTION` (Service Consumption Model)
- **Location**: Business Services $\rightarrow$ Service Consumption Models
- **Analogy**: API Client SDK Generator (like OpenAPI/Swagger spec to TypeScript).
- **Purpose**: Generates ABAP proxy types (`tys_job`, `tyt_job`) and constants from the external EDMX specification.

#### 2. `ZOUTBOUNDSERVICE_REST` (Outbound Service)
- **Location**: Cloud Communication Management $\rightarrow$ Outbound Services
- **Analogy**: Android manifest `uses-permission` declaration.
- **Purpose**: Pure metadata registration declaring outbound HTTP communication intent to ABAP Cloud.

#### 3. `ZJSS_COMM_SCENARIO` (Communication Scenario)
- **Location**: Cloud Communication Management $\rightarrow$ Communication Scenarios
- **Analogy**: Infrastructure Blueprint / Terraform Module.
- **Purpose**: Bundles outbound services and permitted auth methods for admin consumption in Fiori Launchpad.

#### 4. `ZCL_JSS_SERVICE_CALL` (ABAP Class Runner)
- **Location**: Source Code Library $\rightarrow$ Classes
- **Analogy**: MVC Controller.
- **Purpose**: Executable runtime class (`if_oo_adt_classrun`) orchestrating destination lookup, HTTP execution, and console output.

#### 5. `ZJSSSERVICECONSUMPTION` (Generated Proxy Class)
- **Location**: Source Code Library $\rightarrow$ Classes
- **Analogy**: TypeScript `.d.ts` type definition file.
- **Purpose**: Generated class containing ABAP structures and constants for OData V4 client proxy calls.

---

## 💻 ABAP Source Code

```abap
CLASS zcl_jss_service_call DEFINITION
  PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES if_oo_adt_classrun.
ENDCLASS.

CLASS zcl_jss_service_call IMPLEMENTATION.
  METHOD if_oo_adt_classrun~main.
    TRY.
        " 1. Look up BTP destination 'JSS' via Destination Service
        " Requires active Communication Arrangement SAP_COM_0276
        DATA(lo_dest) = cl_http_destination_provider=>create_by_cloud_destination(
            i_name = 'JSS' ).

        " 2. Create HTTP client — OAuth2 token auto-injected by framework
        DATA(lo_client) = cl_web_http_client_manager=>create_by_http_destination(
            lo_dest ).

        " 3. Append /jobs to destination base URL (/scheduler)
        DATA(lo_request) = lo_client->get_http_request( ).
        lo_request->set_uri_path( i_uri_path = '/scheduler/jobs' ).

        " 4. Execute GET request — Bearer token automatically attached
        DATA(lo_response) = lo_client->execute( if_web_http_client=>get ).
        DATA(lv_status) = lo_response->get_status( ).
        DATA(lv_body) = lo_response->get_text( ).

        " 5. Output response to ADT Console (F9)
        out->write( |Status: { lv_status-code } { lv_status-reason }| ).
        out->write( lv_body ).

      CATCH cx_web_http_client_error INTO DATA(lx_http).
        out->write( |HTTP Error: { lx_http->get_text( ) }| ).
      CATCH cx_http_dest_provider_error INTO DATA(lx_dest).
        out->write( |Destination Error: { lx_dest->get_text( ) }| ).
      CATCH cx_root INTO DATA(lx_root).
        out->write( |Error: { lx_root->get_text( ) }| ).
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```

---

## 📊 Artifacts Summary

| # | Artifact Name | Type | Created In |
| :--- | :--- | :--- | :--- |
| **1** | `ZJSSSERVICECONSUMPTION` | Service Consumption Model | Eclipse ADT |
| **2** | `ZOUTBOUNDSERVICE_REST` | Outbound Service (HTTP) | Eclipse ADT |
| **3** | `ZJSS_COMM_SCENARIO` | Communication Scenario | Eclipse ADT |
| **4** | Communication System | Scenario `SAP_COM_0276` | Fiori Launchpad |
| **5** | Communication Arrangement | Scenario `SAP_COM_0276` | Fiori Launchpad |
| **6** | Destination `JSS` | `OAuth2ClientCredentials` | BTP Cockpit |
| **7** | `ZCL_JSS_SERVICE_CALL` | ABAP Class | Eclipse ADT |

---

## 📝 License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.