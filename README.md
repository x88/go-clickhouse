# go-clickhouse 

Golang [Yandex ClickHouse](https://clickhouse.yandex/) HTTP connector

ClickHouse manages extremely large volumes of data in a stable and sustainable manner.
It currently powers Yandex.Metrica, world’s second largest web analytics platform,
with over 13 trillion database records and over 20 billion events a day, generating
customized reports on-the-fly, directly from non-aggregated data. This system was
successfully implemented at CERN’s LHCb experiment to store and process metadata on
10bn events with over 1000 attributes per event registered in 2011.

## Examples

#### Query rows

```go
conn := clickhouse.NewConn("localhost:8123", clickhouse.NewHttpTransport())
query := clickhouse.NewQuery("SELECT name, date FROM clicks")
iter := query.Iter(conn)
var (
    name string
    date string
)
for iter.Scan(&name, &date) {
    //
}
if iter.Error() != nil {
    log.Panicln(iter.Error())
}
```

#### Single insert
```go
conn := clickhouse.NewConn("localhost:8123", clickhouse.NewHttpTransport())
query, err := clickhouse.BuildInsert("clicks",
    clickhouse.Columns{"name", "date", "sourceip"},
    clickhouse.Row{"Test name", "2016-01-01 21:01:01", clickhouse.Func{"IPv4StringToNum", "192.0.2.192"}},
)
if err == nil {
    err = query.Exec(conn)
    if err == nil {
        //
    }
}
```

#### Auth and multiple insert
```go
conn := clickhouse.NewConn("localhost:8123", clickhouse.NewHttpTransport())
conn.AddParam("user", "{username}")
conn.AddParam("password", "{password}")

queryStr := `INSERT INTO clicks FORMAT TabSeparated
1	2017-09-27	10
2	2017-09-27	11
3	2017-09-27	12`

query := clickhouse.NewQuery(queryStr)
iter := query.Iter(conn)

if iter.Error() != nil {
    //
}
```

#### Fetch rows
```go
conn := clickhouse.NewConn("localhost:8123", clickhouse.NewHttpTransport())

queryStr := `SELECT visit_id, visit_number FROM clicks ORDER BY created_at DESC LIMIT 5`
query := clickhouse.NewQuery(queryStr)

type fetch struct {
    VisitId     string  `json:"visit_id"`
    VisitNumber int     `json:"visit_number"`
}
fetchObj := []fetch{}

err := query.ExecScan(dbConnection, &fetchObj)

if err != nil {
    //
}

/**
([]fetch)
  0(fetch)
    VisitId(string) "ufifgdwp0y0wfiqp-7887"
    VisitNumber(int) 1
  1(fetch)
    VisitId(string) "ufifgdwp0y0wfiww-5356"
    VisitNumber(int) 1
  2(fetch)
    VisitId(string) "ufifgdwp0y0wfiwt-408"
    VisitNumber(int) 2
...
*/
```

#### External data for query processing

[See documentation for details](https://clickhouse.yandex/reference_en.html#External%20data%20for%20query%20processing) 

```go
conn := clickhouse.NewConn("localhost:8123", clickhouse.NewHttpTransport())
query := clickhouse.NewQuery("SELECT Num, Name FROM extdata")
query.AddExternal("extdata", "Num UInt32, Name String", []byte("1	first\n2	second")) // tab separated


iter := query.Iter(conn)
var (
    num  int
    name string
)
for iter.Scan(&num, &name) {
    //
}
if iter.Error() != nil {
    log.Panicln(iter.Error())
}
```

## Cluster

Cluster is useful if you have several servers with same `Distributed` table (master). In this case you can send
requests to random master to balance load.

* `cluster.Check()` pings all connections and filters active ones
* `cluster.ActiveConn()` returns random active connection
* `cluster.OnCheckError()` is called when any connection fails

**Important**: You should call method `Check()` at least once after initialization, but we recommend
to call it continuously, so `ActiveConn()` will always return filtered active connection.

```go
http := clickhouse.NewHttpTransport()
conn1 := clickhouse.NewConn("host1", http)
conn2 := clickhouse.NewConn("host2", http)

cluster := clickhouse.NewCluster(conn1, conn2)
cluster.OnCheckError(func (c *clickhouse.Conn) {
    log.Fatalf("Clickhouse connection failed %s", c.Host)
})
// Ping connections every second
go func() {
    for {
        cluster.Check()
        time.Sleep(time.Second)
    }
}()
```

## Transport options

### Timeout

```go
t := clickhouse.NewHttpTransport()
t.Timeout = time.Second * 5

conn := clickhouse.NewConn("host", t)
```
