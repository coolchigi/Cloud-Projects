# Deployment Guide - Part 2 (CloudFormation Automation)

## Overview

This guide shows you how to deploy both S3 Vectors and OpenSearch Serverless Knowledge Base instances using CloudFormation templates.

## Prerequisites

1. **AWS Account** with Bedrock access in us-east-1 region
1. **Bedrock Model Access** enabled:
- Go to Bedrock Console → Model access
- Request access for: `amazon.titan-embed-text-v1` (or your chosen embedding model)
- Wait for “Access granted” status
1. **S3 Bucket** with your PDF documents already created and uploaded
1. **AWS CLI** installed (optional, for command-line deployment)
1. **IAM Permissions** to create:
- CloudFormation stacks
- Bedrock resources
- IAM roles
- S3 Vectors OR OpenSearch Serverless resources

-----

## Deployment Option 1: S3 Vectors (Lower Cost)

### Step 1: Download Template

Save `bedrock-kb-s3-vectors.yaml` to your local machine.

### Step 2: Deploy via AWS Console

1. **Open CloudFormation Console:**
- Go to: https://console.aws.amazon.com/cloudformation
- Region: **us-east-1** (N. Virginia)
1. **Create Stack:**
- Click **Create stack** → **With new resources**
- Choose **Upload a template file**
- Click **Choose file** and select `bedrock-kb-s3-vectors.yaml`
- Click **Next**
1. **Specify Stack Details:**
- **Stack name:** `bedrock-kb-s3-vectors-stack` (or your choice)
- **Parameters:**
  - `DataSourceBucketName`: Your S3 bucket name (e.g., `my-pdf-documents`)
  - `KnowledgeBaseName`: `kb-s3-vectors-chat-pdf` (default OK)
  - `EmbeddingModel`: `amazon.titan-embed-text-v1` (default OK)
  - `ChunkMaxTokens`: `512` (default OK)
  - `ChunkOverlapPercentage`: `20` (default OK)
- Click **Next**
1. **Configure Stack Options:**
- Tags: (optional) Add tags for cost tracking
- Permissions: Leave default
- Stack failure options: Leave default
- Click **Next**
1. **Review:**
- Check **I acknowledge that AWS CloudFormation might create IAM resources**
- Click **Submit**
1. **Wait for Completion:**
- Status changes: `CREATE_IN_PROGRESS` → `CREATE_COMPLETE`
- Takes ~3-5 minutes

### Step 3: Sync Data Source

1. **Go to Bedrock Console:**
- Navigate to: Knowledge bases → Your KB
1. **Sync Data:**
- In the **Data source** section, click **Sync**
- Wait for sync to complete (depends on document count/size)
- Status: `Syncing` → `Ready`
1. **Test Knowledge Base:**
- Click **Test knowledge base** button
- Toggle **Generate responses** ON
- Ask a question about your PDFs

### Step 4: Record Outputs

Go to CloudFormation → Stacks → Your stack → **Outputs** tab

Copy these values for cost comparison:

- `KnowledgeBaseId`
- `KnowledgeBaseArn`
- `DataSourceId`

-----

## Deployment Option 2: OpenSearch Serverless (Higher Cost)

### Step 1: Download Template

Save `bedrock-kb-opensearch.yaml` to your local machine.

### Step 2: Deploy via AWS Console

Follow the same console steps as S3 Vectors, but:

- Use `bedrock-kb-opensearch.yaml` template
- Stack name: `bedrock-kb-opensearch-stack`
- Parameters:
  - `DataSourceBucketName`: Your S3 bucket name
  - `KnowledgeBaseName`: `kb-opensearch-chat-pdf`
  - `CollectionName`: `bedrock-kb-collection` (default OK)
  - `VectorIndexName`: `bedrock-vector-index` (default OK)
  - Other parameters: use defaults

### Step 3: Create Vector Index (CRITICAL - Manual Step)

**After stack creation completes, you MUST manually create the vector index:**

1. **Go to OpenSearch Service Console:**
- Navigate to: Serverless → Collections
- Region: **us-east-1**
1. **Open Collection:**
- Click on collection name: `bedrock-kb-collection`
1. **Create Vector Index:**
- Go to **Indexes** tab
- Click **Create vector index**
1. **Configure Index:**
- **Vector index name:** `bedrock-vector-index` (must match template parameter)
- **Vector field details:**
  - **Vector field name:** `bedrock-knowledge-base-default-vector`
  - **Dimensions:**
    - `1536` if using `amazon.titan-embed-text-v1`
    - `1024` if using `amazon.titan-embed-text-v2:0`
    - Check your embedding model’s dimensions
  - **Engine:** `FAISS`
  - **Distance metric:** `Euclidean`
- Click **Create**
1. **Verify Index Created:**
- Index appears in the list
- Status: Active

### Step 4: Sync Data Source

Same as S3 Vectors:

1. Go to Bedrock Console → Knowledge bases
1. Select your KB → Sync
1. Wait for completion
1. Test with a query

### Step 5: Record Outputs

Copy from CloudFormation Outputs:

- `KnowledgeBaseId`
- `OpenSearchCollectionArn`
- `OpenSearchCollectionEndpoint`

-----

## Deployment via AWS CLI

### For S3 Vectors:

