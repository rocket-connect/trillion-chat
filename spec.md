# Infinite-Context Chat Storage Specification

**Version:** 1.1  
**Date:** November 2025  
**License:** MIT

## Abstract

A specification for managing unlimited-length chat conversations with Large Language Models using graph database storage, vector embeddings, and intelligent context retrieval. This system enables conversations to persist indefinitely without performance degradation or context loss.

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Data Model](#3-data-model)
4. [Index Strategies](#4-index-strategies)
5. [Tool Interface](#5-tool-interface)
6. [Implementation Guide](#6-implementation-guide)
7. [Configuration](#7-configuration)
8. [Performance Considerations](#8-performance-considerations)

## 1. Overview

### 1.1 Problem Statement

LLM context windows are finite resources. Traditional chat implementations send entire conversation histories with each request, leading to:

- Linear growth in token costs
- Hard limits on conversation length
- Wasted tokens on irrelevant historical messages
- Forced conversation splits or truncation

### 1.2 Solution

Store all messages in a graph database with vector embeddings, presenting the LLM with a navigable index of available context. The LLM retrieves only what it needs through tool calls.

### 1.3 Key Principles

- **Retrievable State**: Messages are mutable and editable
- **Unlimited Storage**: No artificial limits on message count
- **Selective Loading**: LLM decides what context to retrieve
- **Adaptive Indexing**: Different strategies for different scales

## 2. Architecture

### 2.1 Components

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Message   │────▶│   Embedding  │────▶│   Vector    │
│   Input     │     │   Model      │     │   Index     │
└─────────────┘     └──────────────┘     └─────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │  Neo4j Graph │
                    │   Database   │
                    └──────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │    Index     │
                    │   Builder    │
                    └──────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │      LLM     │
                    │  + Tools     │
                    └──────────────┘
```

### 2.2 Flow

1. User sends message
2. System generates embedding and stores in graph
3. System searches for relevant historical context
4. Index builder creates navigable context map
5. LLM receives index + recent messages + current message
6. LLM uses tools to retrieve specific messages as needed
7. LLM generates response

## 3. Data Model

### 3.1 Message Node

```cypher
CREATE (m:Message {
  id: String,                 // ULID or UUID
  content: String,            // Full message text
  snippet: String,            // Preview text
  role: String,               // 'user' | 'assistant' | 'system'
  timestamp: DateTime,        // ISO-8601
  parent_id: String,          // Parent message ID (nullable)
  embedding: List<Float>,     // Vector (1536 dimensions)
  token_count: Integer,       // Approximate tokens
  metadata: Map,              // Extensible metadata
  edited: Boolean,            // Edit flag
  deleted: Boolean,           // Soft delete flag
  edit_history: List<Map>,    // Previous versions
  is_chunk: Boolean,          // True if part of chunked content
  chunk_index: Integer,       // Position in sequence (nullable)
  chunk_parent_id: String     // Original message ID (nullable)
})
```

### 3.2 Tool Call Node

```cypher
CREATE (t:ToolCall {
  id: String,                 // ULID or UUID
  tool_name: String,          // Tool identifier
  arguments: String,          // JSON arguments
  result: String,             // JSON result
  timestamp: DateTime,        // ISO-8601
  message_id: String,         // Triggering message
  embedding: List<Float>,     // Vector embedding (required)
  token_count: Integer,       // Approximate tokens
  is_chunk: Boolean,          // True if result was chunked
  chunk_index: Integer,       // Position in sequence (nullable)
  chunk_parent_id: String     // Original tool call ID (nullable)
})
```

### 3.3 Content Chunking Strategy

For messages or tool results exceeding a configured threshold (default: 4000 tokens), implementations should chunk the content:

**Chunking Rules:**

- Each chunk maintains the same role, timestamp, and parent relationships
- Chunks are linked via `chunk_parent_id` to the logical message/tool call
- `chunk_index` indicates position (0-based)
- Each chunk receives its own embedding for granular retrieval
- Snippet is generated from first chunk only

**Retrieval Behavior:**

- Tools can retrieve individual chunks or all chunks of a parent
- Index presents chunks as separate searchable units
- LLM decides whether to load full content or specific chunks

**Example Chunking:**

```
Original: 10,000 token assistant response
Becomes:
  - Chunk 0: Tokens 0-4000 (chunk_parent_id: original_id)
  - Chunk 1: Tokens 4000-8000 (chunk_parent_id: original_id)
  - Chunk 2: Tokens 8000-10000 (chunk_parent_id: original_id)
```

### 3.4 Relationships

```cypher
// Message chain
(child:Message)-[:REPLIES_TO]->(parent:Message)

// Chunk relationships
(chunk:Message)-[:CHUNK_OF]->(parent:Message)

// Tool calls
(tool:ToolCall)-[:CALLED_BY]->(message:Message)

// Tool chunk relationships
(chunk:ToolCall)-[:CHUNK_OF]->(parent:ToolCall)

// Topic clustering (optional, implementation-defined)
(message:Message)-[:BELONGS_TO]->(topic:Topic)
```

### 3.5 Indexes

```cypher
// Unique constraints
CREATE CONSTRAINT message_id_unique
FOR (m:Message) REQUIRE m.id IS UNIQUE;

CREATE CONSTRAINT toolcall_id_unique
FOR (t:ToolCall) REQUIRE t.id IS UNIQUE;

// Performance indexes
CREATE INDEX message_timestamp
FOR (m:Message) ON (m.timestamp);

CREATE INDEX message_role
FOR (m:Message) ON (m.role);

CREATE INDEX message_chunk_parent
FOR (m:Message) ON (m.chunk_parent_id);

CREATE INDEX toolcall_timestamp
FOR (t:ToolCall) ON (t.timestamp);

CREATE INDEX toolcall_chunk_parent
FOR (t:ToolCall) ON (t.chunk_parent_id);

// Vector indexes
CREATE VECTOR INDEX message_embeddings
FOR (m:Message) ON (m.embedding)
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 1536,
    `vector.similarity_function`: 'cosine'
  }
};

CREATE VECTOR INDEX toolcall_embeddings
FOR (t:ToolCall) ON (t.embedding)
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 1536,
    `vector.similarity_function`: 'cosine'
  }
};
```

## 4. Index Strategies

### 4.1 Index Configuration

```typescript
interface IndexConfig {
  snippetLength: number; // Characters per snippet (default: 100)
  snippetStrategy: "first" | "semantic_core" | "summary";
  maxIndexTokens: number; // Token budget for index (default: 10000)
  indexStrategy:
    | "adaptive"
    | "recent_plus_relevant"
    | "clustered"
    | "hierarchical";
  clusteringThreshold: number; // Similarity threshold (0.0-1.0)
  minClusterSize: number; // Minimum cluster size
  recentWindowSize: number; // Always-included recent messages
  chunkThreshold: number; // Token count to trigger chunking (default: 4000)
  includeToolCalls: boolean; // Include tool calls in search (default: true)
}
```

### 4.2 Adaptive Strategy

The system automatically selects the appropriate index format based on result count:

| Result Count | Strategy     | Description                       |
| ------------ | ------------ | --------------------------------- |
| 0-50         | Full         | Complete messages shown           |
| 51-500       | Snippet      | Recent full + historical snippets |
| 501-5000     | Clustered    | Semantic groups with summaries    |
| 5000+        | Hierarchical | Multi-level navigation structure  |

### 4.3 Index Format Examples

#### Small Result Set (<50 messages)

```
Recent Conversation:
[user]: What's our API authentication strategy?
[assistant]: We're using OAuth2 with JWT tokens...

Historical Context:
[d8f3a2b1] Discussion about API rate limiting...
[7c4e9f2d] OAuth2 implementation details...
```

#### Medium Result Set (51-500 messages)

```
Recent Conversation:
[last 10 messages shown in full]

Historical Matches (retrieve with get_message_by_id):
- d8f3a2b1 [2024-10-15] "We need to implement rate limiting for..."
- 7c4e9f2d [2024-10-12] "OAuth2 configuration should include..."
[... up to token limit]
```

#### Large Result Set (501-5000 messages)

```
Recent Conversation:
[last 5 messages]

Message Clusters:
Cluster: "API Authentication" (47 messages)
Period: 2024-03-15 to 2024-10-22
Summary: Decisions on OAuth2, JWT tokens, API key support
Sample messages:
  - "OAuth2 implementation with refresh tokens..."
  - "API key fallback for legacy clients..."
Retrieve with: get_cluster("api_auth_cluster_id")

Cluster: "Database Design" (89 messages)
[...]
```

#### Huge Result Set (5000+ messages)

```
Temporal Overview:
  Today: 5 messages
  This Week: 45 messages
  This Month: 423 messages
  Older: 8,234 messages

Topic Overview (This Week):
  "API Development": 12 messages
  "Bug Fixes": 18 messages
  "Architecture": 15 messages

Navigation:
  - get_period_messages('today')
  - get_topic_messages('API Development')
  - vector_search(query, limit)
```

## 5. Tool Interface

### 5.1 Core Tools

```typescript
interface ChatTools {
  // Single message retrieval
  get_message_by_id(id: string): Message;

  // Bulk retrieval
  get_messages_by_ids(ids: string[]): Message[];

  // Chunk-aware retrieval
  get_message_with_chunks(id: string): Message[];

  // Semantic search (includes tool calls if configured)
  vector_search(query: string, limit?: number): SearchResult[];

  // Cluster/group retrieval
  get_cluster(cluster_id: string, limit?: number): Message[];

  // Temporal retrieval
  get_period_messages(
    period: "today" | "this_week" | "this_month" | string,
    limit?: number
  ): Message[];

  // Thread navigation
  get_conversation_thread(message_id: string, depth?: number): Message[];

  // Tool call retrieval
  get_tool_call(id: string): ToolCall;
  get_tool_calls_by_message(message_id: string): ToolCall[];

  // Combined search and retrieve
  search_and_retrieve(query: string, auto_limit: number): Message[];
}
```

### 5.2 Tool Response Format

```typescript
interface Message {
  id: string;
  content: string;
  role: "user" | "assistant" | "system";
  timestamp: string;
  parentId: string | null;
  metadata?: Record<string, any>;
  isChunk?: boolean;
  chunkIndex?: number;
  chunkParentId?: string;
}

interface ToolCall {
  id: string;
  toolName: string;
  arguments: string;
  result: string;
  timestamp: string;
  messageId: string;
  isChunk?: boolean;
  chunkIndex?: number;
  chunkParentId?: string;
}

interface SearchResult {
  id: string;
  snippet: string;
  timestamp: string;
  score: number;
  type: "message" | "tool_call";
  isChunk?: boolean;
}
```

## 6. Implementation Guide

### 6.1 Message Storage

```typescript
async function storeMessage(
  content: string,
  role: "user" | "assistant" | "system",
  parentId: string | null = null,
  config: IndexConfig
): Promise<string> {
  const tokenCount = estimateTokens(content);

  // Check if chunking is needed
  if (tokenCount > config.chunkThreshold) {
    return await storeChunkedMessage(content, role, parentId, config);
  }

  const id = ulid();
  const timestamp = new Date().toISOString();
  const snippet = content.slice(0, config.snippetLength);
  const embedding = await generateEmbedding(content);

  await neo4j.run(
    `
    CREATE (m:Message {
      id: $id,
      content: $content,
      snippet: $snippet,
      role: $role,
      timestamp: datetime($timestamp),
      parent_id: $parentId,
      embedding: $embedding,
      token_count: $tokenCount,
      edited: false,
      is_chunk: false
    })
  `,
    { id, content, snippet, role, timestamp, parentId, embedding, tokenCount }
  );

  if (parentId) {
    await neo4j.run(
      `
      MATCH (child:Message {id: $childId})
      MATCH (parent:Message {id: $parentId})
      CREATE (child)-[:REPLIES_TO]->(parent)
    `,
      { childId: id, parentId }
    );
  }

  return id;
}

async function storeChunkedMessage(
  content: string,
  role: "user" | "assistant" | "system",
  parentId: string | null,
  config: IndexConfig
): Promise<string> {
  const parentMessageId = ulid();
  const chunks = chunkContent(content, config.chunkThreshold);
  const timestamp = new Date().toISOString();
  const snippet = chunks[0].slice(0, config.snippetLength);

  for (let i = 0; i < chunks.length; i++) {
    const chunkId = ulid();
    const chunkContent = chunks[i];
    const embedding = await generateEmbedding(chunkContent);
    const tokenCount = estimateTokens(chunkContent);

    await neo4j.run(
      `
      CREATE (m:Message {
        id: $id,
        content: $content,
        snippet: $snippet,
        role: $role,
        timestamp: datetime($timestamp),
        parent_id: $parentId,
        embedding: $embedding,
        token_count: $tokenCount,
        edited: false,
        is_chunk: true,
        chunk_index: $chunkIndex,
        chunk_parent_id: $chunkParentId
      })
    `,
      {
        id: chunkId,
        content: chunkContent,
        snippet: i === 0 ? snippet : "",
        role,
        timestamp,
        parentId,
        embedding,
        tokenCount,
        chunkIndex: i,
        chunkParentId: parentMessageId,
      }
    );

    // Link chunk to parent
    if (i === 0 && parentId) {
      await neo4j.run(
        `
        MATCH (child:Message {id: $childId})
        MATCH (parent:Message {id: $parentId})
        CREATE (child)-[:REPLIES_TO]->(parent)
      `,
        { childId: chunkId, parentId }
      );
    }
  }

  return parentMessageId;
}
```

### 6.2 Tool Call Storage

```typescript
async function storeToolCall(
  toolName: string,
  arguments: any,
  result: any,
  messageId: string,
  config: IndexConfig
): Promise<string> {
  const resultString = JSON.stringify(result);
  const tokenCount = estimateTokens(resultString);

  // Check if chunking is needed
  if (tokenCount > config.chunkThreshold) {
    return await storeChunkedToolCall(
      toolName,
      arguments,
      resultString,
      messageId,
      config
    );
  }

  const id = ulid();
  const timestamp = new Date().toISOString();
  const embedding = await generateEmbedding(
    `${toolName}: ${JSON.stringify(arguments)} -> ${resultString}`
  );

  await neo4j.run(
    `
    CREATE (t:ToolCall {
      id: $id,
      tool_name: $toolName,
      arguments: $arguments,
      result: $result,
      timestamp: datetime($timestamp),
      message_id: $messageId,
      embedding: $embedding,
      token_count: $tokenCount,
      is_chunk: false
    })
  `,
    {
      id,
      toolName,
      arguments: JSON.stringify(arguments),
      result: resultString,
      timestamp,
      messageId,
      embedding,
      tokenCount,
    }
  );

  await neo4j.run(
    `
    MATCH (t:ToolCall {id: $toolId})
    MATCH (m:Message {id: $messageId})
    CREATE (t)-[:CALLED_BY]->(m)
  `,
    { toolId: id, messageId }
  );

  return id;
}

async function storeChunkedToolCall(
  toolName: string,
  arguments: any,
  result: string,
  messageId: string,
  config: IndexConfig
): Promise<string> {
  const parentToolCallId = ulid();
  const chunks = chunkContent(result, config.chunkThreshold);
  const timestamp = new Date().toISOString();

  for (let i = 0; i < chunks.length; i++) {
    const chunkId = ulid();
    const chunkContent = chunks[i];
    const embedding = await generateEmbedding(
      `${toolName} [chunk ${i}]: ${chunkContent}`
    );
    const tokenCount = estimateTokens(chunkContent);

    await neo4j.run(
      `
      CREATE (t:ToolCall {
        id: $id,
        tool_name: $toolName,
        arguments: $arguments,
        result: $result,
        timestamp: datetime($timestamp),
        message_id: $messageId,
        embedding: $embedding,
        token_count: $tokenCount,
        is_chunk: true,
        chunk_index: $chunkIndex,
        chunk_parent_id: $chunkParentId
      })
    `,
      {
        id: chunkId,
        toolName,
        arguments: JSON.stringify(arguments),
        result: chunkContent,
        timestamp,
        messageId,
        embedding,
        tokenCount,
        chunkIndex: i,
        chunkParentId: parentToolCallId,
      }
    );

    // Link first chunk to message
    if (i === 0) {
      await neo4j.run(
        `
        MATCH (t:ToolCall {id: $toolId})
        MATCH (m:Message {id: $messageId})
        CREATE (t)-[:CALLED_BY]->(m)
      `,
        { toolId: chunkId, messageId }
      );
    }
  }

  return parentToolCallId;
}
```

### 6.3 Context Preparation

```typescript
async function prepareContext(
  currentMessage: string,
  config: IndexConfig
): Promise<string> {
  // Search for relevant messages and tool calls
  const searchResults = await vectorSearch(
    currentMessage,
    config.includeToolCalls
  );

  // Get recent messages
  const recentMessages = await getRecentMessages(config.recentWindowSize);

  // Build appropriate index
  const index = await buildAdaptiveIndex(searchResults, recentMessages, config);

  return index;
}
```

### 6.4 Message Updates

```typescript
async function updateMessage(
  id: string,
  newContent: string,
  config: IndexConfig
): Promise<void> {
  const timestamp = new Date().toISOString();

  // Check if this is a chunk or full message
  const existing = await getMessage(id);

  if (existing.isChunk) {
    throw new Error(
      "Cannot edit individual chunks. Edit parent message instead."
    );
  }

  // Re-chunk if necessary
  const tokenCount = estimateTokens(newContent);
  if (tokenCount > config.chunkThreshold) {
    // Delete old chunks if they exist
    await neo4j.run(
      `
      MATCH (chunk:Message {chunk_parent_id: $id})
      DELETE chunk
    `,
      { id }
    );

    // Create new chunked version
    await storeChunkedMessage(
      newContent,
      existing.role,
      existing.parentId,
      config
    );
    return;
  }

  const embedding = await generateEmbedding(newContent);

  await neo4j.run(
    `
    MATCH (m:Message {id: $id})
    SET m.content = $newContent,
        m.embedding = $embedding,
        m.token_count = $tokenCount,
        m.edited = true,
        m.edit_history = m.edit_history + [$editRecord]
  `,
    {
      id,
      newContent,
      embedding,
      tokenCount,
      editRecord: {
        timestamp,
        previousContent: existing.content,
      },
    }
  );
}
```

## 7. Configuration

### 7.1 Environment Variables

```bash
# Database
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=password

# Embeddings
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSIONS=1536

# Index Configuration
INDEX_SNIPPET_LENGTH=100
INDEX_MAX_TOKENS=10000
INDEX_RECENT_WINDOW=10
INDEX_CLUSTERING_THRESHOLD=0.85
INDEX_CHUNK_THRESHOLD=4000
INDEX_INCLUDE_TOOL_CALLS=true
```

### 7.2 Performance Tuning

```yaml
performance:
  batch_embedding_size: 100 # Embed multiple messages at once
  cache_embeddings: true # Cache frequently accessed
  vector_search_timeout: 5000 # Milliseconds
  index_build_timeout: 3000 # Milliseconds

chunking:
  chunk_threshold: 4000 # Tokens before chunking
  chunk_overlap: 200 # Token overlap between chunks

fallbacks:
  on_vector_timeout: recent_only # Fall back to recent messages
  on_index_timeout: simple # Use simple format
  max_retries: 3 # Tool call retries
```

### 7.3 Background Processing Recommendations

Implementations should consider background processes to improve retrieval quality over time:

**Relationship Building:**

- Compute semantic similarity between messages in batches
- Build topic clusters using algorithms like DBSCAN or hierarchical clustering
- Create temporal summaries for time periods

**Index Optimization:**

- Pre-compute frequently accessed message clusters
- Build materialized views for common query patterns
- Generate topic embeddings from message clusters

**Maintenance:**

- Periodic re-embedding of edited messages
- Cleanup of orphaned chunks
- Compression of old embeddings

**Example Background Tasks:**

```typescript
// Run nightly
async function buildSemanticClusters() {
  const messages = await getRecentMessages(1000);
  const clusters = await clusterBySimilarity(messages, 0.85);

  for (const cluster of clusters) {
    await createTopicNode(cluster);
  }
}

// Run weekly
async function optimizeVectorIndex() {
  await neo4j.run(`
    CALL db.index.vector.queryNodes(
      'message_embeddings', 
      10, 
      $embedding
    ) YIELD node, score
    // Analyze query patterns and optimize
  `);
}
```

These background processes are implementation-specific and should be tailored to usage patterns and scale.

## 8. Performance Considerations

### 8.1 Bottlenecks

| Component | Bottleneck                    | Mitigation                                    |
| --------- | ----------------------------- | --------------------------------------------- |
| Storage   | Embedding size (6KB/msg)      | Compression, dimensionality reduction         |
| Search    | Vector similarity computation | Approximate nearest neighbors (ANN)           |
| **Index** | **Token presentation limit**  | **Adaptive strategies, clustering, chunking** |
| Retrieval | Sequential tool calls         | Batch retrieval, predictive loading           |
| Chunking  | Embedding generation overhead | Batch processing, async workflows             |

### 8.2 Scaling Characteristics

- **Storage**: O(n) - Linear with message count
- **Vector Search**: O(log n) with proper indexing
- **Index Building**: O(k) where k = result count
- **Context Window Usage**: O(1) - Constant regardless of history length
- **Chunking**: O(n/c) where c = chunk size (reduces memory per retrieval)

### 8.3 Optimization Strategies

1. **Hierarchical Clustering**: Pre-compute message clusters during quiet periods
2. **Embedding Cache**: Cache embeddings for frequently accessed messages
3. **Progressive Loading**: Start with minimal context, expand as needed
4. **Temporal Partitioning**: Separate hot (recent) and cold (old) storage
5. **Chunk-Aware Retrieval**: Load only relevant chunks instead of full messages
6. **Tool Call Indexing**: Separately searchable tool results for debugging/analysis

## Example Usage

### TypeScript/Node.js

```typescript
import { InfiniteChatStorage } from "./infinite-chat";

const chat = new InfiniteChatStorage({
  neo4jUri: process.env.NEO4J_URI,
  neo4jAuth: {
    user: process.env.NEO4J_USER,
    password: process.env.NEO4J_PASSWORD,
  },
  openaiKey: process.env.OPENAI_API_KEY,
  indexConfig: {
    snippetLength: 100,
    maxIndexTokens: 10000,
    recentWindowSize: 10,
    chunkThreshold: 4000,
    includeToolCalls: true,
  },
});

// Store a message (automatically chunks if needed)
const messageId = await chat.storeMessage(
  "What's our API authentication strategy?",
  "user",
  parentId
);

// Store a tool call with result (automatically chunks if needed)
const toolCallId = await chat.storeToolCall(
  "web_search",
  { query: "OAuth2 best practices" },
  largeSearchResult,
  messageId
);

// Prepare context for LLM
const context = await chat.prepareContext(
  "Tell me about our authentication decisions"
);

// Retrieve specific message with all chunks
const message = await chat.getMessageWithChunks(messageId);
```

### Python

```python
from infinite_chat import InfiniteChatStorage

chat = InfiniteChatStorage(
    neo4j_uri="bolt://localhost:7687",
    neo4j_auth=("neo4j", "password"),
    openai_key=os.getenv("OPENAI_API_KEY"),
    index_config={
        "snippet_length": 100,
        "max_index_tokens": 10000,
        "recent_window_size": 10,
        "chunk_threshold": 4000,
        "include_tool_calls": True
    }
)

# Store a message (automatically chunks if needed)
message_id = await chat.store_message(
    content="What's our API authentication strategy?",
    role='user',
    parent_id=parent_id
)

# Store a tool call with result
tool_call_id = await chat.store_tool_call(
    tool_name='web_search',
    arguments={'query': 'OAuth2 best practices'},
    result=large_search_result,
    message_id=message_id
)

# Prepare context for LLM
context = await chat.prepare_context(
    "Tell me about our authentication decisions"
)

# Retrieve specific message with all chunks
message = await chat.get_message_with_chunks(message_id)
```

## Contributing

This is an open specification. Contributions, implementations, and improvements are welcome. Please submit issues and pull requests to the repository.

## License

MIT License - See LICENSE file for details.
