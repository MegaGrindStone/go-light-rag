# go-light-rag

A Go library implementation of [LightRAG](https://github.com/HKUDS/LightRAG) - an advanced Retrieval-Augmented Generation (RAG) system that uniquely combines vector databases with graph database relationships to enhance knowledge retrieval.

## Overview

`go-light-rag` is a Go library that implements the core components of the LightRAG architecture rather than providing an end-to-end system. The library centers around two essential functions:

- `Insert`: Add documents to the knowledge base with flexible processing options
- `Query`: Retrieve contextually relevant information while preserving raw results

Unlike the original Python implementation which offers a complete RAG solution, this library deliberately separates the document processing pipeline from prompt engineering concerns. This approach gives developers:

1. Full control over document insertion workflows
2. Direct access to retrieved context data
3. Freedom to craft custom prompts tailored to specific use cases
4. Ability to integrate with existing Go applications and workflows

The minimalist API combined with powerful extension points makes `go-light-rag` ideal for developers who need the benefits of hybrid retrieval without being constrained by predefined prompt templates or processing pipelines.

## Architecture

`go-light-rag` is built on well-defined interfaces that enable flexibility, extensibility, and modular design. These interfaces define the contract between components, allowing you to replace or extend functionality without modifying the core logic.

### 1. Language Models (LLM)

The LLM interface abstracts different language model providers with these implementations included:

- **OpenAI**: Full support for GPT models
- **Anthropic**: Integration with Claude models
- **Ollama**: Self-hosted option for open-source models
- **OpenRouter**: Unified access to multiple model providers

Custom implementations can be created by implementing the LLM interface, which requires only a `Chat()` method.

### 2. Storage

The library defines three storage interfaces:

- **GraphStorage**: Manages entity and relationship data
- **VectorStorage**: Provides semantic search capabilities
- **KeyValueStorage**: Stores original document chunks

#### Implementations Provided

- GraphStorage: [Neo4j](https://github.com/neo4j/neo4j-go-driver) (and any compatible graph database)
- VectorStorage: [ChromeM](https://github.com/philippgille/chromem-go)
- KeyValueStorage: [BoltDB](https://github.com/etcd-io/bbolt)

You can implement any of these interfaces to use different storage solutions.

### 3. Handlers

Handlers control document and query processing:

- **DocumentHandler**: Controls chunking, entity extraction, and processing
- **QueryHandler**: Manages keyword extraction and prompt structuring

#### Included Handlers

- **Default**: General-purpose text document processing that follows the official Python implementation. Using the zero-value for Default handler will use the same configuration as the Python implementation.
- **Go**: Specialized handler for Go source code using AST parsing

Custom handlers can embed existing handlers and override only specific methods.

## Usage Examples

### Document Insertion

```go
// Initialize LLM
llm := llm.NewOpenAI(apiKey, model, params, logger)

// Initialize storage components
graphDB, _ := storage.NewNeo4J("bolt://localhost:7687", "neo4j", "password")
embeddingFunc := chromem.NewEmbeddingFuncOpenAI("open_ai_key", chromem.EmbeddingModelOpenAI3Large)
vecDB, _ := storage.NewChromem("vec.db", 5, embeddingFunc)
kvDB, _ := storage.NewBolt("kv.db")

store := storageWrapper{
    Bolt:    kvDB,
    Chromem: vecDB,
    Neo4J:   graphDB,
}

// Use default document handler with zero values to match Python implementation behavior
handler := handler.Default{}

// Insert a document
doc := golightrag.Document{
    ID:      "unique-document-id",
    Content: documentContent,
}

err := golightrag.Insert(doc, handler, store, llm, logger)
```

### Query Processing

```go
// Create a conversation with the user's query
conversation := []golightrag.QueryConversation{
    {
        Role:    golightrag.RoleUser,
        Message: "What do you know about the main characters?",
    },
}

// Execute the query
result, err := golightrag.Query(conversation, handler, store, llm, logger)
if err != nil {
    log.Fatalf("Error processing query: %v", err)
}

// Access the retrieved context
fmt.Printf("Found %d local entities and %d global entities\n", 
    len(result.LocalEntities), len(result.GlobalEntities))

// Process source documents
for _, source := range result.LocalSources {
    fmt.Printf("Source ID: %s\nRelevance: %.2f\nContent: %s\n\n", 
        source.ID, source.Relevance, source.Content)
}

// Or use the convenient String method for formatted results
fmt.Println(result)
```

## Handler Configuration Tips

1. **Choose the right handler for your documents**:
   - `Default` for general text
   - `Go` for Go source code
   - Create custom handlers for specialized content

2. **Optimize chunking parameters**:
   - Larger chunks provide more context but may exceed token limits
   - Smaller chunks process faster but may lose context
   - Balance overlap to maintain concept continuity

3. **Configure concurrency appropriately**:
   - Higher concurrency speeds up processing but increases resource usage
   - Balance according to your hardware capabilities and LLM rate limits

4. **Customize entity types**:
   - Define entity types relevant to your domain
   - Be specific enough to capture important concepts
   - Be general enough to avoid excessive fragmentation

## Benchmarks

`go-light-rag` includes benchmark tests comparing its performance against a NaiveRAG implementation. The benchmarks use the same evaluation prompts as the Python implementation but with different documents and queries.

For detailed benchmark results and methodology, visit the [benchmark directory](tests/).

## Examples and Documentation

For more detailed examples, please refer to the [examples directory](examples/).
