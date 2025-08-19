# FastHTML SSE Progress Tracking Patterns

This document captures lessons learned from integrating `cjm-tqdm-capture` with FastHTML's SSE support, based on updating the `01_basic_example.ipynb` notebook.

## Key Lessons Learned

### 1. SSE Implementation with FastHTML

#### Use FastHTML's Built-in `EventStream`
```python
from fasthtml.common import EventStream

@rt("/stream")
def stream(job_id: str):
    # EventStream accepts a generator (not async generator)
    return EventStream(sse_stream(monitor, job_id, interval=0.1, heartbeat=10.0))
```

**Lesson**: FastHTML's `EventStream` class handles SSE properly. Don't use `StreamingResponse` from Starlette directly.

### 2. Generator Types Matter

#### Problem Encountered
```python
# This FAILS - sse_stream returns regular generator, not async
async def generate():
    async for message in sse_stream(...):  # TypeError!
        yield message
```

#### Solution
```python
# sse_stream returns a regular generator, use it directly
return EventStream(sse_stream(monitor, job_id, interval=0.1, heartbeat=10.0))
```

**Lesson**: Check whether your generator is sync or async. `cjm-tqdm-capture`'s `sse_stream` is synchronous.

### 3. HTMX SSE Extension Setup

#### Required Headers
```python
# Add SSE extension AFTER app creation
app, rt = create_test_app(theme=DaisyUITheme.LIGHT)
sse_script = Script(src="https://unpkg.com/htmx-ext-sse@2.2.1/sse.js")
app.hdrs.append(sse_script)
```

**Lesson**: The SSE extension script must be added to enable HTMX SSE features, but append it after app creation when using `create_test_app`.

### 4. SSE Message Handling Patterns

#### Problem: HTMX Auto-swapping JSON
When using `sse_swap="message"`, HTMX automatically inserts the raw JSON into the DOM:
```python
# This displays JSON in the UI instead of updating progress
hx_ext="sse",
sse_connect=f"/stream?job_id={job_id}",
sse_swap="message"  # ❌ Auto-swaps JSON into DOM
```

#### Solution: Manual Message Handling
Use JavaScript to parse and handle SSE messages:
```python
# Don't use sse_swap, handle messages manually
hx_ext="sse",
sse_connect=f"/stream?job_id={job_id}",
# No sse_swap attribute
Script("...handle messages in JS...")
```

### 5. Global SSE Manager Pattern

#### Problem: Multiple Runs Don't Update
Inline scripts in dynamically loaded content may not execute reliably on subsequent HTMX swaps.

#### Solution: Global Event Manager
```python
@rt("/")
def index():
    return create_test_page(
        "Title",
        content,
        # Global SSE manager loaded once
        Script("""
            window.sseManager = {
                currentSource: null,
                connect: function(jobId) {
                    if (this.currentSource) {
                        this.currentSource.close();
                    }
                    this.currentSource = new EventSource('/stream?job_id=' + jobId);
                    // ... handle messages ...
                }
            };
        """)
    )

@rt("/create_job")
def create_job():
    job_id = str(uuid.uuid4())
    # ... start job ...
    return Div(
        # ... progress elements ...
        Script(f"window.sseManager.connect('{job_id}');")  # Just call the manager
    )
```

**Lessons**:
1. Keep SSE handling code in a global scope (not replaced by HTMX)
2. Use a manager pattern to handle connection lifecycle
3. Always close previous connections before opening new ones
4. Get fresh DOM element references inside event handlers

### 6. Route Simplification

#### Use Function Names as Routes
```python
# Instead of
@rt("/api/jobs", methods=["POST"])
def create_job(): ...

# Use
@rt("/create_job")  # or just @rt if function name matches desired route
def create_job(): ...
```

#### Use Query Parameters
```python
# Instead of
@rt("/api/jobs/{job_id}/stream")
def stream_progress(job_id: str): ...

# Use
@rt("/stream")
def stream(job_id: str): ...  # job_id comes from query param
```

### 7. HTMX Integration Best Practices

