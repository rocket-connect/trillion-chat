# Trillion Chat

> Work in progress 🚧

**A specification for \*infinite-context LLM conversations using Neo4j graph storage**

This repository contains:

- **[Complete Specification](./spec.md)** - Detailed technical specification for \*infinite-context chat storage
- **TypeScript Client Library** - Production-ready implementation of the spec (`@trillionchat/client`)
- **Demonstration UI** - Live demo at [trillionchat.rconnect.tech](https://trillionchat.rconnect.tech) with embedded MCP Connect

Created by [rconnect.tech](https://rconnect.tech)

---

## The Specification

**[Read the full specification →](./spec.md)**

Trillion Chat defines a standard approach for managing \*unlimited-length LLM conversations using graph database storage, vector embeddings, and intelligent context retrieval. The specification eliminates context window limitations by storing all messages in Neo4j and letting LLMs retrieve only what they need through tool calls.

Inspired by Neo4j's [trillion-graph demonstration](https://github.com/neo4j/trillion-graph) that proved real-time query performance against over 200 billion nodes and more than a trillion relationships.

### Key Concepts

**Storage Model**

- All messages stored in Neo4j with vector embeddings
- Automatic chunking for long content
- Tool calls are first-class searchable entities
- Full edit history and soft deletes

**Adaptive Indexing**

- Scales from small to extremely large conversations
- Automatic strategy selection based on conversation size
- Constant token usage regardless of history length

**LLM Tool Interface**

- Semantic search across all history
- Thread-based navigation
- Temporal retrieval
- Bulk operations for efficiency

## Client Library

This repository provides a production-ready TypeScript client library implementing the Trillion Chat specification.

**Package:** `@trillionchat/client`

### Architecture

```
apps/
  ui/          → Demonstration web interface with embedded MCP Connect

packages/
  client/      → Core conversation management and Neo4j storage
  utils/       → Shared utilities and types
```

### Installation

```bash
# Install the client
npm install @trillionchat/client

# Or with pnpm
pnpm add @trillionchat/client
```

### Quick Start

**Try the demo:** [trillionchat.rconnect.tech](https://trillionchat.rconnect.tech)

**Or use the client in your project:**

```typescript
import { TrillionChat } from "@trillionchat/client";

const chat = new TrillionChat({
  neo4j: {
    uri: "bolt://localhost:7687",
    auth: { user: "neo4j", password: "password" },
  },
  openai: {
    apiKey: process.env.OPENAI_API_KEY,
  },
});

await chat.initialize();

// Store a message
const messageId = await chat.storeMessage({
  content: "What's the best way to implement OAuth2?",
  role: "user",
});

// Prepare context for LLM (builds adaptive index)
const context = await chat.prepareContext(
  "Tell me about authentication best practices"
);

// LLM can now use retrieval tools to fetch specific messages
```

## Demonstration UI

**[Try it live at trillionchat.rconnect.tech](https://trillionchat.rconnect.tech)**

The demonstration UI shows the specification in action and embeds [MCP Connect](https://mcp.rconnect.tech) for visual development:

- Live message storage to Neo4j with vector embeddings
- Real-time adaptive index generation
- Visual tool management and token optimization
- Protocol inspection for debugging
- Graph visualization of conversation relationships
- Export conversations in multiple formats

Learn more: [How to MCP Connect to Neo4j](https://www.rconnect.tech/blog/how-to-mcp-connect-neo4j)

## Implementation Highlights

### Index Strategies

The implementation automatically adapts based on conversation size per the specification:

| Messages | Strategy     | Description                    |
| -------- | ------------ | ------------------------------ |
| 0-50     | Full         | Complete messages shown        |
| 51-500   | Snippet      | Recent + historical previews   |
| 501-5000 | Clustered    | Semantic groups with summaries |
| 5000+    | Hierarchical | Multi-level navigation         |

### Configuration

```typescript
const chat = new TrillionChat({
  neo4j: {
    /* ... */
  },
  openai: {
    /* ... */
  },
  indexConfig: {
    snippetLength: 100, // Preview text length
    maxIndexTokens: 10000, // Token budget for index
    recentWindowSize: 10, // Always-included recent messages
    chunkThreshold: 4000, // When to chunk long content
    includeToolCalls: true, // Search tool results
    clusteringThreshold: 0.85, // Similarity for clustering
  },
});
```

### LLM Tools

As specified, the implementation provides:

```typescript
get_message_by_id(id: string)
get_messages_by_ids(ids: string[])
vector_search(query: string, limit?: number)
get_message_with_chunks(id: string)
get_conversation_thread(message_id: string)
get_period_messages(period: string)
```

### Performance

The implementation is designed for efficient operation across different scales. Actual performance will vary based on your Neo4j configuration, hardware, and dataset characteristics.

## Development

```bash
# Clone the repository
git clone https://github.com/rocket-connect/trillion-chat
cd trillion-chat

# Install dependencies
pnpm install

# Start development (UI + API)
pnpm dev

# Run tests
pnpm test

# Build all packages
pnpm build
```

## Documentation

- **[Specification](./spec.md)** - Complete technical specification
- [API Reference](./docs/api.md) - Client API documentation
- [UI Guide](./docs/ui.md) - Web interface features
- [Implementation Notes](./docs/implementation.md) - Design decisions

## Use Cases

**Long-running AI assistants** - Maintain context across \*unlimited time periods

**Research conversations** - Build knowledge graphs from \*thousands of exchanges

**Customer support** - Full conversation history with \*instant semantic recall

**Code review assistants** - Reference \*any discussion from entire project history

**Enterprise knowledge management** - Capture insights from \*millions of conversations

## Related Projects

- [neo4j/trillion-graph](https://github.com/neo4j/trillion-graph) - Neo4j's trillion-scale demonstration
- [MCP Connect](https://mcp.rconnect.tech) - Visual MCP development interface
- [neo4j-contrib/mcp-neo4j](https://github.com/neo4j-contrib/mcp-neo4j) - Neo4j MCP server

## Contributing

We welcome contributions to both the specification and implementation!

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

## License

MIT License - see [LICENSE](./LICENSE)

## Acknowledgments

Built by [rconnect.tech](https://rconnect.tech)

Special thanks to Neo4j for trillion-scale graph technology and Anthropic for Claude and the Model Context Protocol.

---

[@trillionchat](https://twitter.com/trillionchat) | [Read the Spec](./spec.md) | [rconnect.tech](https://rconnect.tech)
