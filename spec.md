trade-bot (better name pending) is a event driven real time system that ingests prices, executes algorithmic trading strategies on it and sends generated signals to subscribed consumers.

On a high level, this project consists of a ingestion service that subscribes to data sources, a pubsub system that sends events to and from services, a strategy service that calculates indicator values and executes algorithmic trading strategies on them to generate trading signals, a time series database that stores price and signal data, a dashboard based on the tsdb to visualize events and prices.

---

## Why I'm building this
This is a revamp of an old project I did, for which the complete code is not available. It was ambitious, yet poorly designed and built, had flaky logic and was incredibly inefficient, held together by hopes and dreams.

Nevertheless it did solve a very important problem that I faced - the algo trading platform I used to use had a proprietary language, and errors were not easy to debug, I had minimal control over the whol environment.

I'm building this from scratch again because this is an incredible learning opportunity, I will be able to apply a lot of the things I got to learn since I built this project two years ago, see the mistakes I did while building it then and improve on those and of course learn new things.

---

## Versioning
There are three major versions planned for this project.

v1 is the mvp with hardcoded strategies, where to stop a strategy the container for the service needs to be stopped. All strategies run on a single container, and the system is entirely read only.

v2 enables privileged users to define parameters of a particular strategy and execute it while others are still running, features environment variables to stop a particular strategy executing, different containers for each strategy.

v3 adds write permissions for privileged users - who will be able to write their own code for the strategies they want to run and execute.

---

## Components
### Ingestion service
Takes in data from a source (websocket or polling), processes it, turns the data into candles if not already that way, pushes processed data into the `price.{symbol}` topic.

Language: Python

Candle schema (published to NATS):
```json
{
    "symbol": "EURUSD",
    "source":"binance",
    "timestamp": 1718000000,
    "open": 1.0821,
    "high": 1.0834,
    "low": 1.0819,
    "close": 1.0829,
    "volume": 1024.0
}
```
Timestamp is the unix epoch of the first tick in the candle window. All fields are named.

All data received (whether ticks or full candles) have a timestamp field embedded within, which will be used as the source of truth for the whole system.

The ingestor is built behind a `PriceSource` abstract interface so that swapping data providers requires only a new implementation, not changes to the pipeline.
```python
class PriceSource(ABC):
	@abstractmethod 
	def stream(self) -> Iterator[Tick]: 
		pass
```

A `Tick` contains: `ts` (unix epoch from the data source), `price` (the `last` field or equivalent mid price), `volume` (sum of trade volume in the tick, or tick count if unavailable). 

**CandleBuilder:** 
Receives ticks from the source, accumulates them into candles, emits a completed candle when the current candle is sealed. Two sealing modes depending on data source:
- Interval mode (default): A candle is sealed when the first tick of a new time interval arrives. The interval boundary is computed as `floor(tick.ts / interval_seconds) * interval_seconds`. When a tick's computed interval is greater than the current candle's interval, the current candle is complete and is emitted before starting a new one. The candle is never emitted at a wall-clock boundary — only a newer tick triggers emission. This means there is always exactly one candle of lag, which is correct and expected.

- Tick count mode: A candle is sealed after a fixed number of ticks defined by the data source. A counter increments per tick and resets to zero on seal. In both modes: open is the price of the first tick, close is the price of the last tick before sealing, high and low are the running max and min of price across all ticks in the window, volume is the sum of tick volumes. 

- Edge cases: - First tick ever: initialise a new candle, do not emit. 
- Gap in data (market close, weekend, reconnect): the builder does not synthesise missing candles. On reconnect the next tick starts a fresh candle. Gaps will be visible in the TSDB and dashboard as missing data points, which is correct.
- Reconnection: The ingestion service is responsible for handling websocket drops and polling failures. On reconnect it resumes publishing. Gap detection is the responsibility of the operator reading the dashboard, not the pipeline.

A candle's timestamp is the timestamp of the first event received (from data source). This is taken to be the source of truth for the entire system, and avoids synchronization issues.

### Strategy service
Language: Go

This is the most complex component. It maintains a single event loop that receives candles, updates all indicators, and evaluates all active strategies in sequence.

#### Types
```go
type Candle struct {
    Symbol    string
    Timestamp int64
    Open      float32
    High      float32
    Low       float32
    Close     float32
    Volume    float32
}

type Signal struct {
    Symbol     string
    Timestamp  int64
    Type       uint8
    StrategyID string
    Context    SignalContext
}

type SignalContext struct {
    IndicatorValues map[string]float32
    CandleClose     float32
    CandleTimestamp int64
    Message         string
}
```

