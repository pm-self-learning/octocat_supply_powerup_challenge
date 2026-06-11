---
name: frontend-performance-audit
description: "Audit a frontend route or component for known performance anti-patterns specific to this codebase. Use when: reviewing a new page or component before PR; investigating slow load times on a route; checking React Query configuration; verifying a component follows server-first data fetching patterns; identifying N+1 fetch patterns or unnecessary re-renders in the provider stack."
argument-hint: "Path to the file or route folder to audit (e.g. frontend/src/components/admin/AdminProducts.tsx)"
---

# Frontend Performance Audit

You are a senior frontend engineer auditing a React + Vite + Tailwind component or route in the OctoCAT Supply Chain Management app.

Report each finding with: **exact file and line**, **severity**, **what is wrong**, and a **concrete fix**.

---

## When to Use This Skill

Trigger when:

- Reviewing a new page or component before a PR merges
- Investigating slow load times on a route
- Checking React Query `staleTime` / caching configuration
- Verifying a component follows server-first data fetching patterns
- Identifying N+1 fetch patterns, unnecessary re-renders, or provider-stack bloat

---

## Execution

Follow these steps in order for every audit.

### Step 1 — Read the target file(s)

Read the file (or every file in the folder) passed as the argument. If no argument was given, ask the user for a path before continuing.

Also read:

- `frontend/src/api/config.ts` — base URL and endpoint map
- `frontend/src/context/AuthContext.tsx` and `frontend/src/context/ThemeContext.tsx` — shared context shape and re-render surface

### Step 2 — N+1 Fetch Detection

Look for loops, `Promise.all`, or `map` calls that fire one HTTP request **per item** when a single batched or joined request would suffice.

**Flag immediately if:**

- A `useEffect` or `useQuery` fetches a list, then iterates over results making one `axios.get` per row (e.g., fetching `/suppliers/:id` for every product)
- A child component triggers its own fetch using a prop ID, and that component is rendered in a list

**Preferred fix:** fetch the full joined resource from the API in a single call, or use a single `useQuery` with a batched endpoint. Example pattern seen in this codebase:

```tsx
// ❌ N+1 anti-pattern (AdminProducts.tsx)
const productsWithSuppliers = await Promise.all(
  productsData.map(async (product) => {
    const supplierResponse = await axios.get(
      `${api.baseURL}/suppliers/${product.supplierId}`,
    );
    return { ...product, supplier: supplierResponse.data };
  }),
);

// ✅ Preferred: single endpoint that returns joined data, or two parallel queries
const [products, suppliers] = await Promise.all([
  axios.get(`${api.baseURL}/products`),
  axios.get(`${api.baseURL}/suppliers`),
]);
```

### Step 3 — React Query Adoption

Check every component that fetches data. Bare `useEffect` + `useState` for server data is a red flag in this codebase.

**Flag if:**

- Data is fetched inside `useEffect` without React Query
- Loading and error states are managed with separate `useState` variables
- There is no `staleTime` set (default `0` means every focus/mount refetches)
- `queryKey` arrays do not include dynamic params the query depends on

**Preferred fix:**

```tsx
// ❌ Ad-hoc pattern
const [products, setProducts] = useState([]);
useEffect(() => {
  axios.get(...).then(res => setProducts(res.data));
}, []);

// ✅ React Query pattern
const { data: products, isPending, error } = useQuery({
  queryKey: ['products'],
  queryFn: () => axios.get(`${api.baseURL}/products`).then(r => r.data),
  staleTime: 60_000,
});
```

### Step 4 — Re-render & Provider Stack

Check `App.tsx` and any route wrapper for:

- Context providers wrapping too wide a subtree (whole app) when only a narrow subtree needs the value
- Context values that are recreated on every render (object or function literals inline in the `value` prop)
- Components consuming context but only needing a slice of it — these will re-render on any context change

**Flag if:**

- `value={{ ... }}` or `value={[state, setter]}` is written inline in a context `Provider` without `useMemo`
- A list item component reads from a heavy context (e.g. `ThemeContext`) on every render

### Step 5 — Component Size & Mixing Concerns

Check for components that:

- Exceed ~150 LOC and mix data fetching + complex layout + formatting logic
- Declare local interface types that duplicate types already defined elsewhere
- Contain commented-out `TODO` blocks for core functionality (e.g., cart)

Flag each violation and suggest the extraction boundary (e.g., "extract `SupplierBadge` from `AdminProducts`").

### Step 6 — Lazy Loading & Bundle Size

Check the route definitions in `App.tsx`:

- Are all routes eagerly imported at the top of the file?
- Are any large admin or rarely-visited views loaded lazily with `React.lazy` + `Suspense`?

**Flag if:** Admin routes (e.g., `/admin/products`) are eagerly bundled into the main chunk.

**Preferred fix:**

```tsx
const AdminProducts = React.lazy(
  () => import("./components/admin/AdminProducts"),
);
// wrap route element in <Suspense fallback={<Spinner />}>
```

### Step 7 — Loop & Mutation Safety

Scan for `for` loops that mutate state or arrays. Check loop direction and termination conditions — an off-by-one or wrong increment direction can create an infinite loop or silently corrupt data.

**Flag immediately if:**

- A `for` loop uses `i++` when iterating downward (causing infinite loop)
- An array index is used without bounds checking
- State is mutated directly inside a loop rather than producing a new array

### Step 8 — Self-Verification

For every finding, re-read the relevant lines and confirm:

1. Is the issue genuinely present, not already guarded by a condition or abstraction above?
2. Is the proposed fix compatible with the existing TypeScript types?
3. Does the fix preserve the component's current public API (props, exports)?

Discard or downgrade any finding that does not survive this check.

---

## Severity Guide

| Severity  | Meaning                                                 | Example                                             |
| --------- | ------------------------------------------------------- | --------------------------------------------------- |
| 🔴 HIGH   | Causes measurable slowdown, crashes, or data corruption | N+1 fetch in a list, infinite loop                  |
| 🟠 MEDIUM | Degrades UX or causes avoidable re-renders              | Missing `staleTime`, no React Query adoption        |
| 🟡 LOW    | Structural debt or missed optimisation                  | Large component, eager bundle, inline context value |
| ⚪ INFO   | Worth noting, no direct perf impact                     | Duplicate type declaration, `TODO` comment          |

---

## Output Format

Produce a **findings table** first:

| #   | File                | Line(s) | Severity | Issue                                   |
| --- | ------------------- | ------- | -------- | --------------------------------------- |
| 1   | `AdminProducts.tsx` | 52–64   | 🔴 HIGH  | N+1 supplier fetch inside `Promise.all` |

Then for each finding, a **finding card**:

```
### Finding 1 — N+1 Supplier Fetch
File: frontend/src/components/admin/AdminProducts.tsx  Lines: 52–64
Severity: 🔴 HIGH

What: One HTTP request is made per product to fetch its supplier. With 100 products this fires 100 + 1 requests on every mount.

Fix: Replace with two parallel top-level queries (products + suppliers) and join client-side by supplierId, or request a joined endpoint from the API.

Before:
  [show the current code snippet]

After:
  [show the proposed replacement]
```

Close with a **summary sentence**: "X findings: Y HIGH, Z MEDIUM, W LOW."

If no issues are found, state clearly: "No performance anti-patterns detected in `<path>`." and list what was checked.

---

## Output Rules

- Always produce the findings table before the cards
- Never auto-apply any changes — present fixes for human review only
- Include exact file path and line numbers for every finding
- Explain the _impact_ in plain English (what will a user actually experience?)
- Fixes must be TypeScript-compatible and preserve existing prop/export contracts
