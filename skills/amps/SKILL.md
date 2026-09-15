---
name: amps
description: Configure a 60East AMPS (Advanced Message Processing System) server and connect to it from JavaScript/Node.js — installation, XML server config (transports, SOW, transaction log, production settings), and the AMPS JS client (connect, publish, subscribe, SOW queries, queues, HA/failover, auth). Use when the user mentions AMPS, crankuptheamps.com, ampServer, or wants to set up pub/sub or a SOW-backed topic against an AMPS instance.
---

# AMPS (Advanced Message Processing System)

AMPS (60East Technologies) is a hybrid messaging engine + queryable database: pub/sub and
competing-consumer queues, a "State of the World" (SOW) cache/document store per topic,
transaction-log record & replay, and built-in aggregation — all driven by one XML config file.

Full docs: https://crankuptheamps.com/documentation (server guides under `/docs`, client SDKs
under `/clients`). This skill condenses the parts needed to stand up a server and talk to it
from JavaScript; fetch a specific `/docs/...` or `/clients/amps-client-javascript/...` page for
anything not covered here.

## 0. Local dev instance (this machine)

A minimal AMPS server is already installed and running under WSL — reuse it instead of
installing/starting a new one unless the task specifically calls for a different config.

- `$AMPSDIR` = `~/amps/AMPS-5.3.5.130-Release-Linux`
- Config file: `~/code/amp-server/amps_config.xml` (Windows-side path
  `~\code\amp-server\amps_config.xml` is the same file via the WSL/Windows filesystem bridge)
- Instance name: `AMPS-Sample`. Running as the `ampServer` process, started with:
  `$AMPSDIR/bin/ampServer ~/code/amp-server/amps_config.xml`
- Transports: TCP/`amps` protocol on port **9007**, WebSocket/`amps` protocol on port **9008**
  — no `MessageType` restriction on either transport (accepts any known type, e.g. `json`).
  JS client connection string: `ws://localhost:9008/amps/json`
- Admin/Galvanometer UI: http://localhost:8085/#/ (`Admin/InetAddr` 8085, using the `any-ws`
  transport for queries)
