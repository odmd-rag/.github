# RAG Pipeline Schema Contracts

This document describes the Schema-as-Artifact pattern implemented across the RAG pipeline to ensure robust data contract validation between services while maintaining zero coupling.

## Overview

The RAG pipeline implements a sophisticated schema management system where:
- **Producers** deploy their data schemas as S3 artifacts during service deployment
- **Consumers** download and validate against these schemas at runtime
- **Zero coupling** is maintained - schema changes don't require contractsLib updates
- **Version consistency** is ensured through Git SHA-based schema versioning

## Architecture Pattern

### Schema-as-Artifact Flow
```
Producer Service Build Time:
1. Generate JSON schemas from TypeScript/Zod definitions (src/schemas/)
2. Get current Git SHA via execSync('git rev-parse HEAD')
3. Deploy schemas to S3 using BucketDeployment: schema/{schemaName}/{gitSHA}.json
4. Share schema S3 URL through contractsLib OdmdCrossRefProducer

Consumer Service Build Time:
1. Receive schema S3 URL from contractsLib contract resolution
2. Generate TypeScript types from downloaded schema (design-time)

Consumer Service Runtime:
1. Download schema from S3 URL
2. Validate incoming data against schema
3. Process validated data with type safety
```

### Storage Pattern
- **Bucket Reuse**: Existing service buckets are reused with `schema/{schemaName}/{gitSHA}.json` key pattern
- **Versioning**: Git SHA ensures schema immutability and traceability
- **Namespacing**: Schema name prevents conflicts within the same bucket

## Implementation Details

### Producer Service Implementation

#### 1. Schema Definition (src/schemas/)
```typescript
// src/schemas/processed-content.schema.ts
import { z } from 'zod';

export const ProcessedContentSchema = z.object({
  documentId: z.string(),
  title: z.string(),
  chunks: z.array(z.object({
    id: z.string(),
    content: z.string(),
    metadata: z.record(z.unknown()).optional()
  })),
  processingTimestamp: z.string(),
  totalChunks: z.number()
});

export type ProcessedContent = z.infer<typeof ProcessedContentSchema>;
```

#### 2. Schema Generation Script
```typescript
// scripts/generate-schemas.ts
import { execSync } from 'child_process';
import { writeFileSync } from 'fs';
import { zodToJsonSchema } from 'zod-to-json-schema';
import { ProcessedContentSchema } from '../src/schemas/processed-content.schema';

const gitSha = execSync('git rev-parse HEAD').toString().trim();

const jsonSchema = zodToJsonSchema(ProcessedContentSchema, {
  name: 'ProcessedContent',
  $refStrategy: 'none'
});

writeFileSync(
  `dist/schemas/processed-content-${gitSha}.json`,
  JSON.stringify(jsonSchema, null, 2)
);
```

#### 3. CDK Stack Integration
```typescript
// lib/document-processing-stack.ts
import { BucketDeployment, Source } from 'aws-cdk-lib/aws-s3-deployment';
import { execSync } from 'child_process';

export class DocumentProcessingStack extends Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);

    const gitSha = execSync('git rev-parse HEAD').toString().trim();
    
    // Deploy schema to S3
    const schemaDeployment = new BucketDeployment(this, 'ProcessedContentSchemaDeployment', {
      sources: [Source.asset('dist/schemas')],
      destinationBucket: this.processedContentBucket,
      destinationKeyPrefix: `schema/processed-content/${gitSha}`,
      retainOnDelete: true // Keep schemas for audit trail
    });

    // Schema URL for sharing
    const schemaS3Url = `s3://${this.processedContentBucket.bucketName}/schema/processed-content/${gitSha}/processed-content-${gitSha}.json`;
    
    // Share through contractsLib
    this.processedContentStorage.addSchemaUrl(schemaS3Url);
  }
}
```

#### 4. ContractsLib Producer Update
```typescript
// contractsLib-rag/src/services/document-processing.ts
export class ProcessedContentStorageProducer extends OdmdCrossRefProducer<RagDocumentProcessingEnver> {
  constructor(owner: RagDocumentProcessingEnver, id: string) {
    super(owner, id, {
      children: [
        { pathPart: 'processed-content-bucket' },
        { pathPart: 'processed-content-schema-s3-url' } // New schema URL sharing
      ]
    });
  }

