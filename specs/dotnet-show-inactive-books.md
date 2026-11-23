# Change specification: Show inactive books on the BookCatalog list

## Metadata

- **Owner:** Aaron Arendt
- **Status:** Approved
- **Source root:** `legacy-src/dotnet-bookcatalog/shared-legacy-app`
- **Related issue:** not created

## Problem and outcomes

### Problem

Catalog maintainers who deactivate a book cannot find it again from the UI.
`BooksController.Index` filters the list with `db.Books.Where(b => b.IsActive)`
(`src/BookCatalog.Web/Controllers/BooksController.cs`, `Index`), so inactive
books never appear on `/Books`. `Edit` (POST) can set `IsActive = false`
(`BooksController.cs`, `Edit`), after which the book is reachable only by typing
its `/Books/Details/{id}` URL; `Details`, `Edit` and `Delete` all use
`db.Books.Find(id)` and are unaffected by the filter.

`src/BookCatalog.Web/Views/Books/Index.cshtml` already has a Status column that
renders `ACTIVE` or an `inactive` badge (`badge-inactive`, defined in
`src/BookCatalog.Web/Content/Site.css`), but the inactive branch can never run
because the controller supplies only active books. The seed data includes one
inactive record, "The Matrix: The Shooting Script"
(`src/BookCatalog.Web/Models/ApplicationDbContext.cs`, `SeedBooks`).

Affected users: catalog maintainers who need to review, re-activate or delete
inactive books. The application has no authentication in the files inspected;
whether inactive books may be shown to all users is an unresolved question.

### Desired outcomes

1. A user can choose to see inactive books on the list page with a "Show
   Inactive" checkbox; without that choice the list is unchanged.
2. Inactive books shown in the list are visibly marked as inactive, and the
   existing View, Edit and Del links work for them.

### Success measures

- With the checkbox unchecked, `/Books` shows only active books, as today.
- With the checkbox checked, every inactive book is also listed and marked.

## Functional requirements

| ID | Requirement | Evidence or rationale |
| --- | --- | --- |
| FR-1 | The list page shall provide a "Show Inactive" checkbox that is unchecked by default. | Decision recorded with the maintainer in planning. |
| FR-2 | The checkbox shall submit by HTTP GET using the query parameter `showInactive`, so the state is visible in the URL. | Decision recorded in planning. |
| FR-3 | When `showInactive` is a valid `true`, the list shall include inactive books together with active books, ordered by title. | `BooksController.cs:Index` orders by `Title`. |
| FR-4 | When `showInactive` is absent or is any value other than a valid `true`, the list shall show active books only and shall not return an error. | Decision recorded in planning. |
| FR-5 | Each inactive book in the list shall display the existing inactive marker. | `Views/Books/Index.cshtml` Status column; `Content/Site.css` `.badge-inactive`. |
| FR-6 | After submitting the form, the checkbox shall reflect the requested state. | Usability of a GET filter. |
| FR-7 | The "Showing N record(s)" text shall equal the number of rows displayed. | `Index.cshtml` uses `Model.Count()`. |
| FR-8 | The checked state shall not be remembered across visits (no cookie or session). | Decision recorded in planning. |

## Constraints and out of scope

### Constraints

- **Allowed paths:**
  - `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Controllers/BooksController.cs` (the `Index` action only)
  - `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Views/Books/Index.cshtml`
  - `specs/dotnet-show-inactive-books.md` (this specification)
- **Primary validation command:** `msbuild BookCatalog.sln -restore -p:Configuration=Release`, run from `legacy-src/dotnet-bookcatalog/shared-legacy-app`.
- **Compatibility:** Preserve ASP.NET MVC 5, .NET Framework 4.8, EF6, `packages.config` and the solution layout; the default `/Books` view, routes, columns and CRUD links stay unchanged. (`legacy-src/dotnet-bookcatalog/AGENTS.md`; `.github/instructions/dotnet.instructions.md`)
- **Dependencies:** No new package, configuration change, or schema change.
- **Data and security:** Use only disposable LocalDB data; add no secrets; the query value must not be concatenated into a query or emitted unencoded.

### Exclusions (out of scope)

- Changes to Create, Edit, Details, Delete, the `Book` model, the database schema or seed data.
- Soft-delete behavior, cookies or session persistence, and JavaScript-only filtering.
- Removing or changing the User-Agent display (`ViewBag.UserAgent`, `Index.cshtml`).
- Changes to `Content/Site.css` or any file outside the allowed paths.
- A new test project, ASP.NET Core migration, or other modernization.

