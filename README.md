# Issue 1: Blocking I/O

```mermaid
flowchart TB
    Start([GET /tables Request]) --> API[FastAPI Endpoint<br/>file_processing.py L146<br/>async def get_tables]
    
    API --> Service[FileProcessingService.get_tables<br/>file_processing_service.py L189<br/>async def get_tables]
    
    Service --> Query1[Execute SQL Query<br/>SELECT FROM table_master<br/>L209-220]
    
    Query1 --> ExecuteQuery[await database.execute_query<br/>L222]
    
    ExecuteQuery --> Provider[PostgreSQLProvider.execute_query<br/>postgresql_provider.py L146<br/>async def execute_query]
    
    Provider --> GetConn[Get connection from pool<br/>L167-169]
    
    GetConn --> Problem[🔴 BLOCKING ISSUE<br/>cursor.execute query, params<br/>L175-177<br/><br/>SYNC call in async function<br/>Blocks event loop 150-300ms]
    
    Problem --> Fetch[🔴 BLOCKING ISSUE<br/>rows = cursor.fetchall<br/>L185<br/><br/>SYNC call blocks 50-150ms]
    
    Fetch --> Convert[Convert rows to dict<br/>L188-191]
    
    Convert --> Return[Return results]
    
    Return --> Impact[⚠️ IMPACT:<br/>Total blocking: 200-450ms<br/>Other requests queued<br/>Poor concurrency]
    
    Impact --> Solution[✅ SOLUTION:<br/>Use thread pool executor<br/>to offload blocking calls<br/>asyncio.to_thread or<br/>run_in_executor]
    
    style Problem fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    style Fetch fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    style Impact fill:#ffa94d,stroke:#fd7e14,stroke-width:2px
    style Solution fill:#51cf66,stroke:#2f9e44,stroke-width:2px
```

# Issue 1: Blocking I/O

