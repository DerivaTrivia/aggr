
# SignificantTrades [![Build Status](https://travis-ci.org/Tucsky/aggr.svg?branch=master)](https://travis-ci.org/Tucsky/aggr)

  

Cryptocurrency market trades aggregator.<br>

Currently supporting BitMEX, Bitfinex, Binance & Binance Futures, Coinbase, Bitstamp, Deribit, Huobi, Okex, Hitbtc, Poloniex, Bybit and FTX ([see src/exchanges/](src/exchanges) for detail)

  

![screenshot](https://i.imgur.com/nHJxsdL.gif)

  

## What it do
This tool shows **markets orders** LIVE on the crypto market(s) of your choice.

- Show live trades from exchanges on a specific pair
- Filter noise by aggregating trades with the same timestamp
- Calculate rolling sums over defined periods
- Chart whatever is received from api (so only trades data for now)
- Dynamic audio based on trade volume and side 

Checkout [CHANGELOG.md](CHANGELOG.md) for details about the recent updates.

## How it works

![](https://i.imgur.com/chxtEwb.png)

The app is written in Vue.js, use the javascript WebSocket interface to connect to the exchanges API and listen to the trades events.

Within a worker, the raw trades are dispatched from each Exchange to the Aggregator, whose purpose is to group the trades by time, market and side of trade. The worker regularly send the result of the aggregation to the UI along with some statistics about the market activity (sums of volume, counts by sides and liquidations).

## How to install & run locally

1. Clone the repo

```bash
git clone https://github.com/Tucsky/aggr
```

2. Install dependencies

```bash
npm install
```

3. Run it

  

Development mode
```bash
npm run serve
```

This will automatically open a browser window at localhost:8080

Otherwise can build the application (production)

```bash
npm run build
```

and access the dist/index.html directly in the browser later without having to run a command
  

## Configuration
SignificantTrades is now using Vue Cli which allows you to configure the client using .env file.

Create a <code>.env.local</code> or <code>.env.development</code> or <code>.env.production</code> file inside <code>/</code> folder.

  
|key| description |default value|
|--|--|--|
|<code>PROXY_URL</code>|Redirect HTTP requests from app through a proxy<br>If the <code>PROXY_URL</code> is set to https://cors.aggr.trade/, the app will retrieve Binance's products through this url : https://cors.aggr.trade/https://api.binance.com/api/v3/exchangeInfo |http://localhost:8080/|
|<code>API_URL</code>|Server instance url.<br>As of now only used to fetch historical data for the chart component.<br>Example: http://localhost:3000/historical/{from}/{to}/{timeframe}/{markets} |null|
|<code>API_SUPPORTED_PAIRS</code>|Markets supported by the server instance provided in <code>API_URL</code><br>Write the full market names (EXCHANGE:pair) like so : COINBASE:BTC-USD,BINANCE:btcusdt|null|


## Implement historical data
You can use this project without historical data just by opening the app in your browser, as getting trades from exchanges is made directly in the browser using websocket api.

However, in order to show historical data you will need to setup your own server that will collect and distribute data on demand.

The current code for the server part now located in a separated repository [aggr-server](https://github.com/Tucsky/aggr-server).

Let's say you have a server instance running on port 3000, start the client with an environment variable `API_URL=http://localhost:3000/{from}/{to}/{timeframe}/{markets} npm run serve`

## Support this project !
BTC [3PK1bBK8sG3zAjPBPD7g3PL14Ndux3zWEz](bitcoin:3PK1bBK8sG3zAjPBPD7g3PL14Ndux3zWEz)<br>
XMR 48NJj3RJDo33zMLaudQDdM8G6MfPrQbpeZU2YnRN2Ep6hbKyYRrS2ZSdiAKpkUXBcjD2pKiPqXtQmSZjZM7fC6YT6CMmoX6<br>
COINBASE
https://commerce.coinbase.com/checkout/c58bd003-5e47-4cfb-ae25-5292f0a0e1e8