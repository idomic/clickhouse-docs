---
sidebar_label: ClickHouse Client API
sidebar_position: 3
keywords: [clickhouse, go, client, high-level, api]
slug: /en/integrations/go/clickhouse-go/clickhouse-api
description: ClickHouse Client API
---

# ClickHouse Client API

All code examples for the ClickHouse Client API can be found [here](https://github.com/ClickHouse/clickhouse-go/tree/main/examples).

## Connecting

The following example, which returns the server version, demonstrates connecting to ClickHouse - assuming ClickHouse is not secured and accessible with the default user.

Note we use the default native port to connect.

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
})
if err != nil {
    return err
}
v, err := conn.ServerVersion()
fmt.Println(v)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/connect.go)

**For all subsequent examples, unless explicitly shown, we assume the use of the ClickHouse `conn` variable has been created and is available.**

### Connection Settings

When opening a connection, an Options struct can be used to control client behavior. The following settings are available:


* `Protocol` - either Native or HTTP. HTTP is only supported currently for the [database/sql API](./database-sql-api).
* `TLS` - TLS options. A non-nil value enables TLS. See [Using TLS](clickhouse-api#using-tls).
* `Addr` - a slice of addresses including port.
* `Auth` - Authentication detail. See [Authentication](clickhouse-api#authentication).
* `DialContext` - custom dial function to determine how connections are established. 
* `Debug` - true/false to enable debugging.
* `Debugf` - provides a function to consume debug output. Requires `debug` to be set to true.
* `Settings` - map of ClickHouse settings. These will be applied to all ClickHouse queries. [Using Context](clickhouse-api#using-context) allows settings to be set per query.
* `Compression` - enable compression for blocks. See [Compression](clickhouse-api#compression).
* `DialTimeout` - the maximum time to establish a connection. Defaults to 1s.
* `MaxOpenConns` - max connections for use at any time. More or fewer connections may be in the idle pool, but only this number can be used at any time. Defaults to MaxIdleConns+5. 
* `MaxIdleConns` - number of connections to maintain in the pool. Connections will be reused if possible. Defaults to 5.
* `ConnMaxLifetime` - maximum lifetime to keep a connection available. Defaults to 1hr. Connections are destroyed after this time, with new connections added to the pool as required.
* `ConnOpenStrategy` - determines how the list of node addresses should be consumed and used to open connections. See [Connecting to Multiple Nodes](clickhouse-api#connecting-to-multiple-nodes).

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
    DialContext: func(ctx context.Context, addr string) (net.Conn, error) {
        dialCount++
        var d net.Dialer
        return d.DialContext(ctx, "tcp", addr)
    },
    Debug: true,
    Debugf: func(format string, v ...interface{}) {
        fmt.Printf(format, v)
    },
    Settings: clickhouse.Settings{
        "max_execution_time": 60,
    },
    Compression: &clickhouse.Compression{
        Method: clickhouse.CompressionLZ4,
    },
    DialTimeout:      time.Duration(10) * time.Second,
    MaxOpenConns:     5,
    MaxIdleConns:     5,
    ConnMaxLifetime:  time.Duration(10) * time.Minute,
    ConnOpenStrategy: clickhouse.ConnOpenInOrder,
})
if err != nil {
    return err
}
```
[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/connect_settings.go)

### Connection Pooling

The client maintains a pool of connections, reusing these across queries as required. At most, `MaxOpenConns` will be used at any time, with the maximum pool size controlled by the `MaxIdleConns`. The client will acquire a connection from the pool for each query execution, returning it to the pool for reuse. A connection is used for the lifetime of a batch and released on `Send()`.

There is no guarantee the same connection in a pool will be used for subsequent queries unless the user sets `MaxOpenConns=1`. This is rarely needed but may be required for cases where users are using temporary tables.

Also, note that the `ConnMaxLifetime` is by default 1hr. This can lead to cases where the load to ClickHouse becomes unbalanced if nodes leave the cluster. This can occur when a node becomes unavailable, connections will balance to the other nodes. These connections will persist and not be refreshed for 1hr by default, even if the problematic node returns to the cluster. Consider lowering this value in heavy workload cases.

## Using TLS

At a low level, all client connect methods (DSN/OpenDB/Open) will use the[ Go tls package](https://pkg.go.dev/crypto/tls) to establish a secure connection. The client knows to use TLS if the Options struct contains a non-nil `tls.Config` pointer.

```go
env, err := GetNativeTestEnvironment()
if err != nil {
    return err
}
cwd, err := os.Getwd()
if err != nil {
    return err
}
t := &tls.Config{}
caCert, err := ioutil.ReadFile(path.Join(cwd, "../../tests/resources/CAroot.crt"))
if err != nil {
    return err
}
caCertPool := x509.NewCertPool()
successful := caCertPool.AppendCertsFromPEM(caCert)
if !successful {
    return err
}
t.RootCAs = caCertPool
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.SslPort)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
    TLS: t,
})
if err != nil {
    return err
}
v, err := conn.ServerVersion()
if err != nil {
    return err
}
fmt.Println(v.String())
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/ssl.go)

This minimal `TLS.Config` is normally sufficient to connect to the secure native port (normally 9440) on a ClickHouse server. If the ClickHouse server does not have a valid certificate (expired, wrong hostname, not signed by a publicly recognized root Certificate Authority), InsecureSkipVerify can be true, but this is strongly discouraged.

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.SslPort)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
    TLS: &tls.Config{
        InsecureSkipVerify: true,
    },
})
if err != nil {
    return err
}
v, err := conn.ServerVersion()
```
[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/ssl_no_verify.go)

If additional TLS parameters are necessary, the application code should set the desired fields in the `tls.Config` struct. That can include specific cipher suites, forcing a particular TLS version (like 1.2 or 1.3), adding an internal CA certificate chain, adding a client certificate (and private key) if required by the ClickHouse server, and most of the other options that come with a more specialized security setup.

## Authentication

Specify an Auth struct in the connection details to specify a username and password.

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
})
if err != nil {
    return err
}
if err != nil {
    return err
}
v, err := conn.ServerVersion()
```
[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/auth.go)

## Connecting to Multiple Nodes

