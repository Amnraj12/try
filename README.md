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
    Start([<b>GET /tables</b>])
    
    Start --> API["<b>API Route</b><br/><br/>📄 api/routes/<br/>file_processing.py<br/>📍 Line 146<br/><br/>async def get_tables<br/>current_user"]
    
    API --> Service["<b>Service Call</b><br/><br/>📄 services/<br/>file_processing_service/<br/>file_processing_service.py<br/>📍 Line 189<br/><br/>async def get_tables<br/>org_slug"]
    
    Service --> BuildQuery["<b>Build SQL</b><br/><br/>📍 Lines 209-220<br/><br/>tables_query =<br/>SELECT table_id,<br/>tablename...<br/>FROM table_master"]
    
    BuildQuery --> Execute["<b>Execute Query</b><br/><br/>📍 Line 222<br/><br/>tables_results =<br/>await self.database<br/>.execute_query<br/>tables_query"]
    
    Execute --> Provider["<b>DB Provider</b><br/><br/>📄 core/database/<br/>postgresql_provider.py<br/>📍 Line 146<br/><br/>async def<br/>execute_query<br/>self, query, params"]
    
    Provider --> GetConn["<b>Get Cursor</b><br/><br/>📍 L172: with connection<br/>📍 L173: raw_connection<br/>📍 L174: cursor =<br/>connection.cursor"]
    
    GetConn --> Block1["<b>🔴 BLOCK 1</b><br/><br/>📍 Lines 177-179<br/><br/>if params:<br/>cursor.execute<br/>query, params<br/>else:<br/>cursor.execute<br/>query<br/><br/>⚠️ SYNC - blocks<br/>150-300ms<br/>⚠️ Event loop frozen"]
    
    Block1 --> Block2["<b>🔴 BLOCK 2</b><br/><br/>📍 Line 186<br/><br/>rows =<br/>cursor.fetchall<br/><br/>⚠️ SYNC - blocks<br/>50-150ms<br/>⚠️ Loop still frozen"]
    
    Block2 --> Convert["<b>Convert</b><br/><br/>📍 Line 189<br/><br/>result_list =<br/>dict zip columns, row<br/>for row in rows"]
    
    Convert --> Return["<b>Return</b><br/><br/>📍 L191: return<br/>Back to service<br/>Line 222"]
    
    Return --> Impact["<b>⚠️ IMPACT</b><br/><br/>Per Query:<br/>200-450ms blocked<br/><br/>Event loop frozen<br/>Other requests wait"]
    
    style Block1 fill:#ff6b6b,stroke:#c92a2a,stroke-width:4px,color:#fff,font-size:15px
    style Block2 fill:#ff6b6b,stroke:#c92a2a,stroke-width:4px,color:#fff,font-size:15px
    style Impact fill:#ffa94d,stroke:#fd7e14,stroke-width:3px,color:#000,font-size:15px
    style Start fill:#4dabf7,stroke:#1971c2,stroke-width:3px,color:#fff,font-size:15px
```