  public get processedContentBucket() {
    return this.children![0]!;
  }

  public get processedContentSchemaS3Url() {
    return this.children![1]!;
  }
}
```

### Consumer Service Implementation

#### 1. Design-Time Code Generation
```typescript
// scripts/generate-types-from-schemas.ts
import { execSync } from 'child_process';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';

async function generateTypes() {
  // Get schema URL from contracts
  const contracts = new RagContracts();
  const processingEnver = contracts.ragDocumentProcessingBuild.dev;
  const schemaUrl = await processingEnver.processedContentStorage.processedContentSchemaS3Url.resolve();
  
  // Download schema
  const s3 = new S3Client({});
  const [bucket, key] = schemaUrl.replace('s3://', '').split('/', 2);
  const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
  const schema = JSON.parse(await response.Body!.transformToString());
  
  // Generate TypeScript types using json-schema-to-typescript
  const types = await compile(schema, 'ProcessedContent');
  writeFileSync('src/types/generated/processed-content.ts', types);
}
```

#### 2. Runtime Schema Validator
```typescript
// src/utils/schema-validator.ts
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import Ajv from 'ajv';

export class SchemaValidator {
  private ajv = new Ajv();
  private schemaCache = new Map<string, any>();

  async validateProcessedContent(data: unknown, schemaUrl: string): Promise<ProcessedContent> {
    let schema = this.schemaCache.get(schemaUrl);
    
    if (!schema) {
      // Download schema from S3
      const s3 = new S3Client({});
      const [bucket, key] = schemaUrl.replace('s3://', '').split('/', 2);
      const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
      schema = JSON.parse(await response.Body!.transformToString());
      this.schemaCache.set(schemaUrl, schema);
    }

    const validate = this.ajv.compile(schema);
    if (!validate(data)) {
      throw new Error(`Schema validation failed: ${JSON.stringify(validate.errors)}`);
    }

    return data as ProcessedContent;
  }
}
```

#### 3. Handler Integration
```typescript
// src/handlers/embedding-processor.ts
import { SchemaValidator } from '../utils/schema-validator';

const validator = new SchemaValidator();

export const handler = async (event: SQSEvent) => {
  const contracts = new RagContracts();
  const processingEnver = contracts.ragDocumentProcessingBuild.dev;
  const schemaUrl = await processingEnver.processedContentStorage.processedContentSchemaS3Url.resolve();

  for (const record of event.Records) {
    const s3Event = JSON.parse(record.body);
    
    // Download and validate processed content
    const processedContent = await validator.validateProcessedContent(
      downloadedData, 
      schemaUrl
    );
    
    // Process with full type safety
    await generateEmbeddings(processedContent);
  }
};
```

## Complete Data Contract Specifications

### 1. Processing → Embedding Service
- **Producer**: `rag-document-processing-service`
- **Consumer**: `rag-embedding-service`
- **Schema**: `ProcessedContent`
- **Location**: `s3://{processedContentBucket}/schema/processed-content/{gitSHA}/processed-content-{gitSHA}.json`
- **Contract Path**: `ragDocumentProcessingBuild.dev.processedContentStorage.processedContentSchemaS3Url`

```typescript
interface ProcessedContent {
  documentId: string;
  title: string;
  chunks: Array<{
    id: string;
    content: string;
    metadata?: Record<string, unknown>;
  }>;
  processingTimestamp: string;
  totalChunks: number;
}
```

### 2. Embedding → Vector Storage Service
- **Producer**: `rag-embedding-service`
- **Consumer**: `rag-vector-storage-service`
- **Schema**: `EmbeddingStatus`
- **Location**: `s3://{embeddingsBucket}/schema/embedding-status/{gitSHA}/embedding-status-{gitSHA}.json`
- **Contract Path**: `ragEmbeddingBuild.dev.embeddingStorage.embeddingStatusSchemaS3Url`

```typescript
interface EmbeddingStatus {
  documentId: string;
  totalChunks: number;
  completedChunks: number;
  chunkReferences: Array<{
    chunkId: string;
    embeddingS3Key: string;
    contentS3Key: string;
  }>;
  embeddingModel: string;
  processingTimestamp: string;
}
```

