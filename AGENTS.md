# FusionPBX Documentation - Agent Guidelines

## Build Commands
- `make html` - Build HTML documentation locally
- `make clean` - Clean build directory
- `make linkcheck` - Check all external links
- `sphinx-build -b html source build/html` - Direct sphinx build

## Documentation Format
- Files use both Markdown (.md) and reStructuredText (.rst) formats
- Markdown files use MyST parser with extensions (colon_fence, tasklist, etc.)
- Place images in `source/_static/images/`
- New pages must be added to appropriate index/toctree

## Style Guidelines
- Use descriptive headings with proper hierarchy (# > ## > ###)
- Keep line length reasonable for readability
- Use code blocks with appropriate language tags
- Include cross-references using MyST syntax: `{doc}path/to/doc`
- Prefer Markdown over RST for new content

## File Organization
- Main content in `source/` directory
- Group related docs in subdirectories (accounts/, applications/, etc.)
- Configuration in `source/conf.py`