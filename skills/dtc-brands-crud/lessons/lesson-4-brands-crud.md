# Lesson 4: Complete Brands CRUD — Backend

**Prerequisite:** `learning-medusa` 1-3 done (Brand Module, link, `GET /admin/brands` with `products.*`). This lesson turns the skeleton into full CRUD.

**Goal:** `POST /admin/brands/:id` (rename) and `DELETE /admin/brands/:id` (soft-delete + unlink)

**Time:** 60-75 min

## Architecture: Updates are POSTs, Deletes Unlink

Medusa's HTTP contract is `GET`/`POST`/`DELETE` only — never `PUT`/`PATCH`. An update is:

```
POST /admin/brands/:id  { name: "New name" }
```

```
┌──────────────────────────────┐
│  POST /admin/brands/:id      │  validate id + body
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│  updateBrandWorkflow         │  retrieve → validate unique → update
│    updateBrandStep           │  StepResponse(brand, {id, previousName})
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│  Brand Module                │  updateBrands({id, name})
└──────────────────────────────┘
```

Delete must also clean the pivot table:

```
DELETE /admin/brands/:id
  → deleteBrandWorkflow
    1. retrieveBrand (404 if missing)
    2. dismissRemoteLinkStep (remove product_brand rows)
    3. softDeleteBrands (sets deleted_at, keeps row)
```

Why soft-delete? Brands may be referenced in order history; hard delete would orphan. Companion vs `deleteCascade` on the link: we do explicit `dismiss` so products survive — `deleteCascade: true` would *delete* linked products.

## Part 1: Update Brand

### Step 4.1: updateBrandStep

Create `src/workflows/steps/update-brand.ts`:

```typescript
import { createStep, StepResponse } from "@medusajs/framework/workflows-sdk"
import { BRAND_MODULE } from "../../modules/brand"
import BrandModuleService from "../../modules/brand/service"

export type UpdateBrandStepInput = {
  id: string
  name: string
}

export const updateBrandStep = createStep(
  "update-brand",
  async ({ id, name }: UpdateBrandStepInput, { container }) => {
    const brandService: BrandModuleService = container.resolve(BRAND_MODULE)

    const existing = await brandService.retrieveBrand(id)
    const previousName = existing.name

    // Optional: uniqueness check
    const [sameName] = await brandService.listBrands({ name })
    if (sameName && sameName.id !== id) {
      throw new Error(`Brand name "${name}" already exists`)
    }

    const updated = await brandService.updateBrands({ id, name })

    return new StepResponse(updated, { id, previousName })
  },
  async ({ id, previousName }, { container }) => {
    if (!id || !previousName) return
    const brandService: BrandModuleService = container.resolve(BRAND_MODULE)
    await brandService.updateBrands({ id, name: previousName })
  }
)
```

Compensation restores `previousName` — note we capture it *before* the update so rollback has it. Second arg of `StepResponse` is the undo payload.

### Step 4.2: updateBrandWorkflow

Create `src/workflows/update-brand.ts`:

```typescript
import { createWorkflow, WorkflowResponse } from "@medusajs/framework/workflows-sdk"
import { updateBrandStep } from "./steps/update-brand"

export type UpdateBrandWorkflowInput = { id: string; name: string }

export const updateBrandWorkflow = createWorkflow(
  "update-brand",
  function (input: UpdateBrandWorkflowInput) {
    const brand = updateBrandStep(input)
    return new WorkflowResponse(brand)
  }
)
```

Same load-time graph rules as Lesson 1: `function`, no `await`.

### Step 4.3: Route + validation

Create `src/api/admin/brands/[id]/validators.ts`:

```typescript
import { z } from "@medusajs/framework/zod"
export const PostAdminUpdateBrand = z.object({ name: z.string().min(1) })
export type PostAdminUpdateBrandType = z.infer<typeof PostAdminUpdateBrand>
```

Update `src/api/admin/brands/[id]/route.ts` — keep your `GET`, add `POST`:

```typescript
export const POST = async (
  req: AuthenticatedMedusaRequest<PostAdminUpdateBrandType>,
  res: MedusaResponse
) => {
  const { result } = await updateBrandWorkflow(req.scope).run({
    input: { id: req.params.id, ...req.validatedBody },
  })
  res.json({ brand: result })
}
```

Extend `src/api/middlewares.ts`:

```typescript
{
  matcher: "/admin/brands/:id",
  method: "POST",
  middlewares: [validateAndTransformBody(PostAdminUpdateBrand)],
},
```

