# Lightweight HTMX and PHP Design Patterns with PostgreSQL and PDO

> The owner's canonical pattern document, preserved in full. Where an example here differs from a locked plugin rule (design-system markup, the CRUD contract, the no-modal rule, HTMX CSRF headers), SKILL.md's "Plugin integration" section states the reconciliation — the structure and layering below always stand.

## Recommended Pattern

Use a **feature-oriented Page Controller pattern** with:

* Procedural PHP endpoints as controllers
* Small query functions that accept a `PDO` connection
* Full-page templates for initial navigation
* Partial templates for HTMX responses
* Shared helpers for database access, rendering, escaping, CSRF protection, and HTMX detection

This is effectively a lightweight MVC structure:

| Responsibility | Implementation |
| --- | --- |
| Controller | Small PHP endpoint |
| Data access | Procedural functions using PDO |
| Full-page view | `page.php` |
| Fragment view | Files under `partials/` |
| Application infrastructure | `bootstrap.php`, `db.php`, and `http.php` |

HTMX works particularly well with this structure because the server returns HTML rather than JSON, and HTMX inserts that HTML into the specified target.

Avoid introducing controllers, repositories, services, DTOs, or entity classes initially. Extract reusable components only when duplicated business logic becomes substantial.

## 1. Directory Structure

