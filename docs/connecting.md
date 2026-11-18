# connecting

```go
cfg, err := pg.ParseURL("postgres://user:password@db.example:5432/app?sslmode=verify-full&sslrootcert=/etc/ssl/db-ca.pem")
if err != nil {
	return err
}
conn, err := pg.Connect(ctx, cfg)
if err != nil {
	return err
}
defer conn.Close()
```

a `Conn` is one session with the server, for one goroutine at a time.

## the url

the url starts `postgres://` or `postgresql://`. it must name a user. the host defaults to `localhost`, the port to 5432 and the database to the user's name.

the query may set these keys and nothing else.

| key | sets |
| :-- | :-- |
| `host` | a unix socket's directory |
| `application_name` | the name that shows in `pg_stat_activity` |
| `sslmode` | `disable` or `verify-full` |
| `sslrootcert` | a pem file of the authorities the server's certificate must chain to |

a key set twice is refused, and so is `sslrootcert` with `sslmode=disable`. an error never repeats any part of the url, since a password with an unescaped `?` would spill into the query.

## a config

`ParseURL` returns a `pg.Config`, which can also be built by hand. its fields are `Host` (a host name or address, or the directory holding the server's unix socket), `Port`, `User`, `Password`, `Database`, `AppName`, `TLS` and `RootCAs` (nil means the system's authorities). printing a config never shows the password.

## tls

- over tcp, tls is on unless `sslmode=disable`. it is `verify-full`, so the chain is checked against `sslrootcert` (or the system's authorities), and the host dialed must be named in the certificate.
- the minimum is tls 1.2, and the client offers alpn `postgresql`.
- the server's answer to the tls request is read straight off the socket, so nothing it sends before the handshake is read as postgres.
- a config's tls mode is `pg.TLSOff`, `pg.TLSVerifyFull`, or unset (`pg.TLSDefault`), which means the default. an unknown mode is refused before dialing.
- tls over a unix socket isn't spoken. a socket's session is in the clear.

## logging in

- over tls, only scram-sha-256-plus bound to the server's certificate (tls-server-end-point) is accepted. md5, trust, unbound scram and cleartext are all refused, so a tls session can't be downgraded to a weaker method.
- a certificate signed with ed25519 can't be bound, so it is refused.
- in the clear, the server may ask for trust, md5 or scram-sha-256. cleartext is always refused, since the password would cross the wire as it is.

## session settings

the client sends its own settings at startup, and writes and reads values under them. they are `client_encoding=UTF8`, `DateStyle=ISO, MDY`, `TimeZone=UTC`, `extra_float_digits=3` and `bytea_output=hex`. a server that reports another value for one, at startup or later, can't be read right, so the session breaks.

`conn.Param(name)` is the latest value the server reported for a setting, such as `server_version`.
