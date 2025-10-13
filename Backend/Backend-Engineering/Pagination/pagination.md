Pagination — A Practical Guide (for Backend & Frontend)

Efficient pagination keeps your app fast, scalable, and pleasant to use. This guide explains when and how to paginate, shows offset vs cursor strategies with SQL + code examples (Spring Boot / Next.js), and lists gotchas, indexing tips, API contracts, and testing checklists you can paste into any project.

Why paginate?

Performance: Avoid sending thousands of rows per request.

Cost: Less DB work, less bandwidth.

UX: Faster first paint; users browse in small chunks.

Stability: Predictable request/response sizes.

Always paginate at the backend. Client-side-only paging after downloading huge datasets defeats the purpose.

API Response Shape (Example)
{
  "count": 35,
  "next": "http://localhost:8000/api/notes?page=2",
  "previous": null,
  "data": [
    {
      "id": 1,
      "collection_data": { "id": 2, "name": "test" },
      "title": "first note",
      "content": "notes",
      "collection": 2
    }
  ]
}


Fields

data: Array of items for this page.

count: Total items (optional—may be expensive).

next/previous: Absolute or relative links to fetch adjacent pages (optional if you return cursors/tokens instead).

Two Core Strategies
1) Offset-based Pagination (a.k.a. page/limit)

Request

GET /comments?page=2&limit=10


Backend logic

offset = (page - 1) * limit


SQL (generic)

SELECT *
FROM comments
ORDER BY id
LIMIT 10 OFFSET 10;  -- page=2, limit=10


Pros

Easy to implement & reason about

Works with arbitrary sorting (by name, price, etc.)

Easy to “jump” to page N

Cons

Slows down for deep pages (DB still scans/steps through skipped rows)

Inconsistent if new rows are inserted while the user is paginating (duplicates/missing rows)

SQL Server flavor

SELECT *
FROM comments
ORDER BY id
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;


Tip: For stable ordering, paginate on indexed columns and include a deterministic ORDER BY.

2) Cursor/Keyset Pagination (a.k.a. “seek” method)

Idea: “Give me the next N rows after this row,” using a stable, indexed column (ID or created_at).

Requirements

Column is unique, NOT NULL, monotonic/unchanging, and indexed
(e.g., (created_at, id) pair to break ties).

Request

GET /comments?after_id=12345&limit=25


SQL

SELECT *
FROM comments
WHERE id > 12345
ORDER BY id ASC
LIMIT 25;


With created_at + id (tie-breaker)

SELECT *
FROM comments
WHERE (created_at, id) > (:cursorCreatedAt, :cursorId)
ORDER BY created_at ASC, id ASC
LIMIT :limit;


Opaque/Continuation Token

Instead of exposing after_id=12345, return cursor=eyJhZnRlcl9pZCI6MTIzNDV9 (base64/JSON).

Prevents users from crafting invalid cursors; lets you encode multiple fields.

Pros

Very fast for deep pagination (no large offsets)

Resistant to inserts (no shifting windows)

Great for infinite scroll and “Load more”

Cons

Not ideal when users need random page jumps (page 7 directly)

A bit more complex to implement (cursors/tokens)

Choosing the Strategy (Quick Decision)

Need infinite scroll / load more? → Cursor

Need jump to page N from the URL or paginator control? → Offset

Dataset can grow/receive inserts while browsing? → Prefer Cursor

Arbitrary sorting (by various fields)? → Offset (or implement compound keyset if feasible)

API Contract Patterns
Offset API
GET /items?page=3&limit=20&sort=created_at:desc


Response

{
  "data": [...],
  "page": 3,
  "limit": 20,
  "count": 12345,
  "next": "/items?page=4&limit=20&sort=created_at:desc",
  "previous": "/items?page=2&limit=20&sort=created_at:desc"
}

Cursor API
GET /items?limit=20&cursor=eyJjcmVhdGVkQXQiOiIyMDI1LTEwLTE0VDA5OjAwOjAwWiIsImlkIjo0MjF9


Response

