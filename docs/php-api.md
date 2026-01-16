# PHP API Reference

This guide covers the complete PHP API for the WordPress AI Client, including all available methods, configuration options, and usage patterns.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Prompt Builder Methods](#prompt-builder-methods)
- [Text Generation](#text-generation)
- [Image Generation](#image-generation)
- [Multimodal Output](#multimodal-output)
- [Configuration Methods](#configuration-methods)
- [Feature Detection](#feature-detection)
- [Error Handling](#error-handling)

## Core Concepts

### Entry Point

The main entry point is the `AI_Client` class:

```php
use WordPress\AI_Client\AI_Client;
```

### Fluent API

The client uses a fluent builder pattern for constructing prompts:

```php
$result = AI_Client::prompt( 'Your prompt here' )
    ->using_temperature( 0.7 )
    ->using_max_output_tokens( 1000 )
    ->generate_text();
```

### Method Naming Convention

PHP methods use WordPress-style `snake_case` naming:
- `generate_text()` - Generate text
- `using_temperature()` - Set temperature
- `as_json_response()` - Request JSON output

## Prompt Builder Methods

### Creating a Prompt Builder

#### `AI_Client::prompt( $prompt = null )`

Creates a new prompt builder.

**Parameters:**
- `$prompt` (string|Message|list<Message>|null) - Initial prompt content

**Returns:** `Prompt_Builder`

**Example:**

```php
// Simple string prompt
$builder = AI_Client::prompt( 'Explain quantum computing.' );

// No initial prompt (can be set later)
$builder = AI_Client::prompt();
$builder->with_prompt( 'Explain quantum computing.' );
```

#### `AI_Client::prompt_with_wp_error( $prompt = null )`

Creates a prompt builder that returns `WP_Error` objects instead of throwing exceptions.

**Parameters:**
- `$prompt` (string|Message|list<Message>|null) - Initial prompt content

**Returns:** `Prompt_Builder_With_WP_Error`

**Example:**

```php
$text = AI_Client::prompt_with_wp_error( 'Hello' )
    ->generate_text();

if ( is_wp_error( $text ) ) {
    wp_die( $text->get_error_message() );
}

echo wp_kses_post( $text );
```

## Text Generation

### `generate_text(): string`

Generates text based on the configured prompt.

**Returns:** `string` - The generated text

**Throws:** `Exception` on failure (unless using `prompt_with_wp_error`)

**Example:**

```php
$text = AI_Client::prompt( 'Write a short story about a robot.' )
    ->generate_text();

echo wp_kses_post( $text );
```

### `generate_result(): GenerationResult`

Generates a full result object with metadata.

**Returns:** `GenerationResult` - Contains the response and metadata

**Example:**

```php
$result = AI_Client::prompt( 'What is WordPress?' )
    ->generate_result();

$message = $result->toMessage();
foreach ( $message->getParts() as $part ) {
    if ( $part->isText() ) {
        echo wp_kses_post( $part->getText() );
    }
}

// Access usage metadata
$usage = $result->getUsage();
$input_tokens = $usage->getInputTokens();
$output_tokens = $usage->getOutputTokens();
```

## Image Generation

### `generate_image(): File`

Generates a single image.

**Returns:** `File` - The generated image file

**Example:**

```php
$image_file = AI_Client::prompt( 'A sunset over the ocean' )
    ->generate_image();

$data_uri = $image_file->getDataUri();
echo '<img src="' . esc_url( $data_uri ) . '" alt="Sunset">';
```

### `generate_images( int $count ): list<File>`

Generates multiple image candidates.

**Parameters:**
- `$count` (int) - Number of images to generate

**Returns:** `list<File>` - Array of generated image files

**Example:**

```php
$images = AI_Client::prompt( 'Abstract art with geometric shapes' )
    ->generate_images( 3 );

foreach ( $images as $image ) {
    echo '<img src="' . esc_url( $image->getDataUri() ) . '" alt="Abstract art">';
}
```

## Multimodal Output

### `as_output_modalities( ModalityEnum ...$modalities ): self`

Configures the prompt to generate multiple types of content.

**Parameters:**
- `$modalities` (ModalityEnum...) - One or more modality types

**Returns:** `self` - For method chaining

**Example:**

```php
use WordPress\AiClient\Messages\Enums\ModalityEnum;

$result = AI_Client::prompt( 'Create a recipe with step-by-step photos.' )
    ->as_output_modalities( ModalityEnum::text(), ModalityEnum::image() )
    ->generate_result();

foreach ( $result->toMessage()->getParts() as $part ) {
    if ( $part->isText() ) {
        echo '<p>' . wp_kses_post( $part->getText() ) . '</p>';
    } elseif ( $part->isFile() && $part->getFile()->isImage() ) {
        echo '<img src="' . esc_url( $part->getFile()->getDataUri() ) . '" alt="">';
    }
}
```

## Configuration Methods

### Model Selection

#### `using_model( Model $model ): self`

Forces the use of a specific model.

**Parameters:**
- `$model` (Model) - The model instance to use

**Returns:** `self` - For method chaining

**Example:**

```php
use WordPress\AiClient\ProviderImplementations\Anthropic\AnthropicProvider as Anthropic;

$text = AI_Client::prompt( 'Explain AI.' )
    ->using_model( Anthropic::model( 'claude-sonnet-4-5' ) )
    ->generate_text();
```

#### `using_model_preference( ...$preferences ): self`

Sets model preferences with automatic fallback.

**Parameters:**
- `$preferences` (array...) - Array pairs of `[provider_id, model_id]`

**Returns:** `self` - For method chaining

**Example:**

```php
$text = AI_Client::prompt( 'Summarize this article.' )
    ->using_model_preference(
        array( 'anthropic', 'claude-sonnet-4-5' ),
        array( 'openai', 'gpt-5.1' ),
        array( 'google', 'gemini-3-pro' )
    )
    ->generate_text();
```

### Output Configuration

#### `using_temperature( float $temperature ): self`

Sets the temperature for response generation.

**Parameters:**
- `$temperature` (float) - Value between 0.0 and 2.0
  - Lower values (0.0-0.3): More deterministic, focused
  - Medium values (0.5-0.9): Balanced creativity
  - Higher values (1.0-2.0): More creative, varied

**Returns:** `self` - For method chaining

**Example:**

```php
// Deterministic output for structured data
$json = AI_Client::prompt( 'List the top 5 programming languages.' )
    ->using_temperature( 0.1 )
    ->generate_text();

// Creative output for stories
$story = AI_Client::prompt( 'Write a short science fiction story.' )
    ->using_temperature( 1.2 )
    ->generate_text();
```

#### `using_max_output_tokens( int $max_tokens ): self`

Sets the maximum number of tokens to generate.

**Parameters:**
- `$max_tokens` (int) - Maximum tokens in the response

**Returns:** `self` - For method chaining

**Example:**

```php
$summary = AI_Client::prompt( 'Summarize the history of WordPress.' )
    ->using_max_output_tokens( 500 )
    ->generate_text();
```

#### `as_json_response( ?array $schema = null ): self`

Requests structured JSON output.

**Parameters:**
- `$schema` (array|null) - JSON Schema definition for validation

**Returns:** `self` - For method chaining

**Example:**

```php
$schema = array(
    'type'       => 'array',
    'items'      => array(
        'type'       => 'object',
        'properties' => array(
            'name'     => array( 'type' => 'string' ),
            'version'  => array( 'type' => 'string' ),
            'category' => array( 'type' => 'string' ),
        ),
        'required'   => array( 'name', 'version', 'category' ),
    ),
);

$json = AI_Client::prompt( 'List 5 popular WordPress plugins.' )
    ->as_json_response( $schema )
    ->generate_text();

$plugins = json_decode( $json, true );
```

### System Instructions

#### `with_system_instruction( string $instruction ): self`

Sets system-level instructions that guide the model's behavior.

**Parameters:**
- `$instruction` (string) - System instruction

**Returns:** `self` - For method chaining

**Example:**

```php
$text = AI_Client::prompt( 'Explain REST APIs.' )
    ->with_system_instruction( 'You are a technical writer creating documentation for developers. Use clear, concise language with code examples.' )
    ->generate_text();
```

## Feature Detection

### Text Generation Support

#### `is_supported_for_text_generation(): bool`

Checks if the current configuration supports text generation.

**Returns:** `bool` - True if supported

**Example:**

```php
$prompt = AI_Client::prompt( 'Explain quantum computing.' )
    ->using_temperature( 0.7 );

if ( $prompt->is_supported_for_text_generation() ) {
    $text = $prompt->generate_text();
    echo wp_kses_post( $text );
} else {
    echo 'Text generation is not available. Please configure an AI provider.';
}
```

### Image Generation Support

#### `is_supported_for_image_generation(): bool`

Checks if the current configuration supports image generation.

**Returns:** `bool` - True if supported

**Example:**

```php
$prompt = AI_Client::prompt( 'A futuristic city.' );

if ( $prompt->is_supported_for_image_generation() ) {
    $image = $prompt->generate_image();
    echo '<img src="' . esc_url( $image->getDataUri() ) . '" alt="Futuristic city">';
} else {
    echo 'Image generation is not available. Please configure a provider like OpenAI DALL-E.';
}
```

## Error Handling

### Exception-Based (Default)

By default, errors throw exceptions:

```php
try {
    $text = AI_Client::prompt( 'Hello' )
        ->generate_text();
    echo wp_kses_post( $text );
} catch ( \WordPress\AiClient\Exceptions\ApiException $e ) {
    // API-related errors (rate limits, invalid keys, etc.)
    error_log( 'AI API Error: ' . $e->getMessage() );
    echo 'An error occurred with the AI service.';
} catch ( \WordPress\AiClient\Exceptions\ValidationException $e ) {
    // Validation errors (invalid parameters, etc.)
    error_log( 'Validation Error: ' . $e->getMessage() );
    echo 'Invalid request configuration.';
} catch ( \Exception $e ) {
    // General errors
    error_log( 'Error: ' . $e->getMessage() );
    echo 'An unexpected error occurred.';
}
```

### WP_Error-Based (Experimental)

Use `prompt_with_wp_error()` for WordPress-style error handling:

```php
$text = AI_Client::prompt_with_wp_error( 'Hello' )
    ->generate_text();

if ( is_wp_error( $text ) ) {
    error_log( 'Error: ' . $text->get_error_message() );
    
    // Get error code
    $error_code = $text->get_error_code();
    
    // Get error data
    $error_data = $text->get_error_data();
    
    echo 'An error occurred: ' . esc_html( $text->get_error_message() );
    return;
}

echo wp_kses_post( $text );
```

## Complete Examples

### Content Generation with Context

```php
function generate_product_description( int $product_id ): string {
    $product = wc_get_product( $product_id );
    
    if ( ! $product ) {
        return '';
    }
    
    $prompt = sprintf(
        'Write a compelling product description for: %s. Category: %s. Price: $%s.',
        $product->get_name(),
        $product->get_categories(),
        $product->get_price()
    );
    
    try {
        return AI_Client::prompt( $prompt )
            ->using_temperature( 0.8 )
            ->using_max_output_tokens( 200 )
            ->with_system_instruction( 'You are a professional copywriter specializing in e-commerce product descriptions.' )
            ->generate_text();
    } catch ( Exception $e ) {
        error_log( 'Product description generation failed: ' . $e->getMessage() );
        return '';
    }
}
```

### Structured Data Extraction

```php
function extract_article_metadata( string $content ): array {
    $schema = array(
        'type'       => 'object',
        'properties' => array(
            'title'       => array( 'type' => 'string' ),
            'summary'     => array( 'type' => 'string' ),
            'tags'        => array(
                'type'  => 'array',
                'items' => array( 'type' => 'string' ),
            ),
            'reading_time' => array( 'type' => 'integer' ),
        ),
        'required'   => array( 'title', 'summary', 'tags', 'reading_time' ),
    );
    
    $prompt = sprintf(
        'Analyze this article and extract metadata:\n\n%s',
        $content
    );
    
    try {
        $json = AI_Client::prompt( $prompt )
            ->using_temperature( 0.2 )
            ->as_json_response( $schema )
            ->generate_text();
        
        return json_decode( $json, true );
    } catch ( Exception $e ) {
        error_log( 'Metadata extraction failed: ' . $e->getMessage() );
        return array();
    }
}
```

### Image Generation with Fallback

```php
function generate_featured_image( string $description ): ?int {
    $prompt = AI_Client::prompt( $description );
    
    // Check support before generating
    if ( ! $prompt->is_supported_for_image_generation() ) {
        return null;
    }
    
    try {
        $image_file = $prompt->generate_image();
        
        // Convert to WordPress attachment
        $upload_dir = wp_upload_dir();
        $filename = 'ai-generated-' . time() . '.png';
        $filepath = $upload_dir['path'] . '/' . $filename;
        
        file_put_contents( $filepath, base64_decode( $image_file->getData() ) );
        
        $attachment_id = wp_insert_attachment(
            array(
                'post_mime_type' => $image_file->getMimeType(),
                'post_title'     => sanitize_file_name( $filename ),
                'post_content'   => '',
                'post_status'    => 'inherit',
            ),
            $filepath
        );
        
        require_once ABSPATH . 'wp-admin/includes/image.php';
        wp_update_attachment_metadata(
            $attachment_id,
            wp_generate_attachment_metadata( $attachment_id, $filepath )
        );
        
        return $attachment_id;
    } catch ( Exception $e ) {
        error_log( 'Image generation failed: ' . $e->getMessage() );
        return null;
    }
}
```

## Best Practices

### 1. Always Check Feature Support

```php
// ✅ Good
$prompt = AI_Client::prompt( 'Generate text' );
if ( $prompt->is_supported_for_text_generation() ) {
    $text = $prompt->generate_text();
}

// ❌ Bad - may fail if no provider configured
$text = AI_Client::prompt( 'Generate text' )->generate_text();
```

### 2. Use Appropriate Temperature

```php
// ✅ Good - low temperature for structured data
$json = AI_Client::prompt( 'Extract data' )
    ->using_temperature( 0.1 )
    ->as_json_response( $schema )
    ->generate_text();

// ✅ Good - higher temperature for creative content
$story = AI_Client::prompt( 'Write a story' )
    ->using_temperature( 1.2 )
    ->generate_text();
```

### 3. Handle Errors Gracefully

```php
// ✅ Good - graceful fallback
try {
    $text = AI_Client::prompt( 'Summarize this' )->generate_text();
} catch ( Exception $e ) {
    error_log( $e->getMessage() );
    $text = 'Summary unavailable';
}

// ❌ Bad - unhandled errors can crash the page
$text = AI_Client::prompt( 'Summarize this' )->generate_text();
```

### 4. Use Model Preferences (Not Specific Models)

```php
// ✅ Good - works across different configurations
$text = AI_Client::prompt( 'Generate text' )
    ->using_model_preference(
        array( 'anthropic', 'claude-sonnet-4-5' ),
        array( 'openai', 'gpt-5' )
    )
    ->generate_text();

// ⚠️ Less flexible - requires specific provider
$text = AI_Client::prompt( 'Generate text' )
    ->using_model( Anthropic::model( 'claude-sonnet-4-5' ) )
    ->generate_text();
```

## Next Steps

- Explore the [JavaScript API](javascript-api.md) for client-side integration
- Learn about [Advanced Usage](advanced-usage.md) patterns
- Review [Troubleshooting](troubleshooting.md) for common issues