#### Form Submission Pattern
```python
Button(
    "Start Task",
    hx_post="/create_job",        # POST to endpoint
    hx_target="#progress-container",  # Replace this element
    hx_swap="innerHTML",           # With inner content only
    cls=button_classes
)
```

#### Return Updated UI from POST
```python
@rt("/create_job")
def create_job():
    job_id = str(uuid.uuid4())
    # Start the job...
    
    # Return the updated UI directly
    return Div(
        Progress(id="progress", ...),
        P("Starting...", id="status"),
        Script(f"window.sseManager.connect('{job_id}');")
    )
```

## Complete Pattern Summary

### For SSE Progress Tracking in FastHTML:

1. **Setup**:
   - Add SSE extension script to app headers
   - Create global SSE manager in main page
   - Use `EventStream` for SSE endpoints

2. **Job Creation**:
   - Use HTMX attributes for form submission
   - Return updated UI with progress elements
   - Trigger SSE connection via global manager

3. **Progress Updates**:
   - Handle SSE messages in JavaScript
   - Parse JSON and update DOM elements
   - Get fresh element references each time

4. **Connection Management**:
   - Close old connections before new ones
   - Handle errors gracefully
   - Clean up on completion

### Example Structure:
```
Main Page:
├── Form with hx_post
├── Progress container (target for updates)
└── Global SSE manager script

Create Job Endpoint:
├── Start background job
├── Generate job_id
└── Return progress UI + connection trigger

Stream Endpoint:
└── Return EventStream(sse_stream(...))
```

## Alternative: Pure HTMX Polling Pattern

### When to Use Polling Instead of SSE

Sometimes pure HTMX polling is simpler and more maintainable than SSE:

```python
@rt("/create_job")
def create_job():
    job_id = str(uuid.uuid4())
    runner.start(job_id, task)
    
    return Div(
        # Progress section with HTMX polling
        Div(
            P("Starting..."),
            hx_get=f"/job_progress?job_id={job_id}",
            hx_trigger="load, every 100ms",
            hx_swap="innerHTML",
            id=f"progress-{job_id}"
        )
    )

@rt("/job_progress")
def job_progress(job_id: str):
    snapshot = monitor.snapshot(job_id)
    
    # Create progress bars dynamically based on actual tqdm bars
    bars_html = []
    for bar_id, bar in snapshot['bars'].items():
        bars_html.append(
            Div(
                P(f"{bar.description}: {bar.progress:.1f}%"),
                Progress(value=str(bar.progress), max="100")
            )
        )
    
    if snapshot['completed']:
        # Return final state WITHOUT polling triggers
        return Div(*bars_html, P("Complete!"))
    
    # Continue polling
    return Div(
        *bars_html,
        hx_get=f"/job_progress?job_id={job_id}",
        hx_trigger="load delay:100ms",
        hx_swap="outerHTML"
    )
```

**Benefits**:
- No JavaScript required
- Server-side HTML generation only
- Automatic cleanup (polling stops when no trigger returned)
- Works with dynamic tqdm bar IDs

### Dynamic Progress Bar Creation

#### Problem: tqdm Bar IDs Are Dynamic
tqdm generates bar IDs like `bar-5`, `bar-6`, not predictable `bar-1`, `bar-2`:

```python
# ❌ Don't pre-create bars with fixed IDs
bars = [
    Progress(id="progress-bar-1"),  # Won't match tqdm's bar-5!
    Progress(id="progress-bar-2"),  # Won't match tqdm's bar-6!
]
```

#### Solution: Create Bars Dynamically
```python
# ✅ Create bars based on actual tqdm data
bars_html = []
for bar_id, bar in snapshot['bars'].items():
    bars_html.append(
        Div(
            P(f"{bar.description}: {bar.progress:.1f}%"),
            Progress(id=f"progress-{job_id}-{bar_id}", ...)
        )
    )
```

### Separating Concerns with Multiple Update Regions

#### Problem: Details/Summary Elements Reset
When a Details element is in an HTMX update region, it loses its open/closed state on each update.

