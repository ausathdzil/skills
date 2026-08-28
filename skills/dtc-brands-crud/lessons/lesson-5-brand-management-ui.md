# Lesson 5: Brand Management UI

**Prerequisite:** Lesson 4 (update/delete routes exist and pass cURL checks).

**Goal:** Replace the read-only brands table with a manageable one: create, edit, delete, and product linking.

**Time:** 60-75 min

## What We're Building

```
Brands page (/app/brands) — DataTable with row actions
  ├─ [Create Brand] → FocusModal → POST /admin/brands
  ├─ Row → Edit → Drawer (prefilled) → POST /admin/brands/:id
  ├─ Row → Delete → Confirm → DELETE /admin/brands/:id
  └─ Row → View → /app/brands/:id → product list + Unlink + Link picker (DataTable selection)
```

**Medusa UI contract:** `FocusModal` for **create**, `Drawer` for **edit** — not interchangeable. This is linted (`form-focusmodal-create`, `form-drawer-edit`).

## Part 1: Create Brand (FocusModal)

### Why FocusModal?

Creation is a *new* entity, often with validation — FocusModal gives a titled header, form area, and footer actions, and traps focus.

### Implementation

In `src/admin/routes/brands/page.tsx`, add above the `DataTable`:

```tsx
import { useState } from "react"
import { useMutation, useQueryClient } from "@tanstack/react-query"
import { FocusModal, Button, Input, Label, toast } from "@medusajs/ui"
import { sdk } from "../../lib/sdk"

const BrandsPage = () => {
  const queryClient = useQueryClient()
  const [open, setOpen] = useState(false)
  const [name, setName] = useState("")

  const createBrand = useMutation({
    mutationFn: (payload: { name: string }) =>
      sdk.client.fetch(`/admin/brands`, { method: "POST", body: payload }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["brands"] })
      toast.success("Brand created")
      setOpen(false)
      setName("")
    },
    onError: (e: Error) => toast.error(e.message),
  })

  return (
    <>
      <FocusModal open={open} onOpenChange={setOpen}>
        <FocusModal.Trigger asChild>
          <Button size="small">Create Brand</Button>
        </FocusModal.Trigger>
        <FocusModal.Content>
          <FocusModal.Header>Create Brand</FocusModal.Header>
          <FocusModal.Body>
            <div className="flex flex-col gap-2">
              <Label size="small" weight="plus">Name</Label>
              <Input value={name} onChange={(e) => setName(e.target.value)} placeholder="Acme" />
            </div>
          </FocusModal.Body>
          <FocusModal.Footer>
            <Button variant="secondary" onClick={() => setOpen(false)}>Cancel</Button>
            <Button
              onClick={() => createBrand.mutate({ name })}
              isLoading={createBrand.isPending}
              disabled={!name.trim() || createBrand.isPending}
            >
              Create
            </Button>
          </FocusModal.Footer>
        </FocusModal.Content>
      </FocusModal>

      {/* ... existing DataTable ... */}
    </>
  )
}
```

**Teaching points:**

- Display query (`["brands", pageIndex]`) vs modal query — here creation has no fetch, but still invalidate the *display* key `["brands"]`, not a modal key.
- `isLoading`/`disabled` on the submit button (`form-disable-pending`, `form-show-loading`).
- `toast` for feedback — part of `@medusajs/ui`.

## Part 2: Edit Brand (Drawer)

### Why Drawer?

Editing is *contextual* — you stay on the page, see the row behind the overlay.

