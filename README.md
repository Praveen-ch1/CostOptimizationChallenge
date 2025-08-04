This solution to optimize costs for your Azure serverless architecture with billing records in Cosmos DB. The key aspects are:
1.Multi-tier Storage Architecture: Implement a hot/cold storage pattern where frequently accessed data (last 3 months) stays in Cosmos DB, while older data moves to cheaper storage.
2.Azure Blob Storage for Archival: Move records older than 3 months to Azure Blob Storage (cool/archive tier) which is significantly cheaper for rarely accessed data.
3.Metadata Index in Cosmos DB: Maintain minimal metadata for all records in Cosmos DB to enable fast lookups, with pointers to the actual data in Blob Storage.
4.Serverless Retrieval Process: Use Azure Functions to seamlessly fetch data from either Cosmos DB or Blob Storage when requested.

**Detailed Solution Components:**
1. Data Partitioning Strategy:

```javascript
// Pseudocode for data partitioning logic
function determineStorageLocation(record) {
    if (record.timestamp > currentDate - 90 days) {
        return "cosmos";  // Hot tier
    } else {
        return "blob";    // Cold tier
    }
} 
```

2. Archival Process Implementation :
```pseudocode
// Pseudocode for archival function
function archiveOldRecords():
    cutoffDate = currentDate - 90 days
    oldRecords = cosmosDB.query("SELECT * FROM records WHERE timestamp < @cutoff", {cutoff: cutoffDate})
    
    foreach record in oldRecords:
        // Upload full record to blob storage
        blobUrl = blobStorage.upload(record.id, record.data)
        
        // Update Cosmos DB with metadata only
        cosmosDB.update(record.id, {
            metadata: record.metadata,
            blobUrl: blobUrl,
            isArchived: true
        })
```

3. Retrieval Process :
```pseudocode
// Pseudocode for record retrieval
function getRecord(recordId):
    record = cosmosDB.get(recordId)
    
    if record.isArchived:
        // Fetch from blob storage if archived
        data = blobStorage.download(record.blobUrl)
        return { ...record.metadata, data: data }
    else:
        // Return directly from Cosmos if recent
        return record
```
**Architecture Diagram:**
```text
┌───────────────────────────────────────────────────────┐
│                   Client Application                  │
└───────────────┬───────────────────────┬───────────────┘
                │                       │
                │ GET /records/{id}     │
                ▼                       ▼
┌───────────────────────────────────────────────────────┐
│                API Gateway (Existing)                 │
└───────────────┬───────────────────────┬───────────────┘
                │                       │
                │                       │
┌───────────────▼───────┐   ┌───────────▼───────────────┐
│   Azure Function      │   │   Azure Function          │
│   (Record Retrieval)  │   │   (Record Archival)       │
└───────────────┬───────┘   └───────────┬───────────────┘
                │                       │
                │ Query                 │ Archive old
                ▼                       ▼ records
┌───────────────────────────────────────────────────────┐
│               Azure Cosmos DB (Hot Tier)             │
│ - Stores recent records (<90 days) fully             │
│ - Stores metadata for all records                    │
│ - Stores blob pointers for archived records          │
└───────────────┬───────────────────────┬───────────────┘
                │                       │
                │                       │
┌───────────────▼───────┐   ┌───────────▼───────────────┐
│  Azure Blob Storage   │   │    Azure Monitor         │
│  (Cool/Archive Tier)  │   │    - Cost tracking       │
│ - Stores full records │   │    - Access patterns     │
│   older than 90 days  │   └─────────────────────────┘
└───────────────────────┘
```
**Implementation Steps**

**1.Set up Blob Storage Containers:**

```bash
az storage container create --name billing-records --account-name <storageaccount> --auth-mode login
```
**2.Create Archive Function (Timer-triggered):**

```javascript
module.exports = async function (context, myTimer) {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - 90);
    
    // Query old records
    const records = await cosmosClient
        .database("billing")
        .container("records")
        .items.query(`SELECT * FROM c WHERE c.timestamp < ${cutoffDate.getTime()}`)
        .fetchAll();
    
    // Archive each record
    for (const record of records.resources) {
        const blobName = `${record.id}.json`;
        await blockBlobClient.upload(blobName, JSON.stringify(record));
        
        // Update Cosmos with metadata
        const { data, ...metadata } = record;
        await cosmosClient
            .database("billing")
            .container("records")
            .item(record.id)
            .replace({
                ...metadata,
                blobUrl: blobName,
                isArchived: true
            });
    }
}
```
**3.Modify Retrieval Function:**

```javascript
module.exports = async function (context, req) {
    const recordId = req.params.id;
    
    // Get record from Cosmos
    const { resource: record } = await cosmosClient
        .database("billing")
        .container("records")
        .item(recordId)
        .read();
    
    if (record.isArchived) {
        // Fetch from blob storage
        const blobClient = containerClient.getBlockBlobClient(record.blobUrl);
        const data = await blobClient.downloadToString();
        return { ...record, data: JSON.parse(data) };
    }
    
    return record;
}
```
**Cost Optimization Analysis**
```
| Storage Option          | Cost per GB/month | Ideal For                          |
|-------------------------|-------------------|------------------------------------|
| Cosmos DB (Hot)         | ~$25              | Recent (<90d), frequent access     |
| Blob Storage Cool       | ~$0.01            | Older data, rare access            |
| Blob Storage Archive    | ~$0.00099         | Very old, almost never accessed    |

```

```
## Estimated Savings
- Assumptions:
  - 2M records at 300KB each = ~600GB total
  - 50% is >90 days old: 300GB can move to Blob Cool
- Cost Analysis:
  - Cosmos savings: 300GB * $25 = $7,500/month
  - Blob cost: 300GB * $0.01 = $3/month
  - **Net savings**: ~$7,497/month

## Failure Scenarios and Mitigations

### 1. Blob Storage Unavailable
-  Implement retry logic with exponential backoff
-  Consider keeping a small cache of recently accessed archived records in Cosmos

### 2. Archival Process Fails
-  Implement dead-letter queue for failed archival operations
-  Set up alerts for archival failures

### 3. Increased Latency for Archived Records
-  For archive tier, implement async retrieval pattern with notifications
-  Consider pre-warming cache for anticipated access patterns

### 4. Data Consistency Issues
-  Implement two-phase updates (write to blob before updating Cosmos metadata)
-  Add verification steps to ensure data integrity

## Deployment Strategy

1. **Phase 1**: Dual-write new records to both Cosmos and Blob Storage (shadow mode)
2. **Phase 2**: Implement retrieval logic to check both sources
3. **Phase 3**: Backfill archival for existing old records
4. **Phase 4**: Fully switch to new model and clean up old data

## Monitoring Recommendations

- Track ratio of hot vs. cold reads
- Monitor latency percentiles for archived record retrieval
- Set up cost alerts for both Cosmos and Blob Storage
- Track archival job completion metrics and failures

## Solution Benefits

 **No API contract changes**  
 **No data loss**  
 **No downtime during implementation**  
 **Simple to maintain**  
 **Meets latency requirements for archived data**
```
