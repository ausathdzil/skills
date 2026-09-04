# Checkpoint: Delete Brand

**Goal:** `DELETE /admin/brands/:id` soft-deletes and unlinks.

**Verify:**

1. `curl -X DELETE http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"` → `{ brand: {...} }` (pre-delete snapshot; `deleted_at` reads null — expected)
2. `curl http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"` → 404 (the real proof of soft-delete)
3. `curl "http://localhost:9000/admin/products/<prod>?fields=+brand.*" -H "Authorization: Bearer $TOKEN"` → product has NO `brand` key (omission, not `null` — compare against a never-branded product)
4. Pivot row soft-deleted: `SELECT (deleted_at IS NOT NULL) FROM product_product_brand_brand WHERE brand_id='<id>'` → `t` (`dismiss` soft-deletes; `count(*)` without the filter lies)
5. Build passes