On the deployment target — Ubuntu 24.04 with Apache — the web root is **`/var/www/html`** (Apache's default DocumentRoot). The application is built directly in that layout: `html/` plays the web-reachable role, and `app/` + `config/` sit beside it under `/var/www/`, **outside** the DocumentRoot, so they are unreachable over HTTP without any vhost changes.

```text
/var/www/
├── app/
│   ├── bootstrap.php
│   ├── db.php
│   ├── http.php
│   │
│   ├── features/
│   │   └── customers/
│   │       └── queries.php
│   │
│   └── views/
│       ├── layout.php
│       │
│       ├── shared/
│       │   ├── flash.php
│       │   └── validation-errors.php
│       │
│       └── customers/
│           ├── page.php
│           └── partials/
│               ├── table.php
│               ├── row.php
│               ├── form.php
│               └── saved.php
│
├── html/                      ← Apache DocumentRoot (/var/www/html)
│   ├── index.php
│   ├── assets/                ← the design-system asset bundle
│   │
│   └── customers/
│       ├── index.php
│       ├── form.php
│       ├── save.php
│       └── delete.php
│
└── config/
    └── application.php
```

This is a **vertical-slice structure**: everything related to the customer interface is grouped under `customers`, while executable endpoints live in the web root at `/var/www/html`. `/var/www/html/customers/save.php` is callable through HTTP; `/var/www/app/views/customers/partials/form.php` is included by PHP and — because it sits outside the DocumentRoot — cannot be reached through the web server at all. Never place `app/` or `config/` inside `html/`.

The endpoint `require` paths are unchanged from a `public/`-style layout because the relative depth is identical: from `/var/www/html/customers/save.php`, `dirname(__DIR__, 2) . '/app/bootstrap.php'` resolves to `/var/www/app/bootstrap.php` (from `/var/www/html/index.php`, it's `dirname(__DIR__) . '/app/bootstrap.php'`).

## 2. HTMX + PHP Patterns

**Pattern A — Dedicated fragment endpoints.** An endpoint that always returns one specific HTML fragment (`hx-get="/customers/list.php"` on a container, often with `hx-trigger="load"`). Use for dropdown options, search results, table bodies, paginated lists, dashboard widgets. Easy to understand; can multiply small endpoint files.

**Pattern B — Dual full-page and partial endpoint.** The same URL returns a complete page for normal navigation or a fragment for an HTMX request, detected via the `HX-Request: true` header:

```php
if (is_htmx_request()) {
    header('Vary: HX-Request');
    echo $tableHtml;
    exit;
}
echo view('layout.php', ['content' => $pageHtml]);
```

`Vary: HX-Request` prevents HTTP caches from confusing the two responses. Use for searchable index pages, pagination, filtered reports, sortable tables, tabbed interfaces. Keeps URLs usable without JavaScript.

**Pattern C — Replace the nearest component.** A control inside a row operates on that row:

```html
<button type="button"
    hx-post="/customers/delete.php"
    hx-vals='{"id": 42}'
    hx-target="closest tr"
    hx-swap="delete"
    hx-confirm="Delete this customer?">
    Delete
</button>
```

`closest tr` keeps the interface independent of generated element IDs. Use for deleting rows, replacing cards, toggling status, inline editing, expanding detail sections.

**Pattern D — Server-triggered refresh.** After a save, the server emits an event; interested regions listen:

```php
header('HX-Trigger: customersChanged');
```

```html
<section id="customers-list-results"
    hx-get="/customers/"
    hx-trigger="customersChanged from:body"
    hx-include="#customers-list-search">
```

The save endpoint doesn't need to know where the table is, how it's structured, or which element to replace. The standard cross-component pattern.

**Pattern E — Out-of-band updates.** One response updates the primary target plus extra elements marked `hx-swap-oob="true"` (a count, a toast, a nav badge). Use sparingly; for ordinary CRUD, Pattern D is easier to maintain.

## 3. Database Connection — `app/db.php`

```php
<?php
declare(strict_types=1);

function db(): PDO
{
    static $pdo = null;
    if ($pdo instanceof PDO) {
        return $pdo;
    }

    $dsn = sprintf(
        'pgsql:host=%s;port=%s;dbname=%s',
        getenv('DB_HOST') ?: '127.0.0.1',
        getenv('DB_PORT') ?: '5432',
        getenv('DB_NAME') ?: 'application'
    );

    $pdo = new PDO($dsn, getenv('DB_USER') ?: 'application', getenv('DB_PASSWORD') ?: '', [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES   => false,
    ]);
    return $pdo;
}
```

Prepared statements for all request-supplied values. No custom PDO subclass — a `db()` function is sufficient.

## 4. HTTP and Rendering Helpers — `app/http.php`

```php
<?php
declare(strict_types=1);

function is_htmx_request(): bool
{
    return ($_SERVER['HTTP_HX_REQUEST'] ?? '') === 'true';
}

function e(mixed $value): string
{
    return htmlspecialchars((string) $value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

/**
 * Render a view located under app/views.
 * Template names must be supplied by application code, never from request parameters.
 */
function view(string $template, array $data = []): string
{
    $viewsRoot = realpath(__DIR__ . '/views');
    if ($viewsRoot === false) {
        throw new RuntimeException('View directory does not exist.');
    }
    $path = realpath($viewsRoot . '/' . ltrim($template, '/'));
    if ($path === false || !str_starts_with($path, $viewsRoot . DIRECTORY_SEPARATOR)) {
        throw new RuntimeException("Invalid view: {$template}");
    }
    extract($data, EXTR_SKIP);
    ob_start();
    try {
        require $path;
        return (string) ob_get_clean();
    } catch (Throwable $exception) {
        ob_end_clean();
        throw $exception;
    }
}

function require_post(): void
{
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        http_response_code(405);
        header('Allow: POST');
        exit('Method Not Allowed');
    }
}

function request_integer(string $name): ?int
{
    $value = $_POST[$name] ?? $_GET[$name] ?? null;
    if ($value === null || $value === '') {
        return null;
    }
    $filtered = filter_var($value, FILTER_VALIDATE_INT);
    return $filtered === false ? null : $filtered;
}
```

Every value originating from a user or the database is escaped with `e()` before insertion into HTML.

`app/bootstrap.php` loads `db.php` + `http.php` and starts the session; as the application grows it also loads authentication/authorization helpers, CSRF helpers, configuration, logging, and error handlers.

## 5. Procedural Query Functions — `app/features/{feature}/queries.php`

SQL lives in small procedural functions that take `PDO` as their first parameter — most of the separation of a repository class without the class hierarchy, and easier to test:

```php
function find_customers(PDO $pdo, string $search = ''): array
{
    $search = trim($search);
    if ($search === '') {
        return $pdo->query(<<<'SQL'
            SELECT id, name, email, created_at
            FROM customers
            ORDER BY name
            LIMIT 100
        SQL)->fetchAll();
    }
    $statement = $pdo->prepare(<<<'SQL'
        SELECT id, name, email, created_at
        FROM customers
        WHERE name ILIKE :search OR email ILIKE :search
        ORDER BY name
        LIMIT 100
    SQL);
    $statement->execute(['search' => '%' . $search . '%']);
    return $statement->fetchAll();
}

function find_customer(PDO $pdo, int $id): ?array
{
    $statement = $pdo->prepare('SELECT id, name, email, created_at FROM customers WHERE id = :id');
    $statement->execute(['id' => $id]);
    $customer = $statement->fetch();
    return $customer === false ? null : $customer;
}

function insert_customer(PDO $pdo, string $name, string $email): array
{
    $statement = $pdo->prepare(<<<'SQL'
        INSERT INTO customers (name, email)
        VALUES (:name, :email)
        RETURNING id, name, email, created_at
    SQL);
    $statement->execute(['name' => $name, 'email' => $email]);
    return $statement->fetch();
}

function update_customer(PDO $pdo, int $id, string $name, string $email): array
{
    $statement = $pdo->prepare(<<<'SQL'
        UPDATE customers
        SET name = :name, email = :email
        WHERE id = :id
        RETURNING id, name, email, created_at
    SQL);
    $statement->execute(['id' => $id, 'name' => $name, 'email' => $email]);
    $customer = $statement->fetch();
    if ($customer === false) {
        throw new RuntimeException('Customer not found.');
    }
    return $customer;
}

function delete_customer(PDO $pdo, int $id): bool
{
    $statement = $pdo->prepare('DELETE FROM customers WHERE id = :id');
    $statement->execute(['id' => $id]);
    return $statement->rowCount() === 1;
}
```

Use `RETURNING` on writes. A query function never touches `$_GET`/`$_POST`, HTMX, HTTP status codes, templates, or redirects — the controller reads the request and passes explicit values.

## 6. The Dual Endpoint — `/var/www/html/customers/index.php`

```php
<?php
declare(strict_types=1);

require_once dirname(__DIR__, 2) . '/app/bootstrap.php';
require_once dirname(__DIR__, 2) . '/app/features/customers/queries.php';

$search = trim((string) ($_GET['q'] ?? ''));
if (mb_strlen($search) > 100) {
    $search = mb_substr($search, 0, 100);
}

$customers = find_customers(db(), $search);
$tableHtml = view('customers/partials/table.php', ['customers' => $customers]);

if (is_htmx_request()) {
    header('Vary: HX-Request');
    echo $tableHtml;
    exit;
}

$pageHtml = view('customers/page.php', ['search' => $search, 'tableHtml' => $tableHtml]);
echo view('layout.php', ['title' => 'Customers', 'content' => $pageHtml]);
```

Normal browser request → complete page with layout. HTMX request → only the table fragment.

The page's search input demonstrates the trigger idiom:

```html
hx-trigger="input changed delay:300ms, search"
```

`input` reacts to typing, `changed` suppresses duplicate requests for the same value, `delay:300ms` waits for a pause, `search` handles the input's clear button. The form also keeps conventional `method`/`action` attributes so it works without HTMX.

## 7. Table and Row Partials

`table.php` renders the empty state or the table, delegating each row:

```php
<?php foreach ($customers as $customer): ?>
    <?= view('customers/partials/row.php', ['customer' => $customer]) ?>
<?php endforeach; ?>
```

`row.php` renders one `<tr id="customer-row-<?= e($customer['id']) ?>">` with escaped cells and its action buttons (Edit per the CRUD contract; Delete via Pattern C).

Partials contain HTML, escaped output, and basic presentation conditions. They do not contain SQL, request processing, authorization decisions, redirect logic, or database connection creation.

## 8. Form Endpoint and Partial

`/var/www/html/customers/form.php` loads the record (or a blank array for add), 404s when an id doesn't resolve, and renders `customers/partials/form.php` with `['customer' => ..., 'errors' => []]`.

The form partial renders its container with a stable id, a validation-errors block when `$errors` is non-empty, a hidden `id` field on edit, and targets its own container so the server can replace it wholesale:

```html
<form method="post" action="/customers/save.php"
      hx-post="/customers/save.php"
      hx-target="#customer-form-container"
      hx-swap="outerHTML">
```

The response replacing it is either the form with validation errors, a success confirmation, or a refreshed form.

## 9. Save Transaction Script — `/var/www/html/customers/save.php`

```php
require_post();
verify_csrf();

$id    = request_integer('id');
$name  = trim((string) ($_POST['name'] ?? ''));
$email = trim((string) ($_POST['email'] ?? ''));

$errors = [];
if ($name === '')  { $errors[] = 'Name is required.'; }
if ($email === '') { $errors[] = 'Email is required.'; }
elseif (filter_var($email, FILTER_VALIDATE_EMAIL) === false) { $errors[] = 'Enter a valid email address.'; }

if ($errors !== []) {
    echo view('customers/partials/form.php', ['customer' => compact('id', 'name', 'email'), 'errors' => $errors]);
    exit;
}

try {
    $customer = $id === null
        ? insert_customer(db(), $name, $email)
        : update_customer(db(), $id, $name, $email);
} catch (PDOException $exception) {
    error_log($exception->getMessage());
    echo view('customers/partials/form.php', [
        'customer' => compact('id', 'name', 'email'),
        'errors'   => ['The customer could not be saved.'],
    ]);
    exit;
}

header('HX-Trigger: customersChanged');
echo view('customers/partials/saved.php', ['customer' => $customer]);
```

Flow: validation errors replace the form → a successful save replaces it with a confirmation (`saved.php`, same container id) → `HX-Trigger` fires `customersChanged` → the list refreshes itself. The save endpoint never knows the table's location or structure.

`/var/www/html/customers/delete.php`: `require_post()`, `verify_csrf()`, validate the id (400), run `delete_customer` (404 on false), return 200 — the requesting button's `hx-target="closest tr" hx-swap="delete"` removes the row.

## 10. Transactions

Use an explicit transaction when one user action performs multiple related operations (e.g. insert + audit-log row):

```php
$pdo = db();
try {
    $pdo->beginTransaction();
    $customer = insert_customer($pdo, $name, $email);
    log_activity($pdo, 'customer', $customer['id'], 'created');
    $pdo->commit();
} catch (Throwable $exception) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }
    throw $exception;
}
```

A single INSERT/UPDATE/DELETE doesn't need an application-level transaction unless it's part of a larger operation.

## 11. Layer Rules

**Controllers** are small transaction scripts: bootstrap → authenticate/authorize → read input → validate → execute query functions → log activity → render page or partial. A controller never becomes a collection of reusable functions; reusable behavior belongs in query files, validation helpers, authorization helpers, HTTP helpers, or domain-specific procedural functions.

**Query functions** contain SQL and database behavior only. Good: `find_customer(PDO $pdo, int $id): ?array`. Avoid: `find_customer_from_request(): void`.

**Partials** contain presentation logic only — conditional rendering (`if ($customers === [])`) and formatting (`date('M j, Y', ...)`) are fine; querying the database inside a partial is not. Views receive data; they never retrieve it.

**Dependency direction:** HTTP endpoint → query functions → PDO → PostgreSQL, and HTTP endpoint → template. Never template → database.

## 12. HTMX Targeting and Response Granularity

Prefer local targets (`hx-target="closest tr"`) over explicit ids for element-local operations — local targets reduce coupling. Use explicit ids when one unrelated part of the page updates another (a form updating a results table, a save updating a notification region, a widget updating a summary counter).

Return the smallest *meaningful* component: a complete badge for a status change, a complete `<tr>` for an edited row, the complete form container for a refreshed form. Never tiny text fragments that depend on surrounding markup — a response should correspond to a recognizable UI component.

## 13. URLs and Progressive Enhancement

Straightforward URL structure (`GET /customers/`, `GET /customers/form.php?id=42`, `POST /customers/save.php`, `POST /customers/delete.php`); Apache rewriting can later expose cleaner URLs (`/customers/42/edit`) without changing the internal architecture.

Where practical, retain ordinary `href`, `action`, and `method` attributes alongside hx-* — easier debugging, meaningful browser behavior, better accessibility, easier automated testing, partial functionality without JavaScript.

## 14. CSRF and Security Rules

CSRF on all state-changing endpoints: `csrf_token()` (per-session `bin2hex(random_bytes(32))`), hidden `csrf_token` input on POST forms, `verify_csrf()` with `hash_equals` immediately after `require_post()`. (The plugin's `verify_csrf` also accepts the `X-CSRF-Token` header that the shell's HTMX listener sends — see php-session-auth.)

At minimum:

* Escape every dynamic HTML value with `e()`
* Prepared statements for database values
* POST for state-changing operations; CSRF tokens on POST forms
* Check authorization in every endpoint; never trust `HX-Request` as an authorization mechanism
* Secure session-cookie settings
* Never expose database exception details to users
* Never pass request values into template filenames
* Validate and constrain IDs, sort fields, limits, and search values
* **Allowlists for dynamic SQL identifiers** — prepared statements protect values, not identifiers:

```php
$allowedSortFields = ['name', 'email', 'created_at'];
$orderBy = (string) ($_GET['order_by'] ?? 'name');
if (!in_array($orderBy, $allowedSortFields, true)) {
    $orderBy = 'name';
}
```

## 15. The Standard, Summarized

```text
Feature folders
+ procedural endpoint controllers
+ procedural PDO query functions
+ full-page templates
+ partial templates
+ dual full-page and partial GET responses
+ closest-element replacements
+ HX-Trigger events for cross-component refreshes
```

```text
Request → small endpoint controller → validation and authorization
        → procedural PDO query function → PostgreSQL
        → page or partial template → HTML response
        → HTMX replaces the target component
```

Clear separation while remaining close to traditional PHP; scales from small internal tools to moderately large SaaS applications without a framework-style object model.