Multiple addresses can be specified via the `Addr` struct.

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{"127.0.0.1:9001", "127.0.0.1:9002", fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
})
if err != nil {
    return err
}
v, err := conn.ServerVersion()
if err != nil {
    return err
}
fmt.Println(v.String())
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/1c0d81d0b1388dbb9e09209e535667df212f4ae4/examples/clickhouse_api/multi_host.go#L26-L45)


Two connection strategies are available: 

* `ConnOpenInOrder` (default)  - addresses are consumed in order. Later addresses are only utilized in case of failure to connect using addresses earlier in the list. This is effectively a failure-over strategy.
* `ConnOpenRoundRobin` - Load is balanced across the addresses using a round-robin strategy.

This can be controlled through the option `ConnOpenStrategy`

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr:             []string{"127.0.0.1:9001", "127.0.0.1:9002", fmt.Sprintf("%s:%d", env.Host, env.Port)},
    ConnOpenStrategy: clickhouse.ConnOpenRoundRobin,
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
})
if err != nil {
    return err
}
v, err := conn.ServerVersion()
if err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/1c0d81d0b1388dbb9e09209e535667df212f4ae4/examples/clickhouse_api/multi_host.go#L50-L67) 

## Execution

Arbitrary statements can be executed via the `Exec` method. This is useful for DDL and simple statements. It should not be used for larger inserts or query iterations.

```go
conn.Exec(context.Background(), `DROP TABLE IF EXISTS example`)
err = conn.Exec(context.Background(), `
    CREATE TABLE IF NOT EXISTS example (
        Col1 UInt8,
        Col2 String
    ) engine=Memory
`)
if err != nil {
    return err
}
conn.Exec(context.Background(), "INSERT INTO example VALUES (1, 'test-1')")
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/exec.go)


Note the ability to pass a Context to the query. This can be used to pass specific query level settings - see [Using Context](clickhouse-api#using-context).

## Batch Insert

To insert a large number of rows, the client provides batch semantics. This requires the preparation of a batch to which rows can be appended. This is finally sent via the `Send()` method. Batches will be held in memory until Send is executed.

```go
conn, err := GetNativeConnection(nil, nil, nil)
if err != nil {
    return err
}
ctx := context.Background()
defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
conn.Exec(context.Background(), "DROP TABLE IF EXISTS example")
err = conn.Exec(ctx, `
    CREATE TABLE IF NOT EXISTS example (
            Col1 UInt8
        , Col2 String
        , Col3 FixedString(3)
        , Col4 UUID
        , Col5 Map(String, UInt8)
        , Col6 Array(String)
        , Col7 Tuple(String, UInt8, Array(Map(String, String)))
        , Col8 DateTime
    ) Engine = Memory
`)
if err != nil {
    return err
}


batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
for i := 0; i < 1000; i++ {
    err := batch.Append(
        uint8(42),
        "ClickHouse",
        "Inc",
        uuid.New(),
        map[string]uint8{"key": 1},             // Map(String, UInt8)
        []string{"Q", "W", "E", "R", "T", "Y"}, // Array(String)
        []interface{}{ // Tuple(String, UInt8, Array(Map(String, String)))
            "String Value", uint8(5), []map[string]string{
                {"key": "value"},
                {"key": "value"},
                {"key": "value"},
            },
        },
        time.Now(),
    )
    if err != nil {
        return err
    }
}
return batch.Send()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/batch.go)

Recommendations for ClickHouse apply [here](https://clickhouse.com/docs/en/about-us/performance/#performance-when-inserting-data). Batches should not be shared across go-routines - construct a separate batch per routine.  

From the above example, note the need for variable types to align with the column type when appending rows. While the mapping is usually obvious, this interface tries to be flexible, and types will be converted provided no precision loss is incurred. For example, the following demonstrates inserting a string into a datetime64.

```go
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
for i := 0; i < 1000; i++ {
    err := batch.Append(
        "2006-01-02 15:04:05.999",
    )
    if err != nil {
        return err
    }
}
return batch.Send()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/type_convert.go)


For a full summary of supported go types for each column type, see [Type Conversions](clickhouse-api#type-conversions).

## Querying Row/s


Users can either query for a single row using the `QueryRow` method or obtain a cursor for iteration over a result set via `Query`. While the former accepts a destination for the data to be serialized into, the latter requires the to call `Scan` on each row.

```go
row := conn.QueryRow(context.Background(), "SELECT * FROM example")
var (
    col1             uint8
    col2, col3, col4 string
    col5             map[string]uint8
    col6             []string
    col7             []interface{}
    col8             time.Time
)
if err := row.Scan(&col1, &col2, &col3, &col4, &col5, &col6, &col7, &col8); err != nil {
    return err
}
fmt.Printf("row: col1=%d, col2=%s, col3=%s, col4=%s, col5=%v, col6=%v, col7=%v, col8=%v\n", col1, col2, col3, col4, col5, col6, col7, col8)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/query_row.go)

```go
rows, err := conn.Query(ctx, "SELECT Col1, Col2, Col3 FROM example WHERE Col1 >= 2")
if err != nil {
    return err
}
for rows.Next() {
    var (
        col1 uint8
        col2 string
        col3 time.Time
    )
    if err := rows.Scan(&col1, &col2, &col3); err != nil {
        return err
    }
    fmt.Printf("row: col1=%d, col2=%s, col3=%s\n", col1, col2, col3)
}
rows.Close()
return rows.Err()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/query_rows.go)

Note in both cases, we are required to pass a pointer to the variables we wish to serialize the respective column values into. These must be passed in the order specified in the `SELECT` statement - by default, the order of column declaration will be used in the event of a `SELECT *` as shown above.

Similar to insertion, the Scan method requires the target variables to be of an appropriate type. This again aims to be flexible, with types converted where possible, provided no precision loss is possible, e.g., the above example shows a UUID column being read into a string variable. For a full list of supported go types for each Column type, see [Type Conversions](clickhouse-api#type-conversions).

Finally, note the ability to pass a Context to the Query and QueryRow methods. This can be used for query level settings - see [Using Context](clickhouse-api#using-context) for further details.

## Async Insert

Asynchronous inserts are supported through the Async method. This allows the user to specify whether the client should wait for the server to complete the insert or respond once the data has been received. This effectively controls the parameter [wait_for_async_insert](https://clickhouse.com/docs/en/operations/settings/settings/#wait-for-async-insert).

```go
conn, err := GetNativeConnection(nil, nil, nil)
if err != nil {
    return err
}
ctx := context.Background()
if err := clickhouse_tests.CheckMinServerServerVersion(conn, 21, 12, 0); err != nil {
    return nil
}
defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
conn.Exec(ctx, `DROP TABLE IF EXISTS example`)
const ddl = `
    CREATE TABLE example (
            Col1 UInt64
        , Col2 String
        , Col3 Array(UInt8)
        , Col4 DateTime
    ) ENGINE = Memory
`
if err := conn.Exec(ctx, ddl); err != nil {
    return err
}
for i := 0; i < 100; i++ {
    if err := conn.AsyncInsert(ctx, fmt.Sprintf(`INSERT INTO example VALUES (
        %d, '%s', [1, 2, 3, 4, 5, 6, 7, 8, 9], now()
    )`, i, "Golang SQL database driver"), false); err != nil {
        return err
    }
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/async.go)

