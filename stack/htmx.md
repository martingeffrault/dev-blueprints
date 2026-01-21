# HTMX (2025)

> **Last updated**: January 2026
> **Versions covered**: 2.x
> **Purpose**: HTML-first approach to building dynamic web applications

---

## Philosophy (2025-2026)

HTMX enables the **hypermedia-driven application (HDA)** architecture — extending HTML with attributes to make AJAX requests and update the DOM, without writing JavaScript.

**Key philosophical shifts:**
- **HTML is the API** — Server returns HTML, not JSON
- **Hypermedia as the engine** — Links and forms, extended
- **Less JavaScript** — Most apps don't need React/Vue
- **Progressive enhancement** — Works without JS (mostly)
- **Server-rendered** — Logic stays on the server
- **Simpler security** — No JSON injection vulnerabilities
- **Long-term stability** — htmx 2025 ≈ htmx 2035

---

## TL;DR

- Server returns HTML fragments, not JSON
- Use `hx-get`, `hx-post`, `hx-put`, `hx-delete` for requests
- Use `hx-target` to specify where to put the response
- Use `hx-swap` to control how content is inserted
- Use `hx-trigger` to control when requests fire
- Serve frontend and backend from the same domain
- Never load untrusted HTML — security is critical
- Great for CRUD apps, not for offline/real-time heavy apps

---

## Best Practices

### Basic Request

```html
<!-- Simple GET request, replace inner content -->
<button hx-get="/api/users" hx-target="#user-list">
  Load Users
</button>

<div id="user-list">
  <!-- Users will be loaded here -->
</div>
```

### Server Response (Return HTML)

```python
# Python/Flask example
@app.route('/api/users')
def get_users():
    users = User.query.all()
    return render_template('partials/user-list.html', users=users)
```

```html
<!-- templates/partials/user-list.html -->
<ul>
  {% for user in users %}
    <li>{{ user.name }} - {{ user.email }}</li>
  {% endfor %}
</ul>
```

### CRUD Operations

```html
<!-- Create -->
<form hx-post="/api/users" hx-target="#user-list" hx-swap="beforeend">
  <input name="name" placeholder="Name" required>
  <input name="email" type="email" placeholder="Email" required>
  <button type="submit">Add User</button>
</form>

<!-- Read (already shown above) -->

<!-- Update -->
<form hx-put="/api/users/{{ user.id }}" hx-target="this" hx-swap="outerHTML">
  <input name="name" value="{{ user.name }}">
  <input name="email" value="{{ user.email }}">
  <button type="submit">Update</button>
</form>

<!-- Delete -->
<button
  hx-delete="/api/users/{{ user.id }}"
  hx-target="closest li"
  hx-swap="outerHTML"
  hx-confirm="Are you sure?"
>
  Delete
</button>
```

### Active Search

```html
<input
  type="search"
  name="q"
  placeholder="Search users..."
  hx-get="/api/users/search"
  hx-trigger="input changed delay:300ms, search"
  hx-target="#search-results"
  hx-indicator="#loading"
>

<span id="loading" class="htmx-indicator">Searching...</span>

<div id="search-results">
  <!-- Results appear here -->
</div>
```

### Infinite Scroll

```html
<div id="posts">
  {% for post in posts %}
    <article>
      <h2>{{ post.title }}</h2>
      <p>{{ post.excerpt }}</p>
    </article>
  {% endfor %}

  <!-- Load more trigger -->
  <div
    hx-get="/api/posts?page={{ next_page }}"
    hx-trigger="revealed"
    hx-swap="outerHTML"
  >
    Loading more...
  </div>
</div>
```

### Click to Edit

```html
<!-- View mode -->
<div id="user-{{ user.id }}">
  <span>{{ user.name }}</span>
  <button
    hx-get="/api/users/{{ user.id }}/edit"
    hx-target="#user-{{ user.id }}"
    hx-swap="outerHTML"
  >
    Edit
  </button>
</div>

<!-- Edit mode (returned from /api/users/:id/edit) -->
<form
  id="user-{{ user.id }}"
  hx-put="/api/users/{{ user.id }}"
  hx-target="this"
  hx-swap="outerHTML"
>
  <input name="name" value="{{ user.name }}">
  <button type="submit">Save</button>
  <button
    type="button"
    hx-get="/api/users/{{ user.id }}"
    hx-target="#user-{{ user.id }}"
    hx-swap="outerHTML"
  >
    Cancel
  </button>
</form>
```