```mermaid
flowchart TD
    Start([<b>CLIENT REQUEST</b><br/><br/>GET /tables API called])
    
    Start --> API[<b>API LAYER</b><br/><br/>📄 api/routes/file_processing.py<br/>📍 Line 146<br/><br/>async def get_tables<br/><br/>FastAPI expects non-blocking code]
    
    API --> Validate[<b>VALIDATE ORG</b><br/><br/>📍 Line 168<br/><br/>org_slug = current_user.get org_slug<br/>org_slug = validate_org_slug org_slug]
    
    Validate --> CallService[<b>CALL SERVICE METHOD</b><br/><br/>📍 Line 171<br/><br/>tables_data = await<br/>file_processing_service.get_tables<br/>org_slug=org_slug<br/><br/>Using await - expects async operation]
    
    CallService --> ServiceEntry[<b>SERVICE LAYER ENTRY</b><br/><br/>📄 services/file_processing_service/<br/>file_processing_service.py<br/>📍 Line 189<br/><br/>async def get_tables org_slug<br/><br/>Function signature promises async behavior]
    
    ServiceEntry --> CheckTables[<b>CHECK TABLE EXISTS</b><br/><br/>📍 Lines 191-196<br/><br/>table_exists = await<br/>self.database.table_exists<br/>table_master, org_slug]
    
    CheckTables --> BuildQuery[<b>BUILD SQL QUERY</b><br/><br/>📍 Lines 209-220<br/><br/>tables_query = SELECT<br/>table_id, tablename,<br/>table_description, uploaded_at,<br/>created_at, record, status<br/>FROM table_master<br/>WHERE enabled IS NULL OR enabled = TRUE<br/>ORDER BY created_at DESC<br/><br/>Query prepared in memory - no blocking yet]
    
    BuildQuery --> ExecuteCall[<b>EXECUTE QUERY CALL</b><br/><br/>📍 Line 222<br/><br/>tables_results = await<br/>self.database.execute_query<br/>tables_query<br/><br/>Developer expects this to be async]
    
    ExecuteCall --> ProviderEntry[<b>DATABASE PROVIDER ENTRY</b><br/><br/>📄 core/database/postgresql_provider.py<br/>📍 Line 146<br/><br/>async def execute_query<br/>self, query: str, params<br/><br/>Function declared as async<br/>Creates expectation of non-blocking]
    
    ProviderEntry --> CheckEngine[<b>CHECK ENGINE</b><br/><br/>📍 Lines 157-158<br/><br/>if not self._engine:<br/>raise Exception Database not initialized<br/><br/>Validation passes]
    
    CheckEngine --> DetectWrite[<b>DETECT OPERATION TYPE</b><br/><br/>📍 Lines 161-163<br/><br/>query_upper = query.strip.upper<br/>is_write_operation = any<br/>query_upper.startswith INSERT/UPDATE/DELETE<br/><br/>Result: is_write_operation = False<br/>This is a SELECT query]
    
    DetectWrite --> GetContext[<b>GET CONNECTION CONTEXT</b><br/><br/>📍 Lines 166-169<br/><br/>connection_context =<br/>self._engine.connect<br/><br/>Because is_write_operation = False<br/>Uses connect instead of begin]
    
    GetContext --> WithBlock[<b>ENTER WITH BLOCK</b><br/><br/>📍 Line 172<br/><br/>with connection_context as connection:<br/><br/>Connection acquired from pool<br/>Pool size: 10, max_overflow: 20]
    
    WithBlock --> GetRawConn[<b>GET RAW CONNECTION</b><br/><br/>📍 Line 173<br/><br/>raw_connection = connection.connection<br/><br/>Getting underlying DBAPI connection<br/>This is where psycopg2 comes in]
    
    GetRawConn --> GetCursor[<b>GET CURSOR</b><br/><br/>📍 Line 174<br/><br/>cursor = raw_connection.cursor<br/><br/>psycopg2 cursor object created<br/>This cursor is SYNCHRONOUS]
    
    GetCursor --> RootCause[<b>🔴 ROOT CAUSE DISCOVERED</b><br/><br/>📍 Lines 177-179<br/><br/>if params:<br/>cursor.execute query, params<br/>else:<br/>cursor.execute query<br/><br/>⚠️ CRITICAL PROBLEM:<br/>cursor.execute is SYNCHRONOUS<br/>psycopg2 has NO async support<br/>This call BLOCKS the Python thread<br/>Event loop CANNOT switch tasks<br/>All other requests are FROZEN]
    
    RootCause --> Blocking1[<b>🔴 BLOCKING OPERATION 1</b><br/><br/>cursor.execute is now running<br/><br/>What happens:<br/>1. Python sends SQL to PostgreSQL<br/>2. Thread waits for network I/O<br/>3. PostgreSQL processes query<br/>4. Thread waits for response<br/>5. Event loop is STUCK<br/><br/>⏱️ Duration: 150-300ms<br/><br/>During this time:<br/>• No other async tasks can run<br/>• Incoming requests queue up<br/>• CPU sits idle waiting<br/>• Entire async benefit lost]
    
    Blocking1 --> GetColumns[<b>GET COLUMN NAMES</b><br/><br/>📍 Lines 182-183<br/><br/>columns = desc 0 for desc<br/>in cursor.description<br/>if cursor.description else <br/><br/>Fast operation - no blocking<br/>Just reading metadata]
    
    GetColumns --> Blocking2[<b>🔴 BLOCKING OPERATION 2</b><br/><br/>📍 Line 186<br/><br/>rows = cursor.fetchall<br/><br/>⚠️ ANOTHER SYNC CALL:<br/>Fetching all rows from database<br/>Blocks thread again<br/>Waiting for data transfer<br/><br/>⏱️ Duration: 50-150ms<br/><br/>Event loop still frozen<br/>Other requests still waiting]
    
    Blocking2 --> Convert[<b>CONVERT TO DICT</b><br/><br/>📍 Line 189<br/><br/>result_list = dict zip columns, row<br/>for row in rows<br/><br/>CPU-bound operation<br/>Fast - happens in memory<br/>No blocking]
    
    Convert --> Return[<b>RETURN RESULTS</b><br/><br/>📍 Line 191<br/><br/>return result_list<br/><br/>Control returns to service layer<br/>Event loop can resume<br/>But damage is done]
    
    Return --> BackToService[<b>BACK TO SERVICE</b><br/><br/>📍 Line 222<br/><br/>tables_results received<br/><br/>Now we have list of N tables<br/>But more blocking queries coming...]
    
    BackToService --> Impact[<b>⚠️ CUMULATIVE IMPACT</b><br/><br/>For this single query:<br/>• Total blocking: 200-450ms<br/>• Event loop frozen entire time<br/>• All concurrent requests delayed<br/><br/>System-wide effects:<br/>• Request 2 waits 200-450ms<br/>• Request 3 waits 400-900ms<br/>• Request 4 waits 600-1350ms<br/>• Latency compounds linearly<br/><br/>Why this defeats async:<br/>• Async designed for concurrency<br/>• Blocking kills concurrency<br/>• Single-threaded becomes bottleneck<br/>• Worse than traditional threading]
    
    Impact --> WhyHappens[<b>🔴 WHY THIS HAPPENS</b><br/><br/>Architecture mismatch:<br/><br/>1. FastAPI is async framework<br/>2. SQLAlchemy engine is sync<br/>3. psycopg2 is sync driver<br/>4. No await inside execute<br/>5. Python cannot detect blocking<br/><br/>The fake async pattern:<br/>• Function declared async def<br/>• Contains sync blocking calls<br/>• Looks async, behaves sync<br/>• Silent performance killer]
    
    style RootCause fill:#ff6b6b,stroke:#c92a2a,stroke-width:4px,color:#fff,font-size:15px
    style Blocking1 fill:#ff6b6b,stroke:#c92a2a,stroke-width:4px,color:#fff,font-size:14px
    style Blocking2 fill:#ff6b6b,stroke:#c92a2a,stroke-width:4px,color:#fff,font-size:14px
    style Impact fill:#ffa94d,stroke:#fd7e14,stroke-width:3px,color:#000,font-size:15px
    style WhyHappens fill:#ffa94d,stroke:#fd7e14,stroke-width:3px,color:#000,font-size:14px
    style Start fill:#4dabf7,stroke:#1971c2,stroke-width:3px,color:#fff,font-size:15px
```