```bash
aws cloudformation create-stack \
  --stack-name bedrock-kb-s3-vectors-stack \
  --template-body file://bedrock-kb-s3-vectors.yaml \
  --parameters \
    ParameterKey=DataSourceBucketName,ParameterValue=my-pdf-documents \
    ParameterKey=KnowledgeBaseName,ParameterValue=kb-s3-vectors-chat-pdf \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name bedrock-kb-s3-vectors-stack \
  --region us-east-1

# Get outputs
aws cloudformation describe-stacks \
  --stack-name bedrock-kb-s3-vectors-stack \
  --region us-east-1 \
  --query 'Stacks[0].Outputs'
```

### For OpenSearch Serverless:

```bash
aws cloudformation create-stack \
  --stack-name bedrock-kb-opensearch-stack \
  --template-body file://bedrock-kb-opensearch.yaml \
  --parameters \
    ParameterKey=DataSourceBucketName,ParameterValue=my-pdf-documents \
    ParameterKey=KnowledgeBaseName,ParameterValue=kb-opensearch-chat-pdf \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# Wait and get outputs (same as above)
```

**Remember:** You still need to manually create the vector index for OpenSearch!

-----

## Customizing the Templates

### Change Embedding Model:

Both templates support multiple models. To use a different one:

1. Update `EmbeddingModel` parameter:
   
   ```yaml
   EmbeddingModel: cohere.embed-multilingual-v3
   ```
1. **IMPORTANT:** Update vector index dimensions for OpenSearch:
- Titan V1: 1536 dimensions
- Titan V2: 1024, 512, or 256 dimensions
- Cohere: 1024 dimensions

### Change Chunking Strategy:

To use different chunking (e.g., semantic):

**Not supported in these templates.** Fixed-size chunking is hardcoded for simplicity. To use other strategies:

- Deploy manually via console, or
- Modify the template’s `VectorIngestionConfiguration` section

### Add Custom Metadata:

Add to the `DataSourceConfiguration` in templates:

```yaml
S3Configuration:
  BucketArn: !Sub 'arn:aws:s3:::${DataSourceBucketName}'
  BucketOwnerAccountId: !Ref AWS::AccountId
  # Add inclusion prefixes to filter files
  InclusionPrefixes:
    - "documents/pdfs/"
```

-----

## Testing Both Instances

### Perform Identical Tests:

1. **Same Questions:** Ask both KBs the same questions
1. **Record Metrics:**
- Response accuracy
- Response time
- Source chunks retrieved
1. **Cost Comparison:**
- Use AWS Cost Explorer
- Filter by tags or resource names
- Compare over 24 hours, 7 days, 30 days

### Sample Test Questions:

```
1. "Summarize the key points from the document"
2. "What does section X say about topic Y?"
3. "List all recommendations mentioned"
4. "Compare the findings between chapter A and B"
```

-----

## Cleanup - CloudFormation Method

### Delete S3 Vectors Stack:

```bash
# Via Console:
# CloudFormation → Stacks → Select stack → Delete

# Via CLI:
aws cloudformation delete-stack \
  --stack-name bedrock-kb-s3-vectors-stack \
  --region us-east-1
```

**Note:** Stack deletion will automatically delete:

- Knowledge Base
- Data Source (with RETAIN policy, so vector data stays)
- IAM Role
- S3 Vector bucket (if created by CloudFormation)

### Delete OpenSearch Stack:

**BEFORE deleting stack:**

1. **Manually delete vector index:**
- OpenSearch Console → Collections → Your collection
- Indexes tab → Delete index
1. **Then delete stack:**

```bash
aws cloudformation delete-stack \
  --stack-name bedrock-kb-opensearch-stack \
  --region us-east-1
```

Stack deletion removes:

- Knowledge Base
- Data Source
- OpenSearch Collection
- Security policies
- IAM Role

-----

## Troubleshooting

### “No model access” Error:

**Solution:** Enable model access in Bedrock Console first.

### “Bucket not found” Error:

**Cause:** S3 bucket doesn’t exist or wrong name.

**Solution:** Verify bucket name and ensure it’s in us-east-1.

### OpenSearch: Stack created but KB failing:

**Cause:** Vector index not created.

**Solution:** Follow Step 3 to manually create index.

### IAM Role Creation Failed:

**Cause:** Insufficient permissions or role name conflict.

**Solution:**

- Check you have IAM role creation permissions
- Delete conflicting role if exists
- Retry stack creation

### Stack DELETE_FAILED:

**Cause:** Resources in use or manual deletions.

**Solution:**

1. Check which resource failed (CloudFormation → Events)
1. Manually delete that resource
1. Retry stack deletion

-----

## Cost Estimation (Approximate)

### S3 Vectors:

- Vector storage: ~$0.023/GB/month
- Embedding generation: ~$0.10 per 1M tokens
- Queries: ~$0.0025 per 10K queries

### OpenSearch Serverless:

- OCU (compute): ~$0.24/OCU-hour (minimum 4 OCUs = ~$691/month)
- Storage: ~$0.024/GB/month
- Embedding: Same as S3

**Expected Savings:** S3 Vectors typically 60-80% cheaper for storage-heavy use cases.

-----

## Next Steps

After deployment and testing:

1. **Document cost findings** for your blog
1. **Take screenshots** of console outputs
1. **Record actual costs** from Cost Explorer
1. **Clean up resources** when done testing
1. **Write Part 2 blog post** with automation benefits