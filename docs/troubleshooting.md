# Troubleshooting

Common issues and solutions for the WordPress AI Client.

## Table of Contents

- [Installation Issues](#installation-issues)
- [Configuration Problems](#configuration-problems)
- [Generation Failures](#generation-failures)
- [Permission Errors](#permission-errors)
- [JavaScript API Issues](#javascript-api-issues)
- [Performance Problems](#performance-problems)
- [Debugging Tips](#debugging-tips)

## Installation Issues

### Composer Install Fails

**Problem**: `composer require wordpress/wp-ai-client` fails with platform requirements error.

**Solution**:

```bash
# Ignore platform requirements
composer require wordpress/wp-ai-client --ignore-platform-reqs

# Or specify PHP version in composer.json
{
    "config": {
        "platform": {
            "php": "7.4"
        }
    }
}
```

### Autoloader Not Found

**Problem**: `Fatal error: Class 'WordPress\AI_Client\AI_Client' not found`

**Solutions**:

1. **Ensure autoloader is included**:

```php
require_once __DIR__ . '/vendor/autoload.php';
```

2. **Regenerate autoloader**:

```bash
composer dump-autoload
```

3. **Check installation**:

```bash
composer show wordpress/wp-ai-client
```

### Settings Page Missing

**Problem**: AI Credentials settings page doesn't appear in WordPress Admin.

**Solutions**:

1. **Verify initialization**:

```php
// Must be called on 'init' hook
add_action( 'init', array( 'WordPress\AI_Client\AI_Client', 'init' ) );
```

2. **Check user role**:
   - Only administrators can see the settings page
   - Log in as an administrator

3. **Clear caches**:
   - Clear object cache
   - Clear opcache: `opcache_reset()`
   - Deactivate caching plugins temporarily

## Configuration Problems

### API Keys Not Working

**Problem**: API credentials configured but generation fails with authentication errors.

**Solutions**:

1. **Verify API key format**:

```php
// Check if credentials are loaded
add_action( 'init', function() {
    if ( current_user_can( 'manage_options' ) ) {
        $credentials = get_option( 'wp_ai_client_credentials', array() );
        error_log( 'AI Credentials: ' . print_r( $credentials, true ) );
    }
} );
```

2. **Test API key directly**:
   - Visit provider's website (e.g., console.anthropic.com)
   - Verify key is active and has proper permissions
   - Check for spending limits or quotas

3. **Check for filter overrides**:

```php
// Temporarily remove filters to test
remove_all_filters( 'wordpress_ai_client_api_credentials' );
```

### Wrong Provider Used

**Problem**: A different provider is being used than expected.

**Solution**:

Use specific model preferences:

```php
// Specify exact provider
$text = AI_Client::prompt( 'Hello' )
    ->using_model_preference(
        array( 'anthropic', 'claude-sonnet-4-5' )
    )
    ->generate_text();

// Or force specific model
use WordPress\AiClient\ProviderImplementations\Anthropic\AnthropicProvider as Anthropic;

$text = AI_Client::prompt( 'Hello' )
    ->using_model( Anthropic::model( 'claude-sonnet-4-5' ) )
    ->generate_text();
```

### Environment Variables Not Loading

**Problem**: API keys defined in environment variables or wp-config.php not being used.

**Solution**:

Ensure the filter is properly configured:

```php
// In wp-config.php or plugin
define( 'ANTHROPIC_API_KEY', 'your-key' );

// In plugin initialization
add_filter(
    'wordpress_ai_client_api_credentials',
    function ( array $credentials ): array {
        if ( defined( 'ANTHROPIC_API_KEY' ) ) {
            $credentials['anthropic'] = array(
                'api_key' => ANTHROPIC_API_KEY,
            );
        }
        return $credentials;
    },
    20 // Higher priority to override
);
```

## Generation Failures

### "No Provider Configured" Error

**Problem**: `AI generation failed: No provider configured for this request.`

**Solutions**:

1. **Check feature support first**:

```php
$prompt = AI_Client::prompt( 'Hello' );

if ( ! $prompt->is_supported_for_text_generation() ) {
    // No provider available
    echo 'Please configure an AI provider in Settings > AI Credentials';
    return;
}

$text = $prompt->generate_text();
```

2. **Verify provider supports requested features**:

```php
// Not all providers support image generation
if ( $prompt->is_supported_for_image_generation() ) {
    $image = $prompt->generate_image();
}
```

3. **Check API credentials are valid**:
   - Test with a simple prompt
   - Verify key hasn't expired
   - Check provider's status page

### Rate Limit Errors

**Problem**: `AI generation failed: Rate limit exceeded.`

**Solutions**:

1. **Implement rate limiting**:

```php
// Check before generating
$user_id = get_current_user_id();
$key = 'ai_rate_limit_' . $user_id;
$count = (int) get_transient( $key );

if ( $count >= 50 ) {
    throw new Exception( 'Rate limit exceeded. Please try again later.' );
}

// Generate and increment
$text = AI_Client::prompt( 'Hello' )->generate_text();
set_transient( $key, $count + 1, HOUR_IN_SECONDS );
```

2. **Implement caching**:

```php
$cache_key = 'ai_response_' . md5( $prompt );
$cached = get_transient( $cache_key );

if ( false !== $cached ) {
    return $cached;
}

$text = AI_Client::prompt( $prompt )->generate_text();
set_transient( $cache_key, $text, HOUR_IN_SECONDS );
```

3. **Contact provider**:
   - Request rate limit increase
   - Upgrade to higher tier

### Invalid JSON Schema

**Problem**: JSON response validation fails.

**Solution**:

Verify schema is valid JSON Schema:

```php
$schema = array(
    'type'       => 'object',
    'properties' => array(
        'title' => array( 'type' => 'string' ),
        'count' => array( 'type' => 'integer' ), // Not 'int'
    ),
    'required'   => array( 'title', 'count' ),
);

try {
    $json = AI_Client::prompt( 'Generate data' )
        ->as_json_response( $schema )
        ->generate_text();
    
    $data = json_decode( $json, true );
} catch ( Exception $e ) {
    error_log( 'Schema validation failed: ' . $e->getMessage() );
}
```

### Timeout Errors

**Problem**: Requests timeout before completion.

**Solutions**:

1. **Increase timeout**:

```php
add_filter(
    'http_request_timeout',
    function ( $timeout ) {
        return 60; // 60 seconds
    }
);
```

2. **Use async processing**:

```php
// Use Action Scheduler for long tasks
as_enqueue_async_action(
    'my_plugin_ai_generation',
    array( 'prompt' => $prompt )
);
```

3. **Reduce output length**:

```php
$text = AI_Client::prompt( 'Summarize briefly' )
    ->using_max_output_tokens( 500 ) // Limit tokens
    ->generate_text();
```

## Permission Errors

### "Permission Denied" Error

**Problem**: REST API returns 403 Forbidden error.

**Solutions**:

1. **Verify user has capability**:

```php
// Check current user
if ( ! current_user_can( 'prompt_ai' ) ) {
    echo 'You need the prompt_ai capability';
}

// Grant to custom role
$role = get_role( 'editor' );
if ( $role ) {
    $role->add_cap( 'prompt_ai' );
}
```

2. **Check REST authentication**:

```javascript
// Ensure nonce is sent
wp.apiFetch( {
    path: '/wp-ai/v1/prompt',
    method: 'POST',
    data: {
        prompt: 'Hello',
    },
} );
```

3. **Verify administrator access**:
   - Log out and log back in
   - Check if user role is actually 'administrator'

### Custom Capability Not Working

**Problem**: Custom capability check not functioning.

**Solution**:

Ensure capability is properly granted:

```php
// Add capability to role
function add_custom_ai_capabilities() {
    $editor = get_role( 'editor' );
    if ( $editor ) {
        $editor->add_cap( 'prompt_ai' );
    }
}
register_activation_hook( __FILE__, 'add_custom_ai_capabilities' );

// Or use filter
add_filter(
    'user_has_cap',
    function ( array $allcaps, array $caps, array $args, WP_User $user ): array {
        if ( in_array( 'prompt_ai', $caps, true ) ) {
            // Custom logic
            if ( get_user_meta( $user->ID, 'ai_access', true ) ) {
                $allcaps['prompt_ai'] = true;
            }
        }
        return $allcaps;
    },
    10,
    4
);
```

## JavaScript API Issues

### `wp.aiClient` is Undefined

**Problem**: `wp.aiClient` is not available in JavaScript.

**Solutions**:

1. **Ensure script is enqueued**:

```php
add_action(
    'admin_enqueue_scripts',
    function () {
        wp_enqueue_script( 'wp-ai-client' );
    }
);
```

2. **Check script loaded**:

```javascript
if ( typeof wp !== 'undefined' && wp.aiClient ) {
    // API is available
} else {
    console.error( 'wp.aiClient is not loaded' );
}
```

3. **Verify dependencies**:
   - Script depends on `@wordpress/api-fetch`
   - Must be loaded in admin or with proper enqueue

### CORS Errors

**Problem**: REST API calls fail with CORS errors.

**Solution**:

This shouldn't happen for same-origin requests. If it does:

```php
// Add CORS headers (only if needed for external requests)
add_action(
    'rest_api_init',
    function () {
        remove_filter( 'rest_pre_serve_request', 'rest_send_cors_headers' );
        add_filter( 'rest_pre_serve_request', function ( $value ) {
            header( 'Access-Control-Allow-Origin: *' );
            header( 'Access-Control-Allow-Methods: GET, POST, OPTIONS' );
            header( 'Access-Control-Allow-Credentials: true' );
            return $value;
        } );
    }
);
```

### Promise Never Resolves

**Problem**: JavaScript promise hangs indefinitely.

**Solutions**:

1. **Add timeout**:

```javascript
const timeout = ( ms ) => new Promise( ( _, reject ) =>
    setTimeout( () => reject( new Error( 'Timeout' ) ), ms )
);

try {
    const text = await Promise.race( [
        wp.aiClient.prompt( 'Hello' ).generateText(),
        timeout( 30000 ), // 30 second timeout
    ] );
} catch ( error ) {
    console.error( 'Request failed or timed out:', error );
}
```

2. **Check network tab**:
   - Open browser DevTools
   - Check Network tab for failed requests
   - Look for error responses

3. **Enable debug mode**:

```php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
```

## Performance Problems

### Slow Generation Times

**Problem**: AI generation takes too long.

**Solutions**:

1. **Reduce output length**:

```php
$text = AI_Client::prompt( 'Summarize' )
    ->using_max_output_tokens( 200 ) // Shorter response
    ->generate_text();
```

2. **Use faster models**:

```php
$text = AI_Client::prompt( 'Quick task' )
    ->using_model_preference(
        array( 'anthropic', 'claude-3-haiku' ) // Faster model
    )
    ->generate_text();
```

3. **Implement caching**:

```php
$cache_key = 'ai_' . md5( $prompt );
$cached = wp_cache_get( $cache_key, 'my_plugin' );

if ( false !== $cached ) {
    return $cached;
}

$text = AI_Client::prompt( $prompt )->generate_text();
wp_cache_set( $cache_key, $text, 'my_plugin', 300 );
```

4. **Use async processing**:

```php
// Process in background
as_enqueue_async_action( 'my_plugin_ai_task', array( 'prompt' => $prompt ) );
```

### High Memory Usage

**Problem**: PHP runs out of memory during generation.

**Solutions**:

1. **Increase memory limit**:

```php
ini_set( 'memory_limit', '256M' );
```

2. **Process in batches**:

```php
// Instead of processing all at once
$posts = get_posts( array( 'posts_per_page' => 10 ) );
foreach ( $posts as $post ) {
    // Generate content for each post
}
```

3. **Clean up after generation**:

```php
$text = AI_Client::prompt( $large_prompt )->generate_text();
unset( $large_prompt ); // Free memory
gc_collect_cycles(); // Force garbage collection
```

## Debugging Tips

### Enable Debug Logging

```php
// In wp-config.php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );
```

### Log AI Requests

```php
add_action(
    'init',
    function () {
        if ( defined( 'WP_DEBUG' ) && WP_DEBUG ) {
            // Log all AI prompts
            add_filter(
                'wordpress_ai_client_before_generate',
                function ( $prompt ) {
                    error_log( 'AI Prompt: ' . print_r( $prompt, true ) );
                    return $prompt;
                }
            );
        }
    }
);
```

### Check Provider Status

Test API directly:

```bash
# Test Anthropic API
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-3-haiku-20240307",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### Debug REST API

```javascript
// Log all requests
wp.apiFetch.use( ( options, next ) => {
    console.log( 'Request:', options );
    return next( options ).then( ( response ) => {
        console.log( 'Response:', response );
        return response;
    } );
} );
```

### Check PHP Error Logs

```bash
# Find WordPress debug log
tail -f /path/to/wp-content/debug.log

# Or check PHP error log
tail -f /var/log/php/error.log
```

### Verify Dependencies

```php
// Check if dependencies are loaded
add_action( 'admin_init', function() {
    if ( ! class_exists( 'WordPress\AiClient\AiClient' ) ) {
        add_action( 'admin_notices', function() {
            echo '<div class="error"><p>PHP AI Client library is not loaded.</p></div>';
        } );
    }
} );
```

## Common Error Messages

### "You must call AI_Client::init()"

**Cause**: Using AI_Client before initialization.

**Solution**:

```php
add_action( 'init', array( 'WordPress\AI_Client\AI_Client', 'init' ) );

// Then use AI_Client in later hooks
add_action(
    'wp_loaded',
    function () {
        $text = AI_Client::prompt( 'Hello' )->generate_text();
    }
);
```

### "Failed to discover HTTP client"

**Cause**: HTTP client discovery strategy not registered.

**Solution**:

Ensure `AI_Client::init()` is called - it registers the HTTP client.

### "Invalid prompt configuration"

**Cause**: Conflicting or invalid configuration options.

**Solution**:

Check that configuration makes sense:

```php
// ❌ Bad - conflicting configuration
$text = AI_Client::prompt( 'Hello' )
    ->as_json_response( $schema )
    ->generate_image(); // Can't generate image with JSON schema

// ✅ Good
$json = AI_Client::prompt( 'Generate data' )
    ->as_json_response( $schema )
    ->generate_text();
```

## Getting Help

If you're still experiencing issues:

1. **Check GitHub Issues**: [github.com/WordPress/wp-ai-client/issues](https://github.com/WordPress/wp-ai-client/issues)
2. **Enable Debug Logging**: Share relevant logs when reporting issues
3. **Provide Context**: Include PHP version, WordPress version, and provider being used
4. **Create Minimal Reproduction**: Isolate the problem in a minimal test case

## Next Steps

- Review [Architecture](architecture.md) to understand how components work
- Check [Advanced Usage](advanced-usage.md) for patterns and best practices
- Explore [Development Guide](development.md) for contributing