## Columnar Insert

Inserts can be inserted in column format. This can provide performance benefits if the data is already orientated in this structure by avoiding the need to pivot to rows.

```go
batch, err := conn.PrepareBatch(context.Background(), "INSERT INTO example")
if err != nil {
    return err
}
var (
    col1 []uint64
    col2 []string
    col3 [][]uint8
    col4 []time.Time
)
for i := 0; i < 1_000; i++ {
    col1 = append(col1, uint64(i))
    col2 = append(col2, "Golang SQL database driver")
    col3 = append(col3, []uint8{1, 2, 3, 4, 5, 6, 7, 8, 9})
    col4 = append(col4, time.Now())
}
if err := batch.Column(0).Append(col1); err != nil {
    return err
}
if err := batch.Column(1).Append(col2); err != nil {
    return err
}
if err := batch.Column(2).Append(col3); err != nil {
    return err
}
if err := batch.Column(3).Append(col4); err != nil {
    return err
}
return batch.Send()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/columnar_insert.go)

## Using Structs

For users, Golang structs provide a logical representation of a row of data in ClickHouse. To assist with this, the native interface provides several convenient functions.

### Select with Serialize

The Select method allows a set of response rows to be marshaled into a slice of structs with a single invocation.

```go
var result []struct {
    Col1           uint8
    Col2           string
    ColumnWithName time.Time `ch:"Col3"`
}

if err = conn.Select(ctx, &result, "SELECT Col1, Col2, Col3 FROM example"); err != nil {
    return err
}

for _, v := range result {
    fmt.Printf("row: col1=%d, col2=%s, col3=%s\n", v.Col1, v.Col2, v.ColumnWithName)
}
```


[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/select_struct.go)

### Scan Struct

ScanStruct allows the marshaling of a single Row from a query into a struct.

```go
var result struct {
    Col1  int64
    Count uint64 `ch:"count"`
}
if err := conn.QueryRow(context.Background(), "SELECT Col1, COUNT() AS count FROM example WHERE Col1 = 5 GROUP BY Col1").ScanStruct(&result); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/scan_struct.go)

### Append Struct

AppendStruct allows a struct to be appended to an existing [batch](clickhouse-api#batch-insert) and interpreted as a complete row. This requires the columns of the struct to align in both name and type with the table. While all columns must have an equivalent struct field, some struct fields may not have an equivalent column representation. These will simply be ignored.

```go
batch, err := conn.PrepareBatch(context.Background(), "INSERT INTO example")
if err != nil {
    return err
}
for i := 0; i < 1_000; i++ {
    err := batch.AppendStruct(&row{
        Col1:       uint64(i),
        Col2:       "Golang SQL database driver",
        Col3:       []uint8{1, 2, 3, 4, 5, 6, 7, 8, 9},
        Col4:       time.Now(),
        ColIgnored: "this will be ignored",
    })
    if err != nil {
        return err
    }
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/append_struct.go)

## Type Conversions

The client aims to be as flexible as possible concerning accepting variable types for both insertion and marshaling of responses. In most cases, an equivalent Golang type exists for a ClickHouse column type, e.g., [UInt64](https://clickhouse.com/docs/en/sql-reference/data-types/int-uint/) to [uint64](https://pkg.go.dev/builtin#uint64). These logical mappings should always be supported. Users may wish to utilize variable types that can be inserted into columns or used to receive a response if the conversion of either the variable or received data takes place first. The client aims to support these conversions transparently, so users do not need to convert their data to align precisely before insertion and to provide flexible marshaling at query time. This transparent conversion does not allow for precision loss. For example, a uint32 cannot be used to receive data from a UInt64 column. Conversely, a string can be inserted into a datetime64 field provided it meets the format requirements.

The type conversions currently supported for primitive types are captured [here](https://github.com/ClickHouse/clickhouse-go/blob/main/TYPES.md).

This effort is ongoing and can be separated into insertion (`Append`/`AppendRow`) and read time (via a `Scan`). Should you need support for a specific conversion, please raise an issue.

## Complex Types

### Date/DateTime types

The ClickHouse go client supports the `Date`, `Date32`, `DateTime`, and `DateTime64` date/datetime types. Dates can be inserted as a string in the format `2006-01-02` or using the native go `time.Time{}` or `sql.NullTime`. DateTimes also support the latter types but require strings to be passed in the format `2006-01-02 15:04:05` with an optional timezone offset e.g. `2006-01-02 15:04:05 +08:00`. `time.Time{}` and `sql.NullTime` are both supported at read time as well as any implementation of of the `sql.Scanner` interface.

Handling of timezone information depends on the ClickHouse type and whether the value is being inserted or read:

* **DateTime/DateTime64**
    * At **insert** time the value is sent to ClickHouse in UNIX timestamp format. If no time zone is provided, the client will assume the client's local time zone. `time.Time{}` or `sql.NullTime` will be converted to epoch accordingly.
    * At **select** time the timezone of the column will be used if set when returning a `time.Time` value. If not, the timezone of the server will be used.
* **Date/Date32**
    * At **insert** time, the timezone of any date is considered when converting the date to a unix timestamp, i.e., it will be offset by the timezone prior to storage as a date, as Date types have no locale in ClickHouse. If this is not specified in a string value, the local timezone will be used.
    * At **select** time, dates are scanned into `time.Time{}` or `sql.NullTime{}` instances will be returned without timezone information.

### Array

Arrays should be inserted as slices. Typing rules for the elements are consistent with those for the [primitive type](clickhouse-api#type-conversions), i.e., where possible elements will be converted.

A pointer to a slice should be provided at Scan time.

```go
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
var i int64
for i = 0; i < 10; i++ {
    err := batch.Append(
        []string{strconv.Itoa(int(i)), strconv.Itoa(int(i + 1)), strconv.Itoa(int(i + 2)), strconv.Itoa(int(i + 3))},
        [][]int64{{i, i + 1}, {i + 2, i + 3}, {i + 4, i + 5}},
    )
    if err != nil {
        return err
    }
}
if err := batch.Send(); err != nil {
    return err
}
var (
    col1 []string
    col2 [][]int64
)
rows, err := conn.Query(ctx, "SELECT * FROM example")
if err != nil {
    return err
}
for rows.Next() {
    if err := rows.Scan(&col1, &col2); err != nil {
        return err
    }
    fmt.Printf("row: col1=%v, col2=%v\n", col1, col2)
}
rows.Close()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/array.go)

