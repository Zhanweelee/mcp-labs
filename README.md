# MCP Labs

Experimental features and enhancements for the MCP (Model Context Protocol) ecosystem.

## Deferred Tool Loading

Progressive Tool Disclosure pattern inspired by Claude Code's "Skills" system. Instead of sending full tool schemas to the LLM on every request, it sends only lightweight metadata (name + description), significantly reducing token usage.

### Key Benefits

| Feature | Description |
|---------|-------------|
| Token Reduction | 60-80% fewer tokens for tool definitions |
| Zero Breaking Changes | 100% backward compatible |
| Opt-in Only | Disabled by default |
| Automatic | Just set a flag, no manual setup |

### How It Works

```
Normal Flow:
  User Request → [Full Tool Schemas] → LLM → Execute Tools
                      ↑
                  Many tokens

Deferred Loading Flow:
  User Request → [Metadata Only] → LLM → Validate → Execute Tools
                      ↑                      ↓
                  Few tokens          If invalid: Retry with full schema
```

## Usage

### Basic Usage (mcp_llm)

```dart
import 'package:mcp_llm/mcp_llm.dart';

// Enable deferred loading when creating LlmClient
final client = LlmClient(
  llmProvider: provider,
  mcpClient: mcpClient,
  useDeferredLoading: true,  // Enable token optimization
);

// Use normally - deferred loading is handled automatically
final response = await client.chat('Search for Flutter tutorials');
```

### Using MCPLlm Factory

```dart
final mcpLlm = MCPLlm();
mcpLlm.registerProvider('claude', ClaudeProviderFactory());

final client = await mcpLlm.createClient(
  providerName: 'claude',
  config: LlmConfiguration(apiKey: 'your-api-key'),
  mcpClient: mcpClient,
  useDeferredLoading: true,  // Enable here
);
```

### Standalone mcp_client Usage (Without mcp_llm)

```dart
import 'package:mcp_client/mcp_client.dart';

final registry = ToolRegistry();

// Fetch and cache tools
final tools = await mcpClient.listTools();
registry.cacheFromTools(tools);

// Get lightweight metadata for your LLM integration
final metadata = registry.getAllMetadata();
final toolsForLlm = metadata.map((m) => m.toJson()).toList();

// Later, get full schema when needed for validation/execution
final fullSchema = registry.getSchema('search');
```

### Using Client Extension

```dart
final registry = ToolRegistry();
final metadata = await client.listToolsMetadata(registry);
// metadata: lightweight for LLM context
// registry: contains full schemas for later lookup
```

## API Reference

### mcp_client Classes

#### ToolMetadata
Lightweight tool representation (name + description only).

```dart
// Create from Tool object
final metadata = ToolMetadata.fromTool(tool);

// Create from Map
final metadata = ToolMetadata.fromMap({'name': 'search', 'description': 'Search the web'});

// Serialize (no inputSchema - token efficient)
final json = metadata.toJson();
```

#### ToolRegistry
Cache layer for tool definitions.

```dart
final registry = ToolRegistry();

// Cache tools
registry.cacheFromTools(tools);      // From Tool list
registry.cacheFromMaps(toolMaps);    // From Map list

// Get metadata (lightweight)
registry.getAllMetadata();           // All metadata
registry.getMetadata('search');      // Specific tool

// Get full schema (for validation/execution)
registry.getSchema('search');

// Status
registry.isInitialized;              // true/false
registry.count;                      // Number of cached tools
registry.toolNames;                  // ['search', 'calculator', ...]
registry.hasTool('search');          // true/false

// Invalidate on tools/list_changed notification
registry.invalidateAll();
```

### mcp_llm Classes

#### ValidationResult
Tool call validation result.

```dart
final valid = ValidationResult.valid();
final invalid = ValidationResult.invalid('Missing required parameter: query');

_logger.debug('isValid: ${result.isValid}');  // true/false
_logger.debug('error: ${result.error}');      // null or error message
```

#### LlmToolMetadata
LLM-specific lightweight metadata.

```dart
const metadata = LlmToolMetadata(name: 'search', description: 'Search for information');
final json = metadata.toJson();  // Excludes inputSchema
```

#### DeferredToolManager
Orchestration layer (automatically created by LlmClient when useDeferredLoading=true).

```dart
// Usually handled internally by LlmClient
// For advanced use:
final manager = DeferredToolManager();
await manager.initialize(mcpClientManager);

manager.getMetadataForLlm();              // For LLM context
manager.getFullSchema('search');          // For validation
manager.validateToolCall('search', args); // Returns ValidationResult
manager.invalidate();                     // On tools/list_changed
```

## Projects

### mcp_client_defer_loading

Enhanced `mcp_client` package with Deferred Tool Loading support.

- `ToolMetadata` - Lightweight tool representation
- `ToolRegistry` - Cache layer with metadata extraction
- `ClientToolMetadataExtension` - Convenient extension

### mcp_llm_defer_loading

Enhanced `mcp_llm` package with Deferred Tool Loading orchestration.

- `ValidationResult` - Tool call validation result
- `LlmToolMetadata` - LLM-specific lightweight metadata
- `DeferredToolManager` - Orchestration layer
- `useDeferredLoading` parameter in `LlmClient`

## Architecture

```
+------------------+     +------------------+
|    mcp_client    |     |     mcp_llm      |
+------------------+     +------------------+
| ToolMetadata     |     | LlmToolMetadata  |
| ToolRegistry     |     | ValidationResult |
|                  |     | DeferredToolMgr  |
+------------------+     +------------------+
   Data Layer            Orchestration Layer
```

## Resource Tool Bridge

MCP resources are visible in system prompts but not callable by LLMs. Synthetic tools (`mcp_read_resource`, `mcp_list_resources`) bridge this gap.

```dart
final client = LlmClient(
  llmProvider: provider,
  mcpClient: mcpClient,
);

// Resources are automatically available as callable tools
final response = await client.chat('Read the config resource');
```

## Multi-Round Tool Calling

Enables the LLM to make multiple sequential tool calls until it has enough information to answer.

```dart
final client = LlmClient(
  llmProvider: provider,
  mcpClient: mcpClient,
  maxToolRounds: 5,  // Allow up to 5 rounds (default: 1, max: 10)
);

final response = await client.chat('Compare weather in Seoul and Tokyo');
// Round 1: LLM calls get_weather(Seoul)
// Round 2: LLM calls get_weather(Tokyo)
// Round 3: LLM returns comparison
```

## Running Tests

```bash
# All packages
cd mcp_llm_resource_tools && dart test
cd mcp_client_defer_loading && dart test
cd mcp_llm_defer_loading && dart test
```

## Design Principles

1. **Zero Breaking Changes**: All existing APIs work unchanged
2. **Opt-in Only**: Disabled by default (`useDeferredLoading: false`)
3. **Automatic Configuration**: Just set the flag, no manual setup needed
4. **Ephemeral Retry**: Retry context doesn't pollute chat history
5. **Layer Separation**: mcp_client for data, mcp_llm for orchestration

## License

See individual package licenses.