### 3. Vector Storage → Home Vector Server
- **Producer**: `rag-vector-storage-service`
- **Consumer**: `home-vector-server`
- **Schema**: `UpsertRequest`
- **Location**: `s3://{vectorMetadataBucket}/schema/upsert-request/{gitSHA}/upsert-request-{gitSHA}.json`
- **Contract Path**: `ragVectorStorageBuild.dev.vectorStorage.upsertRequestSchemaS3Url`

```typescript
interface UpsertRequest {
  chunks: Array<{
    id: string;
    content: string;
    embedding: number[];
    metadata: {
      documentId: string;
      chunkId: string;
      title?: string;
      [key: string]: unknown;
    };
  }>;
}
```

### 4. Vector Storage Final Receipt
- **Producer**: `rag-vector-storage-service`
- **Consumer**: Status tracking systems
- **Schema**: `VectorMetadata`
- **Location**: `s3://{vectorMetadataBucket}/schema/vector-metadata/{gitSHA}/vector-metadata-{gitSHA}.json`
- **Contract Path**: `ragVectorStorageBuild.dev.vectorStorage.vectorMetadataSchemaS3Url`

```typitten
interface VectorMetadata {
  documentId: string;
  totalVectors: number;
  vectorDimensions: number;
  processingTimestamp: string;
  homeServerEndpoint: string;
  status: 'completed' | 'failed';
  errorMessage?: string;
}
```

## Implementation Phases

### Phase 1: Processing → Embedding Contract
1. Update `rag-document-processing-service`:
   - Create `src/schemas/processed-content.schema.ts`
   - Add schema generation script to build process
   - Update CDK stack with BucketDeployment
   - Update contractsLib producer with schema URL

2. Update `rag-embedding-service`:
   - Add design-time type generation
   - Implement runtime schema validation
   - Update handler to use validated types

### Phase 2: Embedding → Vector Storage Contract
1. Update `rag-embedding-service` (producer role)
2. Update `rag-vector-storage-service` (consumer role)

### Phase 3: Vector Storage → Home Server Contract
1. Update `rag-vector-storage-service` (producer role)
2. Update `home-vector-server` (consumer role)

### Phase 4: Vector Storage Final Receipt
1. Complete `rag-vector-storage-service` final status schema

## Benefits

### 1. **Zero Coupling**
- Schema changes don't require contractsLib updates
- Services can evolve schemas independently
- No compilation dependencies between services

### 2. **Version Consistency**
- Git SHA ensures schema immutability
- Deployment traceability to exact source code
- Audit trail of schema evolution

### 3. **Type Safety**
- Design-time code generation provides full IntelliSense
- Runtime validation catches contract violations
- Compile-time errors for schema mismatches

### 4. **Operational Excellence**
- Schemas deployed with service deployments
- No separate schema deployment pipeline needed
- BucketDeployment handles deployment atomicity

## Best Practices

### 1. **Schema Evolution**
- Use semantic versioning in schema names for major changes
- Maintain backward compatibility when possible
- Document breaking changes in schema comments

### 2. **Performance**
- Cache downloaded schemas in memory
- Use schema validation only at service boundaries
- Consider schema pre-warming for critical paths

### 3. **Error Handling**
- Provide detailed validation error messages
- Log schema validation failures for debugging
- Implement fallback behavior for schema download failures

### 4. **Security**
- Use IAM roles for cross-service S3 access
- Encrypt schemas at rest and in transit
- Audit schema access patterns

## Troubleshooting

### Common Issues

1. **Schema Not Found**
   - Verify BucketDeployment completed successfully
   - Check Git SHA matches between producer and consumer
   - Confirm S3 bucket permissions

2. **Validation Failures**
   - Compare actual data structure with schema
   - Check for optional vs required field mismatches
   - Verify data type compatibility

3. **Type Generation Issues**
   - Ensure schema download succeeds during build
   - Check json-schema-to-typescript compatibility
   - Verify generated types compile correctly

## Future Enhancements

1. **Schema Registry**: Centralized schema discovery and governance
2. **Schema Linting**: Automated schema quality checks
3. **Breaking Change Detection**: Automated compatibility analysis
4. **Performance Monitoring**: Schema validation metrics and alerting 