{
  "data": [...],
  "nextCursor": "eyJjcmVhdGVkQXQiOiIyMDI1LTEwLTE0VDA5OjAzOjE0WiIsImlkIjo0NjN9",
  "hasMore": true
}


Return hasMore or next/nextCursor. If hasMore=false and nextCursor=null, you’re at the end.

Backend Examples (Spring Boot)
Offset with Spring Data Pageable
@GetMapping("/comments")
public Page<Comment> list(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(defaultValue = "id,asc") String sort
) {
    String[] parts = sort.split(",");
    Sort s = Sort.by(Sort.Direction.fromString(parts[1]), parts[0]);
    Pageable pageable = PageRequest.of(page, size, s);
    return commentRepository.findAll(pageable);
}

Keyset (cursor) with created_at, id
@GetMapping("/comments")
public CursorResponse<CommentDto> list(
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") int limit
) {
    // decodeCursor -> (createdAt, id)
    Instant createdAt = Instant.EPOCH;
    long id = 0L;

    if (cursor != null) {
        var c = cursorCodec.decode(cursor);
        createdAt = c.createdAt();
        id = c.id();
    }

    // Fetch strictly "after" the cursor in ascending order
    var rows = repo.findNext(createdAt, id, limit + 1); // +1 to detect hasMore

    boolean hasMore = rows.size() > limit;
    var page = hasMore ? rows.subList(0, limit) : rows;

    String nextCursor = hasMore
        ? cursorCodec.encode(page.get(page.size() - 1).getCreatedAt(),
                             page.get(page.size() - 1).getId())
        : null;

    return new CursorResponse<>(toDtos(page), nextCursor, hasMore);
}


Repository (JPA / native)

@Query("""
SELECT c
FROM Comment c
WHERE (c.createdAt > :createdAt)
   OR (c.createdAt = :createdAt AND c.id > :id)
ORDER BY c.createdAt ASC, c.id ASC
""")
List<Comment> findNext(Instant createdAt, Long id, Pageable pageable);


Ensure an index on (created_at, id).

Frontend Examples (Next.js/React)
Offset (page buttons)
const [page, setPage] = useState(1);
const [items, setItems] = useState([]);

useEffect(() => {
  fetch(`/api/items?page=${page}&limit=20`)
    .then(r => r.json())
    .then(d => setItems(d.data));
}, [page]);

Cursor (infinite scroll with IntersectionObserver)
const [cursor, setCursor] = useState<string | null>(null);
const [items, setItems] = useState<any[]>([]);
const [hasMore, setHasMore] = useState(true);

const load = async () => {
  if (!hasMore) return;
  const url = cursor ? `/api/items?limit=20&cursor=${encodeURIComponent(cursor)}` : `/api/items?limit=20`;
  const res = await fetch(url);
  const json = await res.json();
  setItems(prev => {
    const seen = new Set(prev.map(x => x.id));
    const deduped = json.data.filter((x: any) => !seen.has(x.id));
    return [...prev, ...deduped];
  });
  setCursor(json.nextCursor ?? null);
  setHasMore(Boolean(json.nextCursor));
};

// attach to sentinel at the bottom
useEffect(() => {
  const obs = new IntersectionObserver(entries => {
    if (entries[0].isIntersecting) load();
  });
  const el = document.getElementById('sentinel');
  if (el) obs.observe(el);
  return () => obs.disconnect();
}, [cursor, hasMore]);

return (
  <>
    {/* render items */}
    <div id="sentinel" />
  </>
);

Indexing & SQL Tips

Always index columns in your ORDER BY and cursor conditions.

For cursor pagination with (created_at, id), create a composite index on that pair.

For offset pagination, prefer a covering index to reduce lookups (select fewer columns, join by primary key afterward).

Deep offsets are expensive; avoid OFFSET 100000 when possible (use cursor/seek).

For joins, paginate on the driving table after narrowing with filters, then join.

Handling Inserts/Updates/Deletes

Offset: New inserts at the top shift pages → duplicates or skips. Consider deduping on the client; show a “New items” banner and refresh.

Cursor: Stable as long as you paginate by a monotonic key; new rows won’t disrupt already-fetched windows.

