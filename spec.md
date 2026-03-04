trade-bot (better name pending) is a event driven real time system that ingests prices, executes algorithmic trading strategies on it and sends generated signals to subscribed consumers.

On a high level, this project consists of a ingestion service that subscribes to data sources, a pubsub system that sends events to and from services, a strategy service that calculates indicator values and executes algorithmic trading strategies on them to generate trading signals, a time series database that stores price and signal data, a dashboard based on the tsdb to visualize events and prices.

## History
This is a revamp of an old project I did, for which the complete code is not available. It was ambitious, yet poorly designed and built, had flaky logic and was incredibly inefficient, held together by hopes and dreams.

Nevertheless it did solve a very important problem that I faced - the algo trading platform I used to use had a proprietary language, and errors were not easy to debug, I had minimal control over the whol environment.

I'm building this from scratch again because this is an incredible learning opportunity, I will be able to apply a lot of the things I got to learn since I built this project two years ago, see the mistakes I did while building it then and improve on those and of course learn new things.

## Versioning
There are three major versions planned for this project.

v1 is the mvp with hardcoded strategies, where to stop a strategy the container for the service needs to be stopped. All strategies run on a single container, and the system is entirely read only.

v2 enables privileged users to define parameters of a particular strategy and execute it while others are still running, features environment variables to stop a particular strategy executing, different containers for each strategy.

v3 adds write permissions for privileged users - who will be able to write their own code for the strategies they want to run and execute.

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

`IsDuplicate` checks the strategy's internal dedup window. The strategy owns this state. Returns true if a signal of the same type has been seen within the last K signals, where K is defined per strategy.

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

A separate goroutine listens on the NATS subject and pushes deserialized candles into `engine.candles`, a buffered channel. The engine's main loop reads from this channel. This keeps NATS I/O decoupled from strategy execution.

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
Example subjects: `price.EURUSD`, `signal.EURUSD`

v3 will extend this with `indicators.{symbol}.{indicator_id}` subjects published by a dedicated indicator service.

2. signal: transports message published by the strategy service to tsdb and notification services. also has the subtopic based on symbol name. sample message schema given below:
	```json
	{
		"symbol":"EURUSD",
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
### Dashboard
Grafana instance pointed at TimescaleDB as a data source.

Two panels on a single dashboard:

- Price panel: line graph of close price over time, filtered by symbol. Source query against the `prices` table.
- Signal overlay: Grafana annotations sourced from the `signals` table.

```sql
SELECT ts AS time,
       strategy_id || ' ' || CASE type WHEN 1 THEN 'BUY' WHEN 2 THEN 'SELL' END AS text,
       CASE type WHEN 1 THEN 'buy' WHEN 2 THEN 'sell' END AS tags
FROM signals
WHERE $__timeFilter(ts) AND symbol = 'EURUSD'
```

Signals render as vertical lines overlaid on the price chart. No authentication. Public read-only access.
### Notification service
Least priority component. Subscribes to `signal.*`. On receipt, logs the signal as a structured log line to stdout. Abstracted behind an HTTP POST interface so the backing implementation can be swapped to email, Telegram, or webhook later without changing the contract.
Language: Python