### Tabs

```html
<div class="tabs">
  <button
    hx-get="/tabs/profile"
    hx-target="#tab-content"
    class="active"
  >
    Profile
  </button>
  <button
    hx-get="/tabs/settings"
    hx-target="#tab-content"
  >
    Settings
  </button>
  <button
    hx-get="/tabs/notifications"
    hx-target="#tab-content"
  >
    Notifications
  </button>
</div>

<div id="tab-content">
  <!-- Tab content loads here -->
</div>
```

### Modal Dialog

```html
<!-- Trigger -->
<button
  hx-get="/modals/confirm-delete/{{ item.id }}"
  hx-target="#modal-container"
  hx-swap="innerHTML"
>
  Delete
</button>

<div id="modal-container"></div>

<!-- Modal content (returned from server) -->
<div class="modal-backdrop" _="on click remove me">
  <div class="modal" _="on click halt">
    <h2>Confirm Delete</h2>
    <p>Are you sure you want to delete this item?</p>
    <button
      hx-delete="/api/items/{{ item.id }}"
      hx-target="#item-{{ item.id }}"
      hx-swap="outerHTML"
      _="on htmx:afterRequest remove #modal-container's innerHTML"
    >
      Yes, Delete
    </button>
    <button _="on click remove closest .modal-backdrop">
      Cancel
    </button>
  </div>
</div>
```

### Form Validation

```html
<form hx-post="/api/users" hx-target="#form-response">
  <div>
    <label>Email</label>
    <input
      name="email"
      type="email"
      hx-get="/api/validate/email"
      hx-trigger="blur changed"
      hx-target="next .error"
    >
    <span class="error"></span>
  </div>

  <div>
    <label>Username</label>
    <input
      name="username"
      hx-get="/api/validate/username"
      hx-trigger="blur changed delay:200ms"
      hx-target="next .error"
    >
    <span class="error"></span>
  </div>

  <button type="submit">Register</button>
</form>

<div id="form-response"></div>
```

### Dependent Dropdowns

```html
<select
  name="country"
  hx-get="/api/states"
  hx-trigger="change"
  hx-target="#state-select"
>
  <option value="">Select Country</option>
  <option value="us">United States</option>
  <option value="ca">Canada</option>
</select>

<div id="state-select">
  <select name="state" disabled>
    <option value="">Select State</option>
  </select>
</div>
```

### Loading Indicators

```html
<button
  hx-get="/api/slow-operation"
  hx-target="#result"
  hx-indicator="#spinner"
>
  Load Data
</button>

<span id="spinner" class="htmx-indicator">
  <img src="/spinner.gif" alt="Loading...">
</span>

<style>
  .htmx-indicator {
    display: none;
  }
  .htmx-request .htmx-indicator {
    display: inline;
  }
  .htmx-request.htmx-indicator {
    display: inline;
  }
</style>
```

### Out-of-Band Swaps

```html
<!-- Server can update multiple elements at once -->
<!-- Response from server: -->
<div id="main-content">
  <!-- Main content update -->
</div>

<!-- This updates a different element (out of band) -->
<div id="notification-count" hx-swap-oob="true">
  5
</div>
```

### WebSocket Support

```html
<div hx-ws="connect:/chat">
  <div id="chat-messages">
    <!-- Messages appear here -->
  </div>

  <form hx-ws="send:submit">
    <input name="message" placeholder="Type a message...">
    <button type="submit">Send</button>
  </form>
</div>
```

### Boost Navigation

```html
<!-- Boost all links in this container -->
<div hx-boost="true">
  <a href="/page1">Page 1</a>
  <a href="/page2">Page 2</a>
  <a href="/page3">Page 3</a>
</div>

<!-- These become AJAX requests that swap body content -->
```

---

## Anti-Patterns

### ❌ Calling Untrusted HTML APIs

**Why it's bad**: HTML can execute scripts — never execute untrusted code.

```html
<!-- ❌ DON'T — Load HTML from external API -->
<div hx-get="https://external-api.com/widget">
  <!-- DANGEROUS! -->
</div>

<!-- ✅ DO — Only load from your own server -->
<div hx-get="/api/widget">
</div>

<!-- If you need external JSON, use fetch -->
<script>
  fetch('https://external-api.com/data')
    .then(r => r.json())
    .then(data => { /* Safe to use */ });
</script>
```