### Map

Maps should be inserted as Golang maps with keys and values conforming to the type rules defined [earlier](clickhouse-api#type-conversions).

```go
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
var i int64
for i = 0; i < 10; i++ {
    err := batch.Append(
        map[string]uint64{strconv.Itoa(int(i)): uint64(i)},
        map[string][]string{strconv.Itoa(int(i)): {strconv.Itoa(int(i)), strconv.Itoa(int(i + 1)), strconv.Itoa(int(i + 2)), strconv.Itoa(int(i + 3))}},
        map[string]map[string]uint64{strconv.Itoa(int(i)): {strconv.Itoa(int(i)): uint64(i)}},
    )
    if err != nil {
        return err
    }
}
if err := batch.Send(); err != nil {
    return err
}
var (
    col1 map[string]uint64
    col2 map[string][]string
    col3 map[string]map[string]uint64
)
rows, err := conn.Query(ctx, "SELECT * FROM example")
if err != nil {
    return err
}
for rows.Next() {
    if err := rows.Scan(&col1, &col2, &col3); err != nil {
        return err
    }
    fmt.Printf("row: col1=%v, col2=%v, col3=%v\n", col1, col2, col3)
}
rows.Close()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/map.go)

### Tuples

Tuples represent a group of Columns of arbitrary length. The columns can either be explicitly named or only specify a type e.g.

```sql
//unnamed
Col1 Tuple(String, Int64)

//named
Col2 Tuple(name String, id Int64, age uint8)
```

Of these approaches, named tuples offer greater flexibility. While unnamed tuples must be inserted and read using slices, named tuples are also compatible with maps.

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            Col1 Tuple(name String, age UInt8),
            Col2 Tuple(String, UInt8),
            Col3 Tuple(name String, id String)
        ) 
        Engine Memory
    `); err != nil {
    return err
}

defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
// both named and unnamed can be added with slices. Note we can use strongly typed lists and maps if all elements are the same type
if err = batch.Append([]interface{}{"Clicky McClickHouse", uint8(42)}, []interface{}{"Clicky McClickHouse Snr", uint8(78)}, []string{"Dale", "521211"}); err != nil {
    return err
}
if err = batch.Append(map[string]interface{}{"name": "Clicky McClickHouse Jnr", "age": uint8(20)}, []interface{}{"Baby Clicky McClickHouse", uint8(1)}, map[string]string{"name": "Geoff", "id": "12123"}); err != nil {
    return err
}
if err = batch.Send(); err != nil {
    return err
}
var (
    col1 map[string]interface{}
    col2 []interface{}
    col3 map[string]string
)
// named tuples can be retrieved into a map or slices, unnamed just slices
if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2, &col3); err != nil {
    return err
}
fmt.Printf("row: col1=%v, col2=%v, col3=%v\n", col1, col2, col3)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/tuple.go)

Note: typed slices and maps are supported, provide the sub-columns in the named tuple are all of the same types. 

### Nested