```tsx
import { Drawer } from "@medusajs/ui"

const [editBrand, setEditBrand] = useState<Brand | null>(null)
const [editName, setEditName] = useState("")

const updateBrand = useMutation({
  mutationFn: ({ id, name }: { id: string; name: string }) =>
    sdk.client.fetch(`/admin/brands/${id}`, { method: "POST", body: { name } }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["brands"] })
    toast.success("Brand updated")
    setEditBrand(null)
  },
})

<Drawer open={!!editBrand} onOpenChange={(o) => !o && setEditBrand(null)}>
  <Drawer.Content>
    <Drawer.Header><Heading>Edit Brand</Heading></Drawer.Header>
    <Drawer.Body>
      <Input value={editName} onChange={(e) => setEditName(e.target.value)} />
    </Drawer.Body>
    <Drawer.Footer>
      <Button variant="secondary" onClick={() => setEditBrand(null)}>Cancel</Button>
      <Button
        isLoading={updateBrand.isPending}
        disabled={updateBrand.isPending}
        onClick={() => editBrand && updateBrand.mutate({ id: editBrand.id, name: editName })}
      >
        Save
      </Button>
    </Drawer.Footer>
  </Drawer.Content>
</Drawer>
```

Wire row action:

```tsx
const columns = [
  // ... id, name, products
  columnHelper.display({
    id: "actions",
    header: "Actions",
    cell: ({ row }) => (
      <Button
        size="small"
        variant="transparent"
        onClick={() => {
          setEditBrand(row.original)
          setEditName(row.original.name)
        }}
      >
        Edit
      </Button>
    ),
  }),
]
```

## Part 3: Delete with Confirm

Use `Prompt` (or `AlertDialog`) pattern — don't delete on single click.

```tsx
import { Prompt } from "@medusajs/ui"

const deleteBrand = useMutation({
  mutationFn: (id: string) => sdk.client.fetch(`/admin/brands/${id}`, { method: "DELETE" }),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ["brands"] })
    toast.success("Brand deleted")
  },
})

<Prompt>
  <Prompt.Trigger asChild>
    <Button size="small" variant="danger">Delete</Button>
  </Prompt.Trigger>
  <Prompt.Content>
    <Prompt.Header>Delete brand?</Prompt.Header>
    <Prompt.Description>Products will be unlinked, not deleted.</Prompt.Description>
    <Prompt.Footer>
      <Prompt.Cancel>Cancel</Prompt.Cancel>
      <Prompt.Action onClick={() => deleteBrand.mutate(brand.id)}>Delete</Prompt.Action>
    </Prompt.Footer>
  </Prompt.Content>
</Prompt>
```

Explain to the learner: *why* products survive — soft-delete + link dismiss vs `deleteCascade`.

## Part 4 (Stretch): Brand Detail with Product Linking

Create `src/admin/routes/brands/[id]/page.tsx`:

- `useQuery(["brand", id], () => sdk.client.fetch(`/admin/brands/${id}`))` — display on mount
- Table of `brand.products` with **Unlink** (`dismissRemoteLinkStep` via a small `POST /admin/brands/:id/products/:productId/unlink` you add, or reuse `update` with productIds)
- **Link** button → `FocusModal` + `DataTable` selection pattern (`references/table-selection.md`): paginated product picker, `useDataTable` with `rowSelection`, on save call `POST /admin/products` hook path or a dedicated `/admin/brands/:id/link` route that uses `createRemoteLinkStep`

This is the "large dataset selection" pattern — keep display query (`brand + products`) separate from picker query (`products list`).

## Verification

- [ ] Create via FocusModal → table refreshes without reload
- [ ] Edit via Drawer → name changes, `updated_at` changes
- [ ] Delete → row removed, product's `?fields=+brand.*` now `brand: null`
- [ ] (Stretch) Detail page lists products, unlink works

## Common Pitfalls

- Using `Drawer` for create / `FocusModal` for edit — lint will flag `form-*`.
- Single `useQuery` for display + picker — display goes empty on refresh (`data-separate-queries`).
- Forgetting `queryClient.invalidateQueries({ queryKey: ["brands"] })` after mutations — table stale until manual refresh (`data-invalidate-display`).
- Raw `fetch()` instead of `sdk.client.fetch` — misses session cookie → 401 (`data-sdk-always`).
