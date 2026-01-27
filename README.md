# try

'''flowchart TD
    Start([Client Request: GET /tables]) --> API[API Endpoint<br/>file_processing.py:146<br/>async def get_tables]
    
    API --> ValidateOrg[Validate org_slug<br/>file_processing.py:168]
    ValidateOrg --> CallService[Call file_processing_service.get_tables<br/>file_processing.py:171]
    
    CallService --> ServiceMethod[FileProcessingService.get_tables<br/>file_processing_service.py:189]
    
    ServiceMethod --> CheckTableMaster{Check table_master exists<br/>file_processing_service.py:201}
    CheckTableMaster --> QueryTableMaster[Execute Query:<br/>SELECT table_id, tablename,<br/>table_description, record, status<br/>FROM table_master<br/>file_processing_service.py:217-227]
    
    QueryTableMaster --> CallExecuteQuery1[await database.execute_query<br/>file_processing_service.py:229]
    
    %% ISSUE 1: Blocking I/O
    CallExecuteQuery1 --> PostgreSQLProvider1[PostgreSQLProvider.execute_query<br/>postgresql_provider.py:146]
    
    PostgreSQLProvider1 --> GetConnection1[Get connection from pool<br/>postgresql_provider.py:175]
    GetConnection1 --> BlockingExecute1[🔴 ISSUE 1: BLOCKING I/O<br/>cursor.execute - SYNC call<br/>postgresql_provider.py:175-177<br/>⚠️ Blocks event loop 150-300ms]
    
    BlockingExecute1 --> FetchAll1[cursor.fetchall - SYNC<br/>postgresql_provider.py:185<br/>⚠️ Blocks event loop 50-150ms]
    FetchAll1 --> ReturnResults1[Return results]
    
    %% ISSUE 2: N+1 Query Pattern
    ReturnResults1 --> LoopStart{For each table<br/>file_processing_service.py:232}
    
    LoopStart -->|Table 1| QueryFiles1[Execute Query:<br/>SELECT filename, status, size<br/>FROM file_master<br/>WHERE table_id = table1_id<br/>file_processing_service.py:236-247]
    
    QueryFiles1 --> CallExecuteQuery2[await database.execute_query<br/>file_processing_service.py:248-250]
    
    CallExecuteQuery2 --> PostgreSQLProvider2[PostgreSQLProvider.execute_query<br/>postgresql_provider.py:146]
    
    PostgreSQLProvider2 --> BlockingExecute2[🔴 ISSUE 2: N+1 QUERIES<br/>cursor.execute - SYNC call<br/>postgresql_provider.py:175-177<br/>⚠️ Blocks 150-300ms PER TABLE<br/>⚠️ 10 tables = 1500-3000ms total]
    
    BlockingExecute2 --> FetchAll2[cursor.fetchall - SYNC<br/>postgresql_provider.py:185]
    
    FetchAll2 --> BuildTableInfo[Build table_info dict<br/>file_processing_service.py:253-262]
    
    BuildTableInfo --> LoopStart
    
    LoopStart -->|Table 2| QueryFiles2[Query files for table 2<br/>🔴 ANOTHER blocking query]
    QueryFiles2 --> BlockingExecute3[cursor.execute - SYNC<br/>⚠️ Blocks 150-300ms]
    BlockingExecute3 --> LoopStart
    
    LoopStart -->|Table N| QueryFilesN[Query files for table N<br/>🔴 ANOTHER blocking query]
    QueryFilesN --> BlockingExecuteN[cursor.execute - SYNC<br/>⚠️ Blocks 150-300ms]
    BlockingExecuteN --> LoopStart
    
    LoopStart -->|Done| ReturnTables[Return tables_with_files<br/>file_processing_service.py:268]
    
    ReturnTables --> APIResponse[API Response<br/>file_processing.py:175-176]
    
    APIResponse --> End([Client Receives Response])
    
    %% ISSUE 3: Long-term Architecture
    Start -.->|🔴 ISSUE 3: ROOT CAUSE| ArchIssue[ARCHITECTURAL PROBLEM:<br/>Synchronous psycopg2 driver<br/>in async framework<br/>postgresql_provider.py:48-51]
    
    ArchIssue -.-> Impact1[Impact: Fake async pattern<br/>async def functions contain<br/>blocking synchronous I/O]
    
    Impact1 -.-> Impact2[Result: Event loop blocked<br/>Other requests cannot be processed<br/>Poor concurrency performance]
    
    Impact2 -.-> Solution[Solution: Migrate to<br/>asyncpg or psycopg3 async<br/>for true async I/O]
    
    %% Styling
    classDef issueNode fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    classDef blockingNode fill:#ff922b,stroke:#d9480f,stroke-width:2px,color:#fff
    classDef normalNode fill:#4dabf7,stroke:#1971c2,stroke-width:2px,color:#fff
    classDef archNode fill:#9775fa,stroke:#5f3dc4,stroke-width:2px,color:#fff
    
    class BlockingExecute1,BlockingExecute2,BlockingExecute3,BlockingExecuteN issueNode
    class FetchAll1,FetchAll2,QueryFiles2,QueryFilesN blockingNode
    class ArchIssue,Impact1,Impact2,Solution archNode
    
    %% Box groupings
    subgraph "ISSUE 1: Blocking I/O in async function"
        BlockingExecute1
        FetchAll1
    end
    
    subgraph "ISSUE 2: N+1 Query Pattern - Sequential Blocking"
        BlockingExecute2
        BlockingExecute3
        BlockingExecuteN
    end
    
    subgraph "ISSUE 3: Root Architectural Problem"
        ArchIssue
        Impact1
        Impact2
        Solution
    end'''