**Test:**

```bash
curl -X POST http://localhost:9000/admin/brands/<id> \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Acme Co."}'
# → { brand: { name:"Acme Co.", updated_at: ... } }
```

Checkpoint: update succeeds, duplicate name → 400/500 with your error, `updated_at` changes, rerun `GET` shows new name.

## Part 2: Delete Brand (Soft-Delete + Unlink)

### Step 4.4: deleteBrandStep + workflow

Create `src/workflows/steps/delete-brand.ts`:

```typescript
import { createStep, StepResponse } from "@medusajs/framework/workflows-sdk"
import { BRAND_MODULE } from "../../modules/brand"
import BrandModuleService from "../../modules/brand/service"

export const deleteBrandStep = createStep(
  "delete-brand",
  async ({ id }: { id: string }, { container }) => {
    const brandService: BrandModuleService = container.resolve(BRAND_MODULE)
    const brand = await brandService.retrieveBrand(id)
    await brandService.softDeleteBrands(id)
    return new StepResponse(brand, brand)
  },
  async (brand, { container }) => {
    if (!brand?.id) return
    const brandService: BrandModuleService = container.resolve(BRAND_MODULE)
    await brandService.restoreBrands(brand.id)
  }
)
```

Create `src/workflows/delete-brand.ts`:

```typescript
import { createWorkflow, WorkflowResponse, transform } from "@medusajs/framework/workflows-sdk"
import { deleteBrandStep } from "./steps/delete-brand"
import { dismissRemoteLinkStep } from "@medusajs/medusa/core-flows"
import { Modules } from "@medusajs/framework/utils"
import { BRAND_MODULE } from "../modules/brand"

export const deleteBrandWorkflow = createWorkflow(
  "delete-brand",
  function (input: { id: string }) {
    // 1. soft-delete (and capture brand for compensation)
    const brand = deleteBrandStep(input)

    // 2. dismiss links — need brand_id + all linked product_ids
    // Use transform to build the link payload from the brand
    const links = transform({ brand }, ({ brand }) => {
      // brand.products may be needed — fetch via query or pass productIds in input
      // For simplicity, dismiss by brand_id alone if your Medusa version supports it,
      // otherwise query linked products first in a step.
      return []
    })

    // If you have productIds, then:
    // dismissRemoteLinkStep(links)

    return new WorkflowResponse(brand)
  }
)
```

> **Teaching note:** Dismissing links in a workflow is the canonical place to introduce `transform` and `dismissRemoteLinkStep`. The minimal viable version soft-deletes without dismiss and relies on the link rows being ignored (soft-deleted brand → `deleted_at` set → queries filter it). The thorough version adds a preceding `retrieveBrandWithProductsStep` that returns `{ brand, productIds }` so `dismissRemoteLinkStep` can remove pivot rows. Choose one and explain the tradeoff to the learner.

### Step 4.5: DELETE route

Add to `src/api/admin/brands/[id]/route.ts`:

```typescript
export const DELETE = async (req: AuthenticatedMedusaRequest, res: MedusaResponse) => {
  const { result } = await deleteBrandWorkflow(req.scope).run({
    input: { id: req.params.id },
  })
  res.json({ brand: result })
}
```

No body validation needed. Optionally add `validateAndTransformQuery` for `:id` if you want.

**Test:**

```bash
curl -X DELETE http://localhost:9000/admin/brands/<id> \
  -H "Authorization: Bearer $TOKEN"
# → { brand: { id, deleted_at: ... } }

curl http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"
# → 404 (soft-deleted)

curl "http://localhost:9000/admin/products/<prod>?fields=+brand.*" \
  -H "Authorization: Bearer $TOKEN"
# → { product: { ..., brand: null } } — unlinked
```

## Common Pitfalls

- Using `PUT` — Medusa lint `arch-http-methods` will fail; always `POST` to `:id`.
- Forgetting `validateAndTransformBody` on the update route → `req.validatedBody` is empty.
- Hard-deleting (`deleteBrands`) vs soft-deleting (`softDeleteBrands`) — prefer soft for auditability.
- Dismiss order must match `defineLink(product, brand)` — product first, brand second.

## Checkpoint 4

- [ ] `POST /admin/brands/:id` renames, validates uniqueness
- [ ] `DELETE` soft-deletes, product query shows `brand: null`
- [ ] Build passes (`pnpm run build`)
