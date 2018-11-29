# Intrinio Ruby SDK for Real-Time Stock Prices

[Intrinio](https://intrinio.com/) provides real-time stock prices via a two-way WebSocket connection. To get started, [subscribe to a real-time data feed](https://intrinio.com/marketplace/data/prices/realtime) and follow the instructions below.

## Requirements

- Ruby 2.3.1

## Features

* Receive streaming, real-time price quotes (last trade, bid, ask)
* Subscribe to updates from individual securities
* Subscribe to updates for all securities (contact us for special access)

## Installation
```
gem install intrinio-realtime
```

## Example Usage
```ruby
require 'intrinio-realtime'
require 'thread/pool'

# Provide your Intrinio API access keys (found in https://intrinio.com/account)
api_key = "YOUR_INTRINIO_API_KEY"

# Setup a logger
logger = Logger.new($stdout)
logger.level = Logger::INFO

# Specify options
options = {
  api_key: api_key,
  provider: Intrinio::Realtime::IEX,
  channels: ["MSFT","AAPL","GE"],
  logger: logger
}

# Setup a pool of 50 threads to handle quotes
pool = Thread.pool(50)

# Start listening for quotes
Intrinio::Realtime.connect(options) do |quote|
  # Process quote in next available thread
  pool.process do
    logger.info "QUOTE! #{quote}"
    sleep 0.100 # simulate 100ms for I/O operation
  end
end

```

## Handling Quotes

When handling quotes, make sure to do so in a non-blocking fashion. We recommend using the [thread](https://github.com/meh/ruby-thread) gem to setup a thread pool for operations like writing quotes to database (or anything else involving time-consuming I/O). If your handling code blocks, no new quotes will be received until the code is finished. So in order to make sure you receive all incoming quotes, handle the individual quotes with a thread pool. For listening to individual security channels, we recommend a thread pool size of 50. For the $lobby channel, we recommend 500 or higher. If you find that the quotes are not being processed quickly enough, increase the pool size - it is likely that they are being backed up in the thread pool's backlog (which you can check by accessing `pool.backlog`).

## Providers

Currently, Intrinio offers realtime data from the following providers:

* IEX - [Homepage](https://iextrading.com/)
* QUODD - [Homepage](http://home.quodd.com/)
* Cryptoquote - [Homepage](https://www.cryptoquote.io/)

Each has distinct price channels and quote formats, but a very similar API.

## Quote Data Format

Each data provider has a different format for their quote data.

### QUODD

NOTE: Messages from QUOOD reflect _changes_ in market data. Not all fields will be present in every message. Upon subscribing to a channel, you will receive one quote and one trade message containing all fields of the latest data available.

#### Trade Message

```ruby
{ ticker: "AAPL.NB",
  root_ticker: "AAPL",
  protocol_id: 301,
  last_price_4d: 1594850,
  trade_volume: 100,
  trade_exchange: "t",
  change_price_4d: 24950,
  percent_change_4d: 15892,
  trade_time: 1508165070052,
  up_down: "v",
  vwap_4d: 1588482,
  total_volume: 10209883,
  day_high_4d: 1596600,
  day_high_time: 1508164532269,
  day_low_4d: 1576500,
  day_low_time: 1508160605345,
  prev_close_4d: 1569900,
  volume_plus: 6333150,
  ext_last_price_4d: 1579000,
  ext_trade_volume: 100,
  ext_trade_exchange: "t",
  ext_change_price_4d: 9100,
  ext_percent_change_4d: 5796,
  ext_trade_time: 1508160600567,
  ext_up_down: "-",
  open_price_4d: 1582200,
  open_volume: 100,
  open_time: 1508141103583,
  rtl: 30660,
  is_halted: false,
  is_short_restricted: false }
```

* **ticker** - Stock Symbol for the security
* **root_ticker** - Underlying symbol for a particular contract
* **last_price_4d** - The price at which the security most recently traded
* **trade_volume** - The number of shares that that were traded on the last trade
* **trade_exchange** - The market center where the last trade occurred
* **trade_time** - The time at which the security last traded in milliseconds
* **up_down** - Tick indicator - up or down - indicating if the last trade was up or down from the previous trade
* **change_price_4d** - The difference between the closing price of a security on the current trading day and the previous day's closing price.
* **percent_change_4d** - The percentage at which the security is up or down since the previous day's trading
* **total_volume** - The accumulated total amount of shares traded
* **volume_plus** - NASDAQ volume plus the volumes from other market centers to more accurately match composite volume. Used for NASDAQ Basic
* **vwap_4d** - Volume weighted Average Price. VWAP is calculated by adding up the dollars traded for every transaction (price multiplied by number of shares traded) and then dividing by the total shares traded for the day.
* **day_high_4d** - A security's intra-day high trading price.
* **day_high_time** - Time that the security reached a new high
* **day_low_4d** - A security's intra-day low trading price.
* **day_low_time** - Time that the security reached a new low
* **ext_last_price_4d** - Extended hours last price (pre or post market)
* **ext_trade_volume** - The amount of shares traded for a single extended hours trade
* **ext_trade_exchange** - Extended hours exchange where last trade took place (Pre or post market)
* **ext_trade_time** - Time of the extended hours trade in milliseconds
* **ext_up_down** - Extended hours tick indicator - up or down
* **ext_change_price_4d** - Extended hours change price (pre or post market)
* **ext_percent_change_4d** - Extended hours percent change (pre or post market)
* **is_halted** - A flag indicating that the stock is halted and not currently trading
* **is_short_restricted** - A flag indicating the stock is current short sale restricted - meaning you can not short sale the stock when true
* **open_price_4d** - The price at which a security first trades upon the opening of an exchange on a given trading day
* **open_time** - The time at which the security opened in milliseconds
* **open_volume** - The number of shares that that were traded on the opening trade
* **prev_close_4d** - The security's closing price on the preceding day of trading
* **protocol_id** - Internal Quodd ID defining Source of Data
* **rtl** - Record Transaction Level - number of records published that day

#### Quote Message

```ruby
{ ticker: "AAPL.NB",
  root_ticker: "AAPL",
  bid_size: 500,
  ask_size: 600,
  bid_price_4d: 1594800,
  ask_price_4d: 1594900,
  ask_exchange: "t",
  bid_exchange: "t",
  quote_time: 1508165070850,
  protocol_id: 302,
  rtl: 129739 }
```

* **ticker** - Stock Symbol for the security
* **root_ticker** - Underlying symbol for a particular contract
* **ask_price_4d** - The price a seller is willing to accept for a security
* **ask_size** - The amount of a security that a market maker is offering to sell at the ask price
* **ask_exchange** - The market center from which the ask is being quoted
* **bid_price_4d** - A bid price is the price a buyer is willing to pay for a security.
* **bid_size** - The bid size number of shares being offered for purchase at a specified bid price
* **bid_exchange** - The market center from which the bid is being quoted
* **quote_time** - Time of the quote in milliseconds
* **rtl** - Record Transaction Level - number of records published that day
* **protocol_id** - Internal Quodd ID defining Source of Data

### IEX

```ruby
{ type: 'ask',
  timestamp: 1493409509.3932788,
  ticker: 'GE',
  size: 13750,
  price: 28.97 }
```

*   **type** - the quote type
  *    **`last`** - represents the last traded price
  *    **`bid`** - represents the top-of-book bid price
  *    **`ask`** - represents the top-of-book ask price
*   **timestamp** - a Unix timestamp (with microsecond precision)
*   **ticker** - the ticker of the security
*   **size** - the size of the `last` trade, or total volume of orders at the top-of-book `bid` or `ask` price
*   **price** - the price in USD

### Cryptoquote

#### Book Update
```ruby
{ pair_name: "BTCUSD",
  pair_code: "btcusd",
  exchange_name: "Gemini",
  exchange_code: "gemini",
  side: "buy",
  price: 6337.4,
  size: 0.3,
  type: "book_update" }
```

*   **pair_name** - the name of the currency pair
*   **pair_code** - the code of the currency pair
*   **exchange_name** - the name of the exchange
*   **exchange_code** - the code of the exchange
*   **side** - the side of the book this update is for
  *    **`buy`** - this is an update to the buy side of the book
  *    **`sell`** - this is an update to the sell side of the book
*   **price** - the price of this book entry
*   **size** - the size of this book entry
*   **type** - the type of message this is
  *    **`book_update`** - a message that denotes a change to an order book
  *    **`ticker`** - a snapshot of the market as depicted by the Exchange
  *    **`trade`** - a trade message (updating `last_trade_price`, `last_trade_time`, and `last_trade_size`)

#### Ticker
```ruby
{ last_updated: "2018-10-29 23:08:02.277Z",
  pair_name: "BTCUSD",
  pair_code: "btcusd",
  exchange_name: "Binance",
  exchange_code: "binance",
  bid: 6326,
  bid_size: 6.51933000,
  ask: 6326.97,
  ask_size: 6.12643000,
  change: -151.6899999999996,
  change_percent: -2.340895061728389,
  volume: 13777.232772,
  open: 6480,
  high: 6505.01,
  low: 6315,
  last_trade_time: "2018-10-29 23:08:01.834Z",
  last_trade_side: nil,
  last_trade_price: 6326.97000000,
  last_trade_size: 0.00001200,
  type: "ticker" }
```

*   **last_updated** - a UTC timestamp of when the ticker was last updated
*   **pair_name** - the name of the currency pair
*   **pair_code** - the code of the currency pair
*   **exchange_name** - the name of the exchange
*   **exchange_code** - the code of the exchange
*   **ask** - the ask for the currency pair on the exchange
*   **ask_size** - the size of the ask for the currency pair on the exchange
*   **bid** - the bid for the currency pair on the exchange
*   **bid_size** - the size of the bid for the currency pair on the exchange
*   **change** - the notional change in price since the last ticker
*   **change_percent** - the percent change in price since the last ticker
*   **volume** - the volume of the currency pair on the exchange
*   **open** - the opening price of the currency pair on the exchange
*   **high** - the highest price of the currency pair on the exchange
*   **low** - the lowest price of the currency pair on the exchange
*   **last_trade_time** - a UTC timestamp of the last trade for the currency pair on the exchange
*   **last_trade_side** - the side of the last trade
  *    **`buy`** - this is an update to the buy side of the book
  *    **`sell`** - this is an update to the sell side of the book
*   **last_trade_price** - the price of the last trade for the currency pair on the exchange
*   **last_trade_size** - the size of the last trade for the currency pair on the exchange
*   **type** - the type of message this is
  *    **`book_update`** - a message that denotes a change to an order book
  *    **`ticker`** - a snapshot of the market as depicted by the Exchange
  *    **`trade`** - a trade message (updating `last_trade_price`, `last_trade_time`, and `last_trade_size`)

#### Trade
```ruby
{ last_updated: "2018-10-29 23:08:02.277Z",
  pair_name: "BTCUSD",
  pair_code: "btcusd",
  exchange_name: "Gemini",
  exchange_code: "gemini",
  bid: nil,
  bid_size: nil,
  ask: nil,
  ask_size: nil,
  change: -133.40760000000046,
  change_percent: -2.059762280845255,
  volume: 22121.79710206001,
  open: 6476.8445,
  high: 6506.2724,
  low: 6311,
  last_trade_time: "2018-10-29 23:08:01.834Z",
  last_trade_side: "sell",
  last_trade_price: 6343.7124,
  last_trade_size: 1.6045,
  type: "trade" }
```

*   **last_updated** - a UTC timestamp of when the ticker was last updated
*   **pair_name** - the name of the currency pair
*   **pair_code** - the code of the currency pair
*   **exchange_name** - the name of the exchange
*   **exchange_code** - the code of the exchange
*   **ask** - the ask for the currency pair on the exchange
*   **ask_size** - the size of the ask for the currency pair on the exchange
*   **bid** - the bid for the currency pair on the exchange
*   **bid_size** - the size of the bid for the currency pair on the exchange
*   **change** - the notional change in price since the last ticker
*   **change_percent** - the percent change in price since the last ticker
*   **volume** - the volume of the currency pair on the exchange
*   **open** - the opening price of the currency pair on the exchange
*   **high** - the highest price of the currency pair on the exchange
*   **low** - the lowest price of the currency pair on the exchange
*   **last_trade_time** - a UTC timestamp of the last trade for the currency pair on the exchange
*   **last_trade_side** - the side of the last trade
  *    **`buy`** - this is an update to the buy side of the book
  *    **`sell`** - this is an update to the sell side of the book
*   **last_trade_price** - the price of the last trade for the currency pair on the exchange
*   **last_trade_size** - the size of the last trade for the currency pair on the exchange
*   **type** - the type of message this is
  *    **`book_update`** - a message that denotes a change to an order book
  *    **`ticker`** - a snapshot of the market as depicted by the Exchange
  *    **`trade`** - a trade message (updating `last_trade_price`, `last_trade_time`, and `last_trade_size`)

## Channels

### QUODD

To receive price quotes from QUODD, you need to instruct the client to "join" a channel. A channel can be
* A security ticker with data feed designation (`AAPL.NB`, `MSFT.NB`, `GE.NB`, etc)

### IEX

To receive price quotes from IEX, you need to instruct the client to "join" a channel. A channel can be
* A security ticker (`AAPL`, `MSFT`, `GE`, etc)
* The security lobby (`$lobby`) where all price quotes for all securities are posted
* The security last price lobby (`$lobby_last_price`) where only last price quotes for all securities are posted

Special access is required for both lobby channels. [Contact us](mailto:sales@intrinio.com) for more information.

### Cryptoquote

To receive price quotes from Cryptoquote, you need to instruct the client to "join" a channel. A channel can be
* The lobby (`crypto:lobby`) where all message types for all currency pairs are posted
* The type lobby (`crypto:lobby:{message_type}`) where all messages for the given type for all currency pairs are posted (i.e. `crypto:lobby:trade`)
* The pair lobby (`crypto:pair:{pair_code}`) where all message types for the provided currency pair are posted (i.e. `crypto:pair:btcusd`)
* The book_update pair lobby (`crypto:pair:book_update:{pair_code}`) where book_updates for the provided currency pair are posted (i.e. `crypto:pair:book_update:btcusd`)
* The ticker pair lobby (`crypto:pair:ticker:{pair_code}`) where tickers for the provided currency pair are posted (i.e. `crypto:pair:ticker:btcusd`)
* The trade pair lobby (`crypto:pair:trade:{pair_code}`) where trades for the provided currency pair are posted (i.e. `crypto:pair:trade:btcusd`)

## API Keys
You will receive your Intrinio API Key after [creating an account](https://intrinio.com/signup). You will need a subscription to a [realtime data feed](https://intrinio.com/marketplace/data/prices/realtime) as well.

## Documentation

### Methods

`Intrinio::Realtime.connect(options, &block)` - Connects to the Intrinio Realtime feed and provides quotes to the given block.
* **Parameter** `options.api_key`: Your Intrinio API Key
* **Parameter** `options.provider`: The real-time data provider to use (`Intrinio::Realtime::IEX`, `Intrinio:Realtime::CRYPTOQUOTE`, or `Intrinio::Realtime::QUODD`)
* **Parameter** `options.channels`: (optional) An array of channels to join after connecting
* **Parameter** `options.logger`: (optional) A Ruby logger to use for logging
* **Parameter** `&block`: A block that receives a quote as its only parameter. *NOTE*: This block _must_ execute in a non-blocking fashion (by using a thread pool, for example). Otherwise quotes will be dropped until it is done blocking.
```ruby
Intrinio::Realtime.connect(options) do |quote|
  # handle quote in a non-blocking fashion
end
```

---------

`Intrinio::Realtime::Client.new(options)` - Creates a new instance of the Intrinio Realtime Client.
* **Parameter** `options.api_key`: Your Intrinio API Key
* **Parameter** `options.provider`: The real-time data provider to use (`Intrinio::Realtime::IEX`, `Intrinio:Realtime::CRYPTOQUOTE`, or `Intrinio::Realtime::QUODD`)
* **Parameter** `options.channels`: (optional) An array of channels to join after connecting
* **Parameter** `options.logger`: (optional) A Ruby logger to use for logging
```ruby
options = {
  api_key: api_key,
  provider: Intrinio::Realtime::IEX,
  channels: ["MSFT","AAPL","GE"]
}
client = Intrinio::Realtime::Client.new(options)
client.on_quote do |quote|
  # handle quote in a non-blocking fashion
end
client.connect()
```

---------

`client.connect()` - Retrieves an auth token, opens the WebSocket connection, starts the self-healing and heartbeat intervals, joins requested channels.

---------

`client.disconnect()` - Closes the WebSocket, stops the self-healing and heartbeat intervals. You _must_ call this to dispose of the client.

---------

`client.on_quote(&block)` - Invokes the given block when a quote has been received.
* **Parameter** `block` - The block to invoke. The quote will be passed as an argument to the block.
```ruby
client.on_quote do |quote|
  # handle quote in a non-blocking manner
end
```

---------

`client.join(...channels)` - Joins the given channels. This can be called at any time. The client will automatically register joined channels and establish the proper subscriptions with the WebSocket connection.
* **Parameter** `channels` - An argument list or array of channels to join. 
```ruby
client.join("AAPL", "MSFT", "GE")
client.join(["GOOG", "VG"])
client.join("$lobby")
```

---------

`client.leave(...channels)` - Leaves the given channels.
* **Parameter** `channels` - An argument list or array of channels to leave.
```ruby
client.leave("AAPL", "MSFT", "GE")
client.leave(["GOOG", "VG"])
client.leave("$lobby")
```

---------

`client.leave_all()` - Leaves all channels.
