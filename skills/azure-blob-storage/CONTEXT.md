# Azure Blob Storage

Comprehensive guide for Azure Blob Storage operations, SAS tokens, and patterns.

## When to Apply

- Storing unstructured data (files, images, videos)
- Generating SAS tokens for secure access
- Configuring lifecycle management
- Optimizing upload/download performance
- Setting up CDN integration

## Storage Account Structure

```
Storage Account
├── Container 1 (e.g., "uploads")
│   ├── Blob 1
│   ├── Blob 2
│   └── virtual-folder/
│       └── Blob 3
└── Container 2 (e.g., "public")
    └── ...
```

## SDK Setup

### JavaScript/TypeScript
```typescript
import { BlobServiceClient, ContainerClient } from "@azure/storage-blob";

// Connection string
const blobServiceClient = BlobServiceClient.fromConnectionString(
  process.env.AZURE_STORAGE_CONNECTION_STRING
);

// DefaultAzureCredential (recommended for Azure)
import { DefaultAzureCredential } from "@azure/identity";

const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  new DefaultAzureCredential()
);
```

### Python
```python
from azure.storage.blob import BlobServiceClient, BlobClient
from azure.identity import DefaultAzureCredential

# Connection string
blob_service_client = BlobServiceClient.from_connection_string(
    os.environ["AZURE_STORAGE_CONNECTION_STRING"]
)

# DefaultAzureCredential
blob_service_client = BlobServiceClient(
    f"https://{account_name}.blob.core.windows.net",
    credential=DefaultAzureCredential()
)
```

## Basic Operations

### Upload Blob
```typescript
async function uploadBlob(
  containerName: string,
  blobName: string,
  content: Buffer | string
): Promise<void> {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  
  await blockBlobClient.upload(content, content.length, {
    blobHTTPHeaders: {
      blobContentType: "application/json",
    },
    metadata: {
      uploadedBy: "user123",
      version: "1.0",
    },
  });
}

// Upload from file
async function uploadFile(containerName: string, blobName: string, filePath: string) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  
  await blockBlobClient.uploadFile(filePath);
}

// Upload from stream
async function uploadStream(containerName: string, blobName: string, stream: Readable) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  
  await blockBlobClient.uploadStream(stream, 4 * 1024 * 1024, 20, {
    blobHTTPHeaders: { blobContentType: "application/octet-stream" },
  });
}
```

### Download Blob
```typescript
async function downloadBlob(containerName: string, blobName: string): Promise<Buffer> {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blobClient = containerClient.getBlobClient(blobName);
  
  const downloadResponse = await blobClient.download();
  const chunks: Buffer[] = [];
  
  for await (const chunk of downloadResponse.readableStreamBody!) {
    chunks.push(Buffer.from(chunk));
  }
  
  return Buffer.concat(chunks);
}

// Download to file
async function downloadToFile(containerName: string, blobName: string, filePath: string) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blobClient = containerClient.getBlobClient(blobName);
  
  await blobClient.downloadToFile(filePath);
}
```

### List Blobs
```typescript
async function listBlobs(containerName: string, prefix?: string) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  
  const blobs = [];
  for await (const blob of containerClient.listBlobsFlat({ prefix })) {
    blobs.push({
      name: blob.name,
      contentLength: blob.properties.contentLength,
      contentType: blob.properties.contentType,
      lastModified: blob.properties.lastModified,
    });
  }
  
  return blobs;
}

// List with hierarchy (virtual folders)
async function listBlobsHierarchy(containerName: string, prefix: string) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  
  for await (const item of containerClient.listBlobsByHierarchy("/", { prefix })) {
    if (item.kind === "prefix") {
      console.log("Folder:", item.name);
    } else {
      console.log("Blob:", item.name);
    }
  }
}
```

### Delete Blob
```typescript
async function deleteBlob(containerName: string, blobName: string) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blobClient = containerClient.getBlobClient(blobName);
  
  await blobClient.delete();
}

// Soft delete (if enabled)
async function softDeleteBlob(containerName: string, blobName: string) {
  const blobClient = blobServiceClient
    .getContainerClient(containerName)
    .getBlobClient(blobName);
  
  await blobClient.delete({ deleteSnapshots: "include" });
}
```

