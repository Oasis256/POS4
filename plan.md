## Plan: Loyalty Module for NexoPOS

Create a Loyalty Module that allows customers to earn points on completed POS transactions and automatically redeem points when sufficient balance exists (with cashier/customer consent). Points are earned only on the actual money portion of transactions (excluding discounts, coupons, etc.). Points use FIFO expiration and cash has priority over points for payment.

**Key Decision**: Points redemption will be implemented as a **custom payment type** rather than a discount, ensuring proper accounting treatment where books always balance. Points earned create a customer liability, and points redemption creates an OrderPayment record that flows through the existing accounting system.

### Steps

1. **Module Scaffolding**
   - Create module structure under `modules/NsLoyalty` using `php artisan make:module --no-interaction`
   - Configure `config.xml` with namespace `NsLoyalty`, version `1.0.0`, appropriate author/name/description
   - UI is fully internal-facing; no customer-initiated actions

2. **Database Design**
   - Create migration for `nsloyalty_customer_points` table:
     - `id` (primary key)
     - `customer_id` (foreign key to nexopos_users, unique)
     - `points_balance` (integer, default 0)
     - `points_earned_lifetime` (integer, default 0)
     - `points_redeemed_lifetime` (integer, default 0)
     - `created_at`, `updated_at`
   - Create migration for `nsloyalty_point_transactions` table:
     - `id` (primary key)
     - `customer_id` (foreign key)
     - `type` (enum: 'earned', 'redeemed', 'adjusted', 'expired')
     - `points` (integer, positive for earned, negative for redeemed/expired)
     - `points_balance_after` (integer, for audit trail)
     - `related_id` (nullable foreign key to orders for traceability)
     - `description` (text)
     - `expires_at` (nullable timestamp, for earned points with expiration)
     - `created_at`
   - Both migrations will use existence checks (`Schema::hasTable`) for repeat-safety
   - The `expires_at` column supports FIFO - old points are redeemed/expired first

3. **Models**
   - Create `CustomerPointBalance` model with relationships to Customer
   - Create `PointTransaction` model with relationships to Customer and Order (optional)
   - Add proper casts and fillable attributes
   - Consider adding accessors for formatted points display

4. **Services**
   - Create `LoyaltyService` handling:
     - Calculate points earned based on order money subtotal
     - Award points to customer after order completion (FIFO tracking with expiration dates)
     - Automatically redeem points for monetary value when sufficient balance exists and cashier/customer consents verbally
     - Check customer points balance with FIFO expiration logic (oldest points used/expired first)
     - Process point transactions with proper locking and audit trail
     - Enforce that cash/actual money has priority over points (points only cover remaining balance after other payment methods)
     - Limit points redemption to not exceed remaining transaction amount
   - Follow Laravel best practices for service classes with type hints and explicit return types

5. **Events & Listeners**
   - Create listener for `OrderAfterPaymentStatusChangedEvent` (for paid orders only)
   - In listener: calculate and award points when order is successfully paid (FIFO tracking with expiration)
   - Points calculation: only on actual money paid (order.total_with_tax - discount_amounts from coupons/promos)
   - Use Laravel's event system rather than deprecated Hook::addAction patterns
   - Additionally, create a POS-side listener that automatically suggests points redemption when customer has sufficient non-expired points and cashier/customer gives verbal consent

6. **POS Integration**
   - Create POS script (`Resources/ts/pos.ts`) that:
     - Adds loyalty points redemption to payment queue via `ns-pay-queue` filter
     - Shows points balance in POS header or customer display area
     - Automatically suggests points redemption when customer has sufficient balance (FIFO, non-expired points)
     - UI prompts cashier for verbal customer consent before applying points redemption
     - Ensures cash/actual money has priority (points only cover remaining balance after other payment methods)
     - Implements FIFO points expiration (oldest points used/expired first)
     - Limits points redemption to not exceed remaining transaction amount
   - Add assets via `RenderHeaderEvent` and `RenderFooterEvent` listeners on POS route only
   - Use module Tailwind prefix (e.g., `loy:`) for all styling
   - Register POS component for displaying points redemption suggestion during payment flow

7. **Permissions**
   - Create permissions for:
     - `view.loyalty.customer-balance` (view points balance)
     - `redeem.loyalty.points` (redeem points during checkout)
     - `manage.loyalty.settings` (configure earning rates, etc.)
   - Assign appropriate permissions to cashier and manager roles via service provider

