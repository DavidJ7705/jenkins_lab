# Leap UML Walkthrough

**Sprint Review — Speaking Notes**

## Domain Model – UML Class Diagram

*File: domain-uml.md*

### The five entities everything in Leap is built from

This is our domain model. Five classes matter: Account, Order, Instrument, Position, and Asset, which is an abstract class that Stock, Bond, and Etf all extend.

Account holds a cash balance and a status: ACTIVE, INACTIVE, or SUSPENDED. Order represents one buy or sell request. It has a side, a quantity, a price, and an idempotency key, so the same request can't accidentally create two orders. Its status moves through NEW, FILLED, REJECTED, or CANCELLED.

Instrument is reference data: symbol, exchange, currency, and whether it's tradable. Position links an account to an instrument, recording how many shares they hold and at what average cost. It also has helper methods for cost basis and market value.

Here's the key point of this diagram: none of these classes hold a direct reference to each other in code. Everything is joined by matching IDs, like accountId and symbol, not object pointers. Only one of those links is actually enforced in Java: Order to Instrument, inside OrderService and OrderValidationService. The rest — Position, Asset, and Price pointing at Instrument — are enforced only by a database foreign key. Nothing in the Java code checks them yet.

## Persistence Layer – UML Class Diagram

*File: persistance-uml.md*

### Why the services never touch a Map directly

This is the persistence layer, and it follows the repository pattern. Every domain entity — Account, Order, Instrument, Position — has a repository interface with the same shape: save, a find method, delete, and exists.

Right now each interface has exactly one implementation, and it's in-memory: InMemoryAccountRepository, InMemoryOrderRepository, and so on, each backed by a Map. These aren't test doubles. They're the entire persistence layer today, since there's no real database wired in yet.

The point of the interface is that the services, like OrderService and AccountService, only ever depend on it and never on the in-memory class directly. So when we do add a database, we can swap in a new OrderRepository implementation and nothing in OrderService has to change.

## Order Processing Logic – UML Class Diagram

*File: order-logic-uml.md*

### The business classes behind placing and executing an order

This replaces the old BUY-order and full-lifecycle sequence diagrams with a single class diagram. A sequence diagram is great for tracing one call path step by step, but order processing now has two separate concerns — validating/creating an order, and executing it — so a class diagram showing how the pieces are wired together is the clearer picture. Persistence repositories and detailed model fields are covered in the domain and persistence diagrams; this one stays focused on the business classes.

`OrderProcessor` is the orchestrator — `processOrder` is its one public entry point, and it returns an `OrderResult`. It depends on `OrderService` to create and save orders, and on `OrderExecutionStrategy` to pick BUY or SELL behaviour.

`OrderService` creates the `Order` and validates it by calling through to `OrderValidationService`, which in turn checks the `Account` (funds and active status) and the `Instrument` (tradability).

`OrderExecutionStrategy` is the interface both `BuyOrderStrategy` and `SellOrderStrategy` implement: `execute(Order, Account, Instrument)` returns an `OrderResult`. BUY debits cash and increases the position; SELL credits cash and decreases it. To do that, the strategy calls `AccountService` (`credit`/`debit`, which updates the `Account`'s cash balance) and `PositionService`.

`PositionService` is the one class on this diagram that changed recently: `updatePositionAfterBuy` and `updatePositionAfterSell` are still the entry points a strategy calls, but the actual math now lives in two extracted methods, `applyBuy` and `applySell`. `applyBuy` finds or creates the position, then recalculates a weighted-average cost from the existing cost basis plus the new purchase cost. `applySell` subtracts the sold quantity and throws `InsufficientHoldingsException` if there isn't enough to sell. `updatePositionAfterSell` deletes the position outright if that subtraction brings it to zero, rather than leaving a zero-quantity row behind.

Everything ends the same way regardless of side: the strategy marks the order FILLED or REJECTED, returns an `OrderResult` with `isSuccess()` and `getMessage()`, and `OrderProcessor` saves the order one last time with its final status before handing the result back.

## Sell Order – UML Sequence Diagram

*File: sell-sequence-uml.md*

### OrderService.placeOrder validating a SELL order, step by step

This traces OrderService.placeOrder for a SELL order, run through OrderValidationService.

First it checks the account. A null account throws InvalidOrderException. An account that isn't ACTIVE throws AccountNotActiveException. Next it checks the instrument: null throws InstrumentNotFoundException, and one that isn't tradable throws TradingException.

Then it checks holdings. It calls PositionRepository.findByAccountAndSymbol, which returns an Optional Position, empty if the account doesn't hold that symbol at all. It reads the held quantity off that, treating no position as zero, and compares it to the quantity being sold. Selling more than they hold throws InsufficientHoldingsException.

If everything passes, OrderService builds the Order and calls its internal createOrder, which checks whether this idempotency key has been used before. If so, DuplicateOrderException. Otherwise the order saves with status NEW and comes back to the caller.

## Account Management – UML Sequence Diagram

*File: account-sequence.md*

### AccountService.credit and debit — how cash actually moves

This walks through AccountService.credit and debit, the two methods that move cash on an account.

For credit, the amount is validated first. A null amount throws NullPointerException, and one that's zero or negative throws IllegalArgumentException. Then it checks the account: a null account throws NullPointerException, and one that isn't ACTIVE throws AccountNotActiveException.

Once both checks pass, it adds the amount to the cash balance and increments the version number. Version is there for optimistic locking, so two concurrent updates don't silently overwrite each other. The updated account goes back to the caller.

Debit runs the same two checks in the same order, amount then account, with the same exceptions. It adds one more step before touching the balance: if the requested amount is greater than the current cash balance, that's InsufficientFundsException. Otherwise it subtracts the amount and increments the version, same as credit.

In practice, BuyOrderStrategy calls debit when a BUY order executes, and SellOrderStrategy calls credit when a SELL order executes.
