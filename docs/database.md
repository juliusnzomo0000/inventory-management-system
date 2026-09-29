# Inventory Management System — Database Design

## 1. Purpose of the Database

The database serves as the persistent storage layer for the Inventory Management System, a modular monolith application built with FastAPI (backend), PostgreSQL (database), and React/TypeScript (frontend). The database is designed to:

- Maintain an accurate, auditable record of inventory movements
- Support role-based authorization and user management
- Track products, suppliers, categories, and their relationships
- Record purchases, sales/issues, and associated line items
- Calculate current stock through transactional history rather than manual updates
- Support audit logging for compliance and accountability

---

## 2. Design Principles

### 2.1 Inventory as Transaction History (Primary)

Principle: Current inventory is derived from historical transactions, not stored as a single mutable value.

- The database records explicit inventory transactions (receipts, issues, adjustments) as the authoritative source of truth.
- Current stock is calculated as: `Current Stock = Sum of all valid inventory transactions for a product`
- This ensures auditability, prevents silent data loss, and makes inventory fully reconstructible.

### 2.2 Immutability of Transactional Records

Principle: Historical transactions are append-only; they are not silently modified or deleted.

- Inventory transactions, purchase records, and sales records are created once and retained for auditing.
- If a transaction must be reversed or corrected, a new adjusting transaction is created rather than deleting or modifying the original.
- This maintains a clear audit trail and prevents loss of historical context.

### 2.3 Soft Deletion for Active Entities

Principle: Products and categories with historical activity are deactivated rather than deleted.

- Entities are marked as inactive via an `is_active` flag rather than physically removed.
- This preserves referential integrity for historical records and enables searching or reporting on past activity.

### 2.4 Normalization and Consistency

Principle: The schema is normalized to 3NF to reduce redundancy and maintain consistency.

- Product and supplier information is stored once and referenced via foreign keys.
- Purchase and sale headers are separated from line items for flexibility and clarity.
- Denormalization for performance is introduced only where justified and with careful documentation.

### 2.5 Application-Level Authorization

Principle: The database records role and permission information; enforcement occurs in the application layer.

- Users are assigned to roles; roles define permissions.
- The application layer enforces authorization checks before allowing operations.
- The database ensures data consistency but does not implement complex authorization rules via triggers.

### 2.6 Audit Trail Completeness

Principle: Important business actions are logged with context.

- Each audit record identifies who performed an action, what happened, when, and the affected entity.
- Changes to sensitive or important data are captured with before/after values.

---

## 3. Core Tables

### 3.1 users

Purpose: Stores user accounts and authentication credentials.

Important columns:
- `id` (BIGSERIAL, PK)
- `username` (VARCHAR(100), UNIQUE, NOT NULL)
- `email` (VARCHAR(255), UNIQUE, NOT NULL)
- `password_hash` (VARCHAR(255), NOT NULL)
- `first_name` (VARCHAR(100), NOT NULL)
- `last_name` (VARCHAR(100), NOT NULL)
- `is_active` (BOOLEAN, NOT NULL, DEFAULT TRUE)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys: None directly in this table.

Important relationships: `users` is linked to `roles` through `user_roles`, and to `audit_logs`, `purchases`, `sales`, and `inventory_transactions` via `created_by` / `user_id` references.

Constraints and uniqueness:
- `username` unique
- `email` unique

Notes:
- Passwords are securely hashed before storage.
- `is_active` supports soft-deletion of users.

---

### 3.2 roles

Purpose: Stores role definitions.

Important columns:
- `id` (BIGSERIAL, PK)
- `name` (VARCHAR(100), UNIQUE, NOT NULL)
- `description` (TEXT)
- `permissions` (TEXT or JSONB, NOT NULL)
- `is_active` (BOOLEAN, NOT NULL, DEFAULT TRUE)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Unique constraints:
- `name` unique

Notes:
- Roles determine what operations a user is allowed to perform.
- The application enforces authorization; the database stores the role metadata.

---

### 3.3 user_roles

Purpose: Maps users to roles (many-to-many relationship).

