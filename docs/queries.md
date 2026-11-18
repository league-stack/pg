# queries

| method | does |
| :-- | :-- |
| `conn.Query(ctx, sql, args...)` | runs one statement through the extended protocol, with args as `$1`, `$2` and on, and reads every row |
| `conn.Exec(ctx, sql, args...)` | runs one statement and answers the rows it changed |
| `conn.Script(ctx, sql)` | runs sql through the simple protocol, with no arguments and any number of statements |
| `conn.Batch(ctx, statements...)` | sends many statements in one round trip ([batches](batches.md)) |
| `conn.Tx(ctx, work)` | runs work inside a transaction |
| `conn.Wait(ctx)` | answers the next notification on a channel the session listens to |

```go
res, err := conn.Query(ctx, "SELECT id, name FROM projects WHERE owner = $1", owner)
if err != nil {
	return err
}
for _, row := range res.Rows {
	var id int64
	var name string
	if err := row.Scan(&id, &name); err != nil {
		return err
	}
}
```

a `Result` holds the statement's `Columns`, its `Rows`, all read, and its command `Tag`. `res.Affected()` is the rows it changed, from the tag, so `INSERT 0 3` is 3 and `UPDATE 2` is 2.

## values

values travel as text. an argument may be nil (NULL), a string, a bool, an int or uint of any size, a finite float32 or float64, a `time.Time`, a `[]byte` (sent as bytea) or a `[]string` (sent as `text[]`). a nil pointer is NULL and any other pointer is what it points at.

`row.Scan` reads one destination per value. a destination is a pointer to a string, `[]byte`, bool, int or uint of any size, float32, float64, `time.Time` or `[]string`. a pointer to a pointer of one of those takes NULL as nil, and NULL into anything else is an error. a `[]byte` is for bytea. columns count from 1, as arguments do.

the session's time zone is UTC, so every time is written and read in it.

## scripts

`Script` is for what needs the simple protocol, such as migrations, `BEGIN`, `COMMIT` and `LISTEN`. rows are read and dropped. the first error the server sends is the answer, and a `COMMIT` that rolled back a failed transaction is an error too.

## transactions

`Tx` commits when work succeeds, and rolls back when work fails or panics, or when a statement in it failed and work went on. work that commits or rolls back itself is an error. it refuses to begin inside an open transaction, so an inner `Tx` can't commit an outer one.

```go
err := conn.Tx(ctx, func() error {
	_, err := conn.Exec(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from)
	return err
})
```

- `pg.ErrRolledBack` is a commit that found its transaction failed. postgres rolls it back and answers `COMMIT` with a `ROLLBACK` tag, not an error.
- `pg.ErrCommitUnknown` is a `COMMIT` that went out and got no answer, as when ctx ends or the session is cut while `Tx` waits for one. the work may have been applied, so running it again could apply it twice.

`conn.TxStatus()` is the server's transaction status after the last statement, `I` idle, `T` in a transaction or `E` in a failed one.

## listen and notify

`LISTEN` through `Script`, then `Wait` answers the next `pg.Notification` (`PID`, `Channel`, `Payload`), oldest first, waiting for one until ctx ends. a wait that ends before any message starts leaves the session good. a wait is ended through its ctx, not by `Close` from another goroutine.

## errors

an error the server sent is a `*pg.Error`, with `Severity`, `Code` (the sqlstate, such as `23505` for a unique violation), `Message`, `Detail`, `Hint` and `Constraint`.

```go
var pe *pg.Error
if errors.As(err, &pe) && pe.Code == "23505" {
	return errDuplicate
}
```

## a session that can't be used

`conn.Err()` says why a session can't be used any more (it was closed, cut by a cancelled context, or broken by the server), and is nil while it can. a pool drops a session once it has one.

`conn.Close()` ends the session. closing one that's broken or closed already does nothing.
