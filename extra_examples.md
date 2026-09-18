# SOLID & Clean Code in Leap — Extended Examples

Companion to [`solid-clean-code (1).md`](solid-clean-code%20(1).md). That doc has one example per principle for the sprint review talk track; this one goes deeper — more code, more principles per class, and a couple of "what if we did it wrong" counter-examples so the reasoning is concrete, not just asserted.

All code below is quoted directly from the codebase (paths given per snippet) so it stays checkable against source.

---

## Single Responsibility Principle (SRP)

**A class should have one reason to change.**

### Example 1: `OrderValidationService` vs `OrderService` vs `OrderProcessor`

This is actually a three-way split, not two:

- [`OrderValidationService`](src/main/java/com/neueda/leap/services/OrderValidationService.java) — only decides whether an order *would be* legal. No saving, no side effects.
- [`OrderService`](src/main/java/com/neueda/leap/services/OrderService.java) — only builds and persists orders, and enforces idempotency.
- [`OrderProcessor`](src/main/java/com/neueda/leap/services/orderServices/OrderProcessor.java) — only orchestrates: call `OrderService`, pick a strategy, run it.

```java
// OrderValidationService.java — reason to change: a trading rule changes
public void validateBuyOrder(Account account, BigDecimal quantity, BigDecimal price)
        throws InsufficientFundsException, InvalidOrderException {
    validateQuantity(quantity);
    validatePrice(price);

    BigDecimal requiredBalance = quantity.multiply(price);
    BigDecimal availableBalance = account.getCashBalance();

    if (availableBalance.compareTo(requiredBalance) < 0) {
        throw new InsufficientFundsException(
            "Insufficient funds for BUY order. Required: $" + requiredBalance +
            ", Available: $" + availableBalance
        );
    }
}
```

```java
// OrderService.java — reason to change: persistence / idempotency handling changes
private synchronized Order createOrder(Order order) throws DuplicateOrderException {
    if (order.getIdempotencyKey() == null || order.getIdempotencyKey().trim().isEmpty()) {
        throw new IllegalArgumentException("Idempotency key is required");
    }
    if (ordersByIdempotencyKey.containsKey(order.getIdempotencyKey())) {
        throw new DuplicateOrderException(order.getIdempotencyKey(), "idempotencyKey");
    }
    Order savedOrder = orderRepository.save(order);
    ordersByIdempotencyKey.put(savedOrder.getIdempotencyKey(), savedOrder);
    return savedOrder;
}
```

```java
// OrderProcessor.java — reason to change: the execution workflow steps change
public OrderResult processOrder(Account account, Instrument instrument, OrderSide side,
        BigDecimal quantity, BigDecimal price, String idempotencyKey) throws ... {

    Order order = orderService.placeOrder(account, instrument, side, quantity, price, idempotencyKey);

    OrderExecutionStrategy strategy = switch (side) {
        case BUY -> buyStrategy;
        case SELL -> sellStrategy;
        case null -> throw new InvalidOrderException("Order side cannot be null");
    };

    OrderResult result = strategy.execute(order, account, instrument);
    orderService.saveOrder(order);
    return result;
}
```

If validation logic lived inside `OrderService`, a change to a trading rule (say, a minimum order size) would require touching the same class as a change to how orders are persisted — two unrelated stakeholders (compliance vs. infra) editing the same file.

### Example 2: `AccountService` does exactly two things

```java
// AccountService.java
public void credit(Account account, BigDecimal amount) throws AccountNotActiveException { ... }
public void debit(Account account, BigDecimal amount)
        throws AccountNotActiveException, InsufficientFundsException { ... }
```

