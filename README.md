## 🏗️ Smart Processing Flow (End-to-End) Description

This architecture follows a decoupled, near real-time, event-driven pattern using a database queue to ensure the existing mission-critical system remains unaffected by the new messaging service.

### 1\. The Existing System (Source)

| Component | Responsibility | Technology/Database |
| :--- | :--- | :--- |
| **ICashier Application** | Initiates the transaction save. | Existing Desktop App (.EXE) |
| **`DebitCard_Data`** | Stores core transaction data. | SQL Server |
| **`DC_Transaction` Table** | Core table for invoice headers. | SQL Server |
| **`Messaging_Queue` Table** | **The Event Queue.** Holds `TransactionID`s needing processing. | SQL Server (in `DebitCard_Data` DB) |
| **`BusinessTransTrigger`** | **The Publisher.** Inserts new `TransactionID`s into the queue. | SQL Server Trigger |

**Flow Step:** When the ICashier app saves, the existing `BusinessTransTrigger` fires an `INSERT` into the **`Messaging_Queue`** table.

### 2\. The Worker Service (The Bridge / Consumer)

**Project:** `WonderLand.MessageProcessor.Worker`
**Technology:** .NET Worker Service (`IHostedService` pattern)

This service is installed as a Windows Service on the server. Its primary job is to bridge the two database environments and connect the event (the new queue record) to the business logic (the API).

| Sub-Component | Responsibility |
| :--- | :--- |
| **Polling Logic** | Runs a recurring background task (e.g., every 5 seconds). |
| **Queue Repository** | Connects to **`DebitCard_Data`** to read and manage the **`Messaging_Queue`** table. Implements **transactional locking** by updating `Status` from 'New' to 'Processing' to prevent duplicates. |
| **Old DB Repository** | Connects to **`DebitCard_Data`** and executes the **complex multi-table `SELECT` query** (the one provided earlier) using the `TransactionID` to extract all required details (Invoice Number, Phone, Tender, Item details, etc.). |
| **API Client (`HttpClient`)** | Serializes the extracted data into the shared **`InvoiceDataModel`** contract and calls the **WonderLandMessagingAPI** endpoint (`/api/invoices/process-message`). |

**Data Contract (Shared Model):** The Worker must use a shared C\# model, e.g., `InvoiceDataModel`, that matches the output of the complex SQL query. This model is critical for the AI to understand the required JSON payload.

### 3\. The Messaging API (The Orchestrator / Receiver)

**Project:** `WonderLandMessagingAPI`
**Technology:** .NET Web API

This API is the core business logic handler, completely isolated from the existing system's database.

| Sub-Component | Responsibility |
| :--- | :--- |
| **Controller Endpoint** | Receives the **`InvoiceDataModel`** payload from the Worker Service. |
| **New DB Repository** | Connects to the **`WonderLand_MarketingDB`**. **Saves** the received invoice and customer details into the new, separate database (fulfilling requirements 1 and 4). |
| **Messaging Service** | Handles the external communication logic: <br> - Calls the **WhatsApp Business API** (Twilio/Meta) to send the E-slip. <br> - Implements a **fall-back mechanism** to send SMS if WhatsApp delivery fails. |
| **Response** | Returns a simple success/failure status code (e.g., 200/500) to the Worker Service to confirm the message process is complete. |

### 4\. The New Database (Destination)

**Database:** `WonderLand_MarketingDB`
**Purpose:** Storage for marketing and analytics, completely isolated from the production system.

| Table Example | Purpose |
| :--- | :--- |
| `Invoices` | Stores the transaction data received from the Worker Service. |
| `Customers` | Stores customer contact information derived from the invoice data. |
| `MessageLogs` | Tracks the status of every WhatsApp/SMS sent (delivery tracking for Phase 2). |

-----

## 🛠️ Key Instructions for Your AI Assistant

When you prompt your AI for code, make sure to explicitly include these points for each project:

### A. For the `WonderLand.MessageProcessor.Worker`

  * **Goal:** Create a .NET Worker Service using **`IHostedService`** for polling.
  * **Dependencies:** Needs Entity Framework (or Dapper) for SQL connection and `HttpClient` for API calls.
  * **Code Focus:** Provide the code for the main `ExecuteAsync` loop, including the `Task.Delay` for the polling interval (5 seconds).
  * **Data Requirement:** Use the **`InvoiceDataModel`** C\# class as the contract for the extracted data.

### B. For the `WonderLandMessagingAPI`

  * **Goal:** Create a minimal .NET Web API with an endpoint to receive and process the invoice data.
  * **Endpoint Focus:** Create a `POST` endpoint at `/api/invoices/process-message` that accepts the **`InvoiceDataModel`**.
  * **Logic Focus:** The API logic must include calls to:
    1.  A repository to save the data to the **`WonderLand_MarketingDB`**.
    2.  A service stub to simulate the **WhatsApp/SMS API call**.
  * **Contract:** Must define the exact same **`InvoiceDataModel`** class as the Worker.
