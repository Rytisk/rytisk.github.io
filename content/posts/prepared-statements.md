---
title: "I Forgot About Prepared Statements"
date: 2026-05-29
tags: ["go", "mysql"]
---

Currently I'm reading Efficient MySQL Performance book, which I really like so far. Even though I just started reading chapter 4 I already have some takeaways and one thing I thought would be interesting to dive deeper into, because I didn't really know how it worked.

After chapter 1, there's a practice section which suggests to try `pt-query-digest`[^1]. First you need to enable the slow query log on a MySQL instance and then use the `pt-query-digest` tool on that log which then prints out details about queries. I decided to give it a shot since I had never used this Percona tool and I thought it would be interesting.

Although the output would be more interesting on a live MySQL instance, my playground project would have to suffice. This playground project I have is just a docker-compose that runs MySQL, Prometheus, Grafana, and two simple applications I wrote to imitate traffic: `imposter` which is the API that exposes get/put functionality and accesses MySQL and `tester` which continuously calls `imposter` to generate the traffic. Both services expose some metrics, which promo scrapes, and this allows me to make nice Grafana dashboards. I created this project just so I could do some performance testing and in this case, it was a good fit as I just needed a query log from MySQL.

Once I started the containers with `docker-compose up -d` the traffic began to generate. The first thing I had to do was enable the slow query log on MySQL. For this, you need root access.

Using MySQL client, I connected with the root user, and ran these commands to enable the slow query log:
```
SET GLOBAL long_query_time=0;
SET GLOBAL slow_query_log=ON;
```

Then ran this to get the path to the query log file:
```
SELECT @@GLOBAL.slow_query_log_file;
```

Copied the file to the host:
```
docker cp mysql-container:/var/lib/mysql/634260e1cb52-slow.log .
```

And ran the tool:
```
pt-query-digest 634260e1cb52-slow.log
```

Which gave me a report:
```

# 1.9s user time, 20ms system time, 38.41M rss, 415.15G vsz
# Overall: 94.77k total, 6 unique, 1.05k QPS, 0.06x concurrency __________
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time             5s     1us    10ms    55us   167us   197us    27us
# Lock time           26ms       0   683us       0     1us     2us       0
# Rows sent         30.87k       0       1    0.33    0.99    0.47       0
# Rows examine      30.87k       0       1    0.33    0.99    0.47       0
# Query size         3.43M      27      51   37.96   49.17    9.01   31.70

# Profile
# Rank Query ID                            Response time Calls R/Call V/M 
# ==== =================================== ============= ===== ====== ====
#    1 0x75E332F7CE450BCBCCF7D647033FED44   3.8820 74.1% 31600 0.0001  0.00 SELECT kvstore_json
#    2 0xDA556F9115773A1A99AA0165670CE848   1.2613 24.1% 31600 0.0000  0.00 ADMIN PREPARE
# MISC 0xMISC                               0.0923  1.8% 31565 0.0000   0.0 <4 ITEMS>

# Query 1: 405.13 QPS, 0.05x concurrency, ID 0x75E332F7CE450BCBCCF7D647033FED44 at byte 11478527
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         33   31600
# Exec time     74      4s    16us     8ms   122us   332us   322us    54us
# Lock time     99    26ms       0   683us       0     1us     3us     1us
# Rows sent     99  30.86k       1       1       1       1       0       1
# Rows examine  99  30.86k       1       1       1       1       0       1
# Query size    44   1.53M      48      51   50.89   49.17    0.22   49.17
SELECT * FROM kvstore_json WHERE `key` = 'key_7171'\G

# Query 2: 405.13 QPS, 0.02x concurrency, ID 0xDA556F9115773A1A99AA0165670CE848 at byte 16302984
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         33   31600
# Exec time     24      1s     8us    10ms    39us    84us    73us    27us
# Lock time      0       0       0       0       0       0       0       0
# Rows sent      0       0       0       0       0       0       0       0
# Rows examine   0       0       0       0       0       0       0       0
# Query size    26 925.78k      30      30      30      30       0      30
administrator command: Prepare\G

```

Which, as expected, showed that my query took most of the time. What I have not expected was to see `ADMIN PREPARE`. My initial thought was that this is something that the MySQL driver communicates with the server occasionally, but the `Count` matched exactly that of my actual SQL query, so it was evident that this `ADMIN PREPARE` happens every time I make my query.

After a quick search, I got the answer - prepared statements. It was a bit embarrassing I forgot about them. Although I work with MySQL in my day-to-day job, I haven't used prepared statements in a while. Well, I haven't used them explicitly, it turns out I use them all the time, I just didn't realize that. 


## Prepared statements

I had used prepared statements in the past for a tool that had to do lots of queries with different arguments. Instead of just executing the query multiple times, a better approach is to create a prepared statement - and then execute it multiple times by passing different arguments.

This way you optimize your queries. A prepared statement creates an object in MySQL for the lifetime of a connection (session). For the rest of the connection you can execute that prepared statement and pass your query arguments. MySQL parses the query once during the prepare, and so, you avoid the overhead of parsing it each time.

## Why was it in the profile?

I thought I hadn't used them, but turns out the parameterized query I wrote used a prepared statement under the hood. So my query that I executed using `go-sql-driver/mysql`:
```
rows, err := db.Query("SELECT * FROM kvstore_json WHERE `key` = ?", key)
```
first prepares a statement, executes it and then closes it. This is by default in `go-sql-driver/mysql`, but using `interpolateParams=true` configuration value, you can override this behavior. The driver avoids the prepared statement by interpolating params into the query. So even if you use a parameterized query in your client code, the server will get plain text SQL with params interpolated.

Prepared statements are not used only for performance gain. In my use-case (for one-off queries) there's little performance benefit in that regard. The other important benefit is that parameter values are transmitted separately from SQL, which helps avoid SQL injection. `interpolateParams=true` is still considered safe with some caveats[^2]:
> This can not be used together with the multibyte encodings BIG5, CP932, GB2312, GBK or SJIS. These are rejected as they may introduce a SQL injection vulnerability!

But I would still keep it as `false` - there's usually little performance gain from removing that extra roundtrip for typical workloads. Let's be real - in practice, there are a lot more bottlenecks to fix or optimizations to make first. Unless you have a very specific case - you're doing some micro-optimizations or you have a high-latency network, I'd avoid enabling the `interpolateParams`.

[^1]: https://docs.percona.com/percona-toolkit/pt-query-digest.html
[^2]: https://github.com/go-sql-driver/mysql/blob/a065b60ab6d0c8e15468e7709c7f76acf4431647/README.md#interpolateparams
