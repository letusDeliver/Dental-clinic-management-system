# Module 6 — Inventory / Supplies

## Objective
Track clinic consumables (gloves, masks, dental materials, syringes,
cleaning supplies) at a quantity level with movement history and
low-stock visibility, sized appropriately for a single small clinic — not
a full warehouse-management system.

## Scope
- Item catalogue with category.
- Current stock quantity per item.
- Stock movements (in/out) with reason.
- Supplier reference (simple, not a full procurement workflow).
- Low-stock indicator.

## Dependencies
- `foundation.md`, `staff-auth.md` (roles/authz — COMPOUNDER manages,
  ADMIN full, DOCTOR/RECEPTIONIST read-only per the permission matrix).

## Entities / Data Model
Quantity-based (not batch/expiry-tracked in V1 — see Out of Scope).
```prisma
model InventoryCategory {
  id   String @id @default(uuid())
  name String @unique

  @@map("inventory_categories")
}

model Supplier {
  id    String  @id @default(uuid())
  name  String
  phone String?
  email String?

  @@map("suppliers")
}

model InventoryItem {
  id           String            @id @default(uuid())
  name         String
  categoryId   String            @map("category_id")
  unit         String            // e.g. "box", "piece", "liter"
  quantity     Int               @default(0)
  reorderLevel Int               @default(0) @map("reorder_level")
  supplierId   String?           @map("supplier_id")
  isActive     Boolean           @default(true) @map("is_active")
  createdAt    DateTime          @default(now()) @map("created_at")
  updatedAt    DateTime          @updatedAt @map("updated_at")

  @@map("inventory_items")
}

enum StockMovementType {
  IN
  OUT
  ADJUSTMENT
}

model StockMovement {
  id        String            @id @default(uuid())
  itemId    String            @map("item_id")
  type      StockMovementType
  quantity  Int               // always positive; type determines direction
  reason    String?
  loggedBy  String            @map("logged_by") // Staff.id
  createdAt DateTime          @default(now()) @map("created_at")

  @@map("stock_movements")
}
```

## Relationships
`InventoryItem.categoryId` → `InventoryCategory.id`; `.supplierId` →
`Supplier.id`; `StockMovement.itemId` → `InventoryItem.id`;
`.loggedBy` → `Staff.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/inventory/items` | any authenticated staff | List items with current quantity + low-stock flag |
| POST | `/api/v1/inventory/items` | ADMIN/COMPOUNDER | Create item |
| PATCH | `/api/v1/inventory/items/:id` | ADMIN/COMPOUNDER | Update item (reorder level, supplier, active flag) |
| POST | `/api/v1/inventory/items/:id/movements` | ADMIN/COMPOUNDER | Log a stock movement (in/out/adjustment) |
| GET | `/api/v1/inventory/items/:id/movements` | ADMIN/COMPOUNDER | Movement history for an item |
| GET | `/api/v1/inventory/suppliers` | ADMIN/COMPOUNDER | List suppliers |
| POST | `/api/v1/inventory/suppliers` | ADMIN | Create supplier |

## Business Rules
- `InventoryItem.quantity` is a maintained running total, updated
  transactionally whenever a `StockMovement` is logged — not recomputed
  from history on every read (for read performance), but must always be
  kept consistent with the sum of movements.
- An `OUT` or negative `ADJUSTMENT` movement that would drop `quantity`
  below zero is rejected (`422`) — no negative stock.
- `quantity <= reorderLevel` marks the item low-stock in list responses
  (`data[].isLowStock: boolean`), computed at read time.
- Deactivating an item (`isActive=false`) hides it from default lists but
  preserves movement history.

## State Machines
Not applicable — items don't have a lifecycle beyond active/inactive.

## Security / Privacy / Compliance
No PII involved. Standard staff authentication/authorization only; no
special compliance controls beyond the permission matrix.

## Transactions / Concurrency
Logging a movement + updating `InventoryItem.quantity` must be one
transaction to avoid lost updates under concurrent movement logging on the
same item (e.g. `UPDATE ... SET quantity = quantity + :delta` inside the
transaction, not read-modify-write in application code).

## Edge Cases
- Concurrent `OUT` movements on the same item that would jointly overdraw
  stock → the transactional decrement + a `CHECK`-style application
  validation (or a DB check constraint `quantity >= 0`) ensures only the
  movements that keep quantity non-negative succeed; the rest get `422`.
- Logging a movement for an inactive item → `422`.

## Acceptance Criteria
- Stock quantity is always consistent with the sum of its movements.
- Negative stock is never reachable via the API.
- Low-stock items are correctly flagged based on `reorderLevel`.
- DOCTOR/RECEPTIONIST can read but not create items or movements.

## Testing Requirements
- Unit: low-stock flag computation.
- Integration: concurrent movement logging on the same item does not
  produce negative or inconsistent quantity.
- Integration: role-based access sweep.

## Out of Scope
- Batch/lot tracking.
- Expiry-date tracking and expiry alerts (add only if the clinic
  confirms it's needed for specific item types — do not build
  speculatively).
- Purchase orders / procurement workflow beyond a simple supplier
  reference field.
- Barcode scanning integration.

## Cross-Module Contracts
Published for later modules:
- `InventoryItem.id`, `.quantity`, `.reorderLevel` — used by
  `reporting.md` for inventory-usage reports.

## Implementation Notes
Keep this module intentionally simple — it exists to answer "do we have
enough gloves," not to run a supply chain.
