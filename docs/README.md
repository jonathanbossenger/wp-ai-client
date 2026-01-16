# WordPress AI Client - Developer Documentation

Welcome to the WordPress AI Client developer documentation. This guide will help you integrate AI capabilities into your WordPress plugins and themes.

## Documentation Index

### Getting Started
- [Installation & Setup](getting-started.md) - Get up and running quickly with the WordPress AI Client

### API References
- [PHP API Reference](php-api.md) - Complete guide to using the PHP API
- [JavaScript API Reference](javascript-api.md) - Complete guide to using the JavaScript API

### Advanced Topics
- [Advanced Usage](advanced-usage.md) - Advanced features, patterns, and best practices
- [Architecture](architecture.md) - Technical architecture and design decisions
- [Troubleshooting](troubleshooting.md) - Common issues and solutions

### Contributing
- [Development Guide](development.md) - Contributing, testing, and development workflow

## Quick Start

### PHP Example

```php
use WordPress\AI_Client\AI_Client;

// Initialize the client (on 'init' hook)
add_action( 'init', array( 'WordPress\AI_Client\AI_Client', 'init' ) );

// Generate text
$text = AI_Client::prompt( 'Write a haiku about WordPress.' )
    ->generate_text();
```

### JavaScript Example

```javascript
// Enqueue the script
wp_enqueue_script( 'wp-ai-client' );

// Generate text
const text = await wp.aiClient.prompt( 'Write a haiku about WordPress.' )
    .generateText();
```

## Key Features

- **Unified API** - Work with multiple AI providers (Anthropic, Google, OpenAI) through a single interface
- **WordPress Native** - Built with WordPress coding standards and best practices
- **Fluent Builder** - Intuitive fluent API for building AI prompts
- **Type Safe** - Full type hints for PHP 7.4+ and TypeScript definitions
- **Secure** - Capability-based access control for AI features
- **Flexible** - Support for text, images, and multimodal output

## Support

- **Issues**: [GitHub Issues](https://github.com/WordPress/wp-ai-client/issues)
- **Discussions**: [WordPress AI Team](https://make.wordpress.org/ai/)
- **Source Code**: [GitHub Repository](https://github.com/WordPress/wp-ai-client)

## License

WordPress AI Client is licensed under [GPL-2.0-or-later](../LICENSE.md).