`Signal.Type`: 1 = buy, 2 = sell.

`SignalContext.Message` is a human-readable string set by the strategy describing why the signal fired, for example `"RSI crossed below 30, EMA20 above EMA50"`. This is stored as-is in the TSDB context field and surfaced in notifications.

#### Indicator interface

```go
type Indicator interface {
    ID()     string
    Update(candle Candle)
    Value(i int) (float32, bool)
    Ready() bool
}
```

`Value(i int)` returns the indicator value i candles ago: 0 is the current value, 1 is one candle prior, and so on. The bool return is false if the requested index does not exist in the history yet, forcing callers to handle the not-ready case explicitly rather than silently operating on a zero value.

`Ready()` returns true once the indicator has received at least `period` candle updates.

Each indicator stores a history queue of depth equal to its period. `Value(i int)` is valid for `0 <= i < period`. Requesting an index outside this range returns `(0, false)`. Strategies must not request history depth greater than the indicator's period, if a strategy needs 50 candles of history from an EMA, it must declare an EMA with period >= 50.


**On extra parameters (smoothing, etc.):**
Different indicator types need different constructor parameters. EMA needs a smoothing factor, ATR needs nothing beyond period, a custom indicator might need several values. This is handled by passing a `map[string]any` of extra parameters to the constructor, validated at instantiation time.
```go
type IndicatorSpec struct {
    ID     string
    Kind   string
    Period int
    Params map[string]any
}
```

Example: an EMA with custom smoothing declared as:

```go
IndicatorSpec{
    ID:     "ema_20",
    Kind:   "EMA",
    Period: 20,
    Params: map[string]any{"smoothing": 2.0},
}
```

The engine passes `Params` to the indicator constructor. Each indicator type defines which params it expects and uses defaults for anything absent. Unknown params are ignored. This keeps `IndicatorSpec` open to extension without changing its structure.

#### Strategy interface
```go
type Strategy interface {
    ID()      string
    Declare() []IndicatorSpec
    Init(indicators map[string]Indicator)
    Evaluate(candle Candle) *Signal
    IsDuplicate(signal *Signal) bool
}
```

`Declare` returns the list of indicators this strategy needs. Called once at startup, to register the indicators needed with the event loop (which takes care of updates and dependencies).

`Init` receives a map of already-instantiated indicator pointers keyed by indicator ID. The strategy stores these references internally for use in `Evaluate`. Called once after the engine has fulfilled all declarations.

`Evaluate` is called once per candle after all indicators have been updated. Returns nil if no signal.

`IsDuplicate` checks the strategy's internal dedup window. The dedup window size K is a field on the strategy's own config struct, set at instantiation time and fixed for the lifetime of the strategy. It is not a global constant, each strategy defines an appropriate K based on how much noise is acceptable. A signal is considered duplicate if any of the last K signals from this strategy had the same type.


#### Startup and registration
Initiation step - this happens once before any candles are processed:
```
1. Instantiate all strategies defined in the registry.

2. For each strategy:
     call Declare() to get []IndicatorSpec
     for each spec:
         if spec.ID is not already in engine.indicators:
             create indicator from spec.Kind, spec.Period, spec.Params
             register in engine.indicators[spec.ID]
         append spec.ID to engine.deps[strategy.ID()]

3. Compute MAX_LOOKBACK = max(spec.Period across all registered indicator specs)

4. For each strategy:
     build map of indicator ID → Indicator pointer from engine.indicators
     call strategy.Init(that map)

5. Set engine.candlesReceived = 0

6. Begin consuming candles from NATS.
````

Two strategies declaring a spec with the same ID receive a pointer to the same indicator instance. The indicator is updated once per candle regardless of how many strategies depend on it.

#### Engine

```go
type Engine struct {
    indicators      map[string]Indicator
    strategies      []Strategy
    deps            map[string][]string
    candlesReceived int
    maxLookback     int
    candles         chan Candle
    publisher       Publisher
}
```

A dedicated goroutine runs alongside the main engine loop, responsible only for receiving candle messages from NATS and deserialising them. It pushes completed `Candle` values into a buffered channel owned by the engine. When the buffer is full the goroutine blocks, which applies backpressure to the NATS subscription. NATS will redeliver or slow delivery accordingly. If the buffer remains full for an extended period it indicates the engine is genuinely too slow, which should be surfaced as a log warning on each blocked send.


`OnCandle` runs in the main loop:
```
func OnCandle(candle Candle):

    // phase 1: update all indicators
    for each indicator in engine.indicators:
        indicator.Update(candle)

    engine.candlesReceived++

    // warm-up gate: do not evaluate until all indicators have enough history
    if engine.candlesReceived < engine.maxLookback:
        return

    // phase 2: evaluate strategies
    for each strategy in engine.strategies:
        all deps ready = true
        for each indicator ID in engine.deps[strategy.ID()]:
            if not engine.indicators[id].Ready():
                all deps ready = false
                break
        if not all deps ready:
            continue

        signal = strategy.Evaluate(candle)
        if signal == nil:
            continue
        if strategy.IsDuplicate(signal):
            continue

        engine.publisher.Publish(signal)
