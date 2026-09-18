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

## Order Placement – UML Sequence Diagram

*File: order-sequence-uml.md*

### OrderService.placeOrder validating a BUY order, step by step

This traces OrderService.placeOrder for a BUY order, in the exact order the checks run in code. It's fail-fast: the first check that fails stops everything else.

The client calls placeOrder with the account, instrument, side, quantity, price, and an idempotency key. That hands off to OrderValidationService.validateOrder.

First it checks the account. A null account throws InvalidOrderException. An account that isn't ACTIVE throws AccountNotActiveException. Next it checks the instrument: null throws InstrumentNotFoundException, and one that isn't tradable throws TradingException.

Then come the BUY-specific checks. Quantity and price both have to be positive, or it's an InvalidOrderException. Then it checks affordability, comparing quantity times price against the account's cash balance. If the order costs more than the account has, that's InsufficientFundsException.

If everything passes, OrderService builds the Order and calls its internal createOrder, which checks one more thing: has this idempotency key been used before? If so, DuplicateOrderException, because we never want the same request to create two orders. Otherwise the order saves with status NEW and comes back to the caller.

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

## Order Processing – Business Logic Sequence Diagram

*File: order-processing-business-logic.md*

### The full lifecycle through OrderProcessor: validation, execution, and rollback

This is the full order lifecycle, end to end, through OrderProcessor, the class that ties together everything we've covered so far.

It starts with processOrder, which hands off to OrderService.placeOrder. That runs the same validation from the BUY and SELL diagrams: bad account, bad instrument, insufficient funds or holdings, bad input. Any failure stops everything, and OrderProcessor rejects the order right there.

If validation passes, the order already exists with status NEW and is saved. OrderProcessor then picks a strategy based on side, either BuyOrderStrategy or SellOrderStrategy, and calls execute with the order, account, and instrument.

For a BUY: the strategy debits the account for price times quantity. Then PositionManager.updatePositionAfterBuy finds or creates the position, recalculates a weighted-average cost, and saves it.

For a SELL: it's credit instead of debit. updatePositionAfterSell finds the position and subtracts the quantity. If that brings it to zero, the position gets deleted outright rather than left at zero. Otherwise it's just saved with the new quantity.

Either way, once cash and position updates succeed, the strategy marks the order FILLED and returns a result. OrderProcessor saves the order one more time with its final status and returns that result to the caller.