A Nested field is equivalent to an Array of named Tuples. Usage depends on whether the user has set [flatten_nested](https://clickhouse.com/docs/en/operations/settings/settings/#flatten-nested) to 1 or 0.

By setting flatten_nested to 0, Nested columns stay as a single array of tuples. This allows users to use slices of maps for insertion and retrieval and arbitrary levels of nesting. The map's key must equal the column's name, as shown in the example below.

Note: since the maps represent a tuple, they must be of the type `map[string]interface{}`. The values are currently not strongly typed.

```go
conn, err := GetNativeConnection(clickhouse.Settings{
    "flatten_nested": 0,
}, nil, nil)
if err != nil {
    return err
}
ctx := context.Background()
defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
conn.Exec(context.Background(), "DROP TABLE IF EXISTS example")
err = conn.Exec(ctx, `
    CREATE TABLE example (
        Col1 Nested(Col1_1 String, Col1_2 UInt8),
        Col2 Nested(
            Col2_1 UInt8, 
            Col2_2 Nested(
                Col2_2_1 UInt8, 
                Col2_2_2 UInt8
            )
        )
    ) Engine Memory
`)
if err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
var i int64
for i = 0; i < 10; i++ {
    err := batch.Append(
        []map[string]interface{}{
            {
                "Col1_1": strconv.Itoa(int(i)),
                "Col1_2": uint8(i),
            },
            {
                "Col1_1": strconv.Itoa(int(i + 1)),
                "Col1_2": uint8(i + 1),
            },
            {
                "Col1_1": strconv.Itoa(int(i + 2)),
                "Col1_2": uint8(i + 2),
            },
        },
        []map[string]interface{}{
            {
                "Col2_2": []map[string]interface{}{
                    {
                        "Col2_2_1": uint8(i),
                        "Col2_2_2": uint8(i + 1),
                    },
                },
                "Col2_1": uint8(i),
            },
            {
                "Col2_2": []map[string]interface{}{
                    {
                        "Col2_2_1": uint8(i + 2),
                        "Col2_2_2": uint8(i + 3),
                    },
                },
                "Col2_1": uint8(i + 1),
            },
        },
    )
    if err != nil {
        return err
    }
}
if err := batch.Send(); err != nil {
    return err
}
var (
    col1 []map[string]interface{}
    col2 []map[string]interface{}
)
rows, err := conn.Query(ctx, "SELECT * FROM example")
if err != nil {
    return err
}
for rows.Next() {
    if err := rows.Scan(&col1, &col2); err != nil {
        return err
    }
    fmt.Printf("row: col1=%v, col2=%v\n", col1, col2)
}
rows.Close()
```

[Full Example - `flatten_tested=0`](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/nested.go#L28-L118)

If the default value of 1 is used for `flatten_nested`, nested columns are flattened to separate arrays. This requires using nested slices for insertion and retrieval. While arbitrary levels of nesting may work, this is not officially supported. 

```go
conn, err := GetNativeConnection(nil, nil, nil)
if err != nil {
    return err
}
ctx := context.Background()
defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
conn.Exec(ctx, "DROP TABLE IF EXISTS example")
err = conn.Exec(ctx, `
    CREATE TABLE example (
        Col1 Nested(Col1_1 String, Col1_2 UInt8),
        Col2 Nested(
            Col2_1 UInt8, 
            Col2_2 Nested(
                Col2_2_1 UInt8, 
                Col2_2_2 UInt8
            )
        )
    ) Engine Memory
`)
if err != nil {
    return err
}


batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
var i uint8
for i = 0; i < 10; i++ {
    col1_1_data := []string{strconv.Itoa(int(i)), strconv.Itoa(int(i + 1)), strconv.Itoa(int(i + 2))}
    col1_2_data := []uint8{i, i + 1, i + 2}
    col2_1_data := []uint8{i, i + 1, i + 2}
    col2_2_data := [][][]interface{}{
        {
            {i, i + 1},
        },
        {
            {i + 2, i + 3},
        },
        {
            {i + 4, i + 5},
        },
    }
    err := batch.Append(
        col1_1_data,
        col1_2_data,
        col2_1_data,
        col2_2_data,
    )
    if err != nil {
        return err
    }
}
if err := batch.Send(); err != nil {
    return err
}
```

[Full Example - `flatten_nested=1`](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/nested.go#L123-L180)


Note: Nested columns must have the same dimensions. For example, in the above example, `Col_2_2` and `Col_2_1` must have the same number of elements.

Due to a more straightforward interface and official support for nesting, we recommend `flatten_nested=0`.

### JSON

The JSON type utilizes schema inference to automatically create arbitrary levels of tuples, nested objects, and arrays in order to represent JSON data. Other than a root JSON column, the user does not need to define any column types - these will be automatically inferred from the data and created as required. For further details, see [Working with JSON](https://clickhouse.com/docs/en/guides/developer/working-with-json/json-semi-structured). 


This feature is only available in versions later than 22.3. It represents the preferred mechanism for handling arbitrary semi-structured JSON. To provide maximum flexibility, the go client allows JSON to be inserted using a struct, map, or string. JSON columns can be marshaled back into either a struct or map. Examples of each approach are shown below.

Note the need to set `allow_experimental_object_type=1` since JSON is experimental. 

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            Col1 JSON,
            Col2 JSON,
            Col3 JSON
        ) 
        Engine Memory
    `); err != nil {
    return err
}


type User struct {
    Name     string `json:"name"`
    Age      uint8  `json:"age"`
    Password string `ch:"-"`
}


defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
// we can insert JSON as either a string, struct or map
col1Data := `{"name": "Clicky McClickHouse", "age": 40, "password": "password"}`
col2Data := User{
    Name:     "Clicky McClickHouse Snr",
    Age:      uint8(80),
    Password: "random",
}
col3Data := map[string]interface{}{
    "name":     "Clicky McClickHouse Jnr",
    "age":      uint8(10),
    "password": "clicky",
}
// both named and unnamed can be added with slices
if err = batch.Append(col1Data, col2Data, col3Data); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}
// we can scan JSON into either a map or struct
var (
    col1 map[string]interface{}
    col2 map[string]interface{}
    col3 User
)
// named tuples can be retrieved into a map or slices, unnamed just slices
if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2, &col3); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/faef77ec2faf87667b2bd8f0bc930d1418f81f3e/examples/clickhouse_api/json.go#L43-L96)


Maps can be strongly typed. Maps and structs can also be nested in any combination. See the more complex example [here](https://github.com/ClickHouse/clickhouse-go/blob/faef77ec2faf87667b2bd8f0bc930d1418f81f3e/examples/clickhouse_api/json.go#L102):

If inserting structs, the client supports using `json` tags to control the name of a field when serialized. Fields can also be ignored if using the special value of `-` e.g.

```go
type User struct {
  Name     string `json:"name"`
  Age      uint8  `json:"age"`
  Password string `json:"-"`
}
```

The “json” tag is respected by the standard [golang json encoding](https://pkg.go.dev/encoding/json) package. The client also supports the “ch” tag equivalent to “json”. For example, the following is equivalent to the above for the client:

```go
type User struct {
  Name     string `ch:"name"`
  Age      uint8  `ch:"age"`
  Password string `ch:"-"`
}
```

#### Important Notes

* No current support for structs and maps which have pointers inside them. 
* If inserting structs or maps, types must be consistent within a batch, i.e., a field must be consistent within a batch. Note that ClickHouse is able to downcast/coerce types, e.g., if a field is an int and then sent as a String, the field will be converted to a String - see [here](https://clickhouse.com/docs/en/guides/developer/working-with-json/json-semi-structured#changing-columns) for further examples. The client will not do this since it would require visibility of the complete dataset. If you need flexible types within a batch, insert the JSON as a string, e.g., you can't be sure if a JSON field is a string or number. This will defer type decisions and processing to the server.
* Within a batch, we don't allow mixed formats for the same column e.g. strings with maps/structs. Maps and structs are equivalent and can be added to the same column. The examples above use strings for a different JSON column.
* Dimensions and types within a slice must be the same e.g., an interface{} slice with a struct and list is not supported. This is the same behavior as ClickHouse, where lists of objects with different dimensions are not supported.
* At query time, we do best-effort filling structs passed. Any data for which there is no field is currently ignored.

### Geo Types

The client supports the geo types Point, Ring, Polygon, and Multi Polygon. These fields are in Golang using the package [github.com/paulmach/orb](https://github.com/paulmach/orb).

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            point Point,
            ring Ring,
            polygon Polygon,
            mPolygon MultiPolygon
        ) 
        Engine Memory
    `); err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}

if err = batch.Append(
    orb.Point{11, 22},
    orb.Ring{
        orb.Point{1, 2},
        orb.Point{1, 2},
    },
    orb.Polygon{
        orb.Ring{
            orb.Point{1, 2},
            orb.Point{12, 2},
        },
        orb.Ring{
            orb.Point{11, 2},
            orb.Point{1, 12},
        },
    },
    orb.MultiPolygon{
        orb.Polygon{
            orb.Ring{
                orb.Point{1, 2},
                orb.Point{12, 2},
            },
            orb.Ring{
                orb.Point{11, 2},
                orb.Point{1, 12},
            },
        },
        orb.Polygon{
            orb.Ring{
                orb.Point{1, 2},
                orb.Point{12, 2},
            },
            orb.Ring{
                orb.Point{11, 2},
                orb.Point{1, 12},
            },
        },
    },
); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}

