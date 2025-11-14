# CLAUDE.md - Memegen Codebase Guide

This document provides a comprehensive guide to the Memegen codebase for AI assistants working with this repository.

## Project Overview

**Memegen** is an API to programmatically generate memes based solely on requested URLs. It's a stateless service that encodes all information needed to generate meme images directly in the URL structure.

- **Live API**: https://api.memegen.link
- **Version**: 11.1
- **License**: MIT
- **Language**: Python 3.13
- **Framework**: Sanic (async web framework)
- **Deployment**: Heroku with CDN caching

## Repository Structure

```
memegen/
├── app/                    # Main application code
│   ├── main.py            # Application entry point, Sanic app initialization
│   ├── config.py          # App configuration, blueprints, error handlers
│   ├── settings.py        # Environment settings and constants
│   ├── helpers.py         # Helper functions
│   ├── types.py           # Type definitions
│   ├── models/            # Data models
│   │   ├── template.py    # Template model (uses datafiles for YAML config)
│   │   ├── text.py        # Text positioning/styling model
│   │   ├── font.py        # Font handling
│   │   └── overlay.py     # Overlay positioning model
│   ├── views/             # API route handlers (blueprints)
│   │   ├── images.py      # Image generation endpoints
│   │   ├── templates.py   # Template listing endpoints
│   │   ├── fonts.py       # Font listing endpoints
│   │   ├── examples.py    # Example gallery
│   │   ├── clients.py     # Client helpers
│   │   ├── shortcuts.py   # URL shortcuts
│   │   ├── helpers.py     # View helper functions
│   │   └── schemas.py     # API schemas
│   ├── utils/             # Utility modules
│   │   ├── images.py      # Image processing (Pillow, pilmoji)
│   │   ├── text.py        # Text manipulation
│   │   ├── html.py        # HTML generation
│   │   ├── http.py        # HTTP utilities
│   │   ├── meta.py        # Metadata/version handling
│   │   └── urls.py        # URL manipulation
│   ├── static/            # Static files (favicon, robots.txt)
│   └── tests/             # Test suite
│       ├── test_*.py      # Test files
│       └── images/        # Generated test images for comparison
├── templates/             # Meme template definitions (200+ templates)
│   ├── <template_id>/    # Each template has its own directory
│   │   ├── default.png   # Default background image
│   │   ├── config.yml    # Template configuration (text positions, etc.)
│   │   └── *.png         # Alternative styles/backgrounds
├── fonts/                 # Custom font files
├── docs/                  # MkDocs documentation
├── scripts/               # Utility scripts
│   ├── check_deployment.py
│   └── simulate_load.py
├── bin/                   # Binary/executable scripts
├── .circleci/            # CircleCI CI/CD configuration
├── .github/              # GitHub configuration (dependabot, funding)
├── Makefile              # Development workflow automation
├── pyproject.toml        # Poetry dependencies and tool configuration
├── poetry.lock           # Locked dependency versions
├── Containerfile         # Docker container definition
├── Procfile              # Heroku production processes
├── Procfile.dev          # Local development processes
└── mkdocs.yml            # Documentation site configuration
```

## Key Technologies

### Core Stack
- **Python 3.13**: Modern Python with type hints
- **Sanic 25.3**: High-performance async web framework
- **sanic-ext**: OpenAPI/Swagger documentation
- **Pillow 12.0**: Image manipulation
- **pilmoji**: Emoji rendering on images
- **datafiles**: YAML-based configuration management

### Image Processing
- **Pillow**: Core image manipulation
- **pilmoji**: Emoji support in text
- **spongemock**: SpongeBob mocking text transformation
- **webp**: WebP format support

### Development Tools
- **Poetry**: Dependency management
- **pytest**: Testing framework with plugins (pytest-describe, pytest-asyncio, pytest-cov)
- **mypy**: Type checking
- **black**: Code formatting
- **isort**: Import sorting
- **autoflake**: Remove unused imports
- **coveragespace**: Coverage tracking
- **locust**: Load testing

### Production
- **gunicorn**: WSGI server
- **uvicorn**: ASGI server (worker class)
- **bugsnag**: Error tracking
- **aiocache**: Async caching
- **aiohttp**: Async HTTP client

## Development Workflow

### Initial Setup

```bash
# Install system dependencies
make bootstrap

# Verify system dependencies
make doctor

# Install project dependencies
make install
```

### Local Development

```bash
# Start both API server (port 5000) and docs site (port 5001)
make run

# Run with auto-reload on file changes
make dev
```

### Key URLs in Development
- **API Server**: http://localhost:5000/
- **API Docs**: http://localhost:5000/docs/
- **Example Gallery**: http://localhost:5000/examples
- **Test Images (auto-reload)**: http://localhost:5000/test
- **Documentation Site**: http://localhost:5001/

### Testing

```bash
# Run all tests with coverage
make test

# Run fast tests only (skip slow tests)
make test-fast

# Run slow tests only
make test-slow
```

**Test Strategy:**
- Tests are in `app/tests/test_*.py`
- Uses pytest with `--random` flag for test isolation
- Coverage reports in `.cache/htmlcov/`
- Failed tests are re-run first on next test run
- Marks: `@pytest.mark.slow` for slow tests

### Code Quality

```bash
# Format code (autoflake, isort, black)
make format

# Type checking (mypy)
make check

# Run all validation (format, check, test)
make all
```

### Continuous Development

```bash
# Watch mode: runs tests + checks on every file change
make dev
```

## Adding a New Meme Template

Follow this process to add a new template:

1. **Create template directory**: `templates/<template_id>/`
2. **Add background image**: `templates/<template_id>/default.png` (or .jpg)
3. **Create config file**: `templates/<template_id>/config.yml`
4. **Visit test URL**: http://localhost:5000/<template_id>
5. **View generated example**: http://localhost:5000/images/<template_id>
6. **Adjust config.yml** as needed to position/style text
7. **Validate**: http://localhost:5000/templates

### Template Config Structure

```yaml
name: Template Name
source: https://knowyourmeme.com/...
keywords:
  - keyword1
  - keyword2
text:
  - style: upper          # Text transformation (upper, lower, title, etc.)
    color: white         # Text color
    font: thick          # Font ID
    anchor_x: 0.0        # X position (0.0-1.0, relative to image width)
    anchor_y: 0.0        # Y position (0.0-1.0, relative to image height)
    angle: 0.0           # Text rotation
    scale_x: 1.0         # Horizontal scale
    scale_y: 0.2         # Vertical scale (text box height)
    align: center        # Text alignment
    start: 0.0           # Animation start frame (0.0-1.0)
    stop: 1.0            # Animation end frame (0.0-1.0)
example:
  - "Top text"
  - "Bottom text"
overlay:
  - center_x: 0.5       # X position of overlay
    center_y: 0.5       # Y position of overlay
    angle: 0.0          # Rotation
    scale: 0.25         # Size relative to background
```

## URL Structure and Special Characters

### Basic URL Pattern
```
https://api.memegen.link/images/<template_id>/<line1>/<line2>.<format>
```