#### Solution: Separate Update Regions with Different Polling Rates
```python
@rt("/create_job")
def create_job():
    return Div(
        # Fast-updating progress section
        Div(
            hx_get=f"/job_progress?job_id={job_id}",
            hx_trigger="load, every 100ms",
            hx_swap="innerHTML",
            id=f"progress-{job_id}"
        ),
        
        # Separate slower-updating results section
        Div(
            hx_get=f"/job_results?job_id={job_id}",
            hx_trigger="load, every 500ms",
            hx_swap="outerHTML",  # Critical: use outerHTML!
            id=f"results-{job_id}"
        )
    )

@rt("/job_results")
def job_results(job_id: str):
    if not completed:
        # Keep polling with same attributes
        return Div(
            hx_get=f"/job_results?job_id={job_id}",
            hx_trigger="load delay:500ms",
            hx_swap="outerHTML",
            id=f"results-{job_id}"
        )
    
    # Return static content - NO HTMX attributes stops polling
    return Div(
        Details(Summary("View Results"), ...),
        id=f"results-{job_id}"  # Same ID, no polling
    )
```

### Critical: Understanding hx_swap Modes

#### innerHTML vs outerHTML
- **`hx_swap="innerHTML"`**: Replaces content INSIDE the element
  - Keeps the element's attributes (including HTMX polling)
  - Good for updating content that should keep polling
  
- **`hx_swap="outerHTML"`**: Replaces the ENTIRE element
  - Can remove HTMX attributes to stop polling
  - Essential for stopping polling when done

```python
# Initial element with polling
Div(hx_trigger="every 1s", hx_get="/update", hx_swap="outerHTML", id="poll")

# To stop polling, return element WITHOUT HTMX attributes
Div("Done!", id="poll")  # Same ID, but no hx_trigger = polling stops
```

## Common Pitfalls to Avoid

1. ❌ Don't use `async for` with sync generators
2. ❌ Don't use `sse_swap` for complex JSON data
3. ❌ Don't put SSE handling in replaceable content
4. ❌ Don't forget to close old EventSource connections
5. ❌ Don't use Starlette's `StreamingResponse` directly
6. ❌ Don't assume scripts in HTMX-swapped content will execute
7. ❌ Don't pre-create progress bars with fixed IDs - tqdm uses dynamic IDs
8. ❌ Don't put stateful elements (Details/Summary) in fast-polling regions
9. ❌ Don't use `innerHTML` swap when you need to stop polling - use `outerHTML`

## Quality of Life: Smooth UI Transitions

### Problem: Visual Judder When Switching Views

When clicking a button to view different content (e.g., "View" button for a job), users may experience:
1. Brief flash of empty content
2. Elements loading in sequence (progress bars, then results section)
3. Layout shifts as content appears

#### Root Causes
1. **Empty Initial State**: Returning placeholder content that gets immediately replaced
2. **Sequential Loading**: Different parts loading at different times
3. **Delayed Polling Start**: Gap between content swap and first poll response

### Solution 1: Pre-render Initial State

Instead of showing "Loading..." or empty content, fetch and render the actual current state:

```python
@rt("/view_job")
def view_job(job_id: str):
    # ❌ Don't return empty placeholder
    # return Div(P("Loading..."), hx_get=f"/job_progress?job_id={job_id}", ...)
    
    # ✅ Fetch current state and render it immediately
    snapshot = monitor.snapshot(job_id)
    
    # Build actual progress bars with current values
    bars_html = []
    for bar_id, bar in snapshot['bars'].items():
        bars_html.append(
            Div(
                P(f"{bar.description}: {bar.progress:.1f}%"),
                Progress(value=str(bar.progress), max="100")
            )
        )
    
    return Div(
        # Show actual current state immediately
        P(f"Overall: {snapshot['overall_progress']:.1f}%"),
        Progress(value=str(snapshot['overall_progress']), max="100"),
        *bars_html,
        
        # Then continue polling from this state
        hx_get=f"/job_progress?job_id={job_id}" if not snapshot['completed'] else None,
        hx_trigger="load delay:100ms, every 100ms" if not snapshot['completed'] else None
    )
```

### Solution 2: Pre-render All Sections

