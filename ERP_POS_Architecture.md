# Arquitectura Backend: Sistema ERP + POS con NestJS

> **Autor:** Arquitecto de Software Senior  
> **Stack:** NestJS · Prisma · Supabase (PostgreSQL) · Docker · Jest  
> **Versión:** 1.0.0

---

## Tabla de Contenidos

1. [Visión Arquitectónica](#1-visión-arquitectónica)
2. [Estructura del Proyecto](#2-estructura-del-proyecto)
3. [Diseño de Base de Datos — schema.prisma](#3-diseño-de-base-de-datos--schemaprisma)
4. [Descripción de Módulos](#4-descripción-de-módulos)
5. [Flujo Crítico: Pago POS con Cuenta Corriente](#5-flujo-crítico-pago-pos-con-cuenta-corriente)
6. [Estrategia de Testing con Jest](#6-estrategia-de-testing-con-jest)
7. [Infraestructura Docker](#7-infraestructura-docker)
8. [Decisiones de Diseño y Principios SOLID](#8-decisiones-de-diseño-y-principios-solid)

---

## 1. Visión Arquitectónica

El sistema adopta **Clean Architecture** adaptada a NestJS, con separación estricta entre capas:

```
┌─────────────────────────────────────────────────────────┐
│                  HTTP / WebSocket Layer                  │
│              Controllers  ·  Guards  ·  Pipes           │
├─────────────────────────────────────────────────────────┤
│                   Application Layer                      │
│                Services  ·  Use Cases                   │
├─────────────────────────────────────────────────────────┤
│                    Domain Layer                          │
│          Entities  ·  Interfaces  ·  DTOs               │
├─────────────────────────────────────────────────────────┤
│                 Infrastructure Layer                     │
│       Prisma Repositories  ·  Supabase  ·  External     │
└─────────────────────────────────────────────────────────┘
```

**Principios rectores:**
- Cada módulo es un **bounded context** con su propia responsabilidad
- Las transacciones financieras son **atómicas** — se usan transacciones Prisma en todos los flujos de dinero
- Los estados de las órdenes son **máquinas de estado** explícitas, nunca mutaciones libres
- Los repositorios abstraen Prisma — los servicios nunca importan `PrismaClient` directamente

---

## 2. Estructura del Proyecto

```
erp-pos-backend/
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   │
│   ├── config/                         # Configuración global
│   │   ├── app.config.ts
│   │   ├── database.config.ts
│   │   └── jwt.config.ts
│   │
│   ├── common/                         # Shared kernel
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   └── roles.decorator.ts
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── interceptors/
│   │   │   └── transaction.interceptor.ts
│   │   ├── pipes/
│   │   │   └── zod-validation.pipe.ts
│   │   ├── enums/
│   │   │   ├── order-status.enum.ts
│   │   │   ├── payment-method.enum.ts
│   │   │   └── money-type.enum.ts
│   │   └── interfaces/
│   │       └── paginated-response.interface.ts
│   │
│   ├── database/                       # Infraestructura DB
│   │   ├── prisma.service.ts
│   │   └── database.module.ts
│   │
│   ├── modules/
│   │   │
│   │   ├── auth/                       # Autenticación & Autorización
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   └── jwt.strategy.ts
│   │   │   └── dto/
│   │   │       ├── login.dto.ts
│   │   │       └── refresh-token.dto.ts
│   │   │
│   │   ├── employees/                  # Gestión de Empleados
│   │   │   ├── employees.module.ts
│   │   │   ├── employees.controller.ts
│   │   │   ├── employees.service.ts
│   │   │   ├── employees.repository.ts
│   │   │   └── dto/
│   │   │
│   │   ├── customers/                  # Gestión de Clientes
│   │   │   ├── customers.module.ts
│   │   │   ├── customers.controller.ts
│   │   │   ├── customers.service.ts
│   │   │   ├── customers.repository.ts
│   │   │   └── dto/
│   │   │
│   │   ├── suppliers/                  # Gestión de Proveedores
│   │   │   ├── suppliers.module.ts
│   │   │   ├── suppliers.controller.ts
│   │   │   ├── suppliers.service.ts
│   │   │   ├── suppliers.repository.ts
│   │   │   └── dto/
│   │   │
│   │   ├── inventory/                  # Inventario Dual
│   │   │   ├── inventory.module.ts
│   │   │   ├── products/               # Productos Finales
│   │   │   │   ├── products.controller.ts
│   │   │   │   ├── products.service.ts
│   │   │   │   ├── products.repository.ts
│   │   │   │   └── dto/
│   │   │   ├── raw-materials/          # Materias Primas
│   │   │   │   ├── raw-materials.controller.ts
│   │   │   │   ├── raw-materials.service.ts
│   │   │   │   ├── raw-materials.repository.ts
│   │   │   │   └── dto/
│   │   │   └── bill-of-materials/      # Composición del producto
│   │   │       ├── bom.service.ts
│   │   │       └── dto/
│   │   │
│   │   ├── orders/                     # Órdenes de Compra
│   │   │   ├── orders.module.ts
│   │   │   ├── orders.controller.ts
│   │   │   ├── orders.service.ts
│   │   │   ├── orders.repository.ts
│   │   │   ├── state-machine/
│   │   │   │   └── order-state.machine.ts
│   │   │   └── dto/
│   │   │
│   │   ├── pos/                        # Punto de Venta
│   │   │   ├── pos.module.ts
│   │   │   ├── pos.controller.ts
│   │   │   ├── pos.service.ts
│   │   │   └── dto/
│   │   │       ├── process-payment.dto.ts
│   │   │       └── payment-method.dto.ts
│   │   │
│   │   ├── invoices/                   # Facturación
│   │   │   ├── invoices.module.ts
│   │   │   ├── invoices.service.ts
│   │   │   ├── invoices.repository.ts
│   │   │   └── dto/
│   │   │
│   │   ├── finance/                    # Finanzas
│   │   │   ├── finance.module.ts
│   │   │   ├── current-accounts/       # Cuentas Corrientes
│   │   │   │   ├── current-accounts.controller.ts
│   │   │   │   ├── current-accounts.service.ts
│   │   │   │   ├── current-accounts.repository.ts
│   │   │   │   └── dto/
│   │   │   └── cash-register/          # Cierre de Caja
│   │   │       ├── cash-register.controller.ts
│   │   │       ├── cash-register.service.ts
│   │   │       ├── cash-register.repository.ts
│   │   │       └── dto/
│   │   │
│   │   ├── reports/                    # Reportes e Historiales
│   │   │   ├── reports.module.ts
│   │   │   ├── reports.controller.ts
│   │   │   ├── reports.service.ts
│   │   │   └── dto/
│   │   │
│   │   └── dashboard/                  # KPIs y métricas
│   │       ├── dashboard.module.ts
│   │       ├── dashboard.controller.ts
│   │       └── dashboard.service.ts
│   │
│   └── shared/
│       └── services/
│           └── notification.service.ts
│
├── test/
│   ├── unit/
│   │   ├── pos/
│   │   ├── finance/
│   │   └── inventory/
│   ├── integration/
│   │   ├── pos.integration.spec.ts
│   │   └── cash-register.integration.spec.ts
│   └── e2e/
│       └── app.e2e-spec.ts
│
├── docker/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── docker-compose.yml
│
├── .env.example
├── jest.config.ts
└── tsconfig.json
```

---

## 3. Diseño de Base de Datos — schema.prisma

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")  // Supabase requiere directUrl para migraciones
}

// ─────────────────────────────────────────────
// ENUMS
// ─────────────────────────────────────────────

enum Role {
  ADMIN
  EMPLOYEE
  CASHIER
}

enum OrderStatus {
  NO_PAGADO
  PAGADO
  ENTREGADO
}

enum PaymentMethod {
  CASH              // Efectivo
  TRANSFER          // Transferencia bancaria
  DIGITAL_WALLET    // Billetera digital (Mercado Pago, etc.)
  CURRENT_ACCOUNT   // Cuenta corriente del cliente
  MIXED             // Pago mixto (ej: parte efectivo + parte cuenta corriente)
}

enum MoneyType {
  PHYSICAL  // Dinero físico (efectivo)
  VIRTUAL   // Dinero virtual (transferencias, billeteras digitales)
}

enum TransactionType {
  CREDIT    // Ingreso a la cuenta corriente
  DEBIT     // Consumo / descuento de la cuenta corriente
}

enum CashRegisterStatus {
  OPEN
  CLOSED
}

enum SupplyOrderStatus {
  PENDING
  RECEIVED
  CANCELLED
}

// ─────────────────────────────────────────────
// AUTH / USUARIOS
// ─────────────────────────────────────────────

model User {
  id           String    @id @default(cuid())
  email        String    @unique
  passwordHash String    @map("password_hash")
  role         Role      @default(EMPLOYEE)
  isActive     Boolean   @default(true) @map("is_active")
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")

  employee     Employee?
  refreshTokens RefreshToken[]

  @@map("users")
}

model RefreshToken {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String   @map("user_id")
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("refresh_tokens")
}

// ─────────────────────────────────────────────
// ENTIDADES DE NEGOCIO
// ─────────────────────────────────────────────

model Employee {
  id         String    @id @default(cuid())
  userId     String    @unique @map("user_id")
  firstName  String    @map("first_name")
  lastName   String    @map("last_name")
  phone      String?
  address    String?
  hireDate   DateTime  @map("hire_date")
  salary     Decimal   @db.Decimal(12, 2)
  isActive   Boolean   @default(true) @map("is_active")
  createdAt  DateTime  @default(now()) @map("created_at")
  updatedAt  DateTime  @updatedAt @map("updated_at")

  user         User          @relation(fields: [userId], references: [id])
  orders       Order[]       // Órdenes procesadas por este empleado
  cashRegisters CashRegister[]

  @@map("employees")
}

model Customer {
  id             String    @id @default(cuid())
  firstName      String    @map("first_name")
  lastName       String    @map("last_name")
  email          String?   @unique
  phone          String?
  address        String?
  taxId          String?   @unique @map("tax_id")   // CUIT/DNI
  isActive       Boolean   @default(true) @map("is_active")
  createdAt      DateTime  @default(now()) @map("created_at")
  updatedAt      DateTime  @updatedAt @map("updated_at")

  orders          Order[]
  currentAccount  CurrentAccount?

  @@map("customers")
}

model Supplier {
  id          String    @id @default(cuid())
  companyName String    @map("company_name")
  contactName String?   @map("contact_name")
  email       String?   @unique
  phone       String?
  address     String?
  taxId       String?   @unique @map("tax_id")
  isActive    Boolean   @default(true) @map("is_active")
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")

  supplyOrders SupplyOrder[]

  @@map("suppliers")
}

// ─────────────────────────────────────────────
// INVENTARIO DUAL
// ─────────────────────────────────────────────

/// Producto Final (listo para la venta)
model Product {
  id                String    @id @default(cuid())
  sku               String    @unique
  name              String
  description       String?
  salePrice         Decimal   @db.Decimal(12, 2) @map("sale_price")
  stock             Int       @default(0)
  minStock          Int       @default(0) @map("min_stock")   // Alerta de stock mínimo
  imageUrl          String?   @map("image_url")
  isActive          Boolean   @default(true) @map("is_active")
  createdAt         DateTime  @default(now()) @map("created_at")
  updatedAt         DateTime  @updatedAt @map("updated_at")

  billOfMaterials   BillOfMaterial[]
  orderItems        OrderItem[]

  @@map("products")
}

/// Materia Prima / Insumo
model RawMaterial {
  id            String    @id @default(cuid())
  sku           String    @unique
  name          String
  description   String?
  unitCost      Decimal   @db.Decimal(12, 2) @map("unit_cost")  // Costo unitario actualizable
  stock         Decimal   @db.Decimal(12, 4)                     // Permite fracciones (metros, kg)
  unit          String    @default("unit")                       // "unit", "kg", "m", "liter"
  minStock      Decimal   @default(0) @db.Decimal(12, 4) @map("min_stock")
  isActive      Boolean   @default(true) @map("is_active")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  billOfMaterials      BillOfMaterial[]
  supplyOrderItems     SupplyOrderItem[]
  stockMovements       RawMaterialMovement[]

  @@map("raw_materials")
}

/// Lista de Materiales (Bill of Materials) — define la composición de un producto
/// Relación N:M con payload entre Product y RawMaterial
model BillOfMaterial {
  id            String      @id @default(cuid())
  productId     String      @map("product_id")
  rawMaterialId String      @map("raw_material_id")
  quantity      Decimal     @db.Decimal(12, 4)        // Cantidad requerida por unidad de producto
  createdAt     DateTime    @default(now()) @map("created_at")
  updatedAt     DateTime    @updatedAt @map("updated_at")

  product     Product     @relation(fields: [productId], references: [id])
  rawMaterial RawMaterial @relation(fields: [rawMaterialId], references: [id])

  @@unique([productId, rawMaterialId])
  @@map("bill_of_materials")
}

/// Registro de movimientos de materia prima (trazabilidad completa)
model RawMaterialMovement {
  id              String    @id @default(cuid())
  rawMaterialId   String    @map("raw_material_id")
  quantity        Decimal   @db.Decimal(12, 4)   // Positivo = entrada, Negativo = salida
  reason          String    // "SALE", "MANUFACTURE", "SUPPLY", "ADJUSTMENT"
  referenceId     String?   @map("reference_id")  // ID de la orden/proveedor relacionado
  unitCostAtTime  Decimal   @db.Decimal(12, 2) @map("unit_cost_at_time")
  createdAt       DateTime  @default(now()) @map("created_at")
  createdBy       String    @map("created_by")    // userId

  rawMaterial RawMaterial @relation(fields: [rawMaterialId], references: [id])

  @@map("raw_material_movements")
}

// ─────────────────────────────────────────────
// ÓRDENES DE COMPRA Y POS
// ─────────────────────────────────────────────

model Order {
  id            String      @id @default(cuid())
  orderNumber   String      @unique @map("order_number")  // "ORD-2024-00001"
  customerId    String      @map("customer_id")
  employeeId    String      @map("employee_id")
  status        OrderStatus @default(NO_PAGADO)
  subtotal      Decimal     @db.Decimal(12, 2)
  taxAmount     Decimal     @default(0) @db.Decimal(12, 2) @map("tax_amount")
  discountAmount Decimal    @default(0) @db.Decimal(12, 2) @map("discount_amount")
  total         Decimal     @db.Decimal(12, 2)
  notes         String?
  createdAt     DateTime    @default(now()) @map("created_at")
  updatedAt     DateTime    @updatedAt @map("updated_at")
  paidAt        DateTime?   @map("paid_at")
  deliveredAt   DateTime?   @map("delivered_at")

  customer  Customer    @relation(fields: [customerId], references: [id])
  employee  Employee    @relation(fields: [employeeId], references: [id])
  items     OrderItem[]
  payment   Payment?
  invoice   Invoice?

  @@index([status])
  @@index([customerId])
  @@map("orders")
}

model OrderItem {
  id          String    @id @default(cuid())
  orderId     String    @map("order_id")
  productId   String    @map("product_id")
  quantity    Int
  unitPrice   Decimal   @db.Decimal(12, 2) @map("unit_price")   // Precio al momento de la venta
  unitCost    Decimal   @db.Decimal(12, 2) @map("unit_cost")    // Costo calculado al momento de la venta
  subtotal    Decimal   @db.Decimal(12, 2)
  createdAt   DateTime  @default(now()) @map("created_at")

  order   Order   @relation(fields: [orderId], references: [id], onDelete: Cascade)
  product Product @relation(fields: [productId], references: [id])

  @@map("order_items")
}

model Payment {
  id              String        @id @default(cuid())
  orderId         String        @unique @map("order_id")
  method          PaymentMethod
  totalAmount     Decimal       @db.Decimal(12, 2) @map("total_amount")
  
  // Para pagos con CURRENT_ACCOUNT:
  physicalAmount  Decimal       @default(0) @db.Decimal(12, 2) @map("physical_amount")  // Del efectivo de la cta cte
  virtualAmount   Decimal       @default(0) @db.Decimal(12, 2) @map("virtual_amount")   // De transferencias de la cta cte
  
  // Para pagos directos:
  cashAmount      Decimal       @default(0) @db.Decimal(12, 2) @map("cash_amount")
  transferAmount  Decimal       @default(0) @db.Decimal(12, 2) @map("transfer_amount")
  walletAmount    Decimal       @default(0) @db.Decimal(12, 2) @map("wallet_amount")
  
  reference       String?       // Número de transf., comprobante, etc.
  processedAt     DateTime      @default(now()) @map("processed_at")
  processedBy     String        @map("processed_by")  // userId del cajero

  order           Order         @relation(fields: [orderId], references: [id])
  cashRegisterId  String?       @map("cash_register_id")
  cashRegister    CashRegister? @relation(fields: [cashRegisterId], references: [id])

  @@map("payments")
}

// ─────────────────────────────────────────────
// FACTURACIÓN
// ─────────────────────────────────────────────

model Invoice {
  id              String    @id @default(cuid())
  invoiceNumber   String    @unique @map("invoice_number")  // "FAC-2024-00001"
  orderId         String    @unique @map("order_id")
  customerId      String    @map("customer_id")
  subtotal        Decimal   @db.Decimal(12, 2)
  taxAmount       Decimal   @db.Decimal(12, 2) @map("tax_amount")
  total           Decimal   @db.Decimal(12, 2)
  issuedAt        DateTime  @default(now()) @map("issued_at")
  pdfUrl          String?   @map("pdf_url")   // URL en Supabase Storage

  order    Order    @relation(fields: [orderId], references: [id])

  @@map("invoices")
}

// ─────────────────────────────────────────────
// FINANZAS — CUENTAS CORRIENTES
// ─────────────────────────────────────────────

/// Una cuenta corriente por cliente. Tiene saldo físico y virtual separados.
model CurrentAccount {
  id              String    @id @default(cuid())
  customerId      String    @unique @map("customer_id")
  physicalBalance Decimal   @default(0) @db.Decimal(12, 2) @map("physical_balance")
  virtualBalance  Decimal   @default(0) @db.Decimal(12, 2) @map("virtual_balance")
  creditLimit     Decimal   @default(0) @db.Decimal(12, 2) @map("credit_limit")
  isActive        Boolean   @default(true) @map("is_active")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt @map("updated_at")

  customer     Customer                   @relation(fields: [customerId], references: [id])
  transactions CurrentAccountTransaction[]

  // Balance total = physicalBalance + virtualBalance
  @@map("current_accounts")
}

/// Cada movimiento en la cuenta corriente
model CurrentAccountTransaction {
  id               String          @id @default(cuid())
  currentAccountId String          @map("current_account_id")
  type             TransactionType
  moneyType        MoneyType       @map("money_type")  // PHYSICAL o VIRTUAL
  amount           Decimal         @db.Decimal(12, 2)
  balanceAfter     Decimal         @db.Decimal(12, 2) @map("balance_after")  // Snapshot del balance tras la transacción
  description      String
  referenceId      String?         @map("reference_id")  // orderId, invoiceId, etc.
  createdAt        DateTime        @default(now()) @map("created_at")
  createdBy        String          @map("created_by")  // userId

  currentAccount CurrentAccount @relation(fields: [currentAccountId], references: [id])

  @@index([currentAccountId, createdAt])
  @@map("current_account_transactions")
}

// ─────────────────────────────────────────────
// FINANZAS — CIERRE DE CAJA
// ─────────────────────────────────────────────

model CashRegister {
  id                  String             @id @default(cuid())
  employeeId          String             @map("employee_id")
  status              CashRegisterStatus @default(OPEN)
  openingCash         Decimal            @db.Decimal(12, 2) @map("opening_cash")
  
  // Totales calculados al cierre:
  totalCashSales      Decimal            @default(0) @db.Decimal(12, 2) @map("total_cash_sales")
  totalTransferSales  Decimal            @default(0) @db.Decimal(12, 2) @map("total_transfer_sales")
  totalWalletSales    Decimal            @default(0) @db.Decimal(12, 2) @map("total_wallet_sales")
  totalCurrentAccount Decimal            @default(0) @db.Decimal(12, 2) @map("total_current_account")
  totalSales          Decimal            @default(0) @db.Decimal(12, 2) @map("total_sales")
  
  expectedCash        Decimal?           @db.Decimal(12, 2) @map("expected_cash")
  actualCash          Decimal?           @db.Decimal(12, 2) @map("actual_cash")
  cashDifference      Decimal?           @db.Decimal(12, 2) @map("cash_difference")  // actualCash - expectedCash
  
  notes               String?
  openedAt            DateTime           @default(now()) @map("opened_at")
  closedAt            DateTime?          @map("closed_at")

  employee Employee  @relation(fields: [employeeId], references: [id])
  payments Payment[]

  @@map("cash_registers")
}

// ─────────────────────────────────────────────
// ABASTECIMIENTO / COMPRAS A PROVEEDORES
// ─────────────────────────────────────────────

model SupplyOrder {
  id            String            @id @default(cuid())
  orderNumber   String            @unique @map("order_number")  // "SUP-2024-00001"
  supplierId    String            @map("supplier_id")
  status        SupplyOrderStatus @default(PENDING)
  total         Decimal           @db.Decimal(12, 2)
  notes         String?
  expectedAt    DateTime?         @map("expected_at")
  receivedAt    DateTime?         @map("received_at")
  createdAt     DateTime          @default(now()) @map("created_at")
  updatedAt     DateTime          @updatedAt @map("updated_at")
  createdBy     String            @map("created_by")

  supplier Supplier         @relation(fields: [supplierId], references: [id])
  items    SupplyOrderItem[]

  @@map("supply_orders")
}

model SupplyOrderItem {
  id            String    @id @default(cuid())
  supplyOrderId String    @map("supply_order_id")
  rawMaterialId String    @map("raw_material_id")
  quantity      Decimal   @db.Decimal(12, 4)
  unitCost      Decimal   @db.Decimal(12, 2) @map("unit_cost")
  subtotal      Decimal   @db.Decimal(12, 2)

  supplyOrder SupplyOrder @relation(fields: [supplyOrderId], references: [id], onDelete: Cascade)
  rawMaterial RawMaterial @relation(fields: [rawMaterialId], references: [id])

  @@map("supply_order_items")
}
```

---

## 4. Descripción de Módulos

### 4.1 AuthModule

Delega la creación de usuarios a Supabase Auth. Los tokens JWT de Supabase se validan con `JwtStrategy`. El guard de roles se aplica a nivel de controlador o ruta con el decorador `@Roles()`.
