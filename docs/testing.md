# testing

```sh
go test ./...
bash scripts/pg-test.sh
```

`go test ./...` needs no database.

## integration tests

`scripts/pg-test.sh` runs the integration tests against a throwaway cluster with tls on. it makes the cluster in a temp dir on port 55451 (or `PG_TEST_PORT`), with a throwaway authority from `internal/testcerts`, a scram user and an md5 user on one database, then stops and removes it all. `PG_BIN` names postgres's bin directory when it isn't on PATH, and `PG_TEST_FLAGS` passes more flags to `go test`.

the test database's defaults differ from the client's session settings, so the tests show that the client's own settings take effect.

the integration tests are named `TestIntegration…` and read `PG_TEST_URL`, `PG_TEST_MD5_URL`, `PG_TEST_TLS_URL` and `PG_TEST_TLS_DEFAULT_URL`, which the script sets.

## fuzzing

four fuzzers cover the server's answers to a query, the tls answer, the scram messages and a batch's answers.

```sh
go test . -run '^$' -fuzz '^FuzzServerAnswers$' -fuzztime 20s
go test . -run '^$' -fuzz '^FuzzTLSAnswer$' -fuzztime 20s
go test . -run '^$' -fuzz '^FuzzScramServerMessages$' -fuzztime 20s
go test . -run '^$' -fuzz '^FuzzBatchAnswers$' -fuzztime 20s
```

## ci

ci runs format, vet, the tests, the four fuzzers, and the integration tests on postgres 16 with tls on, on every push to main and on pull requests.
