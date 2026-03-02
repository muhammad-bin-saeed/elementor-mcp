# Contributing to Elementor MCP

Thank you for your interest in contributing to Elementor MCP! This project bridges AI agents and Elementor page design through the Model Context Protocol, and community contributions are essential to making it better.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Development Setup](#development-setup)
- [Architecture Overview](#architecture-overview)
- [Adding a New MCP Tool](#adding-a-new-mcp-tool)
- [Coding Standards](#coding-standards)
- [Testing](#testing)
- [Submitting Changes](#submitting-changes)
- [Reporting Bugs](#reporting-bugs)
- [Suggesting Features](#suggesting-features)

## Code of Conduct

This project follows the [WordPress Community Code of Conduct](https://make.wordpress.org/handbook/community-code-of-conduct/). Please be respectful and constructive in all interactions.

## How Can I Contribute?

There are many ways to contribute, regardless of your experience level:

- **Report bugs** — Found something broken? [Open an issue](https://github.com/msrbuilds/elementor-mcp/issues/new).
- **Suggest features** — Have an idea for a new MCP tool? We'd love to hear it.
- **Improve documentation** — Fix typos, clarify instructions, or add examples.
- **Add new widget tools** — Create convenience shortcuts for Elementor widgets not yet covered.
- **Write tests** — Help improve test coverage with PHPUnit tests.
- **Fix bugs** — Browse [open issues](https://github.com/msrbuilds/elementor-mcp/issues) and submit a fix.
- **Share your experience** — Write about how you use the plugin, share prompts that work well.

## Development Setup

### Prerequisites

- PHP 7.4 or later
- WordPress 6.8+ (local development environment)
- Elementor 3.20+ (free or Pro)
- [WordPress MCP Adapter](https://github.com/WordPress/mcp-adapter) plugin
- Composer (for dev dependencies)
- WP-CLI (recommended for testing MCP tools)

### Installation

1. Clone the repository into your WordPress plugins directory:

   ```bash
   cd /path/to/wordpress/wp-content/plugins
   git clone https://github.com/msrbuilds/elementor-mcp.git
   cd elementor-mcp
   ```

2. Install development dependencies:

   ```bash
   composer install
   ```

3. Activate the plugin and its dependencies in WordPress.

4. Verify the MCP server is registered:

   ```bash
   wp mcp-adapter list --path=/path/to/wordpress
   ```

### Testing MCP Tools

Use the MCP Inspector to test tools interactively:

```bash
npx @modelcontextprotocol/inspector wp mcp-adapter serve \
  --server=elementor-mcp-server --user=admin --path=/path/to/wordpress
```

## Architecture Overview

Understanding the plugin's architecture will help you contribute effectively.

```
elementor-mcp/
├── elementor-mcp.php              # Bootstrap: constants, dependency checks, require_once
├── includes/
│   ├── class-plugin.php           # Singleton orchestrator (hooks registration)
│   ├── class-elementor-data.php   # Data access layer (read/write Elementor documents)
│   ├── class-element-factory.php  # Builds valid Elementor JSON structures
│   ├── class-id-generator.php     # 7-char hex unique IDs
│   ├── class-openverse-client.php # Openverse API HTTP client
│   ├── abilities/                 # MCP tools grouped by domain
│   │   ├── class-ability-registrar.php    # Coordinates all ability groups
│   │   ├── class-query-abilities.php      # Read-only discovery tools
│   │   ├── class-page-abilities.php       # Page CRUD
│   │   ├── class-layout-abilities.php     # Container/layout tools
│   │   ├── class-widget-abilities.php     # Widget add/update tools
│   │   ├── class-template-abilities.php   # Template tools
│   │   ├── class-global-abilities.php     # Global settings tools
│   │   ├── class-composite-abilities.php  # build-page composite tool
│   │   ├── class-stock-image-abilities.php # Stock image tools
│   │   ├── class-svg-icon-abilities.php   # SVG icon upload tool
│   │   └── class-custom-code-abilities.php # CSS/JS/snippet tools
│   ├── schemas/                   # JSON Schema generation
│   │   ├── class-schema-generator.php
│   │   └── class-control-mapper.php
│   └── validators/                # Input validation
│       ├── class-element-validator.php
│       └── class-settings-validator.php
```

### Key Concepts

- **Abilities** are MCP tools registered via the WordPress Abilities API (`wp_register_ability()`).
- **Data Layer** wraps Elementor's document system — always use `$this->data` methods, never update `_elementor_data` meta directly.
- **Element Factory** creates valid Elementor JSON element structures with proper IDs.
- **Schema Generator** auto-generates JSON Schema from Elementor's widget controls, so AI agents know what settings each widget accepts.

## Adding a New MCP Tool

This is the most common type of contribution. Here's how to add a new tool:

### 1. Choose the right ability class

Tools are grouped by domain. Add your tool to the appropriate existing class, or create a new one if it doesn't fit.

### 2. Register the ability

In your ability class's `register()` method, call `wp_register_ability()`:

```php
wp_register_ability(
    'elementor-mcp/my-new-tool',
    array(
        'title'               => __( 'My New Tool', 'elementor-mcp' ),
        'description'         => __( 'What this tool does.', 'elementor-mcp' ),
        'category'            => 'elementor-mcp',
        'input_schema'        => array(
            'type'       => 'object',
            'properties' => array(
                'post_id' => array(
                    'type'        => 'integer',
                    'description' => __( 'The page/post ID.', 'elementor-mcp' ),
                ),
            ),
            'required' => array( 'post_id' ),
        ),
        'permission_callback' => array( $this, 'check_edit_permission' ),
        'callback'            => array( $this, 'handle_my_new_tool' ),
    )
);
```

### 3. Implement the handler

```php
public function handle_my_new_tool( array $input ) {
    $post_id = absint( $input['post_id'] );

    // Your logic here using $this->data and $this->factory

    return array(
        'success' => true,
        'message' => 'Tool completed successfully.',
    );
}
```

### 4. Register in the registrar

If you created a new ability class, add it to `class-ability-registrar.php`:

```php
$my_class = new Elementor_MCP_My_Abilities( $this->data, $this->factory );
$my_class->register();
$this->ability_names = array_merge( $this->ability_names, $my_class->get_ability_names() );
```

And add the `require_once` in `elementor-mcp.php`.

### 5. Add to admin tools list

Add the tool entry to `get_all_tools()` in `includes/admin/class-admin.php` under the appropriate category.

## Coding Standards

This project follows WordPress coding standards strictly:

- **Naming**: `snake_case` for functions/variables, `Upper_Snake_Case` for classes, `UPPER_SNAKE` for constants.
- **Prefix everything**: All functions, classes, hooks, and options use the `elementor_mcp` or `Elementor_MCP` prefix.
- **Strings**: All user-facing strings must be translatable using `__()`, `esc_html__()`, etc. with the `elementor-mcp` text domain.
- **Security**: Sanitize all input (`sanitize_text_field`, `absint`), escape all output (`esc_html`, `esc_attr`, `esc_url`), check capabilities before privileged operations.
- **No direct meta updates**: Always use `$this->data->save_elementor_data()` which triggers Elementor CSS regeneration and cache busting.

### PHP Compatibility

- Target PHP 7.4+ (no union types, no named arguments, no enums).
- Use type hints for parameters and return types where supported.

## Testing

### Running Tests

```bash
composer install
vendor/bin/phpunit --configuration phpunit.xml.dist
```

### Writing Tests

- Place test files in the `tests/` directory.
- Follow PHPUnit naming conventions: `Test_My_Feature` class in `tests/test-my-feature.php`.
- Mock Elementor and WordPress functions as needed.

## Submitting Changes

### Pull Request Process

1. **Fork** the repository and create a feature branch from `main`:

   ```bash
   git checkout -b feature/my-new-tool
   ```

2. **Make your changes** following the coding standards above.

3. **Test locally** — ensure your tool works via MCP Inspector and doesn't break existing tools.

4. **Run PHP syntax check** on all modified files:

   ```bash
   php -l includes/abilities/class-my-abilities.php
   ```

5. **Commit** with a clear, descriptive message:

   ```bash
   git commit -m "Add my-new-tool ability for doing X"
   ```

6. **Push** and open a Pull Request against `main`.

### PR Guidelines

- Keep PRs focused — one feature or fix per PR.
- Include a clear description of what the tool does and why it's useful.
- Update the tool count in `README.md` and `CLAUDE.md` if adding new tools.
- Add a changelog entry in `readme.txt`.
- If adding Pro-only tools, make sure they conditionally register when Elementor Pro is active.

## Reporting Bugs

When reporting bugs, please include:

1. **Plugin version** and **Elementor version**.
2. **WordPress version** and **PHP version**.
3. **Steps to reproduce** the issue.
4. **Expected behavior** vs **actual behavior**.
5. **Error logs** if applicable (check `wp-content/debug.log`).
6. **MCP client** you're using (Claude Code, Claude Desktop, Cursor, etc.).

## Suggesting Features

Feature requests are welcome! When suggesting a new MCP tool:

1. **Describe the use case** — what are you trying to build with AI that this tool would enable?
2. **Specify the inputs/outputs** — what parameters should the tool accept and what should it return?
3. **Note if it requires Pro** — does the feature depend on Elementor Pro APIs?
4. **Consider the permission model** — what WordPress capability should be required?

---

Thank you for contributing to Elementor MCP! Every contribution, no matter how small, helps make AI-powered Elementor design better for everyone.