8. **Settings**
   - Create settings page allowing configuration of:
     - Points earning rate (e.g., 1 point per $1 spent)
     - Points redemption value (e.g., 100 points = $1)
     - Minimum points for redemption
     - Whether points can be earned on discounted amounts (configurable, default false)
     - Point expiration rules (optional)

9. **Accounting Integration**
   - By treating points redemption as a payment type (`loyalty-points`), it automatically:
     - Creates OrderPayment records
     - Flows through existing payment processing
     - Integrates with accounting journal system via OrderPayment events
     - Maintains book balance integrity
   - Points earning creates customer liability tracked in `nsloyalty_customer_points`

10. **Testing**
    - Create feature tests for:
      - Points earning on various order types
      - Points redemption during checkout
      - Insufficient points handling
      - Points expiration (if implemented)
    - Use Laravel's testing best practices with factories and assertions

### Relevant Files
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/config.xml` — Module metadata and configuration
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Migrations/2026_10_10_000000_create_nsloyalty_tables.php` — Database schema for points tracking
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Models/CustomerPointBalance.php` — Eloquent model for customer points
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Services/LoyaltyService.php` — Core business logic service
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Listeners/AwardPointsAfterOrder.php` — Event listener for points awarding
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Resources/ts/pos.ts` — POS integration script
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Resources/Views/pos/header.blade.php` — POS header asset loading
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Resources/Views/pos/footer.blade.php` — POS footer asset loading
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Settings/LoyaltySettings.php` — Module configuration page
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Routes/api.php` — API endpoints for points operations
- `/home/Oasis/Compose/Web/WWW/nexo/modules/NsLoyalty/Tests/Feature/LoyaltyTest.php` — Test suite

### Verification
1. **Database Verification**
   - Run migrations: `php artisan module:migrate NsLoyalty`
   - Verify tables created with correct schema
   - Check foreign key constraints

2. **Functional Verification**
   - Create test customer and process POS order
   - Verify points are awarded based on money subtotal only
   - Attempt points redemption during checkout
   - Verify OrderPayment record created with loyalty-points identifier
   - Verify customer points balance updated correctly (FIFO - oldest points used first)
   - Verify accounting entries balance (check OrderPayment -> accounting journals)
   - Verify cash has priority: points redemption only applied for remaining balance after other payments attempted
   - Verify FIFO expiration: oldest points expired/used first
   - Test verbal consent scenario: cashier prompted for consent, consent recorded in order notes
   - Test insufficient points scenario gracefully handled

3. **Edge Case Testing**
   - Test insufficient points scenario
   - Test points earning with various discount types
   - Test points redemption exceeding balance
   - Test concurrent access to points balance

4. **UI Verification**
   - Verify points display in POS header
   - Verify points redemption option appears in payment methods when balance sufficient
   - Verify proper error messaging for insufficient points

### Decisions
- **Points as Payment Type**: Chose to implement points redemption as a custom payment type rather than discount to ensure proper accounting treatment and book balance integrity. This leverages the existing OrderPayment system and accounting journal integration.
- **Points Earning Basis**: Points earned only on actual money paid (order total minus discounts, coupons, etc.) as specified in requirements.
- **Event-Driven Architecture**: Using Laravel events (OrderAfterPaymentStatusChangedEvent) rather than deprecated hooks for better maintainability.
- **POS Integration Approach**: Using ns-pay-queue filter to insert points redemption into payment options, following established patterns from modules like NsAppointments.

### Further Considerations
1. **Point Expiration**: Should points expire after a period of inactivity? If so, need to add expiration logic and scheduled job.
2. **Tiered Earning Rates**: Should different customer groups earn points at different rates? Could be configurable in settings.
3. **Manual Adjustments**: Should administrators be able to manually adjust points balances (for corrections, bonuses, etc.)?
4. **Points Transfer**: Should customers be able to transfer points between accounts? (Likely not for security/fraud reasons)
5. **Redemption Flexibility**: Should points be redeemable for specific products/discounts rather than just monetary value? (Could be phase 2 enhancement)
6. **Reporting**: Should there be reports showing points liability, earning/redemption trends, etc.?