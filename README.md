# Asynchronous Excel Export System (Spring Boot + Oracle + React)

This project implements a high-performance, non-blocking pattern for exporting large datasets to Excel. It is designed to handle "heavy" reports (ranging from seconds to 5+ minutes) without timing out the browser or crashing the server due to memory exhaustion.

## 🚀 High-Level Architecture
The system uses an **Asynchronous Request-Reply Pattern**. The database acts as a state coordinator, allowing the backend to work in a separate thread while the frontend polls for updates.

sequenceDiagram
    autonumber
    participant React as React Frontend
    participant API as Spring Boot Controller
    participant Async as Async Export Service
    participant DB_Poll as Oracle (VW_EXPORT_POLLING)
    participant DB_Bin as Oracle (VW_EXPORT_BINARY)

    Note over React, DB_Bin: Phase 1: Request Initiation
    React->>API: POST /api/export (Params)
    API->>DB_Poll: INSERT Status 'PROCESSING'
    API-->>React: 202 Accepted (Return exportId)
    
    Note over React, DB_Bin: Phase 2: Background Processing (~5 mins)
    API->>Async: Trigger @Async thread
    Async->>Async: Generate Excel via SXSSF (Apache POI)
    Async->>DB_Bin: UPDATE Row: Set BLOB, FileName & Status 'COMPLETED'
    
    Note over React, DB_Bin: Phase 3: Status Polling (every 3s)
    loop Until Status == COMPLETED
        React->>API: GET /api/export/status/{exportId}
        API->>DB_Poll: SELECT (ID, STATUS, UPDATED_ON)
        DB_Poll-->>API: Row Data
        API-->>React: Return Status (Processing...)
    end
    
    Note over React, DB_Bin: Phase 4: Final Download
    React->>API: GET /api/export/download/{exportId}
    API->>DB_Bin: SELECT (ID, FILE_NAME, BLOB_CONTENT)
    DB_Bin-->>API: Binary Payload
    API-->>React: Stream Byte Array (Content-Disposition: attachment)
    React->>React: Browser Triggers File Save

---

## 🛠️ Implementation Details

### 1. Database Layer (Oracle)
The core strategy is **Metadata/Binary Separation**. We use two distinct views to ensure high-frequency polling does not impact database performance or drag heavy binary data across the network.

* **Polling View (`VW_EXPORT_POLLING`)**:
    * **Definition**: `SELECT id, export_id, status, created_by, created_on, updated_on FROM export_master_table`.
    * **Logic**: This view strictly excludes the `BLOB` column. This minimizes the "row width" in the Oracle buffer cache, allowing 3-second status checks from multiple users to be handled entirely in memory.
* **Binary View (`VW_EXPORT_BINARY`)**:
    * **Definition**: `SELECT id, export_id, file_name, blob_content FROM export_master_table`.
    * **Logic**: This view is only queried once per successful export cycle. It includes the `FILE_NAME` so the backend can set the `Content-Disposition` header dynamically without needing to join back to metadata tables.

### 2. Service Layer (Spring Boot)
The backend manages thread isolation and memory-safe streaming to handle 5-minute processing windows reliably.

* **Thread Isolation**: The application uses `@EnableAsync` with a custom `ThreadPoolTaskExecutor`. This ensures long-running exports do not starve the main Tomcat thread pool, keeping the rest of the web application responsive.
* **Streaming Excel (Apache POI SXSSF)**:
    * The service utilizes `SXSSFWorkbook`, the streaming extension of XSSF.
    * **Memory Management**: It maintains a "row access window" (e.g., 100 rows) in memory. Once the limit is reached, older rows are flushed to temporary XML files on disk, preventing `OutOfMemoryError` regardless of data size.
* **Oracle BLOB Streaming**:
    * The download endpoint uses `StreamingResponseBody`.
    * **Logic**: It opens a stream directly from the Oracle `BLOB` column and pipes it to the `HttpServletResponse` output stream. The file is never fully loaded into the JVM heap memory.

### 3. UI Layer (React)
The frontend handles asynchronous state transitions to provide a smooth, non-blocking user experience.

* **Initiation**: On button click, the app calls the export endpoint, receives a `jobId`, and displays a "Processing" spinner.
* **Polling Logic**: A `useEffect` hook initiates a `setInterval` to check the status every 3 seconds via the Polling View.
* **Resolution**:
    * If `STATUS === 'COMPLETED'`, the app clears the interval and triggers the binary download endpoint.
    * If `STATUS === 'FAILED'`, it displays the error message captured in the database record.
* **Safety**: The polling logic includes a timestamp check against `UPDATED_ON` to detect "ghost" processes that may have crashed on the server, allowing for a "Retry" option.

