# Getting Started with WordPress AI Client

This guide will help you install and configure the WordPress AI Client in your WordPress plugin or theme.

## Requirements

- **WordPress**: 6.0 or higher
- **PHP**: 7.4 or higher
- **Composer**: For dependency management

## Installation

### Via Composer (Recommended)

Install the package using Composer:

```bash
composer require wordpress/wp-ai-client
```

### Manual Installation

1. Download the latest release from [GitHub](https://github.com/WordPress/wp-ai-client/releases)
2. Extract the files to your plugin or theme directory
3. Include the autoloader in your code:

```php
require_once __DIR__ . '/vendor/autoload.php';
```

## Initial Setup

### 1. Initialize the Client

The AI Client must be initialized on the WordPress `init` hook. Add this to your plugin or theme:

```php
add_action( 'init', array( 'WordPress\AI_Client\AI_Client', 'init' ) );
```

This initialization step:
- Sets up the HTTP client integration with the PHP AI Client SDK
- Registers the settings screen for API credentials
- Registers REST API endpoints
- Sets up capability checks

### 2. Configure API Credentials

Before you can use the AI Client, you need to configure API keys for at least one AI provider.

#### Option A: Using the Admin Interface

1. Log in to WordPress Admin as an administrator
2. Navigate to **Settings > AI Credentials**
3. Enter your API keys for one or more providers:
   - **Anthropic**: Get your API key from [console.anthropic.com](https://console.anthropic.com/)
   - **Google**: Get your API key from [ai.google.dev](https://ai.google.dev/)
   - **OpenAI**: Get your API key from [platform.openai.com](https://platform.openai.com/)
4. Click **Save Changes**

#### Option B: Programmatic Configuration

You can also set API credentials programmatically using a filter:

```php
add_filter(
    'wordpress_ai_client_api_credentials',
    function ( array $credentials ): array {
        $credentials['anthropic'] = array(
            'api_key' => 'your-api-key-here',
        );
        return $credentials;
    }
);
```

**Security Note**: Never hardcode API keys in your plugin. Use environment variables or constants defined in `wp-config.php`:

```php
// In wp-config.php
define( 'ANTHROPIC_API_KEY', 'your-api-key-here' );

// In your plugin
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

### 3. Verify Installation

Create a simple test to verify the installation:

```php
add_action(
    'wp_loaded',
    function () {
        if ( ! current_user_can( 'manage_options' ) ) {
            return;
        }

        try {
            $prompt = WordPress\AI_Client\AI_Client::prompt( 'Say hello!' );
            
            // Check if text generation is supported
            if ( $prompt->is_supported_for_text_generation() ) {
                $text = $prompt->generate_text();
                error_log( 'AI Client Test: ' . $text );
            } else {
                error_log( 'AI Client: No providers configured for text generation' );
            }
        } catch ( Exception $e ) {
            error_log( 'AI Client Error: ' . $e->getMessage() );
        }
    }
);
```

Check your WordPress debug log to see the results.

## Using the JavaScript API

To use the client-side JavaScript API, you need to enqueue the script:

### 1. Enqueue the Script

```php
add_action(
    'admin_enqueue_scripts',
    function () {
        wp_enqueue_script( 'wp-ai-client' );
    }
);
```

### 2. Use in JavaScript

The API is available globally as `wp.aiClient`:

```javascript
( async function() {
    try {
        const text = await wp.aiClient.prompt( 'Hello from JavaScript!' )
            .generateText();
        console.log( 'AI Response:', text );
    } catch ( error ) {
        console.error( 'AI Error:', error );
    }
} )();
```

## Capability Management

By default, only administrators have access to the AI Client features. The `prompt_ai` capability controls access.

### Granting Capability to Custom Roles

```php
// Grant capability to editors
$role = get_role( 'editor' );
if ( $role ) {
    $role->add_cap( 'prompt_ai' );
}
```

### Custom Capability Checks

You can implement custom capability logic using the `user_has_cap` filter:

```php
add_filter(
    'user_has_cap',
    function ( array $allcaps, array $caps, array $args, WP_User $user ): array {
        // Grant prompt_ai to users with a specific meta key
        if ( in_array( 'prompt_ai', $caps, true ) ) {
            if ( get_user_meta( $user->ID, 'allow_ai_access', true ) ) {
                $allcaps['prompt_ai'] = true;
            }
        }
        return $allcaps;
    },
    10,
    4
);
```

## Next Steps

Now that you have the AI Client installed and configured, you can:

- Learn about the [PHP API](php-api.md) for server-side AI integration
- Explore the [JavaScript API](javascript-api.md) for client-side features
- Discover [Advanced Usage](advanced-usage.md) patterns and best practices
- Review the [Architecture](architecture.md) to understand how it works

## Troubleshooting

If you encounter issues during setup:

1. **Check PHP Version**: Ensure you're running PHP 7.4 or higher
2. **Verify API Keys**: Make sure your API keys are valid and have proper permissions
3. **Check Capabilities**: Ensure the current user has the `prompt_ai` capability
4. **Enable Debug Logging**: Add `define( 'WP_DEBUG_LOG', true );` to `wp-config.php`
5. **Review the [Troubleshooting Guide](troubleshooting.md)** for common issues and solutions

## Common Installation Issues

### Composer Install Fails

If `composer require` fails, try:

```bash
composer require wordpress/wp-ai-client --ignore-platform-reqs
```

### Namespace Not Found

If you get "Class not found" errors:

1. Ensure the autoloader is included: `require_once __DIR__ . '/vendor/autoload.php';`
2. Run `composer dump-autoload` to regenerate the autoloader
3. Check that the package is in your `composer.json` dependencies

### Settings Page Not Showing

If the AI Credentials settings page doesn't appear:

1. Verify that `AI_Client::init()` is called on the `init` hook
2. Check that you're logged in as an administrator
3. Clear any caching plugins or opcache

## Support

If you need help:

- Check the [Troubleshooting Guide](troubleshooting.md)
- Search [GitHub Issues](https://github.com/WordPress/wp-ai-client/issues)
- Ask in the [WordPress AI Team discussions](https://make.wordpress.org/ai/)
