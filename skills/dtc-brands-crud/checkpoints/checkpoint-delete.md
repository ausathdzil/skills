# Checkpoint: Delete Brand

**Goal:** `DELETE /admin/brands/:id` soft-deletes and unlinks.

**Verify:**

1. `curl -X DELETE http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"` → `{ brand: { deleted_at: ... } }`
2. `curl http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"` → 404
3. `curl "http://localhost:9000/admin/products/<prod>?fields=+brand.*" -H "Authorization: Bearer $TOKEN"` → `{ product: { ..., brand: null } }`
4. Build passes