## SAS Tokens

### Generate SAS for Blob
```typescript
import { 
  BlobSASPermissions, 
  generateBlobSASQueryParameters,
  StorageSharedKeyCredential 
} from "@azure/storage-blob";

function generateBlobSAS(
  containerName: string,
  blobName: string,
  permissions: string,
  expiresInMinutes: number
): string {
  const sharedKeyCredential = new StorageSharedKeyCredential(
    accountName,
    accountKey
  );
  
  const sasToken = generateBlobSASQueryParameters({
    containerName,
    blobName,
    permissions: BlobSASPermissions.parse(permissions), // "r" for read, "w" for write
    startsOn: new Date(),
    expiresOn: new Date(Date.now() + expiresInMinutes * 60 * 1000),
  }, sharedKeyCredential).toString();
  
  return `https://${accountName}.blob.core.windows.net/${containerName}/${blobName}?${sasToken}`;
}

// Generate upload URL
const uploadUrl = generateBlobSAS("uploads", "file.pdf", "w", 60);

// Generate download URL
const downloadUrl = generateBlobSAS("uploads", "file.pdf", "r", 60);
```

### User Delegation SAS (More Secure)
```typescript
import { DefaultAzureCredential } from "@azure/identity";

async function generateUserDelegationSAS(
  containerName: string,
  blobName: string
): Promise<string> {
  const blobServiceClient = new BlobServiceClient(
    `https://${accountName}.blob.core.windows.net`,
    new DefaultAzureCredential()
  );
  
  // Get user delegation key
  const startsOn = new Date();
  const expiresOn = new Date(Date.now() + 3600 * 1000);
  
  const userDelegationKey = await blobServiceClient.getUserDelegationKey(
    startsOn,
    expiresOn
  );
  
  const sasToken = generateBlobSASQueryParameters({
    containerName,
    blobName,
    permissions: BlobSASPermissions.parse("r"),
    startsOn,
    expiresOn,
  }, userDelegationKey, accountName).toString();
  
  return `https://${accountName}.blob.core.windows.net/${containerName}/${blobName}?${sasToken}`;
}
```

### SAS Permission Reference
| Permission | Code | Description |
|------------|------|-------------|
| Read | r | Read blob content |
| Add | a | Add block to append blob |
| Create | c | Create new blob |
| Write | w | Write to blob |
| Delete | d | Delete blob |
| List | l | List blobs in container |
| Tags | t | Read/write blob tags |

## Large File Uploads

### Parallel Block Upload
```typescript
async function uploadLargeFile(
  containerName: string,
  blobName: string,
  filePath: string
): Promise<void> {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  
  await blockBlobClient.uploadFile(filePath, {
    blockSize: 4 * 1024 * 1024, // 4 MB blocks
    concurrency: 20, // Parallel uploads
    onProgress: (progress) => {
      console.log(`Uploaded ${progress.loadedBytes} bytes`);
    },
  });
}
```

### Chunked Upload with Progress
```typescript
async function uploadWithProgress(
  containerName: string,
  blobName: string,
  data: Buffer,
  onProgress: (percent: number) => void
): Promise<void> {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  
  const blockSize = 4 * 1024 * 1024; // 4 MB
  const blockIds: string[] = [];
  let uploadedBytes = 0;
  
  for (let i = 0; i < Math.ceil(data.length / blockSize); i++) {
    const start = i * blockSize;
    const end = Math.min(start + blockSize, data.length);
    const chunk = data.slice(start, end);
    
    const blockId = Buffer.from(`block-${i.toString().padStart(6, "0")}`).toString("base64");
    blockIds.push(blockId);
    
    await blockBlobClient.stageBlock(blockId, chunk, chunk.length);
    
    uploadedBytes += chunk.length;
    onProgress((uploadedBytes / data.length) * 100);
  }
  
  await blockBlobClient.commitBlockList(blockIds);
}
```

## Lifecycle Management

### Policy Configuration
```json
{
  "rules": [
    {
      "name": "archiveOldBlobs",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        }
      }
    }
  ]
}
```

### Apply via CLI
```bash
az storage account management-policy create \
  --account-name mystorageaccount \
  --resource-group my-rg \
  --policy @lifecycle-policy.json
