---
title: 'Why the Same ClickHouse Query Passed in the CLI and Failed in Go'
published: 2026-10-15
updated: 2026-10-19
description: 'One ClickHouse query used two analyzers. The fix shipped in 26.8.30.8 and passed tests with the original Go 2.31.0 client.'
image: ''
tags: [ClickHouse, Go, Debugging, Distributed Systems]
category: 'Coding'
draft: false
lang: 'en'
slug: clickhouse-distributed-query-two-analyzers
---

The SQL worked in `clickhouse-client`. The same query failed through a Go service with `NOT_FOUND_COLUMN_IN_BLOCK`.

The error named a column that appeared in the returned data. The difference was its name: the coordinator expected `__table1.event_name`, but a remote server returned `event_name`.

An analyzer resolves SQL names and plans query execution. The coordinator used the new analyzer. A remote server silently switched to the old query interpreter.

Two separate behaviors caused that switch. ClickHouse omitted an explicit analyzer choice from the remote query packet, then interpreted the Go driver's version as an old ClickHouse server version.

This investigation started with ClickHouse `25.8.11.66`. The examples below remove internal names, credentials, and customer data. The original diagnosis dates from 2026-10-15. The release verification below dates from 2026-10-19.

:::tip
Upstream merged [PR #125992](https://github.com/ClickHouse/ClickHouse/pull/125992) into the `26.8` LTS branch. The fix ships in [ClickHouse `26.8.30.8`](https://github.com/ClickHouse/ClickHouse/releases/tag/v26.8.30.8-lts). The [released binary passed the regression and original Go client tests](#testing-the-released-fix-with-the-original-go-driver).
:::

## The failure appeared after the remote query completed

A coordinator is the server that distributes a query. A remote server executes part of the query. It returns data to the coordinator. A block is a batch of columns and rows. A distributed aggregation can return intermediate blocks that the coordinator must combine.

The application sent an HTTP request to an analytics API. The Go service then connected to ClickHouse through its native TCP protocol. The native protocol carries binary messages for ClickHouse connections. The API transport did not describe the database transport.

The query grouped events and requested totals. This reduced query preserves the relevant SQL structure:

```sql
SELECT
    events.event_name AS name,
    sum(events.event_count) AS total
FROM events_dist AS events
GROUP BY 1 WITH TOTALS
ORDER BY name;
```

The remote query completed. The coordinator failed when it adapted the returned block to the expected structure. The observed stack included these functions:

```text
Block::getPositionByName
adaptBlockStructure
RemoteQueryExecutor::processPacket
RemoteSource::tryGenerate
```

[`adaptBlockStructure`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/QueryPipeline/RemoteQueryExecutor.cpp#L307-L358) looks up columns by their expected names. An unqualified name did not satisfy its lookup for `__table1.event_name`.

That evidence placed the failure at the boundary between two servers. It did not indicate that the stored table lacked an event column.

## First, make the comparison equal

The original successful SQL and the API query used different end times. Their totals differed for that reason. I used identical time bounds before comparing the clients.

I also held the coordinator and remote route constant. I disabled the query cache and bounded each diagnostic query. The failing service used `clickhouse-go/v2 v2.31.0`.

The following table summarizes the controlled runs. “Analyzer” shows the effective value recorded for each part of the query. A successful row means that the result matched the other successful runs.

| Request on the same failing route | Coordinator analyzer | Remote analyzer | Result |
| --- | ---: | ---: | --- |
| Go 2.31.0, existing configuration | 1 | 0 | Code 10 |
| Go 2.31.0, explicit `enable_analyzer=1` through the driver | 1 | 0 | Code 10 |
| Go 2.31.0, explicit `enable_analyzer=1` in SQL | 1 | 0 | Code 10 |
| Go 2.31.0, explicit `enable_analyzer=0` | 0 | 0 | Success |
| Official native client 26.4.1 | 1 | 1 | Success |
| Same Go code, reported major version changed from 2 to 26 | 1 | 1 | Success |

Another replica route also succeeded with the unchanged Go driver. The problem depended on the remote path and connection state. It did not affect every replica on every request.

The version-only control gave the strongest clue. The modified Go driver reported `26.31.0`, but it still used native protocol revision `54460`. The SQL, server binary, route, and driver logic stayed the same.

A protocol revision identifies supported message fields. An application version identifies a release of that application. These numbers describe different things.

Changing a reported version was a diagnostic experiment. It is not a production fix. The test isolated a server decision that depended on the application version.

## Two connections used two accounts

The application account did not automatically become the account for the remote connection. The inspected cluster configuration omitted `<user>` and contained no cluster shared secret. In this mode, [`Cluster::Address`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Interpreters/Cluster.cpp#L101-L118) uses `default` when `<user>` is absent.

The following example uses anonymous names. It shows the relevant shape of one replica entry, with unrelated elements omitted:

```xml
<clickhouse>
    <remote_servers>
        <analytics_cluster>
            <shard>
                <replica>
                    <host>replica-a.example.invalid</host>
                    <port>9000</port>
                </replica>
            </shard>
        </analytics_cluster>
    </remote_servers>
</clickhouse>
```

The first connection authenticated as `app_reader`. The coordinator opened a second connection as `default`. Query metadata still described the original application, but that metadata did not authenticate the remote connection.

![A Go application connects as app_reader to a coordinator using analyzer 1. The coordinator connects as default to a remote server, which changes its analyzer to 0.](/images/clickhouse-two-analyzers.svg)

The [full-size diagram](/images/clickhouse-two-analyzers.svg) shows both connections.

A profile is a named collection of configuration values. ClickHouse had both a user named `default` and a base profile named `default`. The shared name caused another source of confusion.

The user named `default` selected a different profile, here called `query_profile`. The application user selected `readonly`. The following reduced examples show that distinction.

The server configuration selected the base profile:

```xml
<!-- config.xml -->
<clickhouse>
    <default_profile>default</default_profile>
</clickhouse>
```

The user configuration selected profiles for each account:

```xml
<!-- Reduced users.xml and users.d content -->
<clickhouse>
    <profiles>
        <default>
            <allow_experimental_analyzer>1</allow_experimental_analyzer>
        </default>
        <readonly>
            <readonly>1</readonly>
        </readonly>
        <query_profile>
            <!-- Other query limits and cache configuration. -->
        </query_profile>
    </profiles>
    <users>
        <app_reader>
            <profile>readonly</profile>
        </app_reader>
        <default>
            <profile>query_profile</profile>
        </default>
    </users>
</clickhouse>
```

[`SettingsProfilesCache`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Access/SettingsProfilesCache.cpp#L120-L145) merges the base profile before the user's own profile. Neither assigned profile overrode the analyzer value. Both accounts therefore started with analyzer `1` in the inspected XML.

The checked-in configuration supported this explanation. I did not read back the complete deployed configuration or all SQL-managed profile overrides. Runtime query records established the account names and the actual analyzer split.

## The coordinator prepared an explicit analyzer choice

ClickHouse already contained code to keep the analyzer consistent across a distributed query. [`MultiplexedConnections`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Client/MultiplexedConnections.cpp#L180-L184) explicitly assigned the coordinator's value before sending remote queries.

The assignment used the older name, `allow_experimental_analyzer`. In this release, [`enable_analyzer` is an alias](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Core/Settings.cpp#L6641-L6643) for the same configuration value. The two names did not select different analyzers.

This assignment appeared in the source:

```cpp
modified_settings.set(
    "allow_experimental_analyzer",
    static_cast<bool>(modified_settings[Setting::allow_experimental_analyzer]));
```

An explicit assignment marks the value as `changed`. That flag records an explicit choice, even when the value equals an existing default. The receiver uses the flag to distinguish an explicit choice from an absent one.

At this point, the coordinator selected `1` and marked it for transmission. Another part of the connection code then removed it.

## The sender removed the choice after the Hello message

A Hello message starts a native connection. The server can include profile configuration in its Hello response. On this release, `apply_settings_from_server=1` permits that advertisement.

The remote account explicitly inherited analyzer `1` from its profile. Its Hello therefore advertised that value. The coordinator recorded the advertised values in `settings_from_server`.

Before sending a query, [`Connection::sendQuery`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Client/Connection.cpp#L882-L905) compared outgoing values with the advertised values. When the two values matched, it cleared the outgoing change:

```cpp
Field value;
if (settings->tryGet(change.name, value) && value == change.value)
{
    // Mark as unchanged so it's not sent.
    modified_settings->setDefaultValue(change.name);
    chassert(!modified_settings->isChanged(change.name));
}
```

This optimization avoided sending a value that the server already knew. For the analyzer, it also removed the coordinator's explicit choice. The remote query packet no longer carried the analyzer assignment.

The distinction matters because equal values do not always carry equal intent. “Use analyzer 1 for this distributed query” carries a stronger instruction than “this account usually uses analyzer 1.”

| Stage | Analyzer value or instruction | Explicit choice preserved? |
| --- | --- | --- |
| Coordinator prepares the remote query | Use `1` | Yes |
| Remote Hello advertises its profile | Profile value is `1` | The coordinator sees an equal value |
| Sender removes the matching assignment | No analyzer entry in the query packet | No |
| Receiver reads `passed_settings` | No explicit analyzer assignment | No |

Adding `enable_analyzer=1` to the application did not avoid this path. The coordinator already held that value. The outgoing comparison still found an equal value and omitted it.

## The compatibility branch compared the wrong version

A secondary query is a query sent by another ClickHouse server. ClickHouse `25.8.11.66` included a compatibility branch for secondary queries from servers older than `23.3.0`. It disabled the analyzer when the received query supplied no explicit analyzer choice.

The [`TCPHandler` branch](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Server/TCPHandler.cpp#L2233-L2244) used these fields:

```cpp
if (query_kind == ClientInfo::QueryKind::SECONDARY_QUERY
    && VersionNumber(client_info.client_version_major,
                     client_info.client_version_minor,
                     client_info.client_version_patch)
        < VersionNumber(23, 3, 0)
    && !passed_settings[Setting::allow_experimental_analyzer].changed)
    passed_settings.set("allow_experimental_analyzer", false);
```

For a distributed query, `client_version_*` described the original application. The Go driver [reported its own release, `2.31.0`](https://github.com/ClickHouse/clickhouse-go/blob/v2.31.0/client_info.go#L28-L34). The comparison treated that independent version as a ClickHouse release older than `23.3.0`.

The field definitions already documented the distinction. [`ClientInfo.h`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Interpreters/ClientInfo.h#L97-L103) directed server compatibility decisions to `connection_client_version_*`. Those fields described the ClickHouse server on the other end of the connection.

The wrong comparison combined with the omitted assignment. The receiver saw an apparently old initiator and no explicit analyzer choice. It set the analyzer to `0`.

The actual peer server ran `25.8.11.66`. The application version carried through the query did not change that fact. The official client reported `26.4.1`, so the same comparison did not select the downgrade branch.

The resulting sequence was:

```text
Coordinator chooses analyzer 1
    -> Remote Hello advertises profile value 1
    -> Sender omits the equal analyzer assignment
    -> Receiver reads original application version 2.31.0
    -> Compatibility branch treats 2.31.0 as an old server
    -> Remote query switches to analyzer 0
    -> Remote returns event_name
    -> Coordinator expects __table1.event_name and throws code 10
```

## Connection reuse exposed a second defect

The same release also assigned `query_kind` after the compatibility branch. The branch read the handler's earlier value. A new connection started with `NO_QUERY`, while a reused connection retained the previous query kind.

That ordering explained a confusing pattern. A first request can pass, then a later request can fail on the same server connection. Changing which connection a query reuses can change the result.

Upstream [PR #118624](https://github.com/ClickHouse/ClickHouse/pull/118624) corrected this ordering. That correction was separate from the comparison against the original application's version. Both details mattered when patching the exact `25.8.11.66` tag.

## A reproduction on the official image

The investigated cluster ran a private build based on `25.8.11.66`. Its build metadata did not identify the exact private source commit. I therefore reproduced the failure on the official `clickhouse/clickhouse-server:25.8.11.66` image.

The [regression fixture](https://github.com/ClickHouse/ClickHouse/pull/125992/files) created a small `Memory` table. It inserted three rows: `('a', 2)`, `('a', 3)`, and `('b', 7)`. A persistent distributed table read that table through two logical shards.

Both logical shards pointed at the same test server. The duplicated input was intentional. The expected result was `a=10`, `b=14`, and a total of `24`.

A proxy forwards messages between two network endpoints. The test proxy changed native message metadata. It reported application version `2.31.0` and retained protocol revision `54460`. It left the SQL and query configuration unchanged.

The fixture also created a dedicated remote account with explicit analyzer `1`. It repeated requests to exercise pooled server connections. On the unmodified official image, the first request produced:

```text
a	10
b	14

	24
```

The second request failed. This was the complete exception from that local run:

```text
Received exception from server (version 25.8.11):
Code: 10. DB::Exception: Received from 127.0.0.1:59201. DB::Exception: Not found column __table1.event_name in block. There are only columns: event_name, sum(event_count): While executing Remote. (NOT_FOUND_COLUMN_IN_BLOCK)
```

A separate control changed the proxy's reported application version to `25.8`. It kept protocol revision `54460` and the same server image. All four distributed queries matched the expected results, with the analyzer enabled and disabled.

That control differed from the earlier Go experiment, which reported `26.31.0`. Both experiments changed application metadata without upgrading the native protocol. Both avoided the failure after that change.

Fresh tests also reproduced the failure on official `26.8.27.9` and `26.9.19.9` binaries. Version-only controls passed on both releases. The 26.9 control kept the analyzer enabled because that release rejects disabling it on an initial query.

The committed test also simulates an old `22.8` initiator and covers explicit analyzer precedence. The complete fixture passes on the released `26.8.30.8` binary, as described below.

The stateless runner can execute the fixture against a `26.8.30.8` server with the ClickHouse test configuration. The shell test expects `clickhouse-client` and Python 3. Run it from the `v26.8.30.8-lts` source tree:

```bash
./tests/clickhouse-test --queries tests/queries --no-random-settings \
    05394_distributed_analyzer_connection_version
```

## A configuration workaround for the Go service

The release tests pass with analyzer `1` and `apply_settings_from_server=1`. This section describes a workaround for affected older versions.

On 25.8, disabling the analyzer for this query succeeded in the controlled run. It made both servers use the old interpreter. That provided a temporary escape, but it did not support the move to the new analyzer.

Enabling the analyzer only in the Go service failed in both tested forms. The driver configuration and SQL `SETTINGS` clause reached the same omission path. A client upgrade alone also does not correct the server's choice of version fields.

A proposed workaround keeps the new analyzer and disables profile advertisement on the remote account. The relevant value is `apply_settings_from_server=0`. The placement of that value matters more than its name suggests.

In this configuration, the remote account is `default`, and its assigned profile is `query_profile`. The following change therefore belongs in that profile on the remote servers. This is an anonymous example for XML-managed users:

```xml
<clickhouse>
    <profiles>
        <query_profile>
            <apply_settings_from_server>0</apply_settings_from_server>
        </query_profile>
    </profiles>
</clickhouse>
```

[`TCPHandler::sendHello`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Server/TCPHandler.cpp#L1951-L1957) sends an empty configuration list when that profile value is false. The coordinator then lacks the advertised analyzer value that caused the omission. Its explicit analyzer assignment remains in the query packet.

The remote receiver now sees `passed_settings[allow_experimental_analyzer].changed=true`. The faulty version comparison still exists, but the final condition no longer matches. The remote preserves analyzer `1`.

The following table compares `apply_settings_from_server=1` with `0`.

| Connection behavior | Value `1` | Value `0` |
| --- | --- | --- |
| Remote Hello advertises analyzer `1` | Yes | No |
| Sender removes the equal analyzer assignment | Yes | No |
| Receiver gets an explicit analyzer choice | No | Yes |
| Faulty compatibility branch can override that choice | Yes | No |

This explanation comes from the source path. I did not apply or test this profile change on the deployed cluster. Its operational result remains untested in this investigation.

The profile value must affect the remote connection's Hello. Setting it only on the application account does not alter the remote account. Sending it inside an application query also comes after the relevant handshake.

A handshake is the initial exchange on a connection. Existing coordinator-to-remote connections already hold their Hello data. The workaround therefore requires fresh connections on that server-to-server path.

Restarting the Go service only resets its own connections to coordinators. It does not guarantee fresh connections from coordinators to remote servers. Operators must account for that separate connection pool when testing the profile change.

The change also stops advertisement of other profile values. It does not disable the server's profile or its query constraints. Tests must cover any client parsing or formatting that depends on advertised configuration.

The [configuration definition](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Core/Settings.cpp#L6797-L6804) also places `apply_settings_from_server` in a user profile. SQL-managed accounts can use SQL administration instead of XML. The inspected accounts used XML, so a generic `ALTER USER` command was not an appropriate replacement.

## The patch and the release boundary

Upstream merged [PR #125992](https://github.com/ClickHouse/ClickHouse/pull/125992) into the `26.8` LTS branch on 2026-10-17. Release [`v26.8.30.8-lts`](https://github.com/ClickHouse/ClickHouse/releases/tag/v26.8.30.8-lts) includes the fix. Development `master` already removed the old analysis path. The patch moves the current query kind before the branch and uses the connecting server's version.

The [upstream issue](https://github.com/ClickHouse/ClickHouse/issues/125989) records the released-version reproductions and version-only controls.

The [released compatibility logic](https://github.com/ClickHouse/ClickHouse/blob/v26.8.30.8-lts/src/Server/TCPHandler.cpp#L2780-L2795) uses the current query kind and the connecting peer:

```cpp
query_kind = state->query_context->getClientInfo().query_kind;

if (query_kind == ClientInfo::QueryKind::SECONDARY_QUERY
    && VersionNumber(
           client_info.connection_client_version_major,
           client_info.connection_client_version_minor,
           client_info.connection_client_version_patch)
        < VersionNumber(23, 3, 0)
    && !passed_settings[Setting::allow_experimental_analyzer].changed)
    passed_settings.set("allow_experimental_analyzer", false);
```

The patch preserves the existing fallback for an actual older server. It also preserves explicit analyzer choices from that server. It changes no native protocol fields.

The following table retains the source findings from 2026-10-15 and adds the release verified on 2026-10-19. Source inspection and binary tests provide different evidence.

| Source | Status of this compatibility path |
| --- | --- |
| [`v25.8.11.66-lts`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.11.66-lts/src/Server/TCPHandler.cpp#L2233-L2244) | Wrong application-version comparison and late query-kind assignment |
| [`v25.8.33.6-lts`](https://github.com/ClickHouse/ClickHouse/blob/v25.8.33.6-lts/src/Server/TCPHandler.cpp#L2251-L2262) | Both defects remain in the inspected source |
| [`v26.8.27.9-lts`](https://github.com/ClickHouse/ClickHouse/blob/v26.8.27.9-lts/src/Server/TCPHandler.cpp#L2780-L2797) | Both defects remain, and the official binary reproduces the failure |
| [`v26.8.30.8-lts`](https://github.com/ClickHouse/ClickHouse/blob/v26.8.30.8-lts/src/Server/TCPHandler.cpp#L2780-L2795) | Both defects are corrected. The released binary passes the regression and Go client tests |
| [`v26.9.19.9-stable`](https://github.com/ClickHouse/ClickHouse/blob/v26.9.19.9-stable/src/Server/TCPHandler.cpp#L2826-L2839) | Query-kind ordering is corrected, but the application-version comparison remains |
| [Inspected master commit](https://github.com/ClickHouse/ClickHouse/tree/17a0ca309467320fcd36bc8850bd193455d84fd8) | The old analyzer compatibility path is removed |

During the original 2026-10-15 investigation, GitHub identified [`v26.9.19.9-stable`](https://github.com/ClickHouse/ClickHouse/releases/tag/v26.9.19.9-stable) as the latest stable release. Upstream [PR #120311](https://github.com/ClickHouse/ClickHouse/pull/120311) removed the old analysis implementation on the 26.10 development line. That removal did not supply a backport for the deployed 25.8 tag.

The earlier 25.8 variant of the changed `TCPHandler.cpp` compiled as a translation unit on macOS arm64 with Clang `23.1.3`. A translation unit is one source file compiled with its headers. This compiled the actual changed code, but it did not produce a complete server.

The local build used the project's `WERROR=OFF` option. The default build stopped on new compiler warnings in existing headers. The patch changed no build rules or compiler flags.

## Testing the released fix with the original Go driver

On 2026-10-19, I tested the official `clickhouse-common-static-26.8.30.8-arm64.tgz` package in isolated Linux ARM64 containers. Its SHA-512 matched the published checksum. The running server reported commit `da45179b6c6761981c969859d168dd93b2bfd550`, matching the release tag.

I reused the setup with two logical shards described above. The account kept `allow_experimental_analyzer=1` and `apply_settings_from_server=1`. Queries used `prefer_localhost_replica=0` to force TCP execution and `use_query_cache=0` to avoid cached results.

The first test used the [fixture shipped in the release](https://github.com/ClickHouse/ClickHouse/blob/v26.8.30.8-lts/tests/queries/0_stateless/05394_distributed_analyzer_connection_version.sh). It matched the complete reference with Bash tracing enabled. Five more executions through the official runner passed, with zero skipped tests.

I then used the actual [`clickhouse-go/v2 v2.31.0`](https://github.com/ClickHouse/clickhouse-go/tree/v2.31.0) driver. The client ran on macOS ARM64 and used a connection pool limit of one. It ran 20 aggregations per mode: the inherited profile default, explicit analyzer `1`, and explicit analyzer `0`.

| Test | Official `26.8.27.9` baseline | Official `26.8.30.8` release |
| --- | --- | --- |
| Complete native-protocol fixture | First aggregation passes. Second fails with code 10 | Full reference matches |
| Five runs through the official runner | Not repeated | Five passed, zero skipped |
| Actual Go 2.31.0 client | First aggregation passes. Second fails with code 10 | All 60 aggregations pass |

Every successful aggregation returned `a=10`, `b=14`, and total `24`. Server query logs recorded client version `2.31.0` and protocol revision `54460`. They contained 60 initial queries and 120 secondary queries, matching two remote shards per aggregation.

The simulated `22.8` initiator also retained its required behavior. Both implicit requests selected analyzer `0`. An explicit assignment of analyzer `1` remained `1`.

The unchanged Go client now succeeds with the profile default and profile advertisement enabled. The older baseline still fails on the second request. This validates the released server fix for the reproduced failure.

The [verification summary in JSON](/data/clickhouse-analyzer-26.8.30.8-verification.json) records the package checksum, versions, configuration, and results. These tests validate the released binary in isolated containers. Production rollout and results through the analytics HTTP API remain unverified.

## What the evidence proves

The earlier translation-unit build did not produce a complete 25.8 server. The later runtime tests used the official 26.8 release package. The following table records the evidence as of 2026-10-19:

| Claim or artifact | Evidence status |
| --- | --- |
| Coordinator and remote used different analyzers | Observed in runtime query records |
| Application version controls the failing branch | Isolated by version-only controls |
| The failure exists outside the private build | Reproduced on official 25.8.11.66, 26.8.27.9, and 26.9.19.9 binaries |
| The 25.8 patch translation unit compiles | Passed with `WERROR=OFF` |
| Shell and embedded Python syntax | Passed, including optimized Python |
| Official full 26.8 server binary with the fix | `26.8.30.8` package checksum and build identity matched |
| Complete regression on the released server | Passed directly and in five runner repetitions |
| Original Go 2.31.0 client against the release | 60 aggregations passed across three analyzer modes |
| Profile workaround with fresh remote connections | Source-supported, not tested |
| Production deployment of the fix | Not performed |

The central error was a server compatibility decision based on an application's release number. The dropped explicit assignment allowed that decision to override the coordinator. The resulting column error appeared later, when two different analyzers met at a distributed query boundary.
