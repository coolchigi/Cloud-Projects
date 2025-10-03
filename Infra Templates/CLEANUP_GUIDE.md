# Cleanup Guide - Part 1 (Manual Console Setup)

## Correct Deletion Order

Follow this exact sequence to avoid errors:

### 1. Change Data Deletion Policy (CRITICAL FIRST STEP)

**Before deleting anything, update the data deletion policy to RETAIN:**

1. Go to Bedrock Console → Knowledge bases
1. Select your knowledge base
1. In the **Data source** section, select the data source
1. Click **Edit**
1. Expand **Advanced settings**
1. Change **Data deletion policy** from `DELETE` to `RETAIN`
1. Click **Submit**

**Why this matters:** If you delete the vector store before the knowledge base, deletion will fail. Setting to RETAIN prevents automatic data deletion attempts.

### 2. Delete the Data Source

1. In Bedrock Console → Knowledge bases
1. Select your knowledge base
1. In the **Data source** section, select the data source
1. Click **Delete**
1. Confirm deletion

### 3. Delete the Knowledge Base

1. In Bedrock Console → Knowledge bases
1. Select your knowledge base
1. Click **Delete**
1. Type `delete` to confirm
1. Click **Delete**

### 4. Delete the Vector Store

**For S3 Vectors:**

1. Go to S3 Console → Buckets
1. Find the S3 vector bucket (created by Bedrock, usually named with “bedrock” prefix)
1. Select it and click **Delete**
1. Follow deletion prompts

**For OpenSearch Serverless:**

1. Go to OpenSearch Service Console → Serverless → Collections
1. Select your collection
1. Click **Delete**
1. Type collection name to confirm
1. Click **Delete**

### 5. Delete Security Policies (OpenSearch only)

1.