Important columns:
- `id` (BIGSERIAL, PK)
- `user_id` (BIGINT, FK → users.id)
- `role_id` (BIGINT, FK → roles.id)
- `assigned_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `user_id` → `users.id`
- `role_id` → `roles.id`

Unique constraints:
- `UNIQUE (user_id, role_id)`

Cardinality:
- Many users can have many roles.

---

### 3.4 categories

Purpose: Organizes products into logical groups.

Important columns:
- `id` (BIGSERIAL, PK)
- `name` (VARCHAR(200), UNIQUE, NOT NULL)
- `description` (TEXT)
- `is_active` (BOOLEAN, NOT NULL, DEFAULT TRUE)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Unique constraints:
- `name` unique

Notes:
- Categories should be deactivated rather than physically deleted if historical records depend on them.

---

### 3.5 products

Purpose: Stores product definitions and attributes.

Important columns:
- `id` (BIGSERIAL, PK)
- `sku` (VARCHAR(50), UNIQUE, NOT NULL)
- `name` (VARCHAR(255), NOT NULL)
- `description` (TEXT)
- `category_id` (BIGINT, FK → categories.id)
- `unit` (VARCHAR(50), NOT NULL)
- `cost_price` (NUMERIC(12,2), NOT NULL)
- `selling_price` (NUMERIC(12,2))
- `reorder_level` (BIGINT, NOT NULL, DEFAULT 0)
- `is_active` (BOOLEAN, NOT NULL, DEFAULT TRUE)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `category_id` → `categories.id`

Unique constraints:
- `sku` unique

Important relationships:
- Products can appear in many `purchase_items`
- Products can appear in many `sale_items`
- Products can have many `inventory_transactions`
- Products can have many `stock_adjustments`

Notes:
- SKU uniqueness is a mandatory business rule.
- Products with historical activity should normally be deactivated rather than deleted.

---

### 3.6 suppliers

Purpose: Stores supplier information.

Important columns:
- `id` (BIGSERIAL, PK)
- `name` (VARCHAR(255), UNIQUE, NOT NULL)
- `contact_person` (VARCHAR(200))
- `phone` (VARCHAR(20))
- `email` (VARCHAR(255))
- `address` (TEXT)
- `notes` (TEXT)
- `is_active` (BOOLEAN, NOT NULL, DEFAULT TRUE)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Unique constraints:
- `name` unique

Notes:
- Suppliers are deactivated instead of removed to preserve purchase history.

---

### 3.7 purchases

Purpose: Stores purchase header information.

Important columns:
- `id` (BIGSERIAL, PK)
- `purchase_number` (VARCHAR(50), UNIQUE, NOT NULL)
- `supplier_id` (BIGINT, FK → suppliers.id)
- `purchase_date` (DATE, NOT NULL)
- `reference` (VARCHAR(255))
- `notes` (TEXT)
- `status` (VARCHAR(50), NOT NULL, DEFAULT 'pending')
- `created_by` (BIGINT, FK → users.id)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `supplier_id` → `suppliers.id`
- `created_by` → `users.id`

Unique constraints:
- `purchase_number` unique

Cardinality:
- One purchase belongs to one supplier.
- One purchase has many purchase items.

Notes:
- Purchase header stores global details; line items are separate in `purchase_items`.
- When stock is received, it must create corresponding inventory transactions.

---

### 3.8 purchase_items

Purpose: Stores products and quantities included in a purchase.

Important columns:
- `id` (BIGSERIAL, PK)
- `purchase_id` (BIGINT, FK → purchases.id)
- `product_id` (BIGINT, FK → products.id)
- `ordered_quantity` (BIGINT, NOT NULL)
- `unit_cost` (NUMERIC(12,2), NOT NULL)
- `received_quantity` (BIGINT, NOT NULL, DEFAULT 0)
- `notes` (TEXT)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `purchase_id` → `purchases.id`
- `product_id` → `products.id`

Important constraints:
- `ordered_quantity > 0`
- `received_quantity >= 0`
- `received_quantity <= ordered_quantity`

Notes:
- `received_quantity` is used to track partial receiving.
- Purchase items should be immutable in a historical sense, or only updated under controlled business rules.

---

### 3.9 sales

Purpose: Stores sale or stock issue header information.

Important columns:
- `id` (BIGSERIAL, PK)
- `sale_number` (VARCHAR(50), UNIQUE, NOT NULL)
- `sale_date` (DATE, NOT NULL)
- `sale_type` (VARCHAR(50), NOT NULL)
- `reference` (VARCHAR(255))
- `notes` (TEXT)
- `status` (VARCHAR(50), NOT NULL, DEFAULT 'completed')
- `created_by` (BIGINT, FK → users.id)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `created_by` → `users.id`

Unique constraints:
- `sale_number` unique

Cardinality:
- One sale has many sale items.

Notes:
- `sale_type` distinguishes between a sale and a stock issue.
- Sales/issues must reduce inventory and create corresponding inventory transactions.

---

### 3.10 sale_items

Purpose: Stores each product in a sale or issue.

Important columns:
- `id` (BIGSERIAL, PK)
- `sale_id` (BIGINT, FK → sales.id)
- `product_id` (BIGINT, FK → products.id)
- `quantity` (BIGINT, NOT NULL)
- `unit_price` (NUMERIC(12,2))
- `notes` (TEXT)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `sale_id` → `sales.id`
- `product_id` → `products.id`

Important constraints:
- `quantity > 0`

Notes:
- `unit_price` may be null for internal stock issues where no pricing is applied.
- Sale items must not result in negative available stock.

---

### 3.11 inventory_transactions

Purpose: Authoritative record of every inventory movement.

Important columns:
- `id` (BIGSERIAL, PK)
- `product_id` (BIGINT, FK → products.id)
- `transaction_type` (VARCHAR(50), NOT NULL)
- `quantity_change` (BIGINT, NOT NULL)
- `reference_type` (VARCHAR(50))
- `reference_id` (BIGINT)
- `notes` (TEXT)
- `created_by` (BIGINT, FK → users.id)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `product_id` → `products.id`
- `created_by` → `users.id`

Important constraints:
- `quantity_change <> 0`

Cardinality:
- Each product can have many transactions.

Importance:
- This table is the primary historical record of stock movement.
- Current stock should not be treated as a manually editable source of truth.
- It records all stock receipts, issues, and adjustments.

Notes:
- Positive values represent stock received or increases.
- Negative values represent stock issued or decreases.
- Historical transactions should never be silently deleted or modified.

---

### 3.12 stock_adjustments

Purpose: Represents a manual or controlled change in stock quantity.

Important columns:
- `id` (BIGSERIAL, PK)
- `adjustment_number` (VARCHAR(50), UNIQUE, NOT NULL)
- `product_id` (BIGINT, FK → products.id)
- `adjustment_type` (VARCHAR(50), NOT NULL)
- `quantity` (BIGINT, NOT NULL)
- `reason` (VARCHAR(255), NOT NULL)
- `notes` (TEXT)
- `created_by` (BIGINT, FK → users.id)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `product_id` → `products.id`
- `created_by` → `users.id`

Unique constraints:
- `adjustment_number` unique

Important constraints:
- `quantity > 0`

Notes:
- Every adjustment should create a corresponding `inventory_transaction` record.
- `adjustment_type` is usually `'increase'` or `'decrease'`.

---

### 3.13 audit_logs

Purpose: Stores records of meaningful actions for auditing and traceability.

Important columns:
- `id` (BIGSERIAL, PK)
- `user_id` (BIGINT, FK → users.id, nullable)
- `action` (VARCHAR(100), NOT NULL)
- `entity_type` (VARCHAR(50), NOT NULL)
- `entity_id` (BIGINT)
- `previous_value` (JSONB)
- `new_value` (JSONB)
- `details` (JSONB)
- `created_at` (TIMESTAMP, NOT NULL, DEFAULT CURRENT_TIMESTAMP)

Primary key: `id`

Foreign keys:
- `user_id` → `users.id`

Notes:
- Audit logs must identify who performed the action, what happened, when it happened, and what entity was affected.
- They should be append-only and immutable.
- `previous_value` and `new_value` support before/after comparisons for authorization-sensitive changes.

---

## 4. Important Relationships Between Tables

### Core relationships

- `users` to `roles`: many-to-many via `user_roles`
- `categories` to `products`: one-to-many
- `suppliers` to `purchases`: one-to-many
- `purchases` to `purchase_items`: one-to-many
- `products` to `purchase_items`: many-to-one
- `sales` to `sale_items`: one-to-many
- `products` to `sale_items`: many-to-one
- `products` to `inventory_transactions`: one-to-many
- `products` to `stock_adjustments`: one-to-many
- `users` to `audit_logs`: one-to-many

### Inventory relationship model

The key inventory flow is:

- Purchase receipt creates `inventory_transactions` with positive `quantity_change`
- Sale/stock issue creates `inventory_transactions` with negative `quantity_change`
- Stock adjustment creates `inventory_transactions` with positive or negative `quantity_change`

The conceptual invariant is:

`Current Stock = Opening Stock + Stock Received - Stock Issued + Adjustments`

This is not stored manually as a single editable field; it is derived from the sum of valid inventory transactions over time.

---

## 5. Primary Keys and Foreign Keys

### Primary keys

| Table | Primary key |
|------|-------------|
| users | `id` |
| roles | `id` |
| user_roles | `id` |
| categories | `id` |
| products | `id` |
| suppliers | `id` |
| purchases | `id` |
| purchase_items | `id` |
| sales | `id` |
| sale_items | `id` |
| inventory_transactions | `id` |
| stock_adjustments | `id` |
| audit_logs | `id` |

### Foreign keys

- `user_roles.user_id` → `users.id`
- `user_roles.role_id` → `roles.id`
- `products.category_id` → `categories.id`
- `purchases.supplier_id` → `suppliers.id`
- `purchases.created_by` → `users.id`
- `purchase_items.purchase_id` → `purchases.id`
- `purchase_items.product_id` → `products.id`
- `sales.created_by` → `users.id`
- `sale_items.sale_id` → `sales.id`
- `sale_items.product_id` → `products.id`
- `inventory_transactions.product_id` → `products.id`
- `inventory_transactions.created_by` → `users.id`
- `stock_adjustments.product_id` → `products.id`
- `stock_adjustments.created_by` → `users.id`
- `audit_logs.user_id` → `users.id`

---

## 6. Cardinality and Important Rules

### One-to-many relationships

- One category can have many products.
- One supplier can have many purchases.
- One purchase can have many purchase items.
- One sale can have many sale items.
- One product can have many inventory transactions.
- One product can have many stock adjustments.
- One user can perform many audit entries.

### Many-to-many relationship

- Users and roles are linked via `user_roles`.

### Inventory transaction semantics

Inventory transaction records are not simply additional metadata—they are the actual historical ledger of inventory movement.

This means:
- There is no single editable `current_stock` column that is treated as authoritative.
- `current_stock` is a derived value, computed from the transaction history.
- All inventory-affecting operations must add a corresponding `inventory_transactions` row.

---

## 7. Important Constraints and Uniqueness Rules

- `products.sku` must be unique.
- `users.username` must be unique.
- `users.email` must be unique.
- `purchases.purchase_number` must be unique.
- `sales.sale_number` must be unique.
- `categories.name` must be unique.
- `suppliers.name` must be unique.
- `roles.name` must be unique.
- `inventory_transactions.quantity_change` cannot be zero.
- `purchase_items.ordered_quantity` must be positive.
- `sale_items.quantity` must be positive.
- `stock_adjustments.quantity` must be positive.
- Products, categories, and suppliers with historical activity should be deactivated instead of deleted.
- Historical inventory transactions cannot be silently modified or deleted.

---

## 8. Inventory Transaction Design

This database design makes `inventory_transactions` the authoritative report of all stock movement. Inventory movement is always represented by an explicit transaction record, regardless of whether it came from a purchase, sale, adjustment, or other future source.

### Transaction record structure

Each `inventory_transactions` record should include:
- `product_id`
- `transaction_type` (`receipt`, `issue`, `adjustment`)
- `quantity_change` (signed integer)
- `reference_type` and `reference_id` linking to source entity
- `notes`
- `created_by`
- `created_at`

### Signed quantity approach

- Receipt: positive value
- Issue: negative value
- Adjustment increase: positive value
- Adjustment decrease: negative value

This allows current stock to be computed with a single `SUM(quantity_change)` per product.

### Example inventory transaction flow

- Purchase of 100 units of Product A results in:
  - `inventory_transactions`: `product_id = A`, `transaction_type = 'receipt'`, `quantity_change = +100`
- Sale of 12 units of Product A results in:
  - `inventory_transactions`: `product_id = A`, `transaction_type = 'issue'`, `quantity_change = -12`
- Adjustment +5 units results in:
  - `inventory_transactions`: `product_id = A`, `transaction_type = 'adjustment'`, `quantity_change = +5`

Resulting stock for Product A is `100 - 12 + 5 = 93`.

---

## 9. How Purchases Affect Inventory

A purchase is not the inventory itself; it is the formal record of a commercial acquisition. The inventory movement is represented by `inventory_transactions` when stock is received.

### Purchase flow

1. Create purchase header in `purchases`
2. Add line items in `purchase_items`
3. Receive stock into inventory
4. Update `purchase_items.received_quantity`
5. Insert one or more `inventory_transactions` for the received quantity

### Purchase inventory logic

When a purchase item is received:
- `purchase_items.received_quantity += received_quantity`
- `inventory_transactions.quantity_change += +received_quantity`
- `inventory_transactions.transaction_type = 'receipt'`
- `reference_type = 'purchase_item'`
- `reference_id = purchase_items.id`

Important rule:
- Inventory movement and purchase record update should happen in the same transaction to maintain consistency.

---

## 10. How Sales and Issues Affect Inventory

Sales and stock issues are inventory-reducing activities. They must be validated against the current available quantity before completion.

### Sale/issue flow

1. Create `sales` header with type and metadata
2. Add `sale_items` for product and quantity
3. Check available quantity for each product
4. Reject if requested quantity exceeds available stock
5. Create `inventory_transactions` with negative `quantity_change`

### Validation logic

For each sale item:
- Compute current stock through the transaction history
- Ensure `requested_quantity <= available_stock`
- If not, reject the transaction without creating negative stock

### Requirement alignment

This directly reflects the requirement:
- "Stock issues must normally be rejected when requested quantity exceeds available stock."

---

## 11. How Stock Adjustments Affect Inventory

Stock adjustments represent manual or exceptional corrections to inventory count, such as:
- counting mismatches
- damage or spoilage
- returns
- corrections after a physical inventory count

### Design principle

Every stock adjustment should be stored in:
- `stock_adjustments`: metadata and reason
- `inventory_transactions`: signed quantity movement

### Example

Product B has a stock adjustment of +8 units due to count correction.

`stock_adjustments` record:
- product_id = B
- adjustment_type = 'increase'
- quantity = 8
- reason = 'Physical count correction'
- created_by = user

Then create:
- `inventory_transactions`: `product_id = B`, `transaction_type = 'adjustment'`, `quantity_change = +8`

This ensures the adjustment is visible in stock history and is auditable.

---

## 12. Audit Logging

Audit logging is critical for traceability and compliance.

### Audit record purpose

Each audit log identifies:
- Who did the action (`user_id`)
- What action happened (`action`)
- Which entity was affected (`entity_type`, `entity_id`)
- When it happened (`created_at`)
- Previous state (`previous_value`)
- New state (`new_value`)
- Additional details (`details`)

### Typical audit examples

- User created a product
- Inventory manager received stock for a purchase
- Staff attempted a sale with insufficient inventory
- Admin updated a supplier record
- Stock adjustment was performed

### Important rule

Audit records should be append-only and immutable.

---

## 13. Data Integrity Considerations

### Referential integrity

- Foreign keys ensure valid references between tables.
- Categories, suppliers, and products are not physically deleted if they are referenced by historical records.

### Inventory correctness

- Inventory is derived from transactions, so there is no risk of stale or manually conflicting current stock values.
- Historical integrity is preserved by making original records append-only.

### Business rule enforcement

- Authorization logic is enforced by the application.
- Data validation occurs on the application side, with the database providing constraints for critical rules.

### Historical safety

- Purchase and sale records are never silently deleted.
- Historical inventory transactions remain intact and auditable.

---

## 14. Transaction and Atomicity Considerations

Because inventory operations can involve multiple tables, database transactions are critical.

### Example: receiving stock

Within one transaction:
- insert or update `purchase_items`
- insert `inventory_transactions`
- update purchase status if relevant
- write audit logs

If any step fails, the entire transaction is rolled back so the database remains consistent.

### Example: completing a sale

Within one transaction:
- validate available inventory
- insert `sale_items`
- insert `inventory_transactions`
- update `sales.status`
- write audit logs

If validation fails, no stock movement occurs.

### Concurrency

To prevent race conditions, stock validation should use transaction locking or serializable isolation when necessary.

This is especially important when multiple users attempt to issue inventory for the same product at the same time.

---

## 15. Indexing Considerations

The following indexes are important for usability and performance:

- `products.sku` unique index
- `products.category_id` index
- `inventory_transactions.product_id` index
- `inventory_transactions.created_at` index
- `purchases.supplier_id` index
- `purchases.purchase_date` index
- `sales.sale_date` index
- `audit_logs.user_id` index
- `audit_logs.entity_type + entity_id` composite index

These indexes support:
- product lookup by SKU
- current stock calculations
- historical transaction queries
- dashboard summaries
- low-stock detection
- audit queries

---

## 16. Entity Relationship Diagram (Text-Based)

```
users
  ├─< user_roles >─ roles
  │
  ├─< purchases
  │
  ├─< sales
  │
  ├─< inventory_transactions
  │
  ├─< stock_adjustments
  │
  └─< audit_logs