## Edge cases

| Type | Condition | Expected behavior |
| --- | --- | --- |
| Normal | `/Books` with no parameter | Active books only; checkbox unchecked. |
| Normal | `/Books?showInactive=true` | Active and inactive books, sorted by title; inactive rows marked; checkbox checked. |
| Failure | `/Books?showInactive=banana` or an empty value | HTTP 200; behaves as the default (active only); no error page. |
| Boundary | Repeated or combined values such as `showInactive=true&showInactive=false` or `true,false` | The first value is used, so `true` first shows inactive books and `false` first does not. No error. |
| Boundary | No active books exist but inactive books do | Default view shows the existing "No books found" box unchanged; it does not mention the checkbox. |
| Compatibility | Redirects to `Index` after Create, Edit or Delete | They continue to show the default view. |
| Compatibility | Existing links to `/Books` and `/Books/Index` | Continue to work unchanged. |

## Acceptance criteria

### AC-1: Default list is unchanged

- **Given** a disposable database that contains at least one active and one inactive book
- **When** a user opens `/Books`
- **Then** only active books appear, ordered by title, with no inactive rows, and the checkbox is unchecked
- **Verification:** Manual IIS Express/LocalDB check, recording the URL and the rows shown

### AC-2: Inactive books can be shown

- **Given** the same database
- **When** a user opens `/Books?showInactive=true`
- **Then** every active book and every inactive book appears ordered by title, each inactive book carries the `inactive` marker, and the checkbox is checked
- **Verification:** Manual check, recording the URL and the rows shown

### AC-3: The checkbox drives the URL

- **Given** the list page is open
- **When** the user toggles the checkbox and submits the form
- **Then** the URL carries the matching `showInactive` value and the list matches AC-1 or AC-2
- **Verification:** Manual check of the resulting URL and list

### AC-4: Invalid value is tolerated

- **Given** the list page is available
- **When** a user opens `/Books?showInactive=banana`
- **Then** the response is HTTP 200 and the list matches AC-1
- **Verification:** Manual browser or HTTP check recording the status code

### AC-4b: Repeated parameter uses the first value

- **Given** the list page is available
- **When** a user opens `/Books?showInactive=true&showInactive=false`, and then `/Books?showInactive=false&showInactive=true`
- **Then** the first request lists inactive books, the second lists active books only, and both return HTTP 200
- **Verification:** Manual check recording each URL, status code and rows shown

### AC-5: Record count is accurate

- **Given** `/Books?showInactive=true` is displayed
- **When** the footer text renders
- **Then** "Showing N record(s)" equals the number of displayed rows
- **Verification:** Manual count against the footer

### AC-6: Existing actions still work for listed inactive books

- **Given** an inactive book is listed with the checkbox checked
- **When** the user follows its `[View]`, `[Edit]` and `[Del]` links
- **Then** they open the same Details, Edit and Delete pages as for an active book
- **Verification:** Manual check; do not confirm a delete against non-disposable data

### AC-7: Scope and build are preserved

- **Given** the implementation is complete
- **When** `git diff --name-only` is reviewed and the primary validation command runs
- **Then** only the allowed paths changed, the User-Agent block is unchanged, and the build completes with exit status 0
- **Verification:** Record the diff file list, the command, its exit status and a concise result

## Non-functional requirements

| ID | Quality | Testable requirement | Verification |
| --- | --- | --- | --- |
| NFR-1 | Reliability | Malformed `showInactive` values never produce an error response. | AC-4 |
| NFR-2 | Security/privacy | The value is bound as a typed parameter and rendered only through Razor encoding; no new data is logged or exposed. | Diff review |
| NFR-3 | Maintainability | No new dependency, configuration, schema or file outside the allowed paths. | `git diff --name-only` |
| NFR-4 | Compatibility | The default response for `/Books` is unchanged. | AC-1 |

## Assumptions, dependencies, and risks