var (
    point    orb.Point
    ring     orb.Ring
    polygon  orb.Polygon
    mPolygon orb.MultiPolygon
)

if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&point, &ring, &polygon, &mPolygon); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/geo.go)

### UUID

The UUID type is supported by the [github.com/google/uuid](https://github.com/google/uuid) package. Users can also send and marshall uuids as strings or any type which implements `sql.Scanner` or `Stringify`.

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            col1 UUID,
            col2 UUID
        ) 
        Engine Memory
    `); err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
col1Data, _ := uuid.NewUUID()
if err = batch.Append(
    col1Data,
    "603966d6-ed93-11ec-8ea0-0242ac120002",
); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}

var (
    col1 uuid.UUID
    col2 uuid.UUID
)

if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/uuid.go)

### Decimal

The Decimal type is supported by [github.com/shopspring/decimal](https://github.com/shopspring/decimal) package.

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
        Col1 Decimal32(3), 
        Col2 Decimal(18,6), 
        Col3 Decimal(15,7), 
        Col4 Decimal128(8), 
        Col5 Decimal256(9)
    ) Engine Memory
    `); err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
if err = batch.Append(
    decimal.New(25, 4),
    decimal.New(30, 5),
    decimal.New(35, 6),
    decimal.New(135, 7),
    decimal.New(256, 8),
); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}

var (
    col1 decimal.Decimal
    col2 decimal.Decimal
    col3 decimal.Decimal
    col4 decimal.Decimal
    col5 decimal.Decimal
)

if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2, &col3, &col4, &col5); err != nil {
    return err
}
fmt.Printf("col1=%v, col2=%v, col3=%v, col4=%v, col5=%v\n", col1, col2, col3, col4, col5)
```    

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/decimal.go)

### Nullable

The go value of Nil represents a ClickHouse NULL. This can be used if a field is declared Nullable. At insert time, Nil can be passed for both the normal and Nullable version of a column. For the former, the default value for the type will be persisted, e.g., an empty string for string. For the nullable version, a NULL value will be stored in ClickHouse. 

At Scan time, the user must pass a pointer to a type that supports nil, e.g., *string, in order to represent the nil value for a Nullable field. In the example below, col1, which is a Nullable(String), thus receives a **string. This allows nil to be represented.

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            col1 Nullable(String),
            col2 String,
            col3 Nullable(Int8),
            col4 Nullable(Int64)
        ) 
        Engine Memory
    `); err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
if err = batch.Append(
    nil,
    nil,
    nil,
    sql.NullInt64{Int64: 0, Valid: false},
); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}

var (
    col1 *string
    col2 string
    col3 *int8
    col4 sql.NullInt64
)

if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2, &col3, &col4); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/nullable.go)

The client additionally supports the `sql.Null*` types e.g. `sql.NullInt64`. These are compatible with their equivalent ClickHouse types.

### Big Ints -  Int128, Int256, UInt128, UInt256

Number types larger than 64 bits are represented using the native go [big](https://pkg.go.dev/math/big) package.

```go
if err = conn.Exec(ctx, `
    CREATE TABLE example (
        Col1 Int128, 
        Col2 UInt128, 
        Col3 Array(Int128), 
        Col4 Int256, 
        Col5 Array(Int256), 
        Col6 UInt256, 
        Col7 Array(UInt256)
    ) Engine Memory`); err != nil {
    return err
}

batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}

col1Data, _ := new(big.Int).SetString("170141183460469231731687303715884105727", 10)
col2Data := big.NewInt(128)
col3Data := []*big.Int{
    big.NewInt(-128),
    big.NewInt(128128),
    big.NewInt(128128128),
}
col4Data := big.NewInt(256)
col5Data := []*big.Int{
    big.NewInt(256),
    big.NewInt(256256),
    big.NewInt(256256256256),
}
col6Data := big.NewInt(256)
col7Data := []*big.Int{
    big.NewInt(256),
    big.NewInt(256256),
    big.NewInt(256256256256),
}

if err = batch.Append(col1Data, col2Data, col3Data, col4Data, col5Data, col6Data, col7Data); err != nil {
    return err
}

if err = batch.Send(); err != nil {
    return err
}

var (
    col1 big.Int
    col2 big.Int
    col3 []*big.Int
    col4 big.Int
    col5 []*big.Int
    col6 big.Int
    col7 []*big.Int
)

if err = conn.QueryRow(ctx, "SELECT * FROM example").Scan(&col1, &col2, &col3, &col4, &col5, &col6, &col7); err != nil {
    return err
}
fmt.Printf("col1=%v, col2=%v, col3=%v, col4=%v, col5=%v, col6=%v, col7=%v\n", col1, col2, col3, col4, col5, col6, col7)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/big_int.go)

## Compression

Support for compression methods depends on the underlying protocol in use. For the native protocol, the client supports `LZ4` and `ZSTD` compression. This is performed at a block level only. Compression can be enabled by including a `Compression` configuration with the connection.

```go
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
    Compression: &clickhouse.Compression{
        Method: clickhouse.CompressionZSTD,
    },
    MaxOpenConns: 1,
})
ctx := context.Background()
defer func() {
    conn.Exec(ctx, "DROP TABLE example")
}()
conn.Exec(context.Background(), "DROP TABLE IF EXISTS example")
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            Col1 Array(String)
    ) Engine Memory
    `); err != nil {
    return err
}
batch, err := conn.PrepareBatch(ctx, "INSERT INTO example")
if err != nil {
    return err
}
for i := 0; i < 1000; i++ {
    if err := batch.Append([]string{strconv.Itoa(i), strconv.Itoa(i + 1), strconv.Itoa(i + 2), strconv.Itoa(i + 3)}); err != nil {
        return err
    }
}
if err := batch.Send(); err != nil {
    return err
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/compression.go)


Additional compression techniques are available if using the standard interface over HTTP. See [database/sql API - Compression](database-sql-api#compression) for further details.

### Parameter Binding

The client supports parameter binding for the Exec, Query, and QueryRow methods. As shown in the example below, this is supported using named, numbered, and positional parameters. We provide examples of these below.

```go
var count uint64
// positional bind
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col1 >= ? AND Col3 < ?", 500, now.Add(time.Duration(750)*time.Second)).Scan(&count); err != nil {
    return err
}
// 250
fmt.Printf("Positional bind count: %d\n", count)
// numeric bind
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col1 <= $2 AND Col3 > $1", now.Add(time.Duration(150)*time.Second), 250).Scan(&count); err != nil {
    return err
}
// 100
fmt.Printf("Numeric bind count: %d\n", count)
// named bind
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col1 <= @col1 AND Col3 > @col3", clickhouse.Named("col1", 100), clickhouse.Named("col3", now.Add(time.Duration(50)*time.Second))).Scan(&count); err != nil {
    return err
}
// 50
fmt.Printf("Named bind count: %d\n", count)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/bind.go)