```

The warm-up gate and the per-strategy `Ready()` check are redundant in steady state but both are kept: the gate prevents early evaluation during the initial fill, the `Ready()` check guards against edge cases where a specific indicator is still behind.

Before any candle is processed, the engine computes `MAX_LOOKBACK` as the maximum period across all registered indicator specs for a strategy. This value represents the minimum number of candles the system must receive before any strategy can be executed. Until `candlesReceived < MAX_LOOKBACK`, the engine updates indicators on each candle but skips strategy evaluation entirely. This prevents strategies from evaluating against partially-filled indicator history, which would produce meaningless signals.

For example: if strategies collectively declare indicators with periods 14, 20, and 50, MAX_LOOKBACK is 50. The first 50 candles are consumed silently. Evaluation begins on candle 51. No rehydration from historical data is performed on startup, if the service restarts mid-session, it waits for a fresh MAX_LOOKBACK candles before signalling again.


If `publisher.Publish` fails, the engine logs the full signal struct and the error at error level and continues to the next strategy. The signal is not retried. For v1 this is acceptable — this is an alert system and a missed signal is preferable to a blocked engine. The log entry provides a manual recovery path if needed.

#### Directory structure
```
algo/
    cmd/
        main.go
    internal/
        engine/
            engine.go
        indicators/
            interface.go
            ema.go
            sma.go
            rsi.go
        strategies/
            interface.go
            registry.go
            ema_crossover.go
        dedup/
            dedup.go
        publisher/
            publisher.go
        types/
            candle.go
            signal.go
    config/
        config.go
````

### NATS Pubsub 
NATS core (no JetStream, no persistence) is used for all inter-service messaging. Messages are delivered via push subscriptions - consumers register a subject subscription and NATS delivers messages as they arrive. Queue groups are not used. Each downstream service (strategy service, TSDB writer) holds its own independent subscription to the same subject and receives every message. 

The topics are:
1. price: transports the prices from ingestion service, consumed by strategy service and the tsdb. Supports sub topics as price.symbol where symbol is ticker of the thing we're trading on (eg. EURUSD). Sample message schema for a message in the price.eurusd topic is given:
```json
{
    "symbol": "EURUSD",
    "source":"binance",
    "timestamp": 1718000000,
    "open": 1.0821,
    "high": 1.0834,
    "low": 1.0819,
    "close": 1.0829,
    "volume": 1024.0
}
```
Example subjects: `price.EURUSD`, `signal.ema_20_ema_50_crossover`

v3 will extend this with `indicators.{symbol}.{indicator_id}` subjects published by a dedicated indicator service.

2. signal: transports message published by the strategy service to tsdb and notification services. also has the subtopic based on strategy id. sample message schema given below:
	```json
	{
		"symbol":"EURUSD",
 		"strategy_id":"ema_20_ema_50_crossover",
		"type":1, //1-buy, 2-sell
		"timestamp":"str",
		"context":json // to be converted to jsonb in the tsdb
	}
	```
### TSDB
Language: Python
A lightweight service that subscribes to `price.*` and `signal.*` NATS subjects and writes received messages to TimescaleDB. This is the only component that writes to the database.

Schema:
```sql
CREATE TABLE prices (
    ts          TIMESTAMPTZ NOT NULL,
    symbol      TEXT NOT NULL,
    open        DOUBLE PRECISION,
    high        DOUBLE PRECISION,
    low         DOUBLE PRECISION,
    close       DOUBLE PRECISION,
    volume      DOUBLE PRECISION
);
SELECT create_hypertable('prices', 'ts');
CREATE INDEX ON prices (symbol, ts DESC);

CREATE TABLE signals (
    ts          TIMESTAMPTZ NOT NULL,
    symbol      TEXT NOT NULL,
    type        SMALLINT NOT NULL,
    strategy_id TEXT NOT NULL,
    context     JSONB
);
SELECT create_hypertable('signals', 'ts');
CREATE INDEX ON signals (symbol, strategy_id, ts DESC);
CREATE INDEX ON signals (type, ts DESC);
```