| Item | Type | Validation or mitigation |
| --- | --- | --- |
| The database is seeded with 6 active and 1 inactive book. | Assumption | Seeding runs only on an empty or recreated database, so an existing LocalDB may differ; AC-1 to AC-6 use relative criteria, and the tester records the data used. |
| MVC 5 model binding of a non-nullable `bool` tolerates invalid input without an exception. | Assumption | **Unverified.** AC-4 must confirm it. |
| A GET checkbox can post `true,false` if paired with a hidden field. | Risk | Use a plain checkbox with value `true`, or verify the binding; implementation-neutral. |
| Razor views are not compiled by the build, so the build can pass with a view error. | Risk | Confirmed: `MvcBuildViews` is `false` (`src/BookCatalog.Web/BookCatalog.Web.csproj:17`). The manual smoke check is therefore the only view check and is required. |
| Inactive books are visible to all users. | Decision | Confirmed by the maintainer in planning; no authentication or access control is added. |
| "First value wins" for repeated `showInactive` parameters. | Assumption | **Unverified** against MVC 5 binding, which may join repeated values as `true,false`; AC-4b must confirm it, and the implementation must make it hold. |
| No automated test project exists, so regressions rely on manual checks. | Risk | Accepted by the maintainer in planning; record manual evidence. |
| Msbuild, Web build tools, LocalDB and IIS Express are installed. | Dependency | The maintainer reported the restore and build succeeded locally; IIS Express and LocalDB were **not independently verified**. |

## Plan-mode review checkpoint

### Evidence

- **Current behavior:** `legacy-src/dotnet-bookcatalog/shared-legacy-app/src/BookCatalog.Web/Controllers/BooksController.cs:Index` filters on `IsActive`; `.../Views/Books/Index.cshtml` renders a Status badge; `.../Content/Site.css` defines `.badge-inactive`; `.../Models/ApplicationDbContext.cs:SeedBooks` seeds one inactive book.
- **Existing tests:** none found. `legacy-src/dotnet-bookcatalog/AGENTS.md` states there is no automated test project, and no test `.cs` files were found.
- **Baseline validation evidence:** `msbuild BookCatalog.sln -restore -p:Configuration=Release` — the maintainer reported it succeeded (self-reported; exit status and output were not captured here). An earlier run failed on a missing Web Applications targets file and was resolved by installing the missing Visual Studio component.

### Unresolved questions

- [x] Should inactive books be visible to all users? **Decision: yes.**
- [x] Outcome for repeated or combined `showInactive` values? **Decision: the first value is used** (see Edge cases and AC-4b).
- [x] Should the empty-state message mention the checkbox? **Decision: no.**
- [x] Is `MvcBuildViews` enabled? **Resolved: no** (`BookCatalog.Web.csproj:17`), so the manual smoke check is the only view check.
- [x] What evidence format is required for the manual checks, and who runs them? **Decision: confirmed by the maintainer.** The implementer runs the checks against a disposable LocalDB database and records, for each AC, the URL requested, the HTTP status, the book titles listed and whether each is marked inactive. The record goes in the pull request description; screenshots are optional. All planning questions are resolved. Approval is pending the human review below.

### Boundary and validation review

- [x] Evidence citations were checked against the repository.
- [x] Allowed paths include all anticipated implementation and test changes and
      nothing unrelated.
- [x] The primary validation command is safe, non-interactive, and runnable
      from the stated source root.
- [x] Acceptance criteria cover the required normal, failure, boundary, and
      compatibility behavior.
- [x] Exclusions prevent unrelated modernization or cleanup.

### Stop conditions

Stop and request human guidance if:

- repository evidence contradicts this specification;
- an unresolved question could materially change behavior or scope;
- satisfying a criterion requires a path outside the allowed paths, including `Content/Site.css`;
- the baseline or primary validation fails for a reason caused by the proposed
  change, or the command is missing, destructive, or irreproducible;
- msbuild, Web build tools, LocalDB or IIS Express is missing and prevents required evidence from being collected;
- a secret, credential, production or non-disposable database, destructive action (such as deleting or recreating the database or creating an empty MDF), dependency change, schema change, or public API change is required but not explicitly approved; or
- implementation or validation cannot satisfy the acceptance criteria without
  expanding the approved contract.

### Human approval

- [x] I reviewed the evidence, unresolved questions, allowed paths, acceptance
      criteria, validation command, exclusions, risks, and stop conditions.
- **Reviewer:** AaronArendt
- **Approval date:** 2026-11-23
- **Approved specification commit:** 9e892e7

Change **Status** to `Approved` only after the checkbox is selected and the
reviewer, date, and commit fields are complete. Planning approval authorizes
issue preparation, not implementation.
- 