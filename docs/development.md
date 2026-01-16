# Development Guide

Guide for contributing to the WordPress AI Client project.

## Table of Contents

- [Getting Started](#getting-started)
- [Development Environment](#development-environment)
- [Code Standards](#code-standards)
- [Testing](#testing)
- [Building Assets](#building-assets)
- [Documentation](#documentation)
- [Pull Request Process](#pull-request-process)
- [Release Process](#release-process)

## Getting Started

### Prerequisites

- **PHP**: 7.4 or higher
- **Node.js**: 16 or higher (check `.nvmrc`)
- **Composer**: Latest version
- **WordPress**: 6.0 or higher

### Initial Setup

1. **Clone the repository**:

```bash
git clone https://github.com/WordPress/wp-ai-client.git
cd wp-ai-client
```

2. **Install PHP dependencies**:

```bash
composer install
```

3. **Install Node dependencies**:

```bash
npm install
```

4. **Build JavaScript assets**:

```bash
npm run build
```

## Development Environment

### Using wp-env (Recommended)

The project uses `@wordpress/env` for local development:

1. **Start the environment**:

```bash
npm run wp-env start
```

2. **Access WordPress**:
   - URL: http://localhost:8888
   - Admin: http://localhost:8888/wp-admin
   - Username: `admin`
   - Password: `password`

3. **Stop the environment**:

```bash
npm run wp-env stop
```

### Configuration

The `.wp-env.json` file configures the development environment:

```json
{
  "core": "WordPress/WordPress#6.4",
  "plugins": [
    "."
  ],
  "env": {
    "tests": {
      "mappings": {
        "wp-content/plugins/wp-ai-client": "."
      }
    }
  }
}
```

### Manual Setup

If not using wp-env:

1. Install WordPress locally
2. Create a symlink or copy the plugin to `wp-content/plugins/`
3. Activate the plugin
4. Configure API credentials in Settings > AI Credentials

## Code Standards

### PHP Standards

Follow WordPress Coding Standards with PSR-4 autoloading exceptions.

#### Key Requirements

1. **File Names**: Match class names for PSR-4 autoloading
2. **Type Hints**: Use explicit type hints for all parameters and return values
3. **PHPDoc**: Complete documentation with `@since`, `@param`, and `@return` tags
4. **Naming Conventions**:
   - Interfaces: `*_Interface`
   - Traits: `*_Trait`
   - Enums: `*_Enum`

#### Example

```php
<?php
/**
 * Class WordPress\AI_Client\Example_Class
 *
 * @since n.e.x.t
 * @package WordPress\AI_Client
 */

namespace WordPress\AI_Client;

/**
 * Example class demonstrating coding standards.
 *
 * @since n.e.x.t
 */
class Example_Class {
    
    /**
     * Processes the given input and returns a result.
     *
     * @since n.e.x.t
     *
     * @param string $input The input to process.
     * @param int    $limit Maximum number of items to process.
     * @return array<string, mixed> The processed result.
     */
    public function process( string $input, int $limit = 10 ): array {
        // Implementation
        return array();
    }
}
```

### JavaScript Standards

Follow WordPress JavaScript Coding Standards.

#### Key Requirements

1. **Use WordPress Scripts**: Leverage `@wordpress/scripts` for building
2. **camelCase**: For variable and function names
3. **JSDoc**: Document functions with types
4. **Modern JavaScript**: Use ES6+ features

#### Example

```javascript
/**
 * Generates AI content based on the provided prompt.
 *
 * @param {string} prompt - The prompt to generate from.
 * @param {Object} options - Generation options.
 * @param {number} options.temperature - Temperature setting (0-2).
 * @return {Promise<string>} The generated text.
 */
async function generateContent( prompt, options = {} ) {
    const { temperature = 0.7 } = options;
    
    return await wp.aiClient.prompt( prompt )
        .usingTemperature( temperature )
        .generateText();
}
```

### Running Linters

#### PHP Linting

```bash
# Run both PHPCodeSniffer and PHPStan
composer lint

# Run individually
composer phpcs    # Code standards
composer phpstan  # Static analysis

# Auto-fix issues
composer phpcbf
```

#### JavaScript Linting

```bash
# Lint JavaScript
npm run lint-js

# Format JavaScript
npm run format-js
```

## Testing

### PHP Tests

The project uses PHPUnit for PHP testing.

#### Running Tests

```bash
# Run all tests (single site)
npm run test-php

# Run all tests (multisite)
npm run test-php-multisite

# Run specific test file
npm run wp-env run tests-cli --env-cwd=wp-content/plugins/wp-ai-client vendor/bin/phpunit tests/phpunit/tests/Example_Test.php

# Run with coverage (requires Xdebug)
npm run wp-env run tests-cli --env-cwd=wp-content/plugins/wp-ai-client vendor/bin/phpunit --coverage-html coverage/
```

#### Writing Tests

Create tests in `tests/phpunit/tests/`:

```php
<?php
/**
 * Tests for Example_Class.
 *
 * @package WordPress\AI_Client
 */

namespace WordPress\AI_Client\Tests;

use WordPress\AI_Client\Example_Class;

/**
 * Test case for Example_Class.
 */
class Example_Test extends \WP_UnitTestCase {
    
    /**
     * Test instance.
     *
     * @var Example_Class
     */
    private Example_Class $instance;
    
    /**
     * Set up test.
     */
    public function set_up(): void {
        parent::set_up();
        $this->instance = new Example_Class();
    }
    
    /**
     * Test that process returns expected result.
     */
    public function test_process_returns_array(): void {
        $result = $this->instance->process( 'test input' );
        $this->assertIsArray( $result );
    }
    
    /**
     * Test that process respects limit parameter.
     */
    public function test_process_respects_limit(): void {
        $result = $this->instance->process( 'test input', 5 );
        $this->assertCount( 5, $result );
    }
}
```

#### Test Structure

```
tests/
├── phpunit/
│   ├── includes/         # Test utilities
│   ├── tests/            # Test cases
│   │   ├── API_Credentials/
│   │   ├── Builders/
│   │   └── ...
│   ├── bootstrap.php     # Test bootstrap
│   └── multisite.xml     # Multisite config
└── phpunit.xml.dist      # PHPUnit config
```

### JavaScript Tests

Currently, the project doesn't have JavaScript unit tests, but you can add them using `@wordpress/scripts`:

```bash
# Add test script to package.json
"test-js": "wp-scripts test-unit-js"

# Run tests
npm run test-js
```

## Building Assets

### Build Commands

```bash
# Development build (unminified, with source maps)
npm run build

# Production build
NODE_ENV=production npm run build

# Watch mode (rebuild on changes)
npm run build -- --watch
```

### Build Output

Assets are compiled to `build/`:

```
build/
├── index.js          # Main JavaScript bundle
├── index.asset.php   # Asset metadata (dependencies, version)
└── index.js.map      # Source map
```

### Asset Registration

The built assets are automatically registered by the plugin:

```php
// In AI_Client::register_client_side_api_script()
wp_register_script(
    'wp-ai-client',
    plugins_url( 'build/index.js', dirname( __FILE__ ) ),
    $asset_metadata['dependencies'],
    $asset_metadata['version'],
    true
);
```

## Documentation

### PHPDoc Standards

All code must be fully documented following WordPress PHPDoc standards.

#### Classes

```php
/**
 * Class WordPress\AI_Client\Example
 *
 * @since n.e.x.t
 * @package WordPress\AI_Client
 */
```

#### Methods

```php
/**
 * Performs an action and returns a result.
 *
 * Description of what the method does, if needed.
 *
 * @since n.e.x.t
 *
 * @param string $param The parameter description.
 * @return bool True on success, false on failure.
 */
public function do_something( string $param ): bool {
    // Implementation
}
```

#### Interface Implementations

Use `{@inheritDoc}` when implementing interface methods:

```php
/**
 * {@inheritDoc}
 */
public function interface_method( string $param ): void {
    // Implementation
}
```

### User Documentation

Developer documentation is in `docs/`:

- `README.md`: Documentation index
- `getting-started.md`: Installation and setup
- `php-api.md`: PHP API reference
- `javascript-api.md`: JavaScript API reference
- `advanced-usage.md`: Advanced patterns
- `architecture.md`: Technical architecture
- `troubleshooting.md`: Common issues
- `development.md`: This file

When adding new features, update relevant documentation.

## Pull Request Process

### Before Submitting

1. **Create a feature branch**:

```bash
git checkout -b feature/your-feature-name
```

2. **Make your changes**:
   - Follow code standards
   - Add tests for new functionality
   - Update documentation

3. **Run all checks**:

```bash
# Lint PHP
composer lint

# Lint JavaScript
npm run lint-js

# Run tests
npm run test-php
```

4. **Commit your changes**:

```bash
git add .
git commit -m "Add feature: description"
```

### Commit Message Format

Follow conventional commits:

```
type(scope): subject

body (optional)

footer (optional)
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Maintenance tasks

**Examples**:

```
feat(builder): add support for streaming responses

Implements streaming response capability using server-sent events.

Closes #123
```

```
fix(credentials): resolve API key validation issue

API keys with special characters were not being properly escaped.
```

### Submitting PR

1. **Push to GitHub**:

```bash
git push origin feature/your-feature-name
```

2. **Create Pull Request**:
   - Go to GitHub repository
   - Click "New Pull Request"
   - Select your branch
   - Fill out the PR template

3. **PR Description**:
   - Describe what changes were made
   - Explain why they were necessary
   - Reference any related issues
   - Include testing instructions

4. **Address Review Feedback**:
   - Respond to reviewer comments
   - Make requested changes
   - Push updates to the same branch

### PR Checklist

- [ ] Code follows project standards
- [ ] Tests added/updated for new functionality
- [ ] Documentation updated
- [ ] All tests passing
- [ ] All linters passing
- [ ] No merge conflicts
- [ ] PR description is complete

## Release Process

### Versioning

The project follows [Semantic Versioning](https://semver.org/):

- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes (backward compatible)

### Release Steps

1. **Update version numbers**:
   - `plugin.php`: Plugin header
   - `composer.json`: Version field
   - `package.json`: Version field

2. **Update changelog**:
   - Document all changes in `CHANGELOG.md`
   - Organize by type (Added, Changed, Fixed, etc.)

3. **Update `@since` tags**:
   - Replace `n.e.x.t` with actual version

4. **Create release branch**:

```bash
git checkout -b release/1.0.0
```

5. **Tag the release**:

```bash
git tag -a 1.0.0 -m "Release 1.0.0"
git push origin 1.0.0
```

6. **Create GitHub release**:
   - Go to Releases on GitHub
   - Create new release from tag
   - Add release notes from changelog

### Post-Release

1. **Merge back to trunk**:

```bash
git checkout trunk
git merge release/1.0.0
git push origin trunk
```

2. **Update development version**:
   - Bump version to next development version
   - Update `@since` tags to `n.e.x.t`

## Development Tips

### Debugging

Enable debug mode:

```php
// In wp-config.php
define( 'WP_DEBUG', true );
define( 'WP_DEBUG_LOG', true );
define( 'WP_DEBUG_DISPLAY', false );
```

### IDE Setup

#### PHPStorm

1. Install WordPress plugin
2. Configure PHPCodeSniffer:
   - Settings > PHP > Quality Tools > PHP_CodeSniffer
   - Set path to `vendor/bin/phpcs`
3. Configure PHPStan:
   - Settings > PHP > Quality Tools > PHPStan
   - Set path to `vendor/bin/phpstan`

#### VS Code

Install extensions:
- PHP Intelephense
- WordPress Snippets
- ESLint
- Prettier

`.vscode/settings.json`:

```json
{
    "phpcs.enable": true,
    "phpcs.standard": "WordPress",
    "editor.formatOnSave": true,
    "eslint.validate": [ "javascript" ]
}
```

### Useful Commands

```bash
# Start development environment
npm run wp-env start

# Run all quality checks
composer lint && npm run lint-js && npm run test-php

# Watch and rebuild assets
npm run build -- --watch

# Access WordPress CLI
npm run wp-env run cli wp --info

# View logs
npm run wp-env logs

# Reset environment (clean slate)
npm run wp-env clean all
npm run wp-env start
```

## Resources

- [WordPress Coding Standards](https://developer.wordpress.org/coding-standards/)
- [PHPDoc Standards](https://developer.wordpress.org/coding-standards/inline-documentation-standards/php/)
- [JavaScript Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/javascript/)
- [wp-env Documentation](https://developer.wordpress.org/block-editor/reference-guides/packages/packages-env/)
- [PHP AI Client](https://github.com/WordPress/php-ai-client)

## Getting Help

- **Issues**: [GitHub Issues](https://github.com/WordPress/wp-ai-client/issues)
- **Discussions**: [WordPress AI Team](https://make.wordpress.org/ai/)
- **Contributing Guidelines**: See `CONTRIBUTING.md`

## Code of Conduct

All contributors must follow the [WordPress Code of Conduct](https://make.wordpress.org/handbook/community-code-of-conduct/).

## License

WordPress AI Client is licensed under [GPL-2.0-or-later](../LICENSE.md).

By contributing, you agree to license your contributions under the same license.