Prevent judder from sections loading separately by rendering everything upfront:

```python
@rt("/view_job")
def view_job(job_id: str):
    snapshot = monitor.snapshot(job_id)
    
    # Pre-render results section based on current state
    if snapshot['completed'] and job_id in results_store:
        # Already complete - render static results
        results_section = Div(
            Details(Summary("View Results"), Pre(results)),
            id=f"results-{job_id}"  # No polling attributes
        )
    elif snapshot['completed']:
        # Complete but results not ready - brief poll
        results_section = Div(
            P("Processing results..."),
            hx_get=f"/job_results?job_id={job_id}",
            hx_trigger="load delay:500ms",
            id=f"results-{job_id}"
        )
    else:
        # Still running - empty div that will poll
        results_section = Div(
            hx_get=f"/job_results?job_id={job_id}",
            hx_trigger="load delay:500ms, every 500ms",
            id=f"results-{job_id}"
        )
    
    # Return everything at once - no sequential loading
    return Div(
        progress_section,  # With current values
        results_section    # Pre-rendered based on state
    )
```

### Solution 3: Use HTMX Swap Transitions

Add smooth transitions to prevent jarring swaps:

```python
Button(
    "View",
    hx_get=f"/view_job?job_id={job_id}",
    hx_target="#content",
    hx_swap="innerHTML transition:true"  # Smooth CSS transition
)
```

### Best Practices for Smooth Transitions

1. **Always Pre-fetch Current State**
   - Don't show placeholders that will be immediately replaced
   - Render actual current values from the start

2. **Render All Sections Together**
   - Include all UI sections in initial response
   - Use conditional rendering based on state

3. **Use Appropriate Swap Strategies**
   ```python
   # For content that should appear smoothly
   hx_swap="innerHTML transition:true"
   
   # For stopping polling cleanly
   hx_swap="outerHTML"  # Replace entire element
   ```

4. **Separate Update Regions by Frequency**
   - Fast updates (100ms): Progress bars
   - Slower updates (500ms): Results, status
   - Static after complete: Final results

5. **Handle Edge Cases**
   - Job not found: Show appropriate message
   - Results not ready: Show processing state
   - Already complete: Don't start unnecessary polling

### Example: Complete Smooth View Transition

```python
@rt("/jobs_table")
def jobs_table():
    # Add transition to View button
    Button(
        "View",
        hx_get=f"/view_job?job_id={job_id}",
        hx_target="#progress-container",
        hx_swap="innerHTML transition:true",  # Smooth transition
        cls="btn btn-xs"
    )

@rt("/view_job")
def view_job(job_id: str):
    snapshot = monitor.snapshot(job_id)
    if not snapshot:
        return P("Job not found")
    
    # Pre-render everything with actual values
    return Div(
        # Header
        H3("Viewing Job"),
        P(f"Job ID: {job_id[:8]}..."),
        
        # Progress with current values (not "Loading...")
        Div(
            render_current_progress(snapshot),
            hx_get=f"/job_progress?job_id={job_id}" if not snapshot['completed'] else None,
            hx_trigger="load delay:100ms, every 100ms" if not snapshot['completed'] else None,
            id=f"progress-{job_id}"
        ),
        
        # Results section pre-rendered based on state
        render_results_section(job_id, snapshot)
    )
```

### Testing for Smooth Transitions

- [ ] No empty/placeholder content flash
- [ ] All sections appear together
- [ ] No layout shift after initial render
- [ ] Smooth visual transition between states
- [ ] Progress continues from actual current values
- [ ] Details/Summary elements maintain their state

## Testing Checklist

When implementing SSE progress tracking:
- [ ] First click starts task and shows progress
- [ ] Subsequent clicks work correctly
- [ ] Progress bar updates smoothly
- [ ] Status text shows current progress
- [ ] Completion is detected and displayed
- [ ] Button is re-enabled after completion
- [ ] Old connections are properly closed
- [ ] Error states are handled gracefully
- [ ] View transitions are smooth without judder
- [ ] No placeholder content flashes
- [ ] All UI sections load together