The `context` column stores the full `SignalContext` as JSONB. Indicator values, the trigger message, and candle close are all queryable if needed.

If a TimescaleDB write fails, the writer logs the failure at error level along with the full raw message payload as a JSON string. The message is not requeued into NATS. The raw payload in the log provides a manual recovery path. Connection failures to TimescaleDB trigger a reconnect loop with exponential backoff. The NATS subscription is paused during reconnection and resumed once the database connection is restored, preventing message loss during a DB outage. During a prolonged outage, NATS core has no persistence, so messages received while the subscription is paused will be lost. This is acceptable for v1.


### Dashboard
Grafana instance pointed at TimescaleDB as a data source.

A candlestick plot for the `price.symbol` subtopic, overlayed with the `signal.strategy_id` through an annotation query. Would prefer to have one panel per strategy, alternatively can have one strategy per symbol with filtering for strategy_ids.


Signals render as vertical lines overlaid on the price chart. No authentication. Public read-only access.
### Notification service
Least priority component. Subscribes to `signal.*`. On receipt, logs the signal as a structured log line to stdout. Abstracted behind an HTTP POST interface so the backing implementation can be swapped to email, Telegram, or webhook later without changing the contract.
Language: Python

### Docker compose
All services are run via a single `docker-compose.yml` at the repository root. Each component is a separate service in the compose file. The services are:

- `nats`: official NATS image, no persistence config needed, exposes port 4222 internally.
- `timescaledb`: official TimescaleDB image on top of Postgres. Mounts a named volume for data persistence. Exposes port 5432 internally. An init SQL script runs the schema creation on first start.
- `grafana`: official Grafana image. Mounts a provisioning directory containing the datasource config (pointing at timescaledb) and the dashboard JSON so the dashboard is available immediately on first start without manual setup.
- `ingestor`: Python service. Built from `./ingestor/Dockerfile`. Depends on `nats`.
- `algo`: Go service. Built from `./algo/Dockerfile`. Depends on `nats`. Single binary in a minimal image.
- `tsdb-writer`: Python service. Built from `./tsdb-writer/Dockerfile`. Depends on `nats` and `timescaledb`.
- `notifier`: Python service. Built from `./notifier/Dockerfile`. Depends on `nats`.

Services that depend on NATS should implement a simple retry loop on startup rather than relying solely on compose `depends_on`, since `depends_on` only waits for the container to start, not for NATS to be ready to accept connections.

The repository structure mirrors this:
```
/
    docker-compose.yml
    ingestor/
        Dockerfile
    algo/
        Dockerfile
    tsdb-writer/
        Dockerfile
    notifier/
        Dockerfile
    grafana/
        provisioning/
            datasources/
            dashboards/
    init.sql
```

---

## Future changes (not for v1-mvp)
v2 makes three structural changes from v1.

1. Strategy parameters move from hardcoded constants to values read from environment variables at startup. Each strategy reads its own set of env vars — period lengths, thresholds, dedup window size. The strategy struct gains a config field populated at instantiation. No code changes are needed to run the same strategy with different parameters — only the environment differs.
2. Each strategy moves to its own container. The algo service in docker-compose is replaced by one service per strategy. Each container runs the same binary but with different environment variables selecting which strategy to activate. A strategy that is not selected by its env var does not register or execute. Stopping a strategy is done by stopping its container with no impact on others.
3. The indicator computation remains inside each strategy container for v2. Deduplication of indicator work across containers is not attempted — if two strategy containers both need EMA20 on EURUSD, they each compute it independently. This is acceptable at v2 scale.

---

v3 changes:

v3 extracts indicator computation into a dedicated indicator service. This service subscribes to `price.*`, computes all known indicators for each symbol, and publishes results to `indicators.{symbol}.{indicator_id}` subjects. User strategy containers subscribe only to the indicator subjects they need - NATS subject filtering handles this at the broker without an intermediate routing service.

User-submitted strategies are deployed as containers with network access restricted to NATS subscribe-only on the `indicators.*` subject hierarchy and publish-only on `signal.*`. They cannot subscribe to raw price data and cannot publish to price or indicator subjects. Resource limits (CPU, memory) are enforced at the container level.

The Declare and Init flow changes in v3: strategies no longer instantiate or own indicator references. Instead Init receives a NATS subscription handle per declared indicator subject, and Evaluate reads the latest value from that subscription's buffer. The indicator service becomes the single source of truth for indicator values.