### ❌ Separating Frontend and Backend

**Why it's bad**: HTMX is designed for same-origin hypermedia.

```html
<!-- ❌ DON'T — Separate repos, different domains -->
Frontend: https://app.example.com
Backend: https://api.example.com/users

<!-- ✅ DO — Same server/domain -->
<div hx-get="/api/users">
</div>
```

### ❌ Using hx-push-url on Destructive Actions

**Why it's bad**: Creates invalid URLs after delete/actions.

```html
<!-- ❌ DON'T -->
<button
  hx-delete="/api/items/123"
  hx-push-url="true"
>
  Delete
</button>
<!-- URL becomes /api/items/123 after delete — broken! -->

<!-- ✅ DO — Only push URL on navigation -->
<button hx-delete="/api/items/123" hx-target="#item-123">
  Delete
</button>
```

### ❌ Complex State Management with HTMX

**Why it's bad**: HTMX is for hypermedia, not client-side state.

```html
<!-- ❌ DON'T — Try to manage complex state client-side -->
<!-- HTMX is not a replacement for Redux/Zustand -->

<!-- ✅ DO — Keep state on the server -->
<!-- Use HTMX for CRUD, navigation, forms -->
<!-- Use React/Vue for complex interactive dashboards -->
```

### ❌ Not Using Template Engine Escaping

**Why it's bad**: XSS vulnerabilities.

```html
<!-- ❌ DON'T — Raw HTML injection -->
{{ user.bio | safe }}
${userInput}

<!-- ✅ DO — Let template engine escape -->
{{ user.bio }}
<%= userInput %>
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **2.0** | Jun 2024 | Extensions moved out, Web Components, DELETE query params |
| **2.x** | 2025 | Stability focus, extensions API |
| **4.0** | Future | fetch() API, explicit inheritance |

### HTMX 2.0 Breaking Changes

**Extensions Moved Out of Core:**
```html
<!-- ❌ OLD — Extensions bundled in htmx -->
<script src="htmx.min.js"></script>
<!-- SSE, WS, etc. were included -->

<!-- ✅ NEW — Load extensions separately -->
<script src="https://unpkg.com/htmx.org@2/dist/htmx.min.js"></script>
<script src="https://unpkg.com/htmx-ext-sse@2/sse.js"></script>
<script src="https://unpkg.com/htmx-ext-ws@2/ws.js"></script>
<script src="https://unpkg.com/htmx-ext-head-support@2/head-support.js"></script>
```

**DELETE Requests Now Use Query Params:**
```html
<!-- ❌ OLD (htmx 1.x) — DELETE with body params -->
<button hx-delete="/api/items/123" hx-vals='{"confirm": true}'>
  <!-- Params were sent in request body -->
</button>

<!-- ✅ NEW (htmx 2.x) — DELETE with query params -->
<button hx-delete="/api/items/123?confirm=true">
  <!-- Or use proper HTTP semantics -->
</button>

<!-- If you need body params (not recommended), use POST with X-HTTP-Method-Override -->
```

**SSE/WebSocket Syntax Changed:**
```html
<!-- ❌ OLD (htmx 1.x) — Space-separated -->
<div hx-sse="connect /chat swap message">

<!-- ✅ NEW (htmx 2.x) — Colon-separated, extension attribute -->
<div hx-ext="sse" sse-connect="/chat" sse-swap="message">
```

**WebSocket Extension:**
```html
<!-- ❌ OLD -->
<div hx-ws="connect /chat">

<!-- ✅ NEW — Explicit extension -->
<div hx-ext="ws" ws-connect="/chat">
  <form ws-send>
    <input name="message">
    <button>Send</button>
  </form>
</div>
```

**Disable Attribute Inheritance:**
```html
<!-- NEW — Disable inheritance globally -->
<script>
  htmx.config.disableInheritance = true;
</script>

<!-- Or per-element -->
<div hx-inherit="false">
  <button hx-get="/api/data">No inherited attrs</button>
</div>
```

**selfRequestsOnly Default Changed:**
```javascript
// NEW DEFAULT — Only same-origin requests allowed
// htmx.config.selfRequestsOnly = true (default)

