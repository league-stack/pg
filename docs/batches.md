# batches

```go
res, err := conn.Batch(ctx,
	pg.Statement{SQL: "SELECT set_config('app.tenant', $1, true)", Args: []any{tenant}},
	pg.Statement{SQL: "SELECT id, name FROM projects"},
)
```

a batch goes out in one write and comes back in one round trip, with one `*pg.Result` per statement, in order.

## one transaction

outside a transaction, postgres runs a batch as one of its own. every statement applies or none does, and `set_config(..., true)` lasts until the batch ends, as long as no statement in it begins, commits or rolls back a transaction. inside `Tx` a batch joins `Tx`'s transaction.

a batch that leaves a transaction open is rolled back and is an error, so a setting made in it can't leak into the session's next use.

`VACUUM` and other statements that can't run in a transaction block only work as a batch's first statement.

## failures

the first statement to fail is a `*pg.BatchError`, with its `Index` counted from 0 and the `Err` behind it. a failure when the batch commits is blamed on its last statement.

## size

every row is read into memory, so batches are for bounded results.