Nothing else is allowed to mutate `cashBalance` directly — `BuyOrderStrategy` and `SellOrderStrategy` both go through `accountService.debit(...)` / `.credit(...)` rather than reaching into `Account` themselves. One class owns cash-balance invariants (must be active, amount must be positive, can't go negative).

### Counter-example — what SRP violation would look like here

```java
// NOT how it's done — hypothetical violation
public class OrderService {
    public Order placeOrder(...) {
        // validation inline
        if (!account.isActive()) throw new AccountNotActiveException(...);
        if (quantity.compareTo(BigDecimal.ZERO) <= 0) throw new InvalidOrderException(...);
        // persistence inline
        Order order = new Order(...);
        database.save(order);
        // notification inline
        emailService.sendConfirmation(account, order);
        return order;
    }
}
```

Three reasons to change (validation rules, persistence mechanism, notification format) baked into one method. Our actual `OrderService` only has the persistence/idempotency reason.

---

## Open/Closed Principle (OCP)

**Open for extension, closed for modification.**

### Example 1: `OrderExecutionStrategy`

```java
// OrderExecutionStrategy.java
public interface OrderExecutionStrategy {
    OrderResult execute(Order order, Account account, Instrument instrument);
}
```

```java
// OrderProcessor.java — dispatches without an if/else on order-type logic
OrderExecutionStrategy strategy = switch (side) {
    case BUY -> buyStrategy;
    case SELL -> sellStrategy;
    case null -> throw new InvalidOrderException("Order side cannot be null");
};
OrderResult result = strategy.execute(order, account, instrument);
```

The `switch` only *selects* a strategy object — it doesn't contain BUY/SELL business logic. That logic lives entirely inside [`BuyOrderStrategy`](src/main/java/com/neueda/leap/services/orderServices/BuyOrderStrategy.java) and [`SellOrderStrategy`](src/main/java/com/neueda/leap/services/orderServices/SellOrderStrategy.java):

```java
// BuyOrderStrategy.java
@Override
public OrderResult execute(Order order, Account account, Instrument instrument) {
    BigDecimal totalCost = order.getPrice().multiply(BigDecimal.valueOf(order.getQuantity()));
    accountService.debit(account, totalCost);
    positionService.updatePositionAfterBuy(account.getAccountId(), order.getSymbol(),
        order.getQuantity(), order.getPrice());
    order.setOrderStatus(OrderStatus.FILLED);
    ...
}
```

**Adding a LIMIT order type** — which the interface's own javadoc calls out as the reason it's built this way — means:

```java
// Hypothetical addition, zero changes to OrderProcessor, BuyOrderStrategy, SellOrderStrategy
public class LimitOrderStrategy implements OrderExecutionStrategy {
    @Override
    public OrderResult execute(Order order, Account account, Instrument instrument) {
        // book the order instead of filling immediately, wait for price match
    }
}
```

`OrderProcessor`'s constructor would take a `LimitOrderStrategy` alongside the existing two, and the `switch` gains one more `case`. Nothing about `BuyOrderStrategy` or `SellOrderStrategy` changes — they're closed for modification.

### Counter-example — what it looks like without the strategy pattern

```java
// NOT how it's done — hypothetical violation
public OrderResult processOrder(Order order, ...) {
    if (order.getSide() == OrderSide.BUY) {
        accountService.debit(account, totalCost);
        positionService.updatePositionAfterBuy(...);
    } else if (order.getSide() == OrderSide.SELL) {
        accountService.credit(account, totalProceeds);
        positionService.updatePositionAfterSell(...);
    } else if (order.getSide() == OrderSide.LIMIT) {   // <- modifying existing method
        // ...
    }
}
```

Every new order type means editing a method that BUY and SELL already depend on — one bug in the new branch risks breaking the existing ones, and it's a bigger diff to review.

---

## Liskov Substitution Principle (LSP)

**Subtypes must be usable wherever their supertype is expected, without surprises.**

### Example 1: Strategy substitutability

```java
// OrderProcessor.java — holds an OrderExecutionStrategy reference, never checks concrete type
OrderResult result = strategy.execute(order, account, instrument);
```

Both implementations honor the same contract — take an `Order`/`Account`/`Instrument`, return an `OrderResult`, never throw past the interface, always leave the order in either `FILLED` or `REJECTED` status:

```java
// BuyOrderStrategy.java — success path
order.setOrderStatus(OrderStatus.FILLED);
return new OrderResult(true, successMessage, null);

// BuyOrderStrategy.java — failure path, rolls back its own side effect first
if (cashDebited) {
    accountService.credit(account, totalCost);   // undo the debit
}
order.setOrderStatus(OrderStatus.REJECTED);
return new OrderResult(false, failureMessage, null);
```

```java
// SellOrderStrategy.java — mirrors the same contract with credit/debit swapped
if (cashCredited) {
    accountService.debit(account, totalProceeds);  // undo the credit
}
order.setOrderStatus(OrderStatus.REJECTED);
return new OrderResult(false, failureMessage, null);
```

`OrderProcessor` never has to ask "is this the buy one or the sell one?" before calling `execute` — both keep the same pre/post-conditions (order ends in a terminal status, account state is only mutated on success, `OrderResult` is always returned rather than an exception escaping).

### Example 2: `Asset` subclasses

```java
// Asset.java — the contract every subclass inherits
public abstract class Asset {
    public String getSymbol() { ... }
    public BigDecimal getPrice() { ... }
    ...
}
```

```java
// Stock.java
public class Stock extends Asset {
    public Stock(String symbol, String name, BigDecimal price, LocalDateTime tradeDate,
            String sector, ... ) {
        super(symbol, name, price, tradeDate);
        ...
    }
}
```

```java
// Bond.java
public class Bond extends Asset {
    public Bond(String symbol, String name, BigDecimal price, LocalDateTime tradeDate,
            String category, ... ) {
        super(symbol, name, price, tradeDate);
        ...
    }
}
```

Anything that only needs `Asset`'s contract (`getSymbol()`, `getPrice()`, `getTradeDate()`) works identically whether it's holding a `Stock`, `Bond`, or `Etf` — none of them narrow the constructor's guarantees, throw on inherited getters, or require extra null-checks the base type didn't promise.

### Counter-example — an LSP violation

```java
// NOT how it's done — hypothetical violation
public class SuspendedStrategy implements OrderExecutionStrategy {
    @Override
    public OrderResult execute(Order order, Account account, Instrument instrument) {
        throw new UnsupportedOperationException("Trading suspended");
        // breaks the implicit contract: execute() is expected to return an
        // OrderResult, not throw — callers that only catch the interface's
        // declared exceptions would blow up
    }
}
```

Swapping this in for `buyStrategy` would break `OrderProcessor` even though it "implements the interface," because it violates the behavioral contract the other implementations honor.

---

## Interface Segregation Principle (ISP)

**No client should be forced to depend on methods it doesn't use.**

### Example 1: `OrderRepository` vs `PositionRepository`

```java
// OrderRepository.java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(String orderId);
    List<Order> findByAccountId(String accountId);
    boolean delete(String orderId);
    boolean exists(String orderId);
}
```

```java
// PositionRepository.java
public interface PositionRepository {
    Position save(Position position);
    Optional<Position> findByAccountAndSymbol(String accountId, String symbol);
    List<Position> findByAccountId(String accountId);
    boolean delete(String accountId, String symbol);
    boolean exists(String accountId, String symbol);
}
```

Note `PositionRepository` keys on `(accountId, symbol)` — a composite key — while `OrderRepository` keys on a single `orderId`. They look similar (`save`, `findByAccountId`, `delete`, `exists`) but aren't the same shape, and keeping them separate means an `OrderRepository` implementer is never forced to implement a `findByAccountAndSymbol(String, String)` it has no use for.

### Example 2: `OrderExecutionStrategy` — a one-method interface

```java
public interface OrderExecutionStrategy {
    OrderResult execute(Order order, Account account, Instrument instrument);
}
```

This is ISP taken to its logical minimum: `BuyOrderStrategy` and `SellOrderStrategy` implement exactly one method each. Compare to a fatter interface:

```java
// NOT how it's done — hypothetical violation
public interface OrderExecutionStrategy {
    OrderResult execute(Order order, Account account, Instrument instrument);
    OrderResult cancel(Order order);
    OrderResult amend(Order order, BigDecimal newQuantity);
    BigDecimal estimateFees(Order order);
}
```

If BUY and SELL never support amendment or fee estimation, every implementer would be stuck writing `throw new UnsupportedOperationException()` stubs for methods it can't honor — exactly the smell ISP exists to prevent.

---

## Dependency Inversion Principle (DIP)

**High-level modules depend on abstractions, not concrete implementations.**

### Example: constructor injection all the way down

```java
// OrderService.java — depends on the interfaces...
private final OrderValidationService validationService;
private final OrderRepository orderRepository;

public OrderService(PositionRepository positionRepository, OrderRepository orderRepository) {
    this.validationService = new OrderValidationService(positionRepository);
    this.orderRepository = orderRepository;
}
```

```java
// OrderValidationService.java — same pattern
private final PositionRepository positionRepository;

public OrderValidationService(PositionRepository positionRepository) {
    this.positionRepository = positionRepository;
}
```

```java
// OrderService.java — the ONLY place a concrete InMemory* type is even mentioned,
// and only as a default convenience constructor, not something service logic reaches for
public OrderService(PositionRepository positionRepository) {
    this(positionRepository, new InMemoryOrderRepository());
}
```

Every method on `OrderService` and `OrderValidationService` — `placeOrder`, `validateOrder`, `validateBuyOrder`, etc. — is written entirely against `OrderRepository` / `PositionRepository` method signatures. Swapping `InMemoryOrderRepository` for a JDBC- or JPA-backed implementation later means writing one new class that implements `OrderRepository` and changing the constructor call site — no service method changes.

### Counter-example — what DIP violation would look like

```java
// NOT how it's done — hypothetical violation
public class OrderService {
    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();

    public Order placeOrder(...) {
        // ... orderRepository.save(order) ...
    }
}
```

Here `OrderService` (high-level policy: "how do we place an order") is directly coupled to `InMemoryOrderRepository` (low-level detail: "how do we store it"). Moving to a real database means editing `OrderService` itself, and unit-testing `OrderService` in isolation (no real repository) becomes impossible without a mocking framework that can intercept `new`.

---

## Clean Code, More Generally

### Fail-fast validation

Every service method checks its inputs before doing anything else:

```java
// OrderService.java
public OrderService(PositionRepository positionRepository, OrderRepository orderRepository) {
    if (positionRepository == null) {
        throw new IllegalArgumentException("PositionRepository cannot be null");
    }
    if (orderRepository == null) {
        throw new IllegalArgumentException("OrderRepository cannot be null");
    }
    ...
}
```

```java
// AccountService.java
public void debit(Account account, BigDecimal amount)
        throws AccountNotActiveException, InsufficientFundsException {
    if (account == null) {
        throw new IllegalArgumentException("Account cannot be null");
    }
    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new IllegalArgumentException("Debit amount must be positive");
    }
    if (!account.isActive()) {
        throw new AccountNotActiveException("Cannot debit an inactive account");
    }
    if (account.getCashBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException(...);
    }
    // only now do we touch state
    account.setCashBalance(account.getCashBalance().subtract(amount));
    ...
}
```

### Small, single-purpose methods

`OrderValidationService` splits validation into named pieces instead of one long method:

```java
private void validateQuantity(BigDecimal quantity) throws InvalidOrderException {
    if (quantity == null || quantity.compareTo(BigDecimal.ZERO) <= 0) {
        throw new InvalidOrderException("Quantity must be positive");
    }
    if (quantity.stripTrailingZeros().scale() > 0) {
        throw new InvalidOrderException("Quantity must be a whole number of shares");
    }
    if (quantity.compareTo(BigDecimal.valueOf(Integer.MAX_VALUE)) > 0) {
        throw new InvalidOrderException("Quantity exceeds the maximum supported value");
    }
}

private void validatePrice(BigDecimal price) throws InvalidOrderException {
    if (price == null || price.compareTo(BigDecimal.ZERO) <= 0) {
        throw new InvalidOrderException("Price must be positive");
    }
}
```

`validateBuyOrder` and `validateSellOrder` both call these instead of repeating the checks inline — each method reads as one coherent step, and a bug in "what counts as a valid quantity" is fixed in exactly one place.

### Specific exceptions instead of generic ones

```java
// TradingException.java — common base
public class TradingException extends Exception {
    public TradingException(String message) { super(message); }
    public TradingException(String message, Throwable cause) { super(message, cause); }
}
```

```java
// InsufficientFundsException.java — carries its own context, own overloads
public class InsufficientFundsException extends TradingException {
    public InsufficientFundsException(BigDecimal required, BigDecimal available, String accountId) {
        super("Insufficient funds in account " + accountId +
              ": required " + required + ", available " + available);
    }
    public InsufficientFundsException(String message) { super(message); }
    public InsufficientFundsException() { super("Insufficient funds"); }
}
```

```java
// InsufficientHoldingsException.java — same pattern, different domain
public class InsufficientHoldingsException extends TradingException {
    public InsufficientHoldingsException(String symbol, BigDecimal requestedQuantity,
                                        BigDecimal availableQuantity, String accountId) {
        super("Insufficient holdings in account " + accountId +
              " for " + symbol + ": requested " + requestedQuantity +
              ", available " + availableQuantity);
    }
}
```

A caller can catch `InsufficientFundsException` specifically to show "top up your account," and separately catch `InsufficientHoldingsException` to show "you don't own that many shares" — versus catching a generic `TradingException` and having to parse the message string to know which happened.

### `BigDecimal` for every money value

Every price, quantity comparison, and balance in `AccountService`, `OrderValidationService`, `BuyOrderStrategy`, and `SellOrderStrategy` uses `BigDecimal`, never `double`:

```java
BigDecimal totalCost = order.getPrice().multiply(BigDecimal.valueOf(order.getQuantity()));
accountService.debit(account, totalCost);
```

```java
if (availableBalance.compareTo(requiredBalance) < 0) { ... }
```

Comparisons go through `.compareTo(...)`, not `==` or `<`/`>`, because `BigDecimal` doesn't overload those operators — and because two `BigDecimal`s representing the same value can differ in scale (`2.50` vs `2.5000`), `.compareTo()` is correct where `.equals()` would not be. Using `double` here would risk rounding errors compounding across trades — e.g. `0.1 + 0.2 != 0.3` in binary floating point — which is unacceptable when the numbers are cash balances.

---

## Quick reference

| Principle | Primary example | Also shown in |
|---|---|---|
| SRP | `OrderValidationService` / `OrderService` / `OrderProcessor` split | `AccountService` (credit/debit only) |
| OCP | `OrderExecutionStrategy` + Buy/Sell strategies | Hypothetical `LimitOrderStrategy` addition |
| LSP | Strategy substitutability in `OrderProcessor` | `Asset` → `Stock` / `Bond` / `Etf` |
| ISP | `OrderRepository` vs `PositionRepository` | Single-method `OrderExecutionStrategy` |
| DIP | `OrderService` / `OrderValidationService` depend on repository interfaces | `InMemoryOrderRepository` only as a default, never referenced in logic |