// To allow cross-origin (not recommended):
htmx.config.selfRequestsOnly = false;
```

**Response Handling Configuration:**
```javascript
// NEW — Configure response code handling
htmx.config.responseHandling = [
  { code: "204", swap: false },           // No swap on 204
  { code: "[23]..", swap: true },          // Swap on 2xx/3xx
  { code: "[45]..", swap: false, error: true }, // Error on 4xx/5xx
];
```

### HTMX Extensions (2.x)

**Head Support (Built-in alternative):**
```html
<!-- Head tag merging is now built into htmx 2.0 -->
<!-- For more control, use the extension: -->
<script src="https://unpkg.com/htmx-ext-head-support@2/head-support.js"></script>
<body hx-ext="head-support">
```

**Multi-Swap Extension:**
```html
<script src="https://unpkg.com/htmx-ext-multi-swap@2/multi-swap.js"></script>

<!-- Swap multiple targets with one response -->
<button
  hx-get="/api/dashboard"
  hx-ext="multi-swap"
  hx-swap="multi:#stats:innerHTML,#chart:outerHTML"
>
  Refresh Dashboard
</button>
```

**Preload Extension:**
```html
<script src="https://unpkg.com/htmx-ext-preload@2/preload.js"></script>

<!-- Preload on hover/focus -->
<a href="/page" hx-ext="preload" preload="mousedown">
  Fast Navigation
</a>
```

### Web Components Support (2.0)

```html
<!-- htmx now works properly in Shadow DOM -->
<my-component>
  #shadow-root
    <button hx-get="/api/data" hx-target="#result">
      Load Data
    </button>
    <div id="result"></div>
</my-component>
```

### HTMX 2025+ Direction

The htmx team prioritizes **stability over new features**:
- Core library will resist new features
- New capabilities added via extensions
- "htmx you write in 2025 will look very similar to htmx you write in 2035"

**HTMX 4.x Preview:**
- All AJAX requests will use native `fetch()` instead of XMLHttpRequest
- Even more explicit attribute inheritance
- Further Web Standards alignment

---

## Quick Reference

| Attribute | Purpose |
|-----------|---------|
| `hx-get` | GET request |
| `hx-post` | POST request |
| `hx-put` | PUT request |
| `hx-patch` | PATCH request |
| `hx-delete` | DELETE request |
| `hx-target` | Element to update |
| `hx-swap` | How to insert content |
| `hx-trigger` | When to trigger request |
| `hx-indicator` | Loading indicator |
| `hx-confirm` | Confirmation dialog |
| `hx-push-url` | Update URL |
| `hx-boost` | Boost links/forms |

| hx-swap Value | Behavior |
|---------------|----------|
| `innerHTML` | Replace inner HTML (default) |
| `outerHTML` | Replace entire element |
| `beforebegin` | Insert before element |
| `afterbegin` | Insert at start of element |
| `beforeend` | Insert at end of element |
| `afterend` | Insert after element |
| `delete` | Delete element |
| `none` | Don't swap |

| hx-trigger | When |
|------------|------|
| `click` | On click (default for most) |
| `change` | On change (select, input) |
| `submit` | On form submit |
| `load` | On page load |
| `revealed` | When scrolled into view |
| `intersect` | IntersectionObserver |
| `every 2s` | Polling |
| `input changed delay:300ms` | Debounced input |

---

## When to Use HTMX

| Good Fit | Not Ideal |
|----------|-----------|
| CRUD applications | Complex dashboards |
| Content sites | Real-time multiplayer |
| Admin panels | Offline-first apps |
| E-commerce | Heavy client-side state |
| Forms & validation | Drag-and-drop editors |
| Server-rendered apps | WebGL/Canvas apps |

---

## Resources

- [Official HTMX Documentation](https://htmx.org/docs/)
- [HTMX Essays](https://htmx.org/essays/)
- [Hypermedia Systems Book](https://hypermedia.systems/)
- [HTMX Patterns](https://hypermedia.systems/htmx-patterns/)
- [When to Use Hypermedia](https://htmx.org/essays/when-to-use-hypermedia/)
- [Web Security with HTMX](https://htmx.org/essays/web-security-basics-with-htmx/)
- [Is HTMX Worth Learning in 2025?](https://www.wearedevelopers.com/en/magazine/537/is-htmx-worth-learning-in-2025-537)