categories
  └─< products

suppliers
  └─< purchases

products
  ├─< purchase_items
  ├─< sale_items
  ├─< inventory_transactions
  ├─< stock_adjustments
  └─< audit_logs (entity_type = 'product')

purchases
  └─< purchase_items

sales
  └─< sale_items

purchase_items
  └─> inventory_transactions (reference_type='purchase_item')

sale_items
  └─> inventory_transactions (reference_type='sale_item')

stock_adjustments
  └─> inventory_transactions (reference_type='stock_adjustment')
```

### More detailed conceptual ER structure

```text
┌───────────────┐        1:N        ┌───────────────┐
│ categories    │────────────────────▶ products      │
└───────────────┘                     └───────────────┘
                                              │
                                              │ 1:N
                                              ▼
                                  ┌────────────────────┐
                                  │ inventory_transactions │
                                  └────────────────────┘
                                              ▲
                                              │
                                              │
                                              │
┌───────────────┐        1:N        ┌───────────────┐
│ suppliers     │────────────────────▶ purchases      │
└───────────────┘                     └───────────────┘
                                               │
                                               │ 1:N
                                               ▼
                                       ┌────────────────┐
                                       │ purchase_items │
                                       └───────���────────┘

┌───────────────┐        1:N        ┌───────────────┐
│ users         │────────────────────▶ sales          │
└───────────────┘                     └───────────────┘
                                               │
                                               │ 1:N
                                               ▼
                                       ┌────────────────┐
                                       │ sale_items     │
                                       └────────────────┘