Edits/Deletes: A previously fetched item may disappear or change. Be resilient client-side (gracefully remove/update items if refetched).

Sorting & Filtering

Offset supports arbitrary sort fields easily (?sort=name:asc).

Cursor requires the same ORDER BY field as the cursor key.
For complex sorts, consider compound cursors (e.g., (price, id)).

Counts: exact vs approximate

Returning count on every request can be expensive on large tables.

Options:

Make count optional (?include=count).

Cache counts periodically.

Return approximate counts for huge datasets.

Use window functions or specialized estimators (DB-specific).

Caching, HTTP, and Links

Support ETag / If-None-Match for unchanged pages.

Use Cache-Control for read-heavy endpoints.

Add Link headers
 for next/prev relations (nice for crawlers/clients).

Prefer idempotent GET for paging.

Client Patterns

Infinite scroll: Use cursor. Trigger next load on intersection; dedupe by ID.

Load more button: Same as infinite scroll without auto-trigger.

Classic paginator: Offset/page numbers. Provide First/Prev/Next/Last.

Deduping snippet

const mergeUnique = (prev: any[], next: any[]) => {
  const map = new Map(prev.map(x => [x.id, x]));
  next.forEach(x => map.set(x.id, x));
  return Array.from(map.values());
};

Error Handling & Edge Cases

Validate limit (max cap, e.g., 100).

Validate page ≥ 1; treat invalid values as 1 or return 400.

Invalid/expired cursor → return 400 with guidance; don’t crash.

Empty pages: return data: [], hasMore: false.

Last page: next/nextCursor is null.

SEO & Accessibility

If content needs to be indexable (e.g., product lists), provide:

Unique URLs for pages (/products?page=3).

Server-rendered or pre-rendered pages for crawlers.

Infinite scroll: ensure accessible keyboard navigation and ARIA live regions for new content.

Do’s & Don’ts

Do

Enforce sane limit caps.

Index your sort/cursor columns.

Keep ordering stable and deterministic.

Return next/previous or nextCursor and hasMore.

Don’t

Download all rows to the client and “paginate” there.

Mix different ORDER BY between requests when using cursors.

Use deep offsets for huge tables—switch to keyset.

Testing Checklist

✅ First page returns expected items in correct order.

✅ Middle and last pages behave correctly.

✅ Large page or invalid cursor returns 400/empty safely.

✅ limit boundaries (1 and max) covered.

✅ Concurrency: insert/delete during pagination doesn’t break order.

✅ Client deduping logic verified for infinite scroll.

Quick Reference: SQL Variants

PostgreSQL/MySQL

-- Offset
SELECT * FROM items ORDER BY id LIMIT :limit OFFSET :offset;

-- Keyset (id)
SELECT * FROM items WHERE id > :cursor ORDER BY id ASC LIMIT :limit;

-- Keyset (created_at, id)
SELECT *
FROM items
WHERE (created_at, id) > (:cursorCreatedAt, :cursorId)
ORDER BY created_at ASC, id ASC
LIMIT :limit;


SQL Server

-- Offset
SELECT *
FROM items
ORDER BY id
OFFSET @offset ROWS FETCH NEXT @limit ROWS ONLY;

-- Keyset (with created_at, id) — emulate pair comparison
SELECT TOP (@limit) *
FROM items
WHERE created_at > @cursorCreatedAt
   OR (created_at = @cursorCreatedAt AND id > @cursorId)
ORDER BY created_at ASC, id ASC;

Example cURL

Offset

curl "http://localhost:8080/api/comments?page=3&limit=20&sort=created_at:desc"


Cursor

curl "http://localhost:8080/api/comments?limit=20&cursor=eyJjcmVhdGVkQXQiOiIyMDI1LTEwLTE0VDA5OjAzOjE0WiIsImlkIjo0NjN9"

TL;DR

Use offset for simple pagers and arbitrary sorts (page 1,2,3…).

Use cursor/keyset for speed at scale + infinite scroll.

Always index your ORDER BY (and cursor) columns.

Keep API contracts consistent (next/prev or nextCursor/hasMore).

Be deliberate about count (omit, cache, or approximate on huge tables).