### Special Character Encoding
- `_` (underscore) → space (` `)
- `-` (dash) → space (` `)
- `__` (double underscore) → underscore (`_`)
- `--` (double dash) → dash (`-`)
- `~n` → newline character
- `~q` → question mark (`?`)
- `~a` → ampersand (`&`)
- `~p` → percentage (`%`)
- `~h` → hashtag (`#`)
- `~s` → slash (`/`)
- `~b` → backslash (`\`)
- `~l` → less-than (`<`)
- `~g` → greater-than (`>`)
- `''` (two single quotes) → double quote (`"`)

### Query Parameters
- `width=<int>`: Scale to specific width
- `height=<int>`: Scale to specific height
- `style=<str>`: Alternate style or custom overlay URL
- `layout=<str>`: Layout mode (default or `top`)
- `font=<str>`: Font selection
- `background=<url>`: Custom background image
- `center=<float>,<float>`: Overlay position
- `scale=<float>`: Overlay scale

### Supported Formats
- `.png`: High quality static images
- `.jpg`/`.jpeg`: Smaller file size static images
- `.gif`: Animated (if background supports it)
- `.webp`: Animated or static with good compression

## Architecture Patterns

### Async-First Design
- All route handlers are async functions
- Use `await asyncio.to_thread()` for CPU-bound operations
- Image processing offloaded to thread pool

### Stateless API
- All parameters encoded in URL
- No server-side sessions
- Enables aggressive CDN caching

### Configuration as Code
- Templates use YAML configuration files
- `datafiles` library provides ORM-like access to YAML
- Changes to YAML automatically propagate

### Error Handling
- Custom `BugsnagErrorHandler` for production error tracking
- Ignored exceptions: `ClientPayloadError`, `MethodNotSupported`, `NotFound`, `UnidentifiedImageError`
- All errors logged and reported to Bugsnag in production

## Environment Variables

### Required in Production
- `DOMAIN`: Server domain (e.g., "api.memegen.link")
- `HEROKU_APP_NAME`: Auto-set on Heroku

### Optional
- `DEBUG`: Enable debug mode ("true"/"false")
- `DEFAULT_STATIC_EXTENSION`: Default format for static images (default: "png")
- `DEFAULT_ANIMATED_EXTENSION`: Default format for animated images (default: "gif")
- `REMOTE_TRACKING_URL`: Analytics tracking endpoint
- `REMOTE_TRACKING_ERRORS_LIMIT`: Max tracking errors before disabling (default: 10)
- `BUGSNAG_API_KEY`: Error reporting API key
- `WEB_CONCURRENCY`: Number of worker processes (Heroku)
- `MAX_REQUESTS`: Max requests before worker restart (Heroku)
- `MAX_REQUESTS_JITTER`: Jitter for max requests (Heroku)

## Settings Constants (app/settings.py)

### Image Settings
- `IMAGES_DIRECTORY`: Where generated images are cached
- `DEFAULT_SIZE`: (600, 600) pixels
- `PREVIEW_SIZE`: (300, 300) pixels
- `MAXIMUM_PIXELS`: 1920 × 1080 (prevents DoS)
- `MAXIMUM_FRAMES`: 20 frames for animations
- `MINIMUM_FRAMES`: 5 frames for animations
- `ALLOWED_EXTENSIONS`: {gif, jpg, jpeg, png, webp}
- `ANIMATED_EXTENSIONS`: {gif, webp}

### Font Settings
- `DEFAULT_FONT`: "thick" (Titillium Web Black)
- `MINIMUM_FONT_SIZE`: 7 pixels

### Watermark Settings
- `DEFAULT_WATERMARK`: "Memegen.link"
- `WATERMARK_HEIGHT`: 20 pixels
- `WATERMARK_ALPHA`: 0.65
- `DISABLED_WATERMARK`: "none"

## API Endpoints

### Main Routes (defined in app/config.py)
1. **examples** (`app.views.examples`): Example gallery
2. **clients** (`app.views.clients`): Client helper tools
3. **fonts** (`app.views.fonts`): Available fonts listing
4. **images** (`app.views.images`): Image generation (core functionality)
5. **templates** (`app.views.templates`): Template listing and details
6. **shortcuts** (`app.views.shortcuts`): Deprecated URL shortcuts

### Key Endpoints
- `GET /`: Redirects to `/docs`
- `GET /docs`: OpenAPI/Swagger documentation
- `GET /templates`: List all templates
- `GET /templates/<id>`: Template details
- `GET /fonts`: List all fonts
- `GET /images/<template_id>/<text>.<format>`: Generate meme image
- `POST /images`: Generate image from JSON payload
- `GET /examples`: Gallery of example memes
- `GET /test`: Development-only test images with auto-reload

## Testing Conventions

### Test Organization
- One test file per module: `test_<module>.py`
- Use `pytest-describe` for BDD-style test grouping
- Use `pytest-expecter` for readable assertions
- Use `sanic-testing` for async route testing

### Test Fixtures
- Test images stored in `app/tests/images/`
- Organized by template ID
- Used for visual regression testing

### Running Specific Tests
```bash
# Run a specific test file
poetry run pytest app/tests/test_views_images.py

# Run a specific test function
poetry run pytest app/tests/test_views_images.py::test_function_name

# Run with verbose output
poetry run pytest -v

# Run with print statements shown
poetry run pytest -s
```

## Common Development Tasks

### Update Dependencies
```bash
# Update all dependencies
poetry update

# Update specific package
poetry update <package-name>

# Add new dependency
poetry add <package-name>

# Add dev dependency
poetry add --group dev <package-name>
```

### Image Optimization
```bash
# Compress all template images
make compress
```

### Clean Build Artifacts
```bash
# Clean temporary files
make clean-tmp

# Clean all generated files
make clean

# Clean everything including test images
make clean-all
```

### Deployment (Maintainers Only)
```bash
# Deploy to staging
make deploy

# Promote staging to production
make promote
```

## CI/CD Pipeline

### CircleCI Jobs

1. **test**: Run tests, type checking, and coverage
   - Python 3.13 Docker image
   - Cache Poetry dependencies
   - Upload coverage to Coveralls
   - Build MkDocs site

2. **build**: Build and test Docker container
   - Build multi-architecture container
   - Start container and validate API responds
   - Cache Docker image

3. **deploy**: Push to container registry (on tags only)
   - Only runs for version tags (v*)
   - Requires Docker credentials

### Workflow
- All jobs run on every push
- Deploy only runs on version tags matching `v*`
- Tests must pass before deploy

## Code Style and Conventions

### Python Style
- **Formatter**: black (line length: 88)
- **Import sorting**: isort (black-compatible profile)
- **Type hints**: Required for all functions
- **Async/await**: Prefer async functions for I/O
- **Docstrings**: Not strictly enforced, but helpful for complex logic

### Type Checking
- **Tool**: mypy
- **Settings**: `ignore_missing_imports = true`
- **Plugins**: datafiles mypy plugin enabled
- **Mode**: `check_untyped_defs = true`

### File Organization
- One class per model file (models/)
- Group related routes in blueprint files (views/)
- Utility functions in appropriate utils/ module
- Tests mirror source structure

### Naming Conventions
- **Files**: lowercase with underscores (`image_utils.py`)
- **Classes**: PascalCase (`Template`, `Text`)
- **Functions**: snake_case (`generate_image`, `get_template`)
- **Constants**: UPPER_SNAKE_CASE (`DEFAULT_FONT`, `MAXIMUM_PIXELS`)
- **Private**: Leading underscore (`_internal_helper`)

## Performance Considerations

### Caching Strategy
- Generated images cached in `images/` directory
- CDN caching in production (Cloudflare)
- `aiocache` for in-memory caching

### Image Generation
- CPU-intensive operations run in thread pool
- Avoid blocking the async event loop
- Pillow operations are not async-native

### Resource Limits
- Maximum image size: 1920×1080 pixels
- Maximum animation frames: 20
- Prevents resource exhaustion attacks

## Debugging

### Enable Debug Mode
```bash
# Set environment variable
export DEBUG=true

# Or in .env file
DEBUG=true
```

### Debug Features
- Auto-reload on code changes
- Detailed error pages
- ASYNCIO debug mode enabled
- Template example auto-update

### Debugging with IPDB
```python
# Breakpoint in code (PYTHONBREAKPOINT=ipdb.set_trace in Makefile)
breakpoint()
```

### View Logs
```bash
# Local development logs to console
make run

# Production logs
heroku logs --tail --app memegen-production
```

## Common Pitfalls

1. **Forgetting to handle async**: All route handlers must be async, use `await` for async operations

2. **Not using thread pool for CPU work**: Image processing blocks the event loop if not offloaded

3. **Template config syntax**: YAML is strict about indentation and types

4. **Special characters in URLs**: Remember to use the encoding patterns (~q, ~h, etc.)

5. **Image size limits**: Large images will be rejected to prevent DoS

6. **Cache invalidation**: Changes to templates require clearing cached images

## Useful Commands Reference

```bash
# Development
make install          # Install dependencies
make run             # Start dev server
make dev             # Watch mode with auto-reload
make test            # Run tests
make check           # Type checking
make format          # Format code
make all             # Format + check + test

# Maintenance
make clean           # Clean build artifacts
make clean-tmp       # Clean temporary files
make compress        # Optimize template images
make doctor          # Check system dependencies

# Deployment
make run-production  # Test production config locally
make deploy          # Deploy to staging
make promote         # Promote staging to production
```

## Resources

- **API Documentation**: https://api.memegen.link/docs/
- **Main Site**: https://memegen.link
- **Source Code**: https://github.com/jacebrowning/memegen
- **Issue Tracker**: https://github.com/jacebrowning/memegen/issues
- **Know Your Meme**: Source for meme template information

## Getting Help

- Read `CONTRIBUTING.md` for contribution guidelines
- Check existing issues on GitHub
- Review OpenAPI documentation at `/docs`
- Test locally with `/test` endpoint

## Version Information

- **Current Version**: 11.1
- **Python**: 3.13+
- **Sanic**: 25.3
- **Pillow**: 12.0

---

**Last Updated**: 2025-11-14

This document should be updated whenever significant architectural changes, new patterns, or major features are added to the codebase.