┌───────────────┐        1:N        ┌────────────────────┐
│ users         │────────────────────▶ stock_adjustments  │
└───────────────┘                     └────────────────────┘
                                               │
                                               │ 1:N
                                               ▼
                                  ┌────────────────────┐
                                  │ inventory_transactions │
                                  └────────────────────┘

┌───────────────┐
│ users         │
│ roles         │
│ user_roles    │
│ audit_logs    │
└───────────────┘
```

---

## 17. Alignment with Requirements

This design is aligned with the documented functional and business requirements:

- Users and roles are modeled via `users`, `roles`, and `user_roles`
- Products, categories, and suppliers are included as core entities
- Purchases are represented by `purchases` and `purchase_items`
- Sales/issues are represented by `sales` and `sale_items`
- Inventory movement is recorded through `inventory_transactions`
- Stock adjustments are recorded separately through `stock_adjustments`
- Audit logging is implemented in `audit_logs`
- Historical integrity is preserved by avoiding silent mutation of transaction records
- Inventory is treated as the sum of valid movements rather than a manually maintained current stock figure
- Low-stock monitoring is supported through product reorder levels and stock calculation logic

---

## 18. Summary

This database design is intentionally normalized and maintainable. It provides a clear separation between:

- master data (`users`, `roles`, `products`, `categories`, `suppliers`)
- business transactions (`purchases`, `sale`, `stock_adjustments`)
- inventory history (`inventory_transactions`)
- audit trail (`audit_logs`)

The most important design principle is that inventory movement is not based on a manually edited `current_stock` value; it is reconstructed from immutable historical inventory transactions. This keeps the system accurate, auditable, and aligned with the core business rules in the requirements.

The document intentionally avoids premature implementation complexity such as warehouses, serial numbers, batch tracking, customers, payments, or forecasting. Those features may be added later when justified by real product requirements.