```

## Access Tiers

| Tier | Use Case | Access Cost | Storage Cost |
|------|----------|-------------|--------------|
| Hot | Frequently accessed | Low | High |
| Cool | Infrequent (30+ days) | Medium | Medium |
| Cold | Rare (90+ days) | High | Low |
| Archive | Long-term (180+ days) | Very High | Very Low |

```typescript
// Set tier on upload
await blockBlobClient.upload(content, content.length, {
  tier: "Cool",
});

// Change tier
await blockBlobClient.setAccessTier("Archive");

// Rehydrate from archive (takes hours)
await blockBlobClient.setAccessTier("Hot", {
  rehydratePriority: "High", // "High" (hours) or "Standard" (up to 15 hours)
});
```

## Event Grid Integration

### Blob Events
```json
{
  "source": "subject",
  "type": "Microsoft.Storage.BlobCreated",
  "data": {
    "api": "PutBlob",
    "clientRequestId": "...",
    "requestId": "...",
    "eTag": "...",
    "contentType": "application/pdf",
    "contentLength": 12345,
    "blobType": "BlockBlob",
    "url": "https://account.blob.core.windows.net/container/blob.pdf"
  }
}
```

### Azure Function Trigger
```typescript
import { app, EventGridEvent, InvocationContext } from "@azure/functions";

app.eventGrid("blobCreated", {
  handler: async (event: EventGridEvent, context: InvocationContext) => {
    const data = event.data as {
      url: string;
      contentType: string;
      contentLength: number;
    };
    
    context.log(`New blob: ${data.url}`);
    
    // Process the blob
    const blobClient = new BlobClient(data.url, new DefaultAzureCredential());
    const content = await blobClient.downloadToBuffer();
    
    // Do something with content
  },
});
```

## CDN Integration

```bash
# Create CDN profile and endpoint
az cdn profile create --name mycdnprofile --resource-group my-rg --sku Standard_Microsoft

az cdn endpoint create \
  --name myendpoint \
  --profile-name mycdnprofile \
  --resource-group my-rg \
  --origin mystorageaccount.blob.core.windows.net \
  --origin-host-header mystorageaccount.blob.core.windows.net

# Access via CDN
# https://myendpoint.azureedge.net/container/blob.jpg
```

## Static Website Hosting

```bash
# Enable static website
az storage blob service-properties update \
  --account-name mystorageaccount \
  --static-website \
  --index-document index.html \
  --404-document 404.html

# Upload website files
az storage blob upload-batch \
  --account-name mystorageaccount \
  --source ./dist \
  --destination '$web'
```

## Common Patterns

### Copy Blob Between Accounts
```typescript
async function copyBlob(
  sourceUrl: string,
  destContainerName: string,
  destBlobName: string
): Promise<void> {
  const destClient = blobServiceClient
    .getContainerClient(destContainerName)
    .getBlobClient(destBlobName);
  
  const copyPoller = await destClient.beginCopyFromURL(sourceUrl);
  await copyPoller.pollUntilDone();
}
```

### Batch Delete
```typescript
async function batchDelete(containerName: string, blobNames: string[]) {
  const containerClient = blobServiceClient.getContainerClient(containerName);
  
  const batchClient = containerClient.getBlobBatchClient();
  const response = await batchClient.deleteBlobs(
    blobNames.map(name => containerClient.getBlobClient(name))
  );
  
  console.log(`Deleted ${response.subResponses.length} blobs`);
}
```

## CLI Reference

```bash
# List containers
az storage container list --account-name mystorageaccount

# List blobs
az storage blob list --container-name mycontainer --account-name mystorageaccount

# Upload
az storage blob upload --file ./file.txt --container-name mycontainer --name file.txt

# Download
az storage blob download --container-name mycontainer --name file.txt --file ./downloaded.txt

# Generate SAS
az storage blob generate-sas \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --name file.txt \
  --permissions r \
  --expiry 2024-12-31

# Set access tier
az storage blob set-tier --container-name mycontainer --name file.txt --tier Cool
```

## References

- [Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Storage SDK for JavaScript](https://learn.microsoft.com/en-us/javascript/api/@azure/storage-blob/)
- [SAS Tokens](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Lifecycle Management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
