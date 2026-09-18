# SOLID & Clean Code in Leap

**Sprint Review — Speaking Notes**

Not textbook definitions — one real example from our own code for each principle, so if someone asks "where's that actually used," there's a specific class and method to point at.

## Single Responsibility

**Example: `PositionManager` vs. `PositionService`**

A good example is `PositionManager` versus `PositionService`. `PositionManager` talks to the repository — it finds positions, creates them, saves them, deletes them when they hit zero. `PositionService` does none of that — it's pure math, `applyBuy` and `applySell`, recalculating average cost or subtracting quantity, with zero repository dependency. Two classes, two separate reasons to change: one changes if we swap storage, the other changes if we change how average cost gets calculated. Same pattern with `AccountService` — it only ever does credit and debit, nothing else touches the cash balance.

## Open/Closed

**Example: `OrderExecutionStrategy`**

`OrderProcessor` picks between `BuyOrderStrategy` and `SellOrderStrategy` through one shared interface, `OrderExecutionStrategy`, instead of an if-else block with the logic inline. If we ever add a LIMIT or STOP order type — which is actually called out in the interface's own comments as the reason it's built this way — we write a new class that implements `OrderExecutionStrategy`. `OrderProcessor` doesn't change at all. It's open to new order types, closed to modification.

## Liskov Substitution

**Example: Strategy swap, Asset subclasses**

Anywhere `OrderProcessor` expects an `OrderExecutionStrategy`, it can be handed a `BuyOrderStrategy` or a `SellOrderStrategy` interchangeably, and it behaves correctly either way — `OrderProcessor` never needs to know or check which one it's holding. Same idea on the model side: `Stock`, `Bond`, and `Etf` all extend `Asset`, and anything that works with an `Asset` works with any of the three without special-casing.

## Interface Segregation

**Example: Narrow repository interfaces**

Our repository interfaces are narrow and specific to each entity, not one bloated interface every repository has to implement. `OrderRepository` doesn't carry `PositionRepository`'s composite-key methods, and vice versa. `OrderExecutionStrategy` is even tighter — one method, `execute`. Implementers aren't forced to fill in methods they don't need.

## Dependency Inversion

**Example: Services depend on interfaces, not InMemory\***

`OrderService` and `OrderValidationService` depend on the `OrderRepository` and `PositionRepository` interfaces, never on `InMemoryOrderRepository` or `InMemoryPositionRepository` directly. That's the whole reason we can swap in a real database later without touching a single line of service code — the high-level logic depends on the abstraction, not the concrete implementation.

## Clean code, more generally

If they don't ask about SOLID specifically:

A few things we're consistent about:
- **Fail-fast validation** — every service method checks its inputs first and throws immediately instead of letting bad state creep further in.
- **Small, single-purpose methods** — `OrderValidationService` splits `validateQuantity` and `validatePrice` out instead of one long method doing everything inline.
- **Specific exceptions instead of generic ones** — everything extends a common `TradingException` base, but each subtype, `InsufficientFundsException`, `InsufficientHoldingsException`, and so on, carries its own clear message.
- **`BigDecimal` for every money value** instead of `double`, specifically to avoid floating-point rounding errors on cash and quantities.
