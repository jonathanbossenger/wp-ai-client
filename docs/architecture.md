# Architecture

This document explains the technical architecture and design decisions behind the WordPress AI Client.

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Core Components](#core-components)
- [Request Flow](#request-flow)
- [Design Patterns](#design-patterns)
- [Security Model](#security-model)
- [Extension Points](#extension-points)

## Overview

The WordPress AI Client is built as a WordPress-native wrapper around the [PHP AI Client](https://github.com/WordPress/php-ai-client), providing a unified API for interacting with multiple AI providers while adhering to WordPress coding standards and best practices.

### Key Design Goals

1. **Provider Agnosticism**: Support multiple AI providers through a uniform interface
2. **WordPress Integration**: Deep integration with WordPress APIs and conventions
3. **Security First**: Capability-based access control and input validation
4. **Developer Experience**: Fluent API design with clear error handling
5. **Flexibility**: Support for various AI capabilities (text, images, multimodal)

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│              WordPress AI Client                     │
├─────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────────────────┐      ┌──────────────────┐    │
│  │  PHP SDK Layer   │      │  JavaScript SDK  │    │
│  │  (includes/)     │      │  (src/, build/)  │    │
│  └────────┬─────────┘      └────────┬─────────┘    │
│           │                          │               │
│           │  ┌──────────────────┐   │              │
│           └──┤   REST API       ├───┘              │
│              │ (REST_API/)      │                   │
│              └────────┬─────────┘                   │
│                       │                              │
│         ┌─────────────┴─────────────┐               │
│         │   Core PHP AI Client      │               │
│         │   (vendor/wordpress/      │               │
│         │    php-ai-client)          │               │
│         └─────────────┬─────────────┘               │
│                       │                              │
│         ┌─────────────┴─────────────┐               │
│         │   HTTP Client Layer       │               │
│         │   (WP HTTP API Adapter)   │               │
│         └─────────────┬─────────────┘               │
└───────────────────────┼─────────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          │    AI Provider APIs       │
          │ (Anthropic, Google, etc.) │
          └───────────────────────────┘
```

## Core Components

### 1. AI_Client (Main Entry Point)

**Location**: `includes/AI_Client.php`

The main facade class providing static entry points for AI operations.

**Responsibilities**:
- Initialize the package components
- Register WordPress hooks and filters
- Provide factory methods for prompt builders
- Register REST API routes
- Set up the HTTP client integration

**Key Methods**:
- `init()`: Initialize the package
- `prompt()`: Create a prompt builder (exception-based)
- `prompt_with_wp_error()`: Create a prompt builder (WP_Error-based)

### 2. Prompt_Builder

**Location**: `includes/Builders/Prompt_Builder.php`

Fluent API builder for constructing AI prompts, using WordPress naming conventions.

**Responsibilities**:
- Translate WordPress-style method names to PHP AI Client methods
- Build and configure AI requests
- Execute generation operations
- Handle feature detection

**Design Pattern**: Proxy/Adapter pattern wrapping the PHP AI Client's PromptBuilder

**Key Features**:
- Method name translation (`using_temperature` → `usingTemperature`)
- WordPress coding standards compliance
- Type hints for IDE support

### 3. API Credentials Manager

**Location**: `includes/API_Credentials/`

Manages AI provider API credentials and settings screen.

**Responsibilities**:
- Store and retrieve API credentials from WordPress options
- Provide admin settings interface
- Wire credentials to the PHP AI Client
- Support programmatic credential configuration via filters

**Storage**: WordPress options API (`wp_ai_client_credentials`)

### 4. REST API Controllers

**Location**: `includes/REST_API/`

Expose AI capabilities via REST endpoints for client-side use.

**Controllers**:

#### AI_Prompt_REST_Controller
- **Endpoint**: `/wp-ai/v1/prompt`
- **Methods**: POST
- **Purpose**: Execute AI prompts from JavaScript
- **Capability**: `prompt_ai`

#### AI_Providers_Models_REST_Controller
- **Endpoint**: `/wp-ai/v1/providers-models`
- **Methods**: GET
- **Purpose**: List available providers and models
- **Capability**: `list_ai_providers_models`

### 5. HTTP Client Integration

**Location**: `includes/HTTP/WP_AI_Client_Discovery_Strategy.php`

PSR-compliant HTTP client using WordPress HTTP API.

**Responsibilities**:
- Adapt WordPress HTTP API to PSR-7/PSR-18 interfaces
- Integrate with PHP AI Client's HTTP discovery
- Handle request/response conversion
- Support WordPress HTTP filters

**Libraries Used**:
- `nyholm/psr7`: PSR-7 HTTP message implementation
- WordPress HTTP API: Underlying HTTP transport

### 6. Capabilities Manager

**Location**: `includes/Capabilities/Capabilities_Manager.php`

Manages WordPress capabilities for AI features.

**Capabilities**:
- `prompt_ai`: Execute AI prompts
- `list_ai_providers_models`: List available AI providers

**Default Grants**: Administrators only

### 7. JavaScript SDK

**Location**: `src/`, compiled to `build/`

Client-side API mirroring the PHP prompt builder interface.

**Key Files**:
- `index.ts`: Main entry point
- `builders/PromptBuilder.ts`: JavaScript prompt builder
- `enums.ts`: Enum definitions (Modality, MessagePartType)
- `types.ts`: TypeScript type definitions

**Communication**: REST API via `@wordpress/api-fetch`

## Request Flow

### PHP Request Flow

```
1. User Code
   └─> AI_Client::prompt( $prompt )
       └─> Creates Prompt_Builder instance
           └─> Configures prompt (temperature, model, etc.)
               └─> Calls generate_text() / generate_image()
                   └─> PHP AI Client processes request
                       └─> HTTP Client (WP HTTP API Adapter)
                           └─> AI Provider API
                               └─> Response processed
                                   └─> Result returned to user code
```

### JavaScript Request Flow

```
1. JavaScript Code
   └─> wp.aiClient.prompt( prompt )
       └─> Creates PromptBuilder instance
           └─> Configures prompt (temperature, model, etc.)
               └─> Calls generateText() / generateImage()
                   └─> Makes REST API call (/wp-ai/v1/prompt)
                       └─> AI_Prompt_REST_Controller
                           └─> Permission check (prompt_ai capability)
                               └─> PHP AI Client processes request
                                   └─> HTTP Client (WP HTTP API Adapter)
                                       └─> AI Provider API
                                           └─> Response processed
                                               └─> JSON returned to JavaScript
                                                   └─> Result returned to user code
```

### Feature Detection Flow

```
1. User Code
   └─> prompt->is_supported_for_text_generation()
       └─> PHP AI Client checks:
           ├─> Are any providers configured?
           ├─> Do configured providers support text generation?
           ├─> Do they support the requested configuration?
           └─> Returns boolean result
```

## Design Patterns

### 1. Facade Pattern

`AI_Client` acts as a facade, simplifying access to the complex underlying PHP AI Client.

```php
// Simple facade
AI_Client::prompt( 'Hello' )->generate_text();

// Instead of complex instantiation
$registry = AiClient::defaultRegistry();
$builder = new PromptBuilder( $registry );
$builder->withPrompt( 'Hello' );
// ... etc
```

### 2. Builder Pattern

`Prompt_Builder` implements the Builder pattern for fluent API construction.

```php
$text = AI_Client::prompt( 'Base prompt' )
    ->using_temperature( 0.7 )
    ->using_max_output_tokens( 1000 )
    ->with_system_instruction( 'Be helpful' )
    ->generate_text();
```

### 3. Proxy/Adapter Pattern

`Prompt_Builder` proxies calls to the PHP AI Client's PromptBuilder, adapting method names.

```php
// WordPress style (snake_case)
->using_temperature( 0.7 )

// Translated to PHP AI Client style (camelCase)
->usingTemperature( 0.7 )
```

### 4. Strategy Pattern

HTTP client uses strategy pattern through PSR discovery, allowing different HTTP implementations.

```php
// WordPress HTTP API strategy
WP_AI_Client_Discovery_Strategy::init();

// PHP AI Client automatically uses the registered strategy
```

### 5. Factory Pattern

`AI_Client` provides factory methods for creating builders.

```php
// Factory for exception-based builder
$builder = AI_Client::prompt( 'Hello' );

// Factory for WP_Error-based builder
$builder = AI_Client::prompt_with_wp_error( 'Hello' );
```

## Security Model

### 1. Capability-Based Access Control

All AI operations require the `prompt_ai` capability.

```php
// Automatically checked in REST API
public function check_permission(): bool {
    return current_user_can( 'prompt_ai' );
}

// Should be checked in custom code
if ( ! current_user_can( 'prompt_ai' ) ) {
    wp_die( 'Permission denied' );
}
```

### 2. REST API Security

REST endpoints use WordPress nonces and capability checks:

```php
// Nonce verification
wp_verify_nonce( $request->get_header( 'X-WP-Nonce' ), 'wp_rest' );

// Capability check
current_user_can( 'prompt_ai' );
```

### 3. Input Validation

All user inputs are sanitized and validated:

```php
// Sanitize prompts
$prompt = sanitize_textarea_field( $user_input );

// Validate schemas
if ( ! $this->validate_json_schema( $schema ) ) {
    return new WP_Error( 'invalid_schema' );
}
```

### 4. Output Escaping

AI-generated content must be escaped before output:

```php
// For HTML output
echo wp_kses_post( $ai_generated_text );

// For attributes
echo '<div title="' . esc_attr( $ai_generated_text ) . '">';
```

### 5. API Key Security

API keys are stored securely and never exposed to clients:

- Stored in WordPress options (can be encrypted)
- Never sent to JavaScript
- Only accessible server-side
- Can be filtered for environment variable support

## Extension Points

### 1. Filters

#### `wordpress_ai_client_api_credentials`

Modify or add API credentials programmatically.

```php
add_filter(
    'wordpress_ai_client_api_credentials',
    function ( array $credentials ): array {
        if ( defined( 'ANTHROPIC_API_KEY' ) ) {
            $credentials['anthropic'] = array(
                'api_key' => ANTHROPIC_API_KEY,
            );
        }
        return $credentials;
    }
);
```

#### `user_has_cap`

Customize capability grants.

```php
add_filter(
    'user_has_cap',
    function ( array $allcaps, array $caps, array $args, WP_User $user ): array {
        if ( in_array( 'prompt_ai', $caps, true ) ) {
            // Custom logic
            $allcaps['prompt_ai'] = custom_check( $user );
        }
        return $allcaps;
    },
    10,
    4
);
```

### 2. Actions

#### `rest_api_init`

Register custom REST endpoints alongside the built-in ones.

```php
add_action(
    'rest_api_init',
    function () {
        register_rest_route(
            'my-plugin/v1',
            '/ai-custom',
            array(
                'methods'             => 'POST',
                'callback'            => 'my_custom_ai_endpoint',
                'permission_callback' => function () {
                    return current_user_can( 'prompt_ai' );
                },
            )
        );
    }
);
```

### 3. Custom Builders

Extend `Prompt_Builder` for custom behavior:

```php
class Custom_Prompt_Builder extends Prompt_Builder {
    public function with_wordpress_context( int $post_id ): self {
        $post = get_post( $post_id );
        $context = sprintf(
            'Post Title: %s\nPost Content: %s',
            $post->post_title,
            wp_trim_words( $post->post_content, 100 )
        );
        
        return $this->with_system_instruction( $context );
    }
}
```

### 4. Custom HTTP Clients

Register custom HTTP client strategies:

```php
use Psr\Http\Client\ClientInterface;

class Custom_HTTP_Client implements ClientInterface {
    // Custom implementation
}

// Register with PSR discovery
// (Advanced use case)
```

## Data Flow Diagrams

### Credential Management

```
WordPress Admin
     │
     ├─> Settings > AI Credentials
     │        │
     │        └─> API_Credentials_Manager
     │                 │
     │                 └─> update_option( 'wp_ai_client_credentials' )
     │
Plugin Code
     │
     └─> Filter: wordpress_ai_client_api_credentials
              │
              └─> API_Credentials_Manager::get_credentials()
                       │
                       ├─> get_option( 'wp_ai_client_credentials' )
                       └─> apply_filters( 'wordpress_ai_client_api_credentials' )
                                │
                                └─> PHP AI Client uses credentials
```

### HTTP Request

```
AI_Client::prompt()->generate_text()
            │
            └─> PHP AI Client
                     │
                     └─> Creates HTTP request (PSR-7)
                              │
                              └─> PSR-18 HTTP Client
                                       │
                                       └─> WP_HTTP_Client (Adapter)
                                                │
                                                └─> wp_remote_post()
                                                         │
                                                         └─> AI Provider API
                                                                  │
                                                                  └─> Response
                                                                       │
                                                                       └─> Convert to PSR-7
                                                                            │
                                                                            └─> Return to user
```

## Performance Considerations

### 1. Lazy Initialization

The package only loads components when needed:

```php
// Only initializes when called
AI_Client::init();
```

### 2. Caching

Response caching is left to the implementation to allow flexibility:

```php
// User code implements caching
$cached = get_transient( 'ai_response_' . md5( $prompt ) );
if ( false === $cached ) {
    $cached = AI_Client::prompt( $prompt )->generate_text();
    set_transient( 'ai_response_' . md5( $prompt ), $cached, HOUR_IN_SECONDS );
}
```

### 3. Async Processing

Long-running tasks should use Action Scheduler:

```php
as_enqueue_async_action(
    'my_plugin_ai_task',
    array( 'prompt' => $prompt )
);
```

## Testing Architecture

### Unit Tests

Located in `tests/phpunit/tests/`, testing individual components:

- Prompt Builder method translation
- Credential management
- Capability checks
- REST API endpoints

### Integration Tests

Testing full request flows with mocked AI responses.

## Future Considerations

### Planned Enhancements

1. **Streaming Support**: Real-time streaming of AI responses
2. **Usage Analytics**: Built-in usage tracking and reporting
3. **Rate Limiting**: Native rate limiting functionality
4. **Response Caching**: Built-in caching layer
5. **Webhook Support**: Async generation with callbacks

### Extensibility

The architecture is designed to accommodate:

- Additional AI providers
- New AI capabilities (audio, video, etc.)
- Custom authentication methods
- Alternative storage backends for credentials

## Next Steps

- Review the [Development Guide](development.md) for contributing
- Explore [Advanced Usage](advanced-usage.md) patterns
- Check the [Troubleshooting Guide](troubleshooting.md) for issues
