# Reference Solution — Transactional Transfer

This example demonstrates a transaction boundary and a rollback test. It assumes a relational database and a configured Spring transaction manager. It omits account lookup implementation details, which must be supplied by the chosen persistence layer.

## Service implementation

```java
@Service
public class TransferService {
    private final AccountRepository accounts;

    public TransferService(AccountRepository accounts) {
        this.accounts = accounts;
    }

    @Transactional
    public void transfer(long fromId, long toId, BigDecimal amount) {
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        Account from = accounts.findById(fromId)
            .orElseThrow(() -> new AccountNotFoundException(fromId));
        Account to = accounts.findById(toId)
            .orElseThrow(() -> new AccountNotFoundException(toId));

        if (from.balance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        from.debit(amount);
        to.credit(amount);
        accounts.save(from);
        accounts.save(to);
    }
}
```

The entities, repository, and domain methods must be implemented for the chosen JPA/JDBC stack. If managed JPA entities are modified inside a transaction, explicit saves may not be necessary; the persistence context can detect changes. Do not blindly copy this fragment without matching the persistence model.

## Design decisions

- The method is a business transaction boundary.
- Validation that can happen before data access is performed early.
- Business failures are unchecked in this example, matching the common default rollback behavior.
- The call must enter through the Spring-managed proxy.
- A remote notification should not be published as an uncoordinated side effect inside this method. Use an outbox if durable event publication is required.

## Rollback test strategy

1. Seed two accounts with known balances.
2. Arrange a failure after the first in-memory mutation but before commit, or use a test hook that throws after the first persistence operation.
3. Invoke the service through its Spring proxy.
4. Clear the persistence context or query in a new transaction.
5. Assert both persisted balances equal their original values.

Avoid testing rollback only by inspecting the same managed entity instance; it may contain in-memory mutations even though the database transaction rolled back.

## Common failure modes

- Catching the exception and returning normally before rollback is triggered.
- Calling the method through self-invocation.
- Selecting the wrong transaction manager for the repository.
- Making remote calls while holding database locks.
- Assuming two local transaction managers atomically cover two databases.
- Ignoring concurrent updates; add optimistic or pessimistic locking according to the business invariant.

## Validation checklist

- [ ] Invalid amount does not mutate accounts.
- [ ] Insufficient funds leaves both balances unchanged.
- [ ] Missing account fails clearly.
- [ ] Forced mid-operation failure rolls back both database updates.
- [ ] Concurrent transfers preserve balance invariants.
- [ ] The integration test uses the production database engine for locking/isolation behavior that matters.
