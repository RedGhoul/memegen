# Memegen Architecture and Design Document

**Version:** 1.0
**Last Updated:** 2025-11-14
**Application Version:** 11.1

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture Overview](#system-architecture-overview)
3. [Component Design](#component-design)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Deployment Architecture](#deployment-architecture)
6. [Technology Stack](#technology-stack)
7. [Security Architecture](#security-architecture)
8. [Performance and Scaling](#performance-and-scaling)
9. [API Design](#api-design)
10. [Image Processing Pipeline](#image-processing-pipeline)
11. [Development Workflow](#development-workflow)
12. [Monitoring and Observability](#monitoring-and-observability)

---

## Executive Summary

**Memegen** is a stateless, high-performance API service for programmatic meme generation. The application follows a URL-based encoding pattern where all meme parameters are embedded directly in the URL, eliminating the need for server-side sessions or databases. This architectural decision enables aggressive CDN caching and horizontal scalability.

### Key Architectural Principles

- **Stateless Design**: No server-side sessions, databases, or persistent state
- **URL-Encoded Parameters**: All meme configuration embedded in URLs
- **Async-First**: Built on async Python with non-blocking I/O
- **CDN-Optimized**: Designed for edge caching with Cloudflare
- **Container-Ready**: Docker-based deployment pipeline
- **Template-Driven**: YAML configuration for extensibility

---

## System Architecture Overview

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         CDN Layer (Cloudflare)                   │
│                    • Static Asset Caching                        │
│                    • DDoS Protection                             │
│                    • TLS Termination                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Load Balancer (Heroku)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Application Layer (Sanic)                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Worker Process 1  │  Worker Process 2  │  Worker Process N │ │
│  │  (Gunicorn + Uvicorn Workers)                             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Blueprint Routes                        │  │
│  │  • Images   • Templates   • Fonts   • Examples            │  │
│  │  • Clients  • Shortcuts                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   Business Logic Layer                     │  │
│  │  • Template Models (datafiles ORM)                        │  │
│  │  • Text Processing                                        │  │
│  │  • Image Generation                                       │  │
│  │  • URL Encoding/Decoding                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                Image Processing Pipeline                   │  │
│  │  • Pillow (CPU-bound, offloaded to thread pool)           │  │
│  │  • Pilmoji (Emoji rendering)                              │  │
│  │  • Dynamic Text Sizing & Wrapping                         │  │
│  │  • Animation Support (GIF, WebP)                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Filesystem Layer                            │
│  • Template Definitions (YAML configs)                           │
│  • Background Images (PNG, JPG, GIF, WebP)                       │
│  • Generated Image Cache                                         │
│  • Font Files                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Architecture Patterns

#### 1. **Stateless Request Handling**
Every request is independent and self-contained. No server-side state means:
- Requests can be routed to any worker
- Easy horizontal scaling
- Simplified deployment and rollbacks
- CDN-friendly caching headers

#### 2. **Async/Await Architecture**
Built on Sanic (async web framework) with careful separation:
- **I/O-bound operations**: Async/await (HTTP requests, file I/O)
- **CPU-bound operations**: Thread pool (`asyncio.to_thread()`)
- Prevents blocking the event loop during image processing

#### 3. **Configuration as Code**
Templates are YAML files managed by the `datafiles` library:
- Git-versioned configuration
- No database migrations needed
- Easy template additions via pull requests
- ORM-like access to YAML data

---

## Component Design

### 1. Application Entry Point (`app/main.py`)

```python
from sanic import Sanic
app = Sanic(name="memegen")
config.init(app)

if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=5000,
        debug=settings.DEBUG,
        auto_reload=True,
        access_log=False,
    )
```

**Responsibilities:**
- Initialize Sanic application
- Apply configuration
- Register blueprints
- Start development or production server

### 2. Configuration Module (`app/config.py`)

**Key Functions:**
- Blueprint registration
- CORS configuration
- OpenAPI/Swagger setup
- Error handling configuration
- Bugsnag integration

**Registered Blueprints:**
1. `examples` - Example gallery
2. `clients` - Client helper tools
3. `fonts` - Font listing
4. `images` - Image generation (core)
5. `templates` - Template CRUD operations
6. `shortcuts` - Legacy URL redirects

### 3. Settings Module (`app/settings.py`)

**Environment Detection:**
```python
if "DOMAIN" in os.environ:
    RELEASE_STAGE = "production" or "staging"
elif "HEROKU_APP_NAME" in os.environ:
    RELEASE_STAGE = "review"
else:
    RELEASE_STAGE = "local"
```

**Key Settings:**
- Image constraints (max pixels: 1920×1080)
- Font defaults
- Watermark configuration
- Analytics tracking
- Error reporting

### 4. Template Model (`app/models/template.py`)

**Design Pattern:** Active Record with `datafiles` ORM

```python
@datafile("../../templates/{self.id}/config{self.variant}.yml", defaults=True)
class Template:
    id: str
    name: str
    source: str | None
    text: list[Text]
    overlay: list[Overlay]
    example: list[str]
```

**Key Features:**
- Lazy loading of template configurations
- Cached properties for performance
- Dynamic template creation from URLs
- Style variant support
- Animation detection

**Template Directory Structure:**
```
templates/<template_id>/
├── config.yml          # Template configuration
├── default.png         # Default background image
├── alternate.png       # Alternative style (optional)
└── default.gif         # Animated version (optional)
```

### 5. Views Layer (`app/views/`)

#### Images Blueprint (`images.py`)
**Core Endpoints:**
- `GET /images/` - List example memes
- `POST /images/` - Create meme from JSON
- `GET /images/<template_id>/<text>.<ext>` - Generate meme

**Processing Flow:**
```
1. Parse URL parameters
2. Validate template exists
3. Decode text from URL
4. Generate or retrieve cached image
5. Return image with caching headers
```

#### Templates Blueprint (`templates.py`)
- List all available templates
- Template detail with metadata
- Template search by keyword

#### Fonts Blueprint (`fonts.py`)
- List available fonts
- Font previews

### 6. Image Processing Utilities (`app/utils/images.py`)

**Core Functions:**
- `load()` - Load and validate images
- `save()` - Save with optimization
- `render_text()` - Text rendering with wrapping
- `embed()` - Overlay image composition
- `merge()` - Animated GIF frame merging
- `pad_top()` - Add padding for layout modes

**Image Processing Pipeline:**
```
1. Load background image (Pillow)
2. For each text line:
   a. Calculate optimal font size
   b. Wrap text if needed
   c. Render with stroke/outline
   d. Apply to image
3. Apply overlays (if any)
4. Add watermark
5. Optimize and save
```

### 7. URL Utilities (`app/utils/urls.py`)

**Special Character Encoding:**
- `_` → space
- `__` → underscore
- `--` → dash
- `~n` → newline
- `~q` → `?`
- `~a` → `&`
- `~p` → `%`
- `~h` → `#`
- `~s` → `/`

**Example:**
```
Input:  "Hello_World__2023"
Output: "Hello World_2023"
```

---

## Data Flow Architecture

### 1. Image Generation Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Client Request                                           │
│    GET /images/fry/not_sure/if_joking.png                   │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Route Handler (views/images.py)                          │
│    • Parse URL: template_id="fry", text=["not sure",        │
│      "if joking"], extension="png"                          │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Template Lookup (models/template.py)                     │
│    • Load templates/fry/config.yml                          │
│    • Get background: templates/fry/default.png              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Cache Check                                              │
│    • Generate fingerprint: slug + config hash               │
│    • Check: images/fry/<fingerprint>.png                    │
└───────────────────────┬─────────────────────────────────────┘
                        │
                 ┌──────┴──────┐
                 │             │
            Cache Hit      Cache Miss
                 │             │
                 │             ▼
                 │   ┌─────────────────────────────────────┐
                 │   │ 5. Image Generation (async thread)   │
                 │   │    • Load background image           │
                 │   │    • Calculate text layout           │
                 │   │    • Render text with Pillow         │
                 │   │    • Apply watermark                 │
                 │   │    • Save to cache                   │
                 │   └───────────┬─────────────────────────┘
                 │               │
                 └───────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Response                                                 │
│    • Set cache headers (CDN: 1 year)                        │
│    • Return image/png                                       │
└─────────────────────────────────────────────────────────────┘
```

### 2. Template Discovery Flow

```
Application Startup
        │
        ▼
datafiles library scans
  templates/ directory
        │
        ▼
For each subdirectory:
  • Read config.yml
  • Create Template object
  • Cache in memory
        │
        ▼
Template.objects.all()
  returns all templates
```

### 3. Custom Background Flow

```
POST /images
{
  "template_id": "custom",
  "text": ["Top", "Bottom"],
  "background": "https://example.com/image.jpg"
}
        │
        ▼
1. Download background image
        │
        ▼
2. Create temporary template
   with fingerprinted ID
        │
        ▼
3. Generate meme with
   downloaded background
        │
        ▼
4. Return URL with
   background parameter
```

---

## Deployment Architecture

### Production Environment (Heroku)

#### Infrastructure Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare CDN                            │
│  • Global edge network                                       │
│  • Cache TTL: 1 year for images                              │
│  • DDoS protection                                           │
│  • SSL/TLS termination                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                 Heroku Platform                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Dyno 1    Dyno 2    Dyno 3    ...    Dyno N         │  │
│  │  (Container instances with auto-scaling)              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
│  Each Dyno runs:                                             │
│  • Gunicorn (WSGI server)                                   │
│  • Uvicorn workers (ASGI)                                   │
│  • WEB_CONCURRENCY worker processes                         │
│  • Ephemeral filesystem (rebuilt on deploy)                 │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              External Services                               │
│  • Bugsnag (Error tracking)                                  │
│  • Coveralls (Code coverage)                                │
│  • GitHub (Source control)                                  │
│  • CircleCI (CI/CD)                                         │
└─────────────────────────────────────────────────────────────┘
```

#### Deployment Process

**Procfile Configuration:**
```
web: gunicorn app.main:app \
  --bind 0.0.0.0:${PORT:-5000} \
  --worker-class uvicorn.workers.UvicornWorker \
  --max-requests ${MAX_REQUESTS:-0} \
  --max-requests-jitter ${MAX_REQUESTS_JITTER:-0}

release: bin/purge
```

**Release Process:**
1. `release` phase: Clear CDN cache
2. `web` phase: Start new dynos
3. Old dynos gracefully terminate

**Environment Variables (Production):**
```bash
DOMAIN=api.memegen.link
BUGSNAG_API_KEY=<secret>
WEB_CONCURRENCY=4              # Worker processes
MAX_REQUESTS=1000              # Restart after N requests
MAX_REQUESTS_JITTER=50         # Random jitter for restarts
```

### Container Deployment (Docker)

**Containerfile Structure:**
```dockerfile
FROM python:3.13.9-bookworm

# Install system dependencies
RUN apt update && apt install --yes webp cmake
RUN pip install poetry

# Create non-root user
RUN useradd -md /opt/memegen -u 1000 memegen
USER memegen

# Copy application files
WORKDIR /opt/memegen
COPY templates /opt/memegen/templates
COPY fonts /opt/memegen/fonts
COPY app /opt/memegen/app
COPY pyproject.toml poetry.lock /opt/memegen/

# Install Python dependencies
RUN poetry install --only=main

# Start application
ENTRYPOINT ["poetry", "run", "gunicorn", \
  "--bind", "0.0.0.0:5000", \
  "--worker-class", "uvicorn.workers.UvicornWorker", \
  "app.main:app"]
```

**Container Deployment Options:**
1. **Docker Hub / Container Registry**
2. **Kubernetes** (with horizontal pod autoscaling)
3. **Cloud Run** (Google Cloud)
4. **ECS/Fargate** (AWS)
5. **Azure Container Instances**

### CI/CD Pipeline (CircleCI)

```
┌─────────────────────────────────────────────────────────────┐
│  Git Push to main/tag                                        │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Job 1: Test                                                 │
│  • Install dependencies                                      │
│  • Run mypy type checking                                    │
│  • Run pytest                                                │
│  • Upload coverage to Coveralls                              │
│  • Build MkDocs site                                         │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Job 2: Build                                                │
│  • Build Docker image                                        │
│  • Tag: latest + git tag (if applicable)                     │
│  • Start container                                           │
│  • Validate API responds                                     │
│  • Cache Docker image                                        │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Job 3: Deploy (tags only: v*)                               │
│  • Load Docker image from cache                              │
│  • Push to container registry                                │
│  • Trigger deployment                                        │
└─────────────────────────────────────────────────────────────┘
```

**Workflow Configuration:**
- **Trigger**: All jobs run on push
- **Deployment Gate**: Only on version tags (`v*`)
- **Caching**: Poetry dependencies, Docker layers
- **Artifacts**: Coverage reports, built images

---

## Technology Stack

### Core Framework
| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Language | Python | 3.13+ | Application runtime |
| Web Framework | Sanic | 25.3 | Async HTTP server |
| ASGI Server | Uvicorn | Latest | Production worker |
| WSGI Server | Gunicorn | Latest | Process manager |
| API Docs | sanic-ext | Latest | OpenAPI/Swagger |

### Image Processing
| Library | Purpose |
|---------|---------|
| Pillow 12.0 | Core image manipulation |
| pilmoji | Emoji rendering in images |
| spongemock | Mocking text transformation |

### Data & Configuration
| Library | Purpose |
|---------|---------|
| datafiles | YAML ORM for templates |
| PyYAML | YAML parsing |
| furl | URL manipulation |

### Development Tools
| Tool | Purpose |
|------|---------|
| Poetry | Dependency management |
| pytest | Testing framework |
| mypy | Type checking |
| black | Code formatting |
| isort | Import sorting |

### Deployment
| Service | Purpose |
|---------|---------|
| Heroku | Application hosting |
| Cloudflare | CDN & caching |
| Docker | Containerization |
| CircleCI | CI/CD pipeline |

### Monitoring & Analytics
| Service | Purpose |
|---------|---------|
| Bugsnag | Error tracking |
| Coveralls | Code coverage |

---

## Security Architecture

### 1. Input Validation

**URL Parameter Validation:**
- Template IDs: Alphanumeric + hyphens only
- Text input: Length limits enforced
- File extensions: Whitelist only (png, jpg, gif, webp)
- Image dimensions: Maximum 1920×1080 pixels
- Animation frames: Maximum 20 frames

**External URL Validation:**
```python
def validate_background_url(url: str) -> bool:
    # Check for valid schema
    if not url.startswith(('http://', 'https://')):
        return False

    # Prevent SSRF attacks
    parsed = furl(url)
    if parsed.host in BLOCKED_HOSTS:
        return False

    return True
```

### 2. Resource Limits

**Preventing DoS:**
```python
MAXIMUM_PIXELS = 1920 * 1080      # 2.07 million pixels
MAXIMUM_FRAMES = 20                # Animation limit
MINIMUM_FONT_SIZE = 7              # Prevent micro-text attacks
```

**Worker Process Limits:**
```
--max-requests 1000                # Restart workers after N requests
--max-requests-jitter 50           # Prevent thundering herd
--timeout 20                       # Request timeout
```

### 3. Error Handling

**Custom Error Handler:**
```python
class BugsnagErrorHandler(ErrorHandler):
    IGNORED_EXCEPTIONS = (
        ClientPayloadError,
        MethodNotSupported,
        NotFound,
        UnidentifiedImageError,
    )

    def default(self, request, exception):
        if should_notify(exception):
            bugsnag.notify(exception)
        return super().default(request, exception)
```

**Error Suppression:**
- 404s not reported (expected user errors)
- Invalid images silently fail
- Client payload errors ignored

### 4. CORS Configuration

```python
app.config.CORS_ORIGINS = "*"
app.config.CORS_SEND_WILDCARD = True
```

**Rationale:** Public API designed for cross-origin access

### 5. File System Security

**Non-Root Container User:**
```dockerfile
RUN useradd -md /opt/memegen -u 1000 memegen
USER memegen
```

**Path Traversal Prevention:**
```python
def safe_path(template_id: str) -> Path:
    # Prevent ../../../etc/passwd attacks
    return ROOT / "templates" / Path(template_id).name
```

### 6. Secrets Management

**Environment Variables Only:**
- `BUGSNAG_API_KEY` - Error tracking
- `REMOTE_TRACKING_URL` - Analytics endpoint
- `DOCKER_PASSWORD` - CI/CD deployment

**No Hardcoded Secrets:**
- All sensitive data via environment
- `.env` files gitignored
- Heroku Config Vars in production

---

## Performance and Scaling

### 1. Caching Strategy

#### Application-Level Caching
```python
# Generated image cache on filesystem
IMAGES_DIRECTORY = ROOT / "images"

# Cache key = fingerprint(text + template_config + style)
def build_path(...) -> Path:
    slug = encode(text_lines)
    identifier = str(text) + font + style + size + watermark
    fingerprint = hash(identifier)
    return Path(template_id) / f"{slug}.{fingerprint}.{extension}"
```

**Cache Behavior:**
- First request: Generate and save
- Subsequent requests: Serve from disk
- Cache invalidation: Purge on deployment

#### CDN Caching (Cloudflare)
```http
Cache-Control: public, max-age=31536000, immutable
```

**Cache TTL:** 1 year (365 days)

**Rationale:** URLs are immutable - changing parameters creates new URL

### 2. Async Architecture

**Event Loop Optimization:**
```python
# ✅ Correct: CPU-bound work in thread pool
image = await asyncio.to_thread(utils.images.render, template, text)

# ❌ Wrong: Blocks event loop
image = utils.images.render(template, text)
```

**Thread Pool:**
- Managed by asyncio
- CPU-bound operations (image processing)
- Does not block async I/O

### 3. Horizontal Scaling

**Stateless Design Enables:**
- Add more Heroku dynos without coordination
- No shared state between workers
- Load balancer can route to any instance

**Scaling Configuration:**
```bash
# Heroku
heroku ps:scale web=10

# Kubernetes
kubectl scale deployment memegen --replicas=10
```

### 4. Performance Metrics

**Benchmarks (Heroku Standard-2X dyno):**
- Simple text meme: ~200-400ms (uncached)
- Cached meme: ~10-50ms
- With CDN: ~10-20ms globally
- Animated GIF: ~1-3 seconds

**Load Testing (Locust):**
```bash
make load
# Simulates traffic to measure throughput
```

### 5. Optimization Techniques

**Image Optimization:**
- WebP support for smaller file sizes
- Progressive JPEGs
- GIF frame reduction
- On-the-fly resizing

**Code Optimization:**
- Cached properties (`@cached_property`)
- Lazy loading of templates
- Efficient text wrapping algorithms
- Font size binary search

---

## API Design

### REST Principles

**Resource-Oriented URLs:**
```
GET  /templates           # List templates
GET  /templates/{id}      # Template details
GET  /images              # Example memes
POST /images              # Create meme
GET  /images/{template}/{text}.{ext}  # Generate meme
GET  /fonts               # List fonts
```

### URL-Encoded Image Generation

**Pattern:**
```
/images/<template_id>/<text_line_1>/<text_line_2>/.../<text_line_n>.<extension>
```

**Examples:**
```
/images/fry/not_sure/if_serious.png
/images/ds/push_this/push_that/confused.jpg
/images/oprah/you_get_a_car/everyone_gets_a_car.gif
```

**Query Parameters:**
```
?width=800              # Scale to width
?height=600             # Scale to height
?style=alternate        # Alternative style
?background=<url>       # Custom background
?font=comic-sans        # Font selection
?watermark=none         # Disable watermark
```

### OpenAPI Documentation

**Swagger UI:** Available at `/docs`

**Schema Definition:**
```python
@openapi.summary("Create a meme from a template")
@openapi.body({"application/json": MemeRequest})
@openapi.response(201, {"application/json": MemeResponse})
async def create(request: Request):
    ...
```

**Request Schema:**
```json
{
  "template_id": "fry",
  "text": ["Not sure", "If serious"],
  "extension": "png",
  "style": "default",
  "font": "thick"
}
```

**Response Schema:**
```json
{
  "url": "https://api.memegen.link/images/fry/not_sure/if_serious.png"
}
```

---

## Image Processing Pipeline

### 1. Background Loading

```python
def load(path: Path) -> Image:
    """Load image from filesystem or download from URL"""
    img = PIL.Image.open(path)

    # Convert to RGB if needed
    if img.mode not in ('RGB', 'RGBA'):
        img = img.convert('RGBA')

    # Validate dimensions
    if img.width * img.height > MAXIMUM_PIXELS:
        raise ValueError("Image too large")

    return img
```

### 2. Text Rendering Algorithm

```python
def render_text(img: Image, text: str, position: Text) -> Image:
    """
    1. Calculate available space
    2. Binary search for optimal font size
    3. Wrap text if needed
    4. Render with stroke/outline
    5. Apply to image
    """

    # Calculate bounding box
    box = (
        position.anchor_x * img.width,
        position.anchor_y * img.height,
        position.scale_x * img.width,
        position.scale_y * img.height,
    )

    # Find optimal font size
    font_size = find_font_size(text, box, font_name)

    # Wrap text if needed
    lines = wrap_text(text, font_size, box.width)

    # Render with outline
    draw = PIL.ImageDraw.Draw(img)
    for line in lines:
        # Draw black stroke
        for offset in [(-2,-2), (-2,2), (2,-2), (2,2)]:
            draw.text(offset, line, fill='black', font=font)

        # Draw white text
        draw.text((0, 0), line, fill='white', font=font)

    return img
```

### 3. Font Size Calculation

**Binary Search Algorithm:**
```python
def find_font_size(text: str, box: Rect, font_name: str) -> int:
    """Binary search for largest font that fits"""
    low, high = MINIMUM_FONT_SIZE, 500

    while low < high:
        mid = (low + high + 1) // 2
        font = load_font(font_name, mid)

        if fits(text, font, box):
            low = mid
        else:
            high = mid - 1

    return low
```

### 4. Overlay Composition

```python
def embed(template: Template, index: int,
          foreground: Path, background: Path):
    """Composite overlay onto background"""

    bg = load(background)
    fg = load(foreground)

    # Get overlay position from config
    overlay = template.overlay[index]

    # Calculate position
    x = int(overlay.center_x * bg.width - fg.width / 2)
    y = int(overlay.center_y * bg.height - fg.height / 2)

    # Scale if needed
    if overlay.scale != 1.0:
        new_size = (
            int(fg.width * overlay.scale),
            int(fg.height * overlay.scale)
        )
        fg = fg.resize(new_size, PIL.Image.LANCZOS)

    # Rotate if needed
    if overlay.angle != 0.0:
        fg = fg.rotate(overlay.angle, expand=True)

    # Composite with alpha
    bg.paste(fg, (x, y), fg)

    save(bg, background)
```

### 5. Animation Support

**GIF Processing:**
```python
def render_animated(template: Template, text_lines: list[str]):
    """Render animated GIF with text"""

    # Load all frames
    frames = []
    with PIL.Image.open(background_path) as img:
        for frame_num in range(img.n_frames):
            img.seek(frame_num)
            frame = img.copy()

            # Render text for this frame
            for i, text in enumerate(text_lines):
                if should_show_text(template.text[i], frame_num, total):
                    frame = render_text(frame, text, template.text[i])

            frames.append(frame)

    # Save as animated GIF
    frames[0].save(
        output_path,
        save_all=True,
        append_images=frames[1:],
        loop=0,
        duration=img.info['duration']
    )
```

### 6. Watermark Application

```python
def add_watermark(img: Image, text: str) -> Image:
    """Add semi-transparent watermark to bottom-right"""

    if text == DISABLED_WATERMARK:
        return img

    # Create watermark image
    wm = PIL.Image.new('RGBA', (img.width, WATERMARK_HEIGHT))
    draw = PIL.ImageDraw.Draw(wm)

    # Render text
    font = load_font('default', 12)
    draw.text((img.width - 100, 5), text, fill=(255,255,255,165), font=font)

    # Composite onto main image
    img.paste(wm, (0, img.height - WATERMARK_HEIGHT), wm)

    return img
```

---

## Development Workflow

### Local Development Setup

```bash
# 1. Install system dependencies
make bootstrap

# 2. Verify system
make doctor

# 3. Install Python dependencies
make install

# 4. Start development server
make run
# API: http://localhost:5000
# Docs: http://localhost:5001
```

### Development Commands

```bash
# Code formatting
make format              # autoflake + isort + black

# Type checking
make check               # mypy

# Testing
make test                # pytest with coverage
make test-fast           # skip slow tests
make test-slow           # only slow tests

# Watch mode (continuous development)
make dev                 # auto-run tests on file changes

# Full validation
make all                 # format + check + test
```

### Testing Strategy

**Test Organization:**
```
app/tests/
├── test_main.py              # Application tests
├── test_models_template.py   # Model tests
├── test_views_images.py      # Route tests
├── test_utils_images.py      # Utility tests
└── images/                   # Reference images
    ├── fry/
    │   └── expected.png
    └── oprah/
        └── expected.gif
```

**Test Execution:**
```bash
# Run with random order for isolation
pytest --random

# Re-run failed tests first
pytest --failed-first

# Coverage report
pytest --cov=app --cov-report=html
```

**Test Marks:**
```python
@pytest.mark.slow
def test_heavy_image_processing():
    """Tests marked as slow are skipped in fast mode"""
    ...
```

### Continuous Integration

**Pre-commit Checks:**
```yaml
# .circleci/config.yml
- run: make doctor    # System dependencies
- run: make install   # Python dependencies
- run: make check     # Type checking
- run: make test      # Unit tests
- run: make site      # Documentation build
```

**Deployment Gates:**
- All tests must pass
- Type checking must pass
- Coverage threshold: (configured in pytest)

---

## Monitoring and Observability

### Error Tracking (Bugsnag)

**Configuration:**
```python
bugsnag.configure(
    api_key=BUGSNAG_API_KEY,
    project_root="/app",
    release_stage=RELEASE_STAGE,  # local/review/staging/production
)
```

**Error Metadata:**
```python
bugsnag.notify(exception, metadata={
    "request": {
        "method": request.method,
        "url": request.url,
        "json": request.json,
        "headers": request.headers,
    }
})
```

**Ignored Exceptions:**
- 404 Not Found
- Invalid image formats
- Client disconnections
- Method not allowed

### Logging

**Structured Logging:**
```python
from sanic.log import logger

logger.info(f"Generating meme: {template_id=} {text=}")
logger.warning(f"Style {style!r} not found for {template_id}")
logger.error(f"Failed to download background: {url}")
```

**Log Levels by Environment:**
- **Local:** DEBUG (all logs)
- **Review:** INFO (skip debug noise)
- **Staging:** INFO
- **Production:** WARNING (errors only)

### Performance Monitoring

**Request Timing:**
```python
@app.middleware('request')
async def add_start_time(request):
    request.ctx.start_time = time.time()

@app.middleware('response')
async def add_response_time(request, response):
    delta = time.time() - request.ctx.start_time
    logger.info(f"{request.path} completed in {delta:.3f}s")
```

**Load Testing:**
```bash
# Simulate traffic with Locust
make load

# Custom load test script
python scripts/simulate_load.py
```

### Health Checks

**Application Health:**
```python
@app.get("/images/preview.jpg")
async def health_check(request):
    """
    Simple health check endpoint
    Used by CircleCI and monitoring systems
    """
    return await render_image(...)
```

**Container Health:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/images/preview.jpg || exit 1
```

---

## Deployment Scenarios

### Scenario 1: Heroku (Current Production)

**Advantages:**
- Managed platform (no infrastructure management)
- Automatic SSL/TLS
- Easy scaling (`heroku ps:scale web=N`)
- Built-in logging and metrics
- Review apps for PRs

**Configuration:**
```bash
# Procfile
web: gunicorn app.main:app \
  --bind 0.0.0.0:${PORT} \
  --worker-class uvicorn.workers.UvicornWorker

# Heroku Config
heroku config:set DOMAIN=api.memegen.link
heroku config:set WEB_CONCURRENCY=4
```

**Deployment:**
```bash
git push heroku main
# or via CircleCI on tag push
```

### Scenario 2: Docker Compose (Development/Self-Hosted)

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  memegen:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DEBUG=false
      - DOMAIN=localhost:5000
    volumes:
      - ./images:/opt/memegen/images
    restart: unless-stopped
```

**Deployment:**
```bash
docker-compose up -d
docker-compose logs -f
```

### Scenario 3: Kubernetes

**Deployment Manifest:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memegen
spec:
  replicas: 3
  selector:
    matchLabels:
      app: memegen
  template:
    metadata:
      labels:
        app: memegen
    spec:
      containers:
      - name: memegen
        image: memegen:latest
        ports:
        - containerPort: 5000
        env:
        - name: DOMAIN
          value: "api.memegen.link"
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: memegen
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: memegen
```

**Horizontal Pod Autoscaling:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memegen-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: memegen
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Scenario 4: Serverless (AWS Lambda / Cloud Run)

**Considerations:**
- **Cold start latency**: Image processing libraries are heavy
- **Execution time limits**: Animation processing may timeout
- **Temporary storage**: Filesystem cache doesn't persist
- **Recommendation**: Use with Redis cache or S3 storage

**Cloud Run Example:**
```bash
# Build and push
gcloud builds submit --tag gcr.io/PROJECT/memegen

# Deploy
gcloud run deploy memegen \
  --image gcr.io/PROJECT/memegen \
  --platform managed \
  --memory 2Gi \
  --timeout 60s \
  --max-instances 100
```

---

## Appendix

### A. Configuration File Format

**Template Config (config.yml):**
```yaml
name: Futurama Fry
source: https://knowyourmeme.com/memes/futurama-fry-not-sure-if
keywords:
  - futurama
  - fry
  - not sure
text:
  - style: upper       # Text transformation
    color: white      # Text color
    font: thick       # Font ID
    anchor_x: 0.0     # X position (0-1)
    anchor_y: 0.0     # Y position (0-1)
    angle: 0.0        # Rotation in degrees
    scale_x: 1.0      # Width scale
    scale_y: 0.5      # Height scale
    align: center     # left/center/right
    start: 0.0        # Animation start (0-1)
    stop: 1.0         # Animation end (0-1)
  - style: upper
    color: white
    font: thick
    anchor_x: 0.0
    anchor_y: 0.5
    scale_y: 0.5
example:
  - "not sure if example"
  - "or actual meme"
overlay: []           # Optional overlay images
```

### B. URL Encoding Reference

| Input | Encoded | Output |
|-------|---------|--------|
| Space | `_` or `-` | ` ` |
| Underscore | `__` | `_` |
| Dash | `--` | `-` |
| Newline | `~n` | `\n` |
| Question mark | `~q` | `?` |
| Ampersand | `~a` | `&` |
| Percent | `~p` | `%` |
| Hash | `~h` | `#` |
| Slash | `~s` | `/` |
| Backslash | `~b` | `\` |
| Less than | `~l` | `<` |
| Greater than | `~g` | `>` |
| Quote | `''` | `"` |

### C. Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DEBUG` | No | `false` | Enable debug mode |
| `DOMAIN` | Production | `localhost:5000` | Server domain |
| `HEROKU_APP_NAME` | Heroku | - | Auto-set on Heroku |
| `BUGSNAG_API_KEY` | No | - | Error tracking |
| `WEB_CONCURRENCY` | No | `1` | Gunicorn workers |
| `MAX_REQUESTS` | No | `0` | Worker restart count |
| `MAX_REQUESTS_JITTER` | No | `0` | Restart jitter |
| `REMOTE_TRACKING_URL` | No | - | Analytics endpoint |

### D. Performance Tuning

**Heroku Dyno Sizing:**
| Dyno Type | Memory | CPU | Recommended Use |
|-----------|--------|-----|-----------------|
| Standard-1X | 512 MB | 1x | Development |
| Standard-2X | 1 GB | 2x | **Production** |
| Performance-M | 2.5 GB | 4x | High traffic |

**Worker Configuration:**
```bash
# Standard-2X dyno
WEB_CONCURRENCY=4           # 4 workers
MAX_REQUESTS=1000           # Restart after 1000 requests
MAX_REQUESTS_JITTER=50      # ±50 jitter

# Performance-M dyno
WEB_CONCURRENCY=8
MAX_REQUESTS=2000
MAX_REQUESTS_JITTER=100
```

### E. Troubleshooting Guide

**Problem: Images not generating**
- Check template exists: `ls templates/<id>/`
- Verify background image: `ls templates/<id>/default.*`
- Check logs for PIL errors

**Problem: Slow performance**
- Enable CDN caching headers
- Check worker count (`WEB_CONCURRENCY`)
- Profile with `make load`

**Problem: Memory errors**
- Reduce `MAXIMUM_PIXELS`
- Lower `MAXIMUM_FRAMES`
- Increase dyno size

**Problem: Deployment failures**
- Verify CircleCI tests pass
- Check Docker credentials
- Validate environment variables

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-14 | Claude | Initial architecture documentation |

---

## References

- [Sanic Documentation](https://sanic.dev/)
- [Pillow Documentation](https://pillow.readthedocs.io/)
- [Heroku Python Guide](https://devcenter.heroku.com/categories/python-support)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [API Design Best Practices](https://restfulapi.net/)

---

**End of Architecture Document**