- Logging: stdout, `error` level only (plus the 00-0015 startup info message)
- **Not yet configured**: no `SOW` topics, no `TransactionLog`, no `Authentication`/
  `Entitlement`, no `Replication` — this is the bare `--sample-config` output. Adding any of
  these means editing `amps_config.xml` and restarting `ampServer` (find the running instance
  with `ps aux | grep ampServer` and check listening ports with `ss -tlnp | grep -E "9007|9008|8085"`
  before assuming it's down or starting a duplicate).

## 1. Install & start the server

Linux only for the eval/download. Installing is just unpacking the distribution — no package
manager step.

```bash
# unpack the downloaded distribution to $AMPSDIR
# layout: $AMPSDIR/bin (ampServer + utilities), lib, sdk, docs, info

# generate a minimal sample config
$AMPSDIR/bin/ampServer --sample-config > $AMPSDIR/amps_config.xml

# start the server with that config
$AMPSDIR/bin/ampServer $AMPSDIR/amps_config.xml
```

A successful start prints a version banner (`AMPS A.B.C.D... - Copyright ...`). AMPS writes
logs/persistence relative to the current working directory, so `cd` into a working dir of your
choice before starting it. Client SDKs are a separate download from the developer page.

## 2. Server configuration (XML)

Every instance needs, at minimum: a unique `Name`, one or more `Transports` for clients to
connect on, and `Logging`. `Admin` enables the monitoring/Galvanometer interface. `SOW` and
`TransactionLog` are opt-in additions.

### Minimal config

```xml
<AMPSConfig>
  <Name>test-AMPS-1</Name>

  <Transports>
    <Transport>
      <Name>any-tcp</Name>
      <Type>tcp</Type>
      <InetAddr>9007</InetAddr>
      <MessageType>json</MessageType>
      <Protocol>amps</Protocol>
    </Transport>
    <Transport>
      <Name>any-ws</Name>
      <Type>tcp</Type>
      <InetAddr>9008</InetAddr>
      <MessageType>json</MessageType>
      <Protocol>websocket</Protocol>
    </Transport>
  </Transports>

  <Admin>
    <InetAddr>8085</InetAddr>
    <SQLTransport>any-ws</SQLTransport>
  </Admin>

  <Logging>
    <Target>
      <Protocol>stdout</Protocol>
      <Level>warning</Level>
    </Target>
  </Logging>
</AMPSConfig>
```

Notes:
- `Name` must be unique within a replicated set — no spaces/shell-special characters.
- The JavaScript client requires `Protocol=websocket` (`ws`/`wss` only) and the `amps` command
  protocol — the transport above with `Protocol>websocket` + `amps` under the hood is what the
  JS client connects to (see §3 connection strings).
- Optional instance-level elements: `Group` (replication group, defaults to `Name`),
  `ProcessName`, `Description`, `Environment`, `Tuning`/`NUMA`, `Externals/SSL`,
  `Externals/Crypto`.

### Adding a SOW topic (cache/document store per topic)

```xml
<SOW>
  <Topic>
    <Name>test-sow</Name>
    <MessageType>json</MessageType>
    <FileName>./sow/%n.sow</FileName>
    <Key>/id</Key>
  </Topic>
</SOW>
```

`Key` is the field(s) that uniquely identify a record within that topic's SOW.

### Adding a transaction log (record & replay)

```xml
<TransactionLog>
  <JournalDirectory>./journals</JournalDirectory>
  <JournalArchiveDirectory>/mnt/high-capacity/journals</JournalArchiveDirectory>
  <JournalSize>100MB</JournalSize>
  <Topic>
    <Name>^/orders</Name>
    <MessageType>json</MessageType>
  </Topic>
</TransactionLog>

<Actions>
  <Action>
    <On>
      <Module>amps-action-on-schedule</Module>
      <Options>
        <Every>21:30</Every>
        <Name>Daily Journal Maintenance Plan</Name>
      </Options>
    </On>
    <Do>
      <Module>amps-action-do-archive-journals</Module>
      <Options><Age>3d</Age></Options>
    </Do>
    <Do>
      <Module>amps-action-do-remove-journals</Module>
      <Options><Age>7d</Age></Options>
    </Do>
  </Action>
</Actions>
```

### Production checklist

For any production instance: unique `Name`, `Admin` interface with a stats-persistence path,
`Logging` at `info` (minimum), at least one `Transport`, and scheduled `Actions` for log/journal
maintenance (as above). Add `Authentication`/`Entitlement` modules for identity and per-topic/
field permissions, and `Replication`/`ReplicationDestinations`/`ReplicationTransports` for HA.

### Config element reference (by category)

| Category | Elements |
|---|---|
| Instance | `Name`, `Group`, `ProcessName`, `Description`, `Environment` |
| Transports | `Transports/Transport` (protocol, type, address, message type) |
| Message Types | `MessageTypes` |
| SOW | `SOW/Topic`, conflated topics, `SOW/Queue`, `SOW/View` |
| Transaction Log | `TransactionLog`, `TransactionLog/Topic` |
| Modules | `Modules` (plug-ins) |
| Security | `Authentication`, `Entitlement` |
| Replication | `Replication`, `ReplicationDestinations`, `ReplicationTransports` |
| Operations | `Actions`, `Logging`, `Admin`/Galvanometer |

Full quick reference: https://crankuptheamps.com/docs/amps-user-guide/configuring-amps/config-quick-ref

## 3. JavaScript client

### Install

```bash
bun add amps
```

(npm/`npm install --save amps` also works — this project prefers Bun per global tooling
convention.) Package name is `amps`; it targets both Node.js and browser use.

### Connection strings

```
ws://localhost:9007/amps/json
wss://127.0.0.1:9007/amps/json
ws://user:password@host:port/amps/json
```

Format: `<ws|wss>://<host>:<port>/<protocol>/<messageType>`. The JS client only supports the
`ws`/`wss` transport and the `amps` protocol (no legacy protocols). Credentials can be embedded
in the URI (`user:password@host`) and are picked up by the default `Authenticator`; implement a
custom `Authenticator` for other auth schemes.

### First program (connect + publish)

```javascript
import { Client } from 'amps'

const uri = 'ws://127.0.0.1:9007/amps/json'

async function main() {
  const client = new Client('examplePublisher')
  try {
    await client.connect(uri)
    client.publish('messages', { hi: 'Hello, World!' })
  } catch (err) {
    console.error(err)
  } finally {
    client.disconnect()
  }
}

main()
```

### Subscribe / unsubscribe

```javascript
const onMessage = message => console.log(message.data)

const client = new Client('test')
await client.connect('ws://127.0.0.1:9007/amps/json')

const subId = await client.subscribe(onMessage, 'messages')
// ... later
await client.unsubscribe(subId)          // or client.unsubscribe('all')
```

Equivalent explicit-`Command` form (same for any command type):

```javascript
const cmd = new Command('subscribe').topic('messages')
const subId = await client.execute(cmd, onMessage)
```

### SOW queries

```javascript
const onMessage = message => {
  switch (message.header.command()) {
    case 'group_begin': console.log('--- begin ---'); break
    case 'sow':          console.log(message.data); break
    case 'group_end':    console.log('--- end ---'); break
  }
}

const queryId = await client.sow(onMessage, 'orders', '/symbol="ROL"')
```

### SOW + subscribe (snapshot, then live updates)

```javascript
const onUpdate = message => {
  const cmd = message.header.command()
  if (cmd === 'sow' || cmd === 'p') addOrUpdate(message)   // initial row or publish
  else if (cmd === 'oof') remove(message)                  // out-of-focus (no longer matches)
}

await client.sowAndSubscribe(
  onUpdate,
  'van_location',        // topic
  '/status = "ACTIVE"',  // filter
  { batchSize: 100, options: 'oof' }
)
```

### Queues (competing consumers)

```javascript
client.autoAck(true)              // let the client ack automatically, or:
client.ack(message)               // manually ack after processing a queue message
```

Subscribing to a queue topic uses the same `subscribe()`/`sowAndSubscribe()` calls as pub/sub —
the server-side `SOW/Queue` config is what makes it competing-consumer instead of fan-out.

### Message types

Built-in: `json`, `fix`/`nvfix` (via `FixTypeHelper`), `binary`. Register a custom type helper
for anything else:

```javascript
class XMLTypeHelper {
  serialize(data)   { return [new XMLSerializer().serializeToString(data)] }
  deserialize(data) { return new DOMParser().parseFromString(data) }
}
TypeHelper.helper('xml', new XMLTypeHelper())

TypeHelper.helper('nvfix').delimiter('%01')   // customize FIX/NVFIX delimiter
```

### High availability / failover

The JS `Client` class combines the roles of `Client` and `HAClient` from other AMPS SDKs.

```javascript
// simplest: fully HA, memory-backed
const client = Client.createMemoryBacked('memory-backed')
await client.connect()
```

For explicit control over failover, resubscription, and reconnect backoff:

```javascript
const client = new Client('ha-client-demo')
client.subscriptionManager(new DefaultSubscriptionManager())   // re-subscribe after failover
client.bookmarkStore(new MemoryBookmarkStore())                 // resume subscriptions

const chooser = new DefaultServerChooser()
chooser.add('ws://primary.amps.example.com:9000/amps/json')
chooser.add('ws://secondary.amps.example.com:9000/amps/json')
client.serverChooser(chooser)

client.delayStrategy(new ExponentialDelayStrategy({
  initialDelay: 200,
  maximumDelay: 5000,
  backoffExponent: 1.5,
  maximumRetryTime: 60000,
}))

client.heartbeat(3)   // seconds; faster failure detection

try {
  await client.connect()
  await client.execute(
    new Command('subscribe').topic('market-data').bookmark(Client.Bookmarks.MOST_RECENT),
    message => { console.log(message.data); client.bookmarkStore().discard(message) }
  )
} finally {
  await client.disconnect()
}
```

`ServerChooser` is an interface (`getCurrentUri`, `getCurrentAuthenticator`, `reportSuccess`,
`reportFailure`, `getError`) — implement a custom one for non-default failover/auth logic.

## 4. Where to go deeper

- Server guides index: https://crankuptheamps.com/docs
- Evaluation guide (install/first steps): https://crankuptheamps.com/docs/amps-eval-guide/eval
- Config quick reference: https://crankuptheamps.com/docs/amps-user-guide/configuring-amps/config-quick-ref
- JS client dev guide index: https://crankuptheamps.com/clients/amps-client-javascript
- Other client SDKs (C/C++, C#, Java, Python) live at `/clients/amps-client-<lang>` with the
  same structure as the JS guide.

When a task needs a page not covered above, fetch the specific `/docs/...` or
`/clients/amps-client-javascript/...` URL rather than guessing at API shape — the client API
(method names, `Command` builder chain, HA classes) is specific to this SDK and changes between
AMPS versions.
