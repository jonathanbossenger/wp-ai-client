# Advanced Usage

This guide covers advanced patterns, best practices, and techniques for building sophisticated AI-powered WordPress features.

## Table of Contents

- [Provider-Agnostic Development](#provider-agnostic-development)
- [Performance Optimization](#performance-optimization)
- [Caching Strategies](#caching-strategies)
- [Rate Limiting](#rate-limiting)
- [Content Moderation](#content-moderation)
- [Streaming Responses](#streaming-responses)
- [Batch Processing](#batch-processing)
- [Custom Capabilities](#custom-capabilities)
- [Multisite Considerations](#multisite-considerations)
- [Security Best Practices](#security-best-practices)

## Provider-Agnostic Development

### Why Provider Agnostic?

Building provider-agnostic features ensures your plugin works on any site, regardless of which AI provider the administrator has configured.

### Best Approach: Model Preferences

Use `using_model_preference()` to specify preferred models while allowing fallback:

```php
$text = AI_Client::prompt( 'Translate this to Spanish: Hello World' )
    ->using_model_preference(
        array( 'anthropic', 'claude-sonnet-4-5' ),
        array( 'openai', 'gpt-5' ),
        array( 'google', 'gemini-3-pro' )
    )
    ->generate_text();
```

### Feature Detection Pattern

Always check support before exposing features:

```php
function is_ai_translation_available(): bool {
    return AI_Client::prompt( 'Translate test' )
        ->is_supported_for_text_generation();
}

function get_ai_translation_status(): array {
    if ( ! is_ai_translation_available() ) {
        return array(
            'available' => false,
            'message'   => 'AI translation requires an AI provider to be configured.',
        );
    }
    
    return array(
        'available' => true,
        'message'   => 'AI translation is ready to use.',
    );
}
```

### Graceful Degradation

Provide fallbacks when AI is unavailable:

```php
function generate_excerpt( string $content ): string {
    $prompt = AI_Client::prompt( "Summarize this in 2 sentences:\n\n{$content}" )
        ->using_temperature( 0.3 );
    
    if ( $prompt->is_supported_for_text_generation() ) {
        try {
            return $prompt->generate_text();
        } catch ( Exception $e ) {
            error_log( 'AI excerpt generation failed: ' . $e->getMessage() );
        }
    }
    
    // Fallback to WordPress default excerpt
    return wp_trim_words( $content, 55 );
}
```

## Performance Optimization

### Async Processing with Action Scheduler

Use Action Scheduler for long-running AI tasks:

```php
use WordPress\AI_Client\AI_Client;

function schedule_content_generation( int $post_id ): void {
    if ( ! function_exists( 'as_enqueue_async_action' ) ) {
        return;
    }
    
    as_enqueue_async_action(
        'my_plugin_generate_content',
        array( 'post_id' => $post_id ),
        'my-plugin-ai'
    );
}

add_action(
    'my_plugin_generate_content',
    function ( int $post_id ) {
        $post = get_post( $post_id );
        if ( ! $post ) {
            return;
        }
        
        try {
            $summary = AI_Client::prompt( $post->post_content )
                ->using_temperature( 0.2 )
                ->generate_text();
            
            update_post_meta( $post_id, '_ai_summary', $summary );
        } catch ( Exception $e ) {
            error_log( "AI generation failed for post {$post_id}: " . $e->getMessage() );
        }
    }
);
```

### Request Batching

Batch multiple prompts when possible:

```php
function generate_multiple_summaries( array $posts ): array {
    $summaries = array();
    
    // Group prompts
    $prompts = array();
    foreach ( $posts as $post ) {
        $prompts[ $post->ID ] = sprintf(
            'Summarize in 1 sentence: %s',
            wp_trim_words( $post->post_content, 100 )
        );
    }
    
    // Process in parallel (if using async processing)
    foreach ( $prompts as $post_id => $prompt ) {
        try {
            $summaries[ $post_id ] = AI_Client::prompt( $prompt )
                ->using_temperature( 0.2 )
                ->generate_text();
        } catch ( Exception $e ) {
            error_log( "Summary generation failed for post {$post_id}" );
            $summaries[ $post_id ] = null;
        }
    }
    
    return $summaries;
}
```

## Caching Strategies

### Transient Caching

Cache AI responses to reduce API calls:

```php
function get_cached_ai_response( string $prompt, int $expiration = HOUR_IN_SECONDS ): ?string {
    $cache_key = 'ai_response_' . md5( $prompt );
    $cached = get_transient( $cache_key );
    
    if ( false !== $cached ) {
        return $cached;
    }
    
    try {
        $response = AI_Client::prompt( $prompt )
            ->using_temperature( 0.1 ) // Low temp for consistency
            ->generate_text();
        
        set_transient( $cache_key, $response, $expiration );
        return $response;
    } catch ( Exception $e ) {
        error_log( 'AI generation failed: ' . $e->getMessage() );
        return null;
    }
}
```

### Post Meta Caching

Store generated content in post meta:

```php
function get_ai_generated_description( int $post_id ): string {
    // Check cache first
    $cached = get_post_meta( $post_id, '_ai_description', true );
    if ( $cached ) {
        return $cached;
    }
    
    $post = get_post( $post_id );
    if ( ! $post ) {
        return '';
    }
    
    try {
        $description = AI_Client::prompt( "Write a compelling description for: {$post->post_title}" )
            ->using_temperature( 0.8 )
            ->generate_text();
        
        update_post_meta( $post_id, '_ai_description', $description );
        return $description;
    } catch ( Exception $e ) {
        error_log( 'AI description generation failed: ' . $e->getMessage() );
        return '';
    }
}

// Invalidate cache when post is updated
add_action(
    'post_updated',
    function ( int $post_id ) {
        delete_post_meta( $post_id, '_ai_description' );
    }
);
```

### Object Caching

Use WordPress object cache for ephemeral data:

```php
function get_ai_suggestion( string $query ): ?string {
    $cache_key = 'ai_suggestion_' . md5( $query );
    $cached = wp_cache_get( $cache_key, 'my_plugin_ai' );
    
    if ( false !== $cached ) {
        return $cached;
    }
    
    try {
        $suggestion = AI_Client::prompt( $query )
            ->using_temperature( 0.5 )
            ->generate_text();
        
        wp_cache_set( $cache_key, $suggestion, 'my_plugin_ai', 300 ); // 5 minutes
        return $suggestion;
    } catch ( Exception $e ) {
        return null;
    }
}
```

## Rate Limiting

### User-Based Rate Limiting

Limit AI requests per user:

```php
class AI_Rate_Limiter {
    private const LIMIT_KEY_PREFIX = 'ai_rate_limit_';
    private const MAX_REQUESTS = 50; // Per hour
    private const TIME_WINDOW = 3600; // 1 hour in seconds
    
    public static function check_rate_limit( int $user_id ): bool {
        $key = self::LIMIT_KEY_PREFIX . $user_id;
        $count = (int) get_transient( $key );
        
        return $count < self::MAX_REQUESTS;
    }
    
    public static function increment_count( int $user_id ): void {
        $key = self::LIMIT_KEY_PREFIX . $user_id;
        $count = (int) get_transient( $key );
        
        if ( false === $count ) {
            set_transient( $key, 1, self::TIME_WINDOW );
        } else {
            set_transient( $key, $count + 1, self::TIME_WINDOW );
        }
    }
    
    public static function get_remaining_requests( int $user_id ): int {
        $key = self::LIMIT_KEY_PREFIX . $user_id;
        $count = (int) get_transient( $key );
        
        return max( 0, self::MAX_REQUESTS - $count );
    }
}

// Usage
function generate_with_rate_limit( string $prompt ): string {
    $user_id = get_current_user_id();
    
    if ( ! AI_Rate_Limiter::check_rate_limit( $user_id ) ) {
        throw new Exception( 'Rate limit exceeded. Please try again later.' );
    }
    
    $text = AI_Client::prompt( $prompt )->generate_text();
    AI_Rate_Limiter::increment_count( $user_id );
    
    return $text;
}
```

### Global Rate Limiting

Limit total site-wide AI requests:

```php
class Site_AI_Rate_Limiter {
    private const OPTION_NAME = 'ai_usage_stats';
    private const MAX_DAILY_REQUESTS = 1000;
    
    public static function can_make_request(): bool {
        $stats = self::get_stats();
        $today = gmdate( 'Y-m-d' );
        
        if ( $stats['date'] !== $today ) {
            // New day, reset counter
            self::reset_stats();
            return true;
        }
        
        return $stats['count'] < self::MAX_DAILY_REQUESTS;
    }
    
    public static function record_request(): void {
        $stats = self::get_stats();
        $today = gmdate( 'Y-m-d' );
        
        if ( $stats['date'] !== $today ) {
            $stats = array(
                'date'  => $today,
                'count' => 0,
            );
        }
        
        $stats['count']++;
        update_option( self::OPTION_NAME, $stats );
    }
    
    private static function get_stats(): array {
        return get_option(
            self::OPTION_NAME,
            array(
                'date'  => gmdate( 'Y-m-d' ),
                'count' => 0,
            )
        );
    }
    
    private static function reset_stats(): void {
        update_option(
            self::OPTION_NAME,
            array(
                'date'  => gmdate( 'Y-m-d' ),
                'count' => 0,
            )
        );
    }
    
    public static function get_remaining_requests(): int {
        $stats = self::get_stats();
        $today = gmdate( 'Y-m-d' );
        
        if ( $stats['date'] !== $today ) {
            return self::MAX_DAILY_REQUESTS;
        }
        
        return max( 0, self::MAX_DAILY_REQUESTS - $stats['count'] );
    }
}
```

## Content Moderation

### Input Validation

Validate and sanitize user input before sending to AI:

```php
function validate_ai_prompt( string $prompt ): array {
    $errors = array();
    
    // Length check
    if ( strlen( $prompt ) < 10 ) {
        $errors[] = 'Prompt must be at least 10 characters.';
    }
    
    if ( strlen( $prompt ) > 5000 ) {
        $errors[] = 'Prompt is too long (max 5000 characters).';
    }
    
    // Content check
    $disallowed_patterns = array(
        '/\b(ignore|disregard)\s+(previous|prior|all)\s+(instructions|prompts)\b/i',
        '/\b(system|admin)\s+prompt\b/i',
    );
    
    foreach ( $disallowed_patterns as $pattern ) {
        if ( preg_match( $pattern, $prompt ) ) {
            $errors[] = 'Prompt contains disallowed content.';
            break;
        }
    }
    
    return $errors;
}

function safe_generate_text( string $prompt ): string {
    $errors = validate_ai_prompt( $prompt );
    if ( ! empty( $errors ) ) {
        throw new Exception( implode( ' ', $errors ) );
    }
    
    return AI_Client::prompt( sanitize_textarea_field( $prompt ) )
        ->generate_text();
}
```

### Output Sanitization

Always sanitize AI-generated content:

```php
function sanitize_ai_text( string $text ): string {
    // Remove any potential scripts or malicious content
    $text = wp_kses_post( $text );
    
    // Trim excessive whitespace
    $text = preg_replace( '/\s+/', ' ', $text );
    
    return trim( $text );
}

function generate_safe_content( string $prompt ): string {
    $text = AI_Client::prompt( $prompt )->generate_text();
    return sanitize_ai_text( $text );
}
```

## Custom Capabilities

### Fine-Grained Access Control

Create custom capabilities for different AI features:

```php
// Add custom capabilities
function add_ai_capabilities(): void {
    $admin = get_role( 'administrator' );
    $editor = get_role( 'editor' );
    
    if ( $admin ) {
        $admin->add_cap( 'prompt_ai' );
        $admin->add_cap( 'generate_ai_images' );
        $admin->add_cap( 'use_ai_premium_models' );
    }
    
    if ( $editor ) {
        $editor->add_cap( 'prompt_ai' );
        $editor->add_cap( 'generate_ai_images' );
    }
}
register_activation_hook( __FILE__, 'add_ai_capabilities' );

// Check capabilities before AI operations
function can_user_generate_images( int $user_id ): bool {
    $user = get_user_by( 'id', $user_id );
    return $user && $user->has_cap( 'generate_ai_images' );
}

function generate_image_with_capability_check( string $prompt ): ?File {
    if ( ! can_user_generate_images( get_current_user_id() ) ) {
        throw new Exception( 'You do not have permission to generate images.' );
    }
    
    return AI_Client::prompt( $prompt )->generate_image();
}
```

### Usage-Based Capabilities

Grant capabilities based on usage:

```php
add_filter(
    'user_has_cap',
    function ( array $allcaps, array $caps, array $args, WP_User $user ): array {
        if ( in_array( 'use_ai_premium_models', $caps, true ) ) {
            // Only allow premium models for users with specific plan
            $user_plan = get_user_meta( $user->ID, 'subscription_plan', true );
            $allcaps['use_ai_premium_models'] = in_array( $user_plan, array( 'premium', 'enterprise' ), true );
        }
        
        return $allcaps;
    },
    10,
    4
);
```

## Multisite Considerations

### Network-Wide Configuration

Set up API credentials for all sites in a multisite network:

```php
// In a network-activated plugin
function get_network_ai_credentials(): array {
    return get_site_option(
        'ai_client_network_credentials',
        array()
    );
}

add_filter(
    'wordpress_ai_client_api_credentials',
    function ( array $credentials ): array {
        if ( is_multisite() ) {
            $network_credentials = get_network_ai_credentials();
            $credentials = array_merge( $credentials, $network_credentials );
        }
        return $credentials;
    }
);
```

### Per-Site Limits

Implement per-site usage limits in multisite:

```php
function get_site_ai_limit( int $blog_id ): int {
    return (int) get_blog_option( $blog_id, 'ai_monthly_limit', 1000 );
}

function check_site_ai_limit( int $blog_id ): bool {
    $limit = get_site_ai_limit( $blog_id );
    $usage = (int) get_blog_option( $blog_id, 'ai_monthly_usage', 0 );
    
    return $usage < $limit;
}

function record_site_ai_usage( int $blog_id ): void {
    $usage = (int) get_blog_option( $blog_id, 'ai_monthly_usage', 0 );
    update_blog_option( $blog_id, 'ai_monthly_usage', $usage + 1 );
}
```

## Security Best Practices

### Input Sanitization

Always sanitize user input:

```php
function secure_ai_prompt( string $user_input ): string {
    // Sanitize input
    $prompt = sanitize_textarea_field( $user_input );
    
    // Validate
    if ( empty( $prompt ) || strlen( $prompt ) > 5000 ) {
        throw new Exception( 'Invalid prompt.' );
    }
    
    // Check for prompt injection attempts
    $dangerous_patterns = array(
        '/ignore\s+previous/i',
        '/disregard\s+all/i',
        '/system\s+message/i',
    );
    
    foreach ( $dangerous_patterns as $pattern ) {
        if ( preg_match( $pattern, $prompt ) ) {
            throw new Exception( 'Prompt contains disallowed content.' );
        }
    }
    
    return $prompt;
}
```

### Capability Checks

Always verify user capabilities:

```php
function verify_ai_access(): void {
    if ( ! current_user_can( 'prompt_ai' ) ) {
        wp_die(
            esc_html__( 'You do not have permission to use AI features.', 'my-plugin' ),
            esc_html__( 'Permission Denied', 'my-plugin' ),
            array( 'response' => 403 )
        );
    }
}

// In REST API
public function check_permission(): bool {
    return current_user_can( 'prompt_ai' );
}
```

### Nonce Verification

Use nonces for form submissions:

```php
function handle_ai_form_submission(): void {
    // Verify nonce
    if ( ! isset( $_POST['ai_nonce'] ) || ! wp_verify_nonce( $_POST['ai_nonce'], 'generate_ai_content' ) ) {
        wp_die( 'Security check failed.' );
    }
    
    // Verify capability
    verify_ai_access();
    
    // Sanitize input
    $prompt = secure_ai_prompt( $_POST['prompt'] ?? '' );
    
    // Generate content
    try {
        $text = AI_Client::prompt( $prompt )->generate_text();
        echo wp_kses_post( $text );
    } catch ( Exception $e ) {
        error_log( 'AI generation failed: ' . $e->getMessage() );
        echo 'Generation failed. Please try again.';
    }
}
```

### Output Escaping

Always escape AI-generated content:

```php
// For HTML output
$text = AI_Client::prompt( 'Generate content' )->generate_text();
echo wp_kses_post( $text );

// For attribute values
$text = AI_Client::prompt( 'Generate title' )->generate_text();
echo '<div title="' . esc_attr( $text ) . '">';

// For JavaScript strings
$text = AI_Client::prompt( 'Generate message' )->generate_text();
echo '<script>const message = ' . wp_json_encode( $text ) . ';</script>';
```

## Next Steps

- Review the [Architecture](architecture.md) to understand the technical implementation
- Check the [Troubleshooting Guide](troubleshooting.md) for common issues
- Explore the [Development Guide](development.md) for contributing
