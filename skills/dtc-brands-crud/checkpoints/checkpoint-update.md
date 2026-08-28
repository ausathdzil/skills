# Checkpoint: Update Brand

**Goal:** `POST /admin/brands/:id` renames a brand, validates uniqueness, and preserves `updated_at`.

**Verify:**

1. Build passes: `pnpm run build` (from `apps/backend`)
2. cURL:

```bash
curl -X POST http://localhost:9000/admin/brands/<id> \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Acme Co."}'
# → { brand: { name:"Acme Co." } }

curl http://localhost:9000/admin/brands/<id> -H "Authorization: Bearer $TOKEN"
# → name is "Acme Co."
```

3. Duplicate name → error (your uniqueness check throws)