### Special Cases

By default, slices will be unfolded into a comma-separated list of values if passed as a parameter to a query. If users require a set of values to be injected with wrapping `[ ]`, ArraySet should be used.

If groups/tuples are required, with wrapping `( )` e.g., for use with IN operators, users can use a GroupSet. This is particularly useful for cases where multiple groups are required, as shown in the example below.

Finally, DateTime64 fields require precision in order to ensure parameters are rendered appropriately. The precision level for the field is unknown by the client, however, so the user must provide it. To facilitate this, we provide the `DateNamed` parameter.

```go
var count uint64
// arrays will be unfolded
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col1 IN (?)", []int{100, 200, 300, 400, 500}).Scan(&count); err != nil {
    return err
}
fmt.Printf("Array unfolded count: %d\n", count)
// arrays will be preserved with []
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col4 = ?", clickhouse.ArraySet{300, 301}).Scan(&count); err != nil {
    return err
}
fmt.Printf("Array count: %d\n", count)
// Group sets allow us to form ( ) lists
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col1 IN ?", clickhouse.GroupSet{[]interface{}{100, 200, 300, 400, 500}}).Scan(&count); err != nil {
    return err
}
fmt.Printf("Group count: %d\n", count)
// More useful when we need nesting
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE (Col1, Col5) IN (?)", []clickhouse.GroupSet{{[]interface{}{100, 101}}, {[]interface{}{200, 201}}}).Scan(&count); err != nil {
    return err
}
fmt.Printf("Group count: %d\n", count)
// Use DateNamed when you need a precision in your time#
if err = conn.QueryRow(ctx, "SELECT count() FROM example WHERE Col3 >= @col3", clickhouse.DateNamed("col3", now.Add(time.Duration(500)*time.Millisecond), clickhouse.NanoSeconds)).Scan(&count); err != nil {
    return err
}
fmt.Printf("NamedDate count: %d\n", count)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/bind_special.go)

## Using Context

Go contexts provide a means of passing deadlines, cancellation signals, and other request-scoped values across API boundaries. All methods on a connection accept a context as their first variable. While previous examples used context.Background(), users can use this capability to pass settings and deadlines and to cancel queries.

Passing a context created `withDeadline` allows execution time limits to be placed on queries. Not this is an absolute time and expiry will only release the connection and send a cancel signal to ClickHouse. `WithCancel` can alternatively be used to cancel a query explicitly.

The helpers  `clickhouse.WithQueryID` and `clickhouse.WithQuotaKey` allow a query id and quota key to be specified. Query ids can be useful for tracking queries in logs and for cancellation purposes. A quota key can be used to impose limits on ClickHouse usage based on a unique key value - see [Quotas Management ](https://clickhouse.com/docs/en/operations/access-rights#quotas-management)for further details. 

Finally, users may wish to ensure a setting is only applied for a specific query - rather than for the entire connection, as shown in [Connection Settings](clickhouse-api#connection-settings).

Examples of the above are shown below.

```go
dialCount := 0
conn, err := clickhouse.Open(&clickhouse.Options{
    Addr: []string{fmt.Sprintf("%s:%d", env.Host, env.Port)},
    Auth: clickhouse.Auth{
        Database: env.Database,
        Username: env.Username,
        Password: env.Password,
    },
    DialContext: func(ctx context.Context, addr string) (net.Conn, error) {
        dialCount++
        var d net.Dialer
        return d.DialContext(ctx, "tcp", addr)
    },
})
if err != nil {
    return err
}
if err := clickhouse_tests.CheckMinServerServerVersion(conn, 22, 6, 1); err != nil {
    return nil
}
// we can use context to pass settings to a specific API call
ctx := clickhouse.Context(context.Background(), clickhouse.WithSettings(clickhouse.Settings{
    "allow_experimental_object_type": "1",
}))

conn.Exec(ctx, "DROP TABLE IF EXISTS example")

// to create a JSON column we need allow_experimental_object_type=1
if err = conn.Exec(ctx, `
    CREATE TABLE example (
            Col1 JSON
        ) 
        Engine Memory
    `); err != nil {
    return err
}

// queries can be cancelled using the context
ctx, cancel := context.WithCancel(context.Background())
go func() {
    cancel()
}()
if err = conn.QueryRow(ctx, "SELECT sleep(3)").Scan(); err == nil {
    return fmt.Errorf("expected cancel")
}

// set a deadline for a query - this will cancel the query after the absolute time is reached.
// queries will continue to completion in ClickHouse
ctx, cancel = context.WithDeadline(context.Background(), time.Now().Add(-time.Second))
defer cancel()
if err := conn.Ping(ctx); err == nil {
    return fmt.Errorf("expected deadline exceeeded")
}

// set a query id to assist tracing queries in logs e.g. see system.query_log
var one uint8
queryId, _ := uuid.NewUUID()
ctx = clickhouse.Context(context.Background(), clickhouse.WithQueryID(queryId.String()))
if err = conn.QueryRow(ctx, "SELECT 1").Scan(&one); err != nil {
    return err
}

