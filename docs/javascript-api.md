# JavaScript API Reference

This guide covers the complete JavaScript API for the WordPress AI Client, providing client-side AI capabilities through REST endpoints.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Loading the API](#loading-the-api)
- [Prompt Builder Methods](#prompt-builder-methods)
- [Text Generation](#text-generation)
- [Image Generation](#image-generation)
- [Multimodal Output](#multimodal-output)
- [Configuration Methods](#configuration-methods)
- [Feature Detection](#feature-detection)
- [Error Handling](#error-handling)

## Core Concepts

### Entry Point

The JavaScript API is available globally as `wp.aiClient`:

```javascript
const { aiClient } = wp;
```

### Async by Design

All generation methods are asynchronous and return Promises:

```javascript
const text = await wp.aiClient.prompt( 'Hello' ).generateText();
```

### Method Naming Convention

JavaScript methods use camelCase naming:
- `generateText()` - Generate text
- `usingTemperature()` - Set temperature
- `asJsonResponse()` - Request JSON output

## Loading the API

### Enqueue the Script

In your PHP code, enqueue the script:

```php
add_action(
    'admin_enqueue_scripts',
    function () {
        wp_enqueue_script( 'wp-ai-client' );
    }
);
```

### Script Dependencies

The script automatically includes these dependencies:
- `@wordpress/api-fetch` - For REST API communication
- `@wordpress/data` - For state management

### Verify Availability

Check if the API is available before using:

```javascript
if ( typeof wp !== 'undefined' && wp.aiClient ) {
    // API is available
    const text = await wp.aiClient.prompt( 'Hello' ).generateText();
}
```

## Prompt Builder Methods

### Creating a Prompt Builder

#### `wp.aiClient.prompt( prompt )`

Creates a new prompt builder.

**Parameters:**
- `prompt` (string|Message|Message[]|null) - Initial prompt content

**Returns:** `PromptBuilder`

**Example:**

```javascript
// Simple string prompt
const builder = wp.aiClient.prompt( 'Explain quantum computing.' );

// No initial prompt (can be set later)
const builder = wp.aiClient.prompt();
builder.withPrompt( 'Explain quantum computing.' );
```

## Text Generation

### `generateText(): Promise<string>`

Generates text based on the configured prompt.

**Returns:** `Promise<string>` - The generated text

**Throws:** Error on failure

**Example:**

```javascript
const text = await wp.aiClient.prompt( 'Write a haiku about WordPress.' )
    .generateText();

console.log( text );
```

### `generateResult(): Promise<GenerationResult>`

Generates a full result object with metadata.

**Returns:** `Promise<GenerationResult>` - Contains the response and metadata

**Example:**

```javascript
const result = await wp.aiClient.prompt( 'What is WordPress?' )
    .generateResult();

const message = result.toMessage();
for ( const part of message.parts ) {
    if ( part.type === wp.aiClient.enums.MessagePartType.TEXT ) {
        console.log( part.text );
    }
}

// Access usage metadata
const usage = result.usage;
console.log( 'Input tokens:', usage.inputTokens );
console.log( 'Output tokens:', usage.outputTokens );
```

## Image Generation

### `generateImage(): Promise<File>`

Generates a single image.

**Returns:** `Promise<File>` - The generated image file

**Example:**

```javascript
const imageFile = await wp.aiClient.prompt( 'A sunset over the ocean' )
    .generateImage();

const dataUri = imageFile.getDataUri();
document.querySelector( '#image' ).src = dataUri;
```

### `generateImages( count ): Promise<File[]>`

Generates multiple image candidates.

**Parameters:**
- `count` (number) - Number of images to generate

**Returns:** `Promise<File[]>` - Array of generated image files

**Example:**

```javascript
const images = await wp.aiClient.prompt( 'Abstract art with geometric shapes' )
    .generateImages( 3 );

const container = document.querySelector( '#images' );
for ( const image of images ) {
    const img = document.createElement( 'img' );
    img.src = image.getDataUri();
    container.appendChild( img );
}
```

## Multimodal Output

### `asOutputModalities( ...modalities ): this`

Configures the prompt to generate multiple types of content.

**Parameters:**
- `modalities` (string...) - One or more modality types (use `wp.aiClient.enums.Modality`)

**Returns:** `this` - For method chaining

**Example:**

```javascript
const { Modality, MessagePartType } = wp.aiClient.enums;

const result = await wp.aiClient.prompt( 'Create a recipe with step-by-step photos.' )
    .asOutputModalities( Modality.TEXT, Modality.IMAGE )
    .generateResult();

for ( const part of result.toMessage().parts ) {
    if ( part.type === MessagePartType.TEXT ) {
        console.log( part.text );
    } else if ( part.type === MessagePartType.FILE && part.file.isImage() ) {
        const img = document.createElement( 'img' );
        img.src = part.file.getDataUri();
        document.body.appendChild( img );
    }
}
```

## Configuration Methods

### Model Selection

#### `usingModel( providerId, modelId ): this`

Forces the use of a specific model.

**Parameters:**
- `providerId` (string) - The provider identifier (e.g., 'anthropic', 'openai')
- `modelId` (string) - The model identifier (e.g., 'claude-sonnet-4-5')

**Returns:** `this` - For method chaining

**Example:**

```javascript
const text = await wp.aiClient.prompt( 'Explain AI.' )
    .usingModel( 'anthropic', 'claude-sonnet-4-5' )
    .generateText();
```

#### `usingModelPreference( ...preferences ): this`

Sets model preferences with automatic fallback.

**Parameters:**
- `preferences` ([string, string]...) - Array pairs of `[providerId, modelId]`

**Returns:** `this` - For method chaining

**Example:**

```javascript
const text = await wp.aiClient.prompt( 'Summarize this article.' )
    .usingModelPreference(
        [ 'anthropic', 'claude-sonnet-4-5' ],
        [ 'openai', 'gpt-5.1' ],
        [ 'google', 'gemini-3-pro' ]
    )
    .generateText();
```

### Output Configuration

#### `usingTemperature( temperature ): this`

Sets the temperature for response generation.

**Parameters:**
- `temperature` (number) - Value between 0.0 and 2.0
  - Lower values (0.0-0.3): More deterministic, focused
  - Medium values (0.5-0.9): Balanced creativity
  - Higher values (1.0-2.0): More creative, varied

**Returns:** `this` - For method chaining

**Example:**

```javascript
// Deterministic output for structured data
const json = await wp.aiClient.prompt( 'List the top 5 programming languages.' )
    .usingTemperature( 0.1 )
    .generateText();

// Creative output for stories
const story = await wp.aiClient.prompt( 'Write a short science fiction story.' )
    .usingTemperature( 1.2 )
    .generateText();
```

#### `usingMaxOutputTokens( maxTokens ): this`

Sets the maximum number of tokens to generate.

**Parameters:**
- `maxTokens` (number) - Maximum tokens in the response

**Returns:** `this` - For method chaining

**Example:**

```javascript
const summary = await wp.aiClient.prompt( 'Summarize the history of WordPress.' )
    .usingMaxOutputTokens( 500 )
    .generateText();
```

#### `asJsonResponse( schema = null ): this`

Requests structured JSON output.

**Parameters:**
- `schema` (object|null) - JSON Schema definition for validation

**Returns:** `this` - For method chaining

**Example:**

```javascript
const schema = {
    type: 'array',
    items: {
        type: 'object',
        properties: {
            name: { type: 'string' },
            version: { type: 'string' },
            category: { type: 'string' },
        },
        required: [ 'name', 'version', 'category' ],
    },
};

const json = await wp.aiClient.prompt( 'List 5 popular WordPress plugins.' )
    .asJsonResponse( schema )
    .generateText();

const plugins = JSON.parse( json );
console.log( plugins );
```

### System Instructions

#### `withSystemInstruction( instruction ): this`

Sets system-level instructions that guide the model's behavior.

**Parameters:**
- `instruction` (string) - System instruction

**Returns:** `this` - For method chaining

**Example:**

```javascript
const text = await wp.aiClient.prompt( 'Explain REST APIs.' )
    .withSystemInstruction( 'You are a technical writer creating documentation for developers. Use clear, concise language with code examples.' )
    .generateText();
```

## Feature Detection

### Text Generation Support

#### `isSupportedForTextGeneration(): Promise<boolean>`

Checks if the current configuration supports text generation.

**Returns:** `Promise<boolean>` - True if supported

**Example:**

```javascript
const prompt = wp.aiClient.prompt( 'Explain quantum computing.' )
    .usingTemperature( 0.7 );

if ( await prompt.isSupportedForTextGeneration() ) {
    const text = await prompt.generateText();
    console.log( text );
} else {
    console.error( 'Text generation is not available. Please configure an AI provider.' );
}
```

### Image Generation Support

#### `isSupportedForImageGeneration(): Promise<boolean>`

Checks if the current configuration supports image generation.

**Returns:** `Promise<boolean>` - True if supported

**Example:**

```javascript
const prompt = wp.aiClient.prompt( 'A futuristic city.' );

if ( await prompt.isSupportedForImageGeneration() ) {
    const image = await prompt.generateImage();
    document.querySelector( '#image' ).src = image.getDataUri();
} else {
    console.error( 'Image generation is not available. Please configure a provider like OpenAI DALL-E.' );
}
```

## Error Handling

### Try-Catch Pattern

Always wrap async calls in try-catch blocks:

```javascript
try {
    const text = await wp.aiClient.prompt( 'Hello' )
        .generateText();
    console.log( text );
} catch ( error ) {
    if ( error.code === 'rest_forbidden' ) {
        console.error( 'You do not have permission to use AI features.' );
    } else if ( error.code === 'ai_generation_failed' ) {
        console.error( 'AI generation failed:', error.message );
    } else {
        console.error( 'Error:', error.message );
    }
}
```

### Common Error Codes

- `rest_forbidden` - User lacks the `prompt_ai` capability
- `ai_generation_failed` - AI generation failed (API error, invalid config, etc.)
- `rest_invalid_param` - Invalid parameters provided
- `ai_no_provider_configured` - No AI providers configured

### Error Handling Example

```javascript
async function generateContent( prompt ) {
    try {
        // Check support first
        const builder = wp.aiClient.prompt( prompt );
        if ( ! await builder.isSupportedForTextGeneration() ) {
            return {
                success: false,
                error: 'No AI provider configured',
            };
        }
        
        // Generate
        const text = await builder.generateText();
        return {
            success: true,
            text: text,
        };
    } catch ( error ) {
        return {
            success: false,
            error: error.message,
            code: error.code,
        };
    }
}

// Usage
const result = await generateContent( 'Write a blog post intro' );
if ( result.success ) {
    document.querySelector( '#output' ).textContent = result.text;
} else {
    document.querySelector( '#error' ).textContent = result.error;
}
```

## Complete Examples

### WordPress Block Editor Integration

```javascript
const { registerBlockType } = wp.blocks;
const { Button, TextareaControl, Spinner } = wp.components;
const { useState } = wp.element;

registerBlockType( 'my-plugin/ai-content', {
    title: 'AI Content Generator',
    icon: 'admin-site',
    category: 'common',
    
    edit: function( props ) {
        const [ prompt, setPrompt ] = useState( '' );
        const [ content, setContent ] = useState( '' );
        const [ loading, setLoading ] = useState( false );
        const [ error, setError ] = useState( null );
        
        const generateContent = async () => {
            setLoading( true );
            setError( null );
            
            try {
                const text = await wp.aiClient.prompt( prompt )
                    .usingTemperature( 0.8 )
                    .generateText();
                
                setContent( text );
                props.setAttributes( { content: text } );
            } catch ( err ) {
                setError( err.message );
            } finally {
                setLoading( false );
            }
        };
        
        return (
            <div className="ai-content-block">
                <TextareaControl
                    label="Prompt"
                    value={ prompt }
                    onChange={ setPrompt }
                    placeholder="Describe what you want to generate..."
                />
                
                <Button
                    isPrimary
                    onClick={ generateContent }
                    disabled={ loading || ! prompt }
                >
                    { loading ? <Spinner /> : 'Generate Content' }
                </Button>
                
                { error && (
                    <div className="error-message">{ error }</div>
                ) }
                
                { content && (
                    <div className="generated-content">
                        { content }
                    </div>
                ) }
            </div>
        );
    },
    
    save: function( props ) {
        return (
            <div>
                { props.attributes.content }
            </div>
        );
    },
} );
```

### Dynamic Form Enhancement

```javascript
document.addEventListener( 'DOMContentLoaded', function() {
    const form = document.querySelector( '#product-form' );
    const generateBtn = document.querySelector( '#generate-description' );
    const productName = document.querySelector( '#product-name' );
    const productDescription = document.querySelector( '#product-description' );
    
    if ( ! form || ! generateBtn ) {
        return;
    }
    
    generateBtn.addEventListener( 'click', async function( e ) {
        e.preventDefault();
        
        const name = productName.value.trim();
        if ( ! name ) {
            alert( 'Please enter a product name first.' );
            return;
        }
        
        // Show loading state
        generateBtn.disabled = true;
        generateBtn.textContent = 'Generating...';
        
        try {
            // Check if AI is available
            const prompt = wp.aiClient.prompt( `Write a compelling product description for: ${ name }` );
            
            if ( ! await prompt.isSupportedForTextGeneration() ) {
                alert( 'AI generation is not available. Please configure an AI provider in Settings > AI Credentials.' );
                return;
            }
            
            // Generate description
            const description = await prompt
                .usingTemperature( 0.8 )
                .usingMaxOutputTokens( 200 )
                .withSystemInstruction( 'You are a professional copywriter specializing in e-commerce product descriptions.' )
                .generateText();
            
            // Update textarea
            productDescription.value = description;
            
        } catch ( error ) {
            console.error( 'Generation failed:', error );
            alert( 'Failed to generate description: ' + error.message );
        } finally {
            // Reset button
            generateBtn.disabled = false;
            generateBtn.textContent = 'Generate with AI';
        }
    } );
} );
```

### Image Gallery Generator

```javascript
async function generateImageGallery( prompt, count = 3 ) {
    const container = document.querySelector( '#image-gallery' );
    const loadingDiv = document.querySelector( '#loading' );
    
    if ( ! container ) {
        return;
    }
    
    // Show loading
    loadingDiv.style.display = 'block';
    container.innerHTML = '';
    
    try {
        // Check support
        const builder = wp.aiClient.prompt( prompt );
        if ( ! await builder.isSupportedForImageGeneration() ) {
            throw new Error( 'Image generation is not available on this site.' );
        }
        
        // Generate images
        const images = await builder.generateImages( count );
        
        // Display images
        for ( const image of images ) {
            const imgElement = document.createElement( 'img' );
            imgElement.src = image.getDataUri();
            imgElement.alt = prompt;
            imgElement.className = 'gallery-image';
            
            const wrapper = document.createElement( 'div' );
            wrapper.className = 'gallery-item';
            wrapper.appendChild( imgElement );
            
            // Add download button
            const downloadBtn = document.createElement( 'button' );
            downloadBtn.textContent = 'Download';
            downloadBtn.onclick = function() {
                const link = document.createElement( 'a' );
                link.href = image.getDataUri();
                link.download = `ai-generated-${ Date.now() }.png`;
                link.click();
            };
            wrapper.appendChild( downloadBtn );
            
            container.appendChild( wrapper );
        }
    } catch ( error ) {
        console.error( 'Image generation failed:', error );
        container.innerHTML = `<div class="error">Error: ${ error.message }</div>`;
    } finally {
        loadingDiv.style.display = 'none';
    }
}

// Usage
document.querySelector( '#generate-gallery-btn' ).addEventListener( 'click', function() {
    const prompt = document.querySelector( '#gallery-prompt' ).value;
    generateImageGallery( prompt, 4 );
} );
```

### Chat Interface

```javascript
class AIChatInterface {
    constructor( containerId ) {
        this.container = document.getElementById( containerId );
        this.messages = [];
        this.render();
    }
    
    render() {
        this.container.innerHTML = `
            <div class="chat-messages" id="chat-messages"></div>
            <div class="chat-input-wrapper">
                <textarea id="chat-input" placeholder="Type your message..."></textarea>
                <button id="chat-send">Send</button>
            </div>
        `;
        
        document.getElementById( 'chat-send' ).addEventListener( 'click', () => this.sendMessage() );
        document.getElementById( 'chat-input' ).addEventListener( 'keypress', ( e ) => {
            if ( e.key === 'Enter' && ! e.shiftKey ) {
                e.preventDefault();
                this.sendMessage();
            }
        } );
    }
    
    addMessage( text, isUser = false ) {
        this.messages.push( { text, isUser } );
        
        const messagesDiv = document.getElementById( 'chat-messages' );
        const messageDiv = document.createElement( 'div' );
        messageDiv.className = `chat-message ${ isUser ? 'user' : 'ai' }`;
        messageDiv.textContent = text;
        messagesDiv.appendChild( messageDiv );
        messagesDiv.scrollTop = messagesDiv.scrollHeight;
    }
    
    async sendMessage() {
        const input = document.getElementById( 'chat-input' );
        const message = input.value.trim();
        
        if ( ! message ) {
            return;
        }
        
        // Add user message
        this.addMessage( message, true );
        input.value = '';
        
        // Show typing indicator
        const typingDiv = document.createElement( 'div' );
        typingDiv.className = 'chat-message ai typing';
        typingDiv.textContent = 'Typing...';
        document.getElementById( 'chat-messages' ).appendChild( typingDiv );
        
        try {
            // Build conversation context
            const conversationPrompt = this.messages
                .map( m => `${ m.isUser ? 'User' : 'Assistant' }: ${ m.text }` )
                .join( '\n' );
            
            // Generate response
            const response = await wp.aiClient.prompt( conversationPrompt )
                .usingTemperature( 0.9 )
                .withSystemInstruction( 'You are a helpful WordPress assistant. Provide clear and concise answers.' )
                .generateText();
            
            // Remove typing indicator
            typingDiv.remove();
            
            // Add AI response
            this.addMessage( response, false );
            
        } catch ( error ) {
            typingDiv.remove();
            this.addMessage( `Error: ${ error.message }`, false );
        }
    }
}

// Initialize chat
const chat = new AIChatInterface( 'chat-container' );
```

## Best Practices

### 1. Always Check Feature Support

```javascript
// ✅ Good
const prompt = wp.aiClient.prompt( 'Generate text' );
if ( await prompt.isSupportedForTextGeneration() ) {
    const text = await prompt.generateText();
}

// ❌ Bad - may fail if no provider configured
const text = await wp.aiClient.prompt( 'Generate text' ).generateText();
```

### 2. Handle Errors Gracefully

```javascript
// ✅ Good
try {
    const text = await wp.aiClient.prompt( 'Hello' ).generateText();
} catch ( error ) {
    console.error( error );
    showUserFriendlyError( error );
}

// ❌ Bad
const text = await wp.aiClient.prompt( 'Hello' ).generateText();
```

### 3. Show Loading States

```javascript
// ✅ Good
button.disabled = true;
button.textContent = 'Generating...';
try {
    const text = await wp.aiClient.prompt( 'Hello' ).generateText();
} finally {
    button.disabled = false;
    button.textContent = 'Generate';
}
```

### 4. Use Appropriate Debouncing

```javascript
// ✅ Good - debounce user input
let debounceTimer;
inputField.addEventListener( 'input', function() {
    clearTimeout( debounceTimer );
    debounceTimer = setTimeout( async () => {
        const suggestion = await wp.aiClient.prompt( this.value )
            .generateText();
        showSuggestion( suggestion );
    }, 500 );
} );
```

## Enums Reference

### `wp.aiClient.enums.Modality`

Available output modality types:
- `Modality.TEXT` - Text output
- `Modality.IMAGE` - Image output
- `Modality.AUDIO` - Audio output (if supported)

### `wp.aiClient.enums.MessagePartType`

Message part types:
- `MessagePartType.TEXT` - Text content
- `MessagePartType.FILE` - File content (images, etc.)

## Next Steps

- Review the [PHP API](php-api.md) for server-side integration
- Learn about [Advanced Usage](advanced-usage.md) patterns
- Explore [Troubleshooting](troubleshooting.md) for common issues
