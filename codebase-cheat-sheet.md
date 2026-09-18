# Leap Codebase Cheat Sheet

**Sprint Review — Prep Notes**

You don't need to know Java to explain this codebase — you need to know what each folder is for and how a request moves through them. The syntax is just how the logic gets written down; the logic itself is the same business rules already covered in the diagrams.

## The folders, translated

Everything lives under `src/main/java/com/neueda/leap`.

| Folder | Description |
|--------|-------------|
| **models/** | The "nouns." Account, Order, Position, Instrument, and Asset (with Stock, Bond, Etf underneath it). Mostly fields plus getters and setters — the only real logic is a couple of small calculations like Position's cost basis. |
| **enums/** | Closed lists of allowed values. AccountStatus is ACTIVE, INACTIVE, or SUSPENDED. OrderSide is BUY or SELL. OrderStatus is NEW, FILLED, REJECTED, or CANCELLED. If someone asks what states a thing can be in, the answer is always one of these lists. |
| **repositories/** | The "filing cabinets." One interface per entity describing save, find, delete, and exists, and one in-memory implementation of each, backed by a Map, since there's no real database yet. |
| **services/** | The "rulebook." Where the actual business logic lives — OrderService, OrderValidationService, AccountService, PositionManager, PositionService. |
| **services/orderServices/** | The "execution engine." OrderProcessor, plus BuyOrderStrategy and SellOrderStrategy, which are the only two places that actually move cash and shares. |
| **exceptions/** | The named failure reasons. Every one of them extends a single base class, TradingException, so a method that can fail says exactly how, in its own signature. |
| **dtos/** | The "wire shapes" — request and response objects for an API layer. Not part of the trading logic itself, just the shape of data going in and out. |

## Java syntax decoder

You'll see these patterns constantly. Here's what they actually mean.

| You'll see | It means |
|------------|----------|
| `class X implements Y` | X is one concrete way of doing what the interface Y promises — like InMemoryOrderRepository implementing OrderRepository. |
| `class X extends Y` | X is a specific kind of Y. Stock extends Asset. Every specific exception extends TradingException. |
| `private final Type name;` | A field that gets set once, in the constructor, and never reassigned after that. |
| `public X(...) { ... }` | The constructor — the checks here are what has to be true for an X to exist at all. |
| `throws SomeException` | This method can fail in this specific, named way. The caller has to catch it or pass it further up. |
| `Optional<Position>` | A value that might not exist. `.map(...)` transforms it if present, `.orElse(x)` supplies a fallback if it's empty. |
| `BigDecimal` | A money-safe number type. Never a plain double, because doubles round money incorrectly. |
| `@Override` | This method is filling in a promise made by an interface or a parent class. |

## If they point at a file

One line each — enough to answer "what does this do?" cold.

### Models
- **Account.java** — A bank-account-style record: id, holder, cash balance, status, plus version and timestamp fields used for optimistic locking.
- **Order.java** — One buy or sell request. Its constructor defaults the status to NEW and the version to zero.
- **Position.java** — How much of a symbol an account holds, and at what average cost. Two positions are equal if they share the same account and symbol, nothing else.
- **Instrument.java** — Reference data for something tradable — symbol, exchange, currency, whether it's currently tradable.
- **Asset.java / Stock.java / Bond.java / Etf.java** — Asset is abstract, so it can never be created on its own. Stock, Bond, and Etf are the concrete kinds, each adding its own extra fields.

### Repositories
- **AccountRepository, OrderRepository, InstrumentRepository, PositionRepository** — Interfaces — just contracts. No logic lives in them.
- **InMemory*Repository classes** — The only implementations that exist right now. Each one is a Map wrapped in save, find, delete, and exists.

### Services
- **AccountService.java** — Two methods, credit and debit. The only place cashBalance ever gets touched directly.
- **OrderValidationService.java** — Every rule that can reject an order lives here, and nowhere else.
- **OrderService.java** — Creates and saves orders, and guards against the same idempotency key being used twice.
- **PositionManager.java** — Talks to the repository. Finds or creates positions, and deletes one outright once it hits zero.
- **PositionService.java** — Pure math, no repository calls. applyBuy recalculates the average cost, applySell subtracts and checks for overselling.

### Order Processing
- **OrderProcessor.java** — The conductor. Calls OrderService to validate and create the order, then picks a strategy to execute it.
- **BuyOrderStrategy.java / SellOrderStrategy.java** — The only two places that move cash and shares together, with a manual rollback in the catch block if something fails partway through.

### Other
- **Main.java** — Still just prints "Hello world." Nothing wires the trading flow to a real entry point yet.

## The exception family

Why every method signature is covered in `throws`.

Every failure in the trading logic is a subclass of one base class, `TradingException`, which itself extends Java's built-in Exception. That makes it a "checked" exception — any method that can throw one has to declare it, which is why you see long `throws` lists on methods like `placeOrder` and `validateOrder`. It's not clutter, it's the method being explicit about every way it can fail.

The children: `InvalidOrderException` (bad input — null account, non-positive quantity or price), `AccountNotActiveException` (account isn't ACTIVE), `InstrumentNotFoundException` (instrument doesn't exist), `TradingException` itself also covers instrument-not-tradable, `InsufficientFundsException` (not enough cash), `InsufficientHoldingsException` (not enough shares to sell), and `DuplicateOrderException` (idempotency key reused).

---

*Companion to the UML walkthrough. Verified against `src/main/java/com/neueda/leap` on `develop`.*