conn.Exec(context.Background(), "DROP QUOTA IF EXISTS foobar")
defer func() {
    conn.Exec(context.Background(), "DROP QUOTA IF EXISTS foobar")
}()
ctx = clickhouse.Context(context.Background(), clickhouse.WithQuotaKey("abcde"))
// set a quota key - first create the quota
if err = conn.Exec(ctx, "CREATE QUOTA IF NOT EXISTS foobar KEYED BY client_key FOR INTERVAL 1 minute MAX queries = 5 TO default"); err != nil {
    return err
}

type Number struct {
    Number uint64 `ch:"number"`
}
for i := 1; i <= 6; i++ {
    var result []Number
    if err = conn.Select(ctx, &result, "SELECT number FROM numbers(10)"); err != nil {
        return err
    }
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/context.go)


## Progress/Profile/Log Information

Progress, Profile, and Log information can be requested on queries. Progress information will report statistics on the number of rows and bytes that have been read and processed in ClickHouse. Conversely, Profile information provides a summary of data returned to the client, including totals of bytes, rows, and blocks. Finally, log information provides statistics on threads, e.g., memory usage and data speed.

Obtaining this information requires the user to use [Context](clickhouse-api#using-context), to which the user can pass call-back functions.  

```go
totalRows := uint64(0)
// use context to pass a call back for progress and profile info
ctx := clickhouse.Context(context.Background(), clickhouse.WithProgress(func(p *clickhouse.Progress) {
    fmt.Println("progress: ", p)
    totalRows += p.Rows
}), clickhouse.WithProfileInfo(func(p *clickhouse.ProfileInfo) {
    fmt.Println("profile info: ", p)
}), clickhouse.WithLogs(func(log *clickhouse.Log) {
    fmt.Println("log info: ", log)
}))

rows, err := conn.Query(ctx, "SELECT number from numbers(1000000) LIMIT 1000000")
if err != nil {
    return err
}
for rows.Next() {
}

fmt.Printf("Total Rows: %d\n", totalRows)
rows.Close()
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/progress.go)


## Dynamic Scanning

Users may need to read tables for which they do not know the schema or type of the fields being returned. This is common in cases where ad-hoc data analysis is performed or generic tooling is written. To achieve this, column-type information is available on query responses. This can be used with Go reflection to create runtime instances of correctly typed variables which can be passed to Scan.

```go
const query = `
SELECT
        1     AS Col1
    , 'Text' AS Col2
`
rows, err := conn.Query(context.Background(), query)
if err != nil {
    return err
}
var (
    columnTypes = rows.ColumnTypes()
    vars        = make([]interface{}, len(columnTypes))
)
for i := range columnTypes {
    vars[i] = reflect.New(columnTypes[i].ScanType()).Interface()
}
for rows.Next() {
    if err := rows.Scan(vars...); err != nil {
        return err
    }
    for _, v := range vars {
        switch v := v.(type) {
        case *string:
            fmt.Println(*v)
        case *uint8:
            fmt.Println(*v)
        }
    }
}
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/dynamic_scan_types.go)


## External tables

[External tables](https://clickhouse.com/docs/en/engines/table-engines/special/external-data/) allow the client to send data to ClickHouse, with a SELECT query. This data is put in a temporary table and can be used in the query itself for evaluation.

To send external data to the client with a query, the user must build an external table via ext.NewTable before passing this via the context.

```go
table1, err := ext.NewTable("external_table_1",
    ext.Column("col1", "UInt8"),
    ext.Column("col2", "String"),
    ext.Column("col3", "DateTime"),
)
if err != nil {
    return err
}

for i := 0; i < 10; i++ {
    if err = table1.Append(uint8(i), fmt.Sprintf("value_%d", i), time.Now()); err != nil {
        return err
    }
}

table2, err := ext.NewTable("external_table_2",
    ext.Column("col1", "UInt8"),
    ext.Column("col2", "String"),
    ext.Column("col3", "DateTime"),
)

for i := 0; i < 10; i++ {
    table2.Append(uint8(i), fmt.Sprintf("value_%d", i), time.Now())
}
ctx := clickhouse.Context(context.Background(),
    clickhouse.WithExternalTable(table1, table2),
)
rows, err := conn.Query(ctx, "SELECT * FROM external_table_1")
if err != nil {
    return err
}
for rows.Next() {
    var (
        col1 uint8
        col2 string
        col3 time.Time
    )
    rows.Scan(&col1, &col2, &col3)
    fmt.Printf("col1=%d, col2=%s, col3=%v\n", col1, col2, col3)
}
rows.Close()

var count uint64
if err := conn.QueryRow(ctx, "SELECT COUNT(*) FROM external_table_1").Scan(&count); err != nil {
    return err
}
fmt.Printf("external_table_1: %d\n", count)
if err := conn.QueryRow(ctx, "SELECT COUNT(*) FROM external_table_2").Scan(&count); err != nil {
    return err
}
fmt.Printf("external_table_2: %d\n", count)
if err := conn.QueryRow(ctx, "SELECT COUNT(*) FROM (SELECT * FROM external_table_1 UNION ALL SELECT * FROM external_table_2)").Scan(&count); err != nil {
    return err
}
fmt.Printf("external_table_1 UNION external_table_2: %d\n", count)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/external_data.go)

## Open Telemetry

ClickHouse allows a [trace context](https://clickhouse.com/docs/en/operations/opentelemetry/) to be passed as part of the native protocol. The client allows a Span to be created via the function `clickhouse.withSpan` and passed via the Context to achieve this.

```go
var count uint64
rows := conn.QueryRow(clickhouse.Context(context.Background(), clickhouse.WithSpan(
    trace.NewSpanContext(trace.SpanContextConfig{
        SpanID:  trace.SpanID{1, 2, 3, 4, 5},
        TraceID: trace.TraceID{5, 4, 3, 2, 1},
    }),
)), "SELECT COUNT() FROM (SELECT number FROM system.numbers LIMIT 5)")
if err := rows.Scan(&count); err != nil {
    return err
}
fmt.Printf("count: %d\n", count)
```

[Full Example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/open_telemetry.go)

Full details on exploiting tracing can be found under [OpenTelemetry support](https://clickhouse.com/docs/en/operations/opentelemetry/).

