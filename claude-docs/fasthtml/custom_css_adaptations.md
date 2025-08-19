# FastHTML Adaptations for Custom CSS Libraries

This document outlines what to ignore from the official FastHTML documentation when using `cjm-fasthtml-tailwind` and `cjm-fasthtml-daisyui` instead of the built-in PicoCSS.

## What to IGNORE from FastHTML Documentation

### 1. PicoCSS References
- **IGNORE**: Any mention of PicoCSS being included by default
- **IGNORE**: `pico=False` parameter in `fast_app()` - not needed with `create_test_app()`
- **IGNORE**: Pico-specific components like `Container`, `Article`, `Card`, `Group` that rely on PicoCSS styling
- **INSTEAD**: Use DaisyUI components with explicit class builders

### 2. MonsterUI References
- **IGNORE**: All MonsterUI imports and components
- **IGNORE**: `Theme.blue.headers()` and other MonsterUI theme utilities
- **IGNORE**: MonsterUI-specific components like `LabelInput`, `DivLAligned`, `UkIconLink`
- **INSTEAD**: Use `cjm-fasthtml-daisyui` and `cjm-fasthtml-tailwind` utilities

### 3. CSS Framework Assumptions
- **IGNORE**: "Semantic HTML gets styled automatically" - this only applies to PicoCSS
- **IGNORE**: References to automatic styling of `Container`, `Article`, etc.
- **INSTEAD**: Always explicitly apply CSS classes using the factory functions from the custom libraries

### 4. Built-in CSS Utilities
- **IGNORE**: FastHTML's built-in `Container` wrapper in `Titled`
- **IGNORE**: Assumptions about default margins, padding, or spacing
- **INSTEAD**: Explicitly control all spacing with Tailwind utilities

## What to KEEP from FastHTML Documentation

### 1. Core FastHTML Patterns
- ✅ Route decorators (`@rt`, `@rt("/path")`)
- ✅ Special route names (`index` for `/`)
- ✅ Query parameters over path parameters
- ✅ `.to()` method for URL generation
- ✅ Return value handling (FT components, tuples, JSON)
- ✅ `serve()` without `if __name__ == "__main__"`

### 2. FastHTML Components
- ✅ All HTML element functions (`Div`, `Button`, `Input`, `Form`, etc.)
- ✅ FastTags (FT) system and rendering
- ✅ Component composition patterns
- ✅ Attribute handling (`cls` parameter, `_for` → `for`, etc.)

### 3. Request Handling
- ✅ Request/response patterns
- ✅ Session and cookie handling
- ✅ Form data binding with dataclasses
- ✅ File upload handling
- ✅ Exception handlers

### 4. Real-time Features
- ✅ WebSocket support
- ✅ Server-Sent Events (SSE) with `EventStream`
- ✅ HTMX integration patterns
- ✅ HTMX extensions handling

### 5. Database Integration
- ✅ Fastlite patterns (if using SQLite)
- ✅ CRUD operations
- ✅ Table creation with dataclasses

## Key Differences in Practice

### Creating Apps
**FastHTML Docs:**
```python
app, rt = fast_app()  # Includes PicoCSS by default
```

**With Custom CSS:**
```python
from cjm_fasthtml_daisyui.core.testing import create_test_app
from cjm_fasthtml_daisyui.core.themes import DaisyUITheme

app, rt = create_test_app(theme=DaisyUITheme.LIGHT)
```

### Creating Pages
**FastHTML Docs:**
```python
return Titled("Page Title", content)  # Auto-wraps in Container
```

**With Custom CSS:**
```python
from cjm_fasthtml_daisyui.core.testing import create_test_page

return create_test_page("Page Title", content)
```

### Styling Components
**FastHTML Docs:**
```python
Button("Click me")  # Styled by PicoCSS
Card(content)      # Semantic HTML with PicoCSS styling
```

**With Custom CSS:**
```python
from cjm_fasthtml_daisyui.components.actions.button import btn, btn_colors
from cjm_fasthtml_tailwind.core.base import combine_classes

Button("Click me", cls=combine_classes(btn, btn_colors.primary))
# Must explicitly apply all styles
```

### Headers and Resources
**FastHTML Docs:**
```python
app, rt = fast_app(hdrs=[...])  # May include PicoCSS resources
```

**With Custom CSS:**
```python
app, rt = create_test_app(theme=DaisyUITheme.LIGHT)
# Then append additional headers if needed:
app.hdrs.append(Script(src="..."))
```

## Important Notes

1. **No Automatic Styling**: Unlike PicoCSS, DaisyUI and Tailwind require explicit class application
2. **Use `combine_classes()`**: Always use this utility to merge multiple CSS classes
3. **Test Utilities Available**: Use `create_test_app()`, `create_test_page()`, and `start_test_server()` for Jupyter notebooks
4. **Theme Support**: DaisyUI themes are handled through the `theme` parameter in `create_test_app()`
5. **No Mixed Frameworks**: Don't try to use PicoCSS components alongside DaisyUI/Tailwind

## Summary

When using `cjm-fasthtml-tailwind` and `cjm-fasthtml-daisyui`:
- Ignore all PicoCSS and MonsterUI references
- Keep all core FastHTML functionality and patterns
- Always explicitly apply CSS classes using the factory functions
- Use the provided test utilities for Jupyter integration
- Follow the `FASTHTML_CSS_UTILITIES_GUIDE.md` CSS utilities guide for proper component styling