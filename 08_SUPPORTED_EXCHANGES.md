# Supported Exchanges

## Overview

Obscura supports **100+ cryptocurrency exchanges** via the CCXT library, enabling copy trading across centralized exchanges (CEX) and decentralized exchanges (DEX).

---

## 🏆 Certified Exchanges (Tier 1)

These exchanges are fully tested, certified by CCXT, and recommended for production use.

### Centralized Exchanges (CEX)

| Exchange | ID | Spot | Futures | Margin | WebSocket | Testnet |
|----------|:--:|:----:|:-------:|:------:|:---------:|:-------:|
| **Binance** | `binance` | ✅ | ✅ | ✅ | ✅ | ✅ |
| Binance USDⓈ-M Futures | `binanceusdm` | - | ✅ | - | ✅ | ✅ |
| Binance COIN-M Futures | `binancecoinm` | - | ✅ | - | ✅ | ✅ |
| **Bybit** | `bybit` | ✅ | ✅ | ✅ | ✅ | ✅ |
| **OKX** | `okx` | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Gate.io** | `gate` | ✅ | ✅ | ✅ | ✅ | ✅ |
| **KuCoin** | `kucoin` | ✅ | - | ✅ | ✅ | ✅ |
| KuCoin Futures | `kucoinfutures` | - | ✅ | - | ✅ | ✅ |
| **Bitget** | `bitget` | ✅ | ✅ | ✅ | ✅ | ✅ |
| **MEXC** | `mexc` | ✅ | ✅ | - | ✅ | - |
| **HTX (Huobi)** | `htx` | ✅ | ✅ | ✅ | ✅ | - |
| **BingX** | `bingx` | ✅ | ✅ | - | ✅ | - |
| **BitMEX** | `bitmex` | - | ✅ | - | ✅ | ✅ |
| **Crypto.com** | `cryptocom` | ✅ | - | - | ✅ | - |
| **BitMart** | `bitmart` | ✅ | ✅ | - | ✅ | - |
| **CoinEx** | `coinex` | ✅ | ✅ | - | ✅ | - |
| **HashKey** | `hashkey` | ✅ | - | - | ✅ | - |
| **WOO X** | `woo` | ✅ | ✅ | - | ✅ | - |

### Decentralized Exchanges (DEX)

| Exchange | ID | Type | Chain(s) | WebSocket |
|----------|:--:|:----:|:--------:|:---------:|
| **Hyperliquid** | `hyperliquid` | Perps DEX | Arbitrum L1 | ✅ |
| **dYdX** | `dydx` | Perps DEX | dYdX Chain (Cosmos) | ✅ |
| **Paradex** | `paradex` | Perps DEX | StarkNet | ✅ |
| **Apex Pro** | `apex` | Perps DEX | StarkEx | ✅ |
| **WOOFi Pro** | `woofipro` | DEX Aggregator | Multi-chain | ✅ |
| **Derive** | `derive` | Options DEX | Lyra Chain | ✅ |

---

## 🔧 Supported Exchanges (Tier 2)

These exchanges have WebSocket (Pro) support and are tested but not CCXT-certified.

| Exchange | ID | Spot | Futures | WebSocket |
|----------|:--:|:----:|:-------:|:---------:|
| Coinbase | `coinbase` | ✅ | - | ✅ |
| Coinbase Exchange | `coinbaseexchange` | ✅ | - | ✅ |
| Coinbase International | `coinbaseinternational` | ✅ | ✅ | ✅ |
| Kraken | `kraken` | ✅ | - | ✅ |
| Kraken Futures | `krakenfutures` | - | ✅ | ✅ |
| Bitstamp | `bitstamp` | ✅ | - | ✅ |
| Gemini | `gemini` | ✅ | - | ✅ |
| Bitfinex | `bitfinex` | ✅ | ✅ | ✅ |
| Phemex | `phemex` | ✅ | ✅ | ✅ |
| Deribit | `deribit` | - | ✅ | ✅ |
| Upbit | `upbit` | ✅ | - | ✅ |
| Poloniex | `poloniex` | ✅ | - | ✅ |
| LBank | `lbank` | ✅ | - | ✅ |
| WhiteBit | `whitebit` | ✅ | ✅ | ✅ |
| XT.com | `xt` | ✅ | ✅ | ✅ |
| Bitrue | `bitrue` | ✅ | ✅ | ✅ |
| Alpaca | `alpaca` | ✅ | - | ✅ |
| Backpack | `backpack` | ✅ | ✅ | ✅ |
| Blofin | `blofin` | ✅ | ✅ | ✅ |
| Toobit | `toobit` | ✅ | ✅ | ✅ |

---

## 🧪 Beta Exchanges (Tier 3)

These exchanges are supported via CCXT but have limited testing. Use with caution.

<details>
<summary>View 50+ Beta Exchanges</summary>

| Exchange | ID | Type |
|----------|:--:|:----:|
| AscendEX | `ascendex` | CEX |
| Bequant | `bequant` | CEX |
| BigONE | `bigone` | CEX |
| Bit2C | `bit2c` | CEX |
| Bitbank | `bitbank` | CEX |
| Bitbns | `bitbns` | CEX |
| Bitflyer | `bitflyer` | CEX |
| Bithumb | `bithumb` | CEX |
| Bitopro | `bitopro` | CEX |
| Bitvavo | `bitvavo` | CEX |
| BL3P | `bl3p` | CEX |
| Blockchain.com | `blockchaincom` | CEX |
| BTC Markets | `btcmarkets` | CEX |
| BTC Trade UA | `btctradeua` | CEX |
| BTC Turk | `btcturk` | CEX |
| Bybit | `bybit` | CEX |
| CEX.IO | `cex` | CEX |
| CoinCatch | `coincatch` | CEX |
| CoinCheck | `coincheck` | CEX |
| CoinFalcon | `coinfalcon` | CEX |
| CoinMate | `coinmate` | CEX |
| CoinOne | `coinone` | CEX |
| Coinspot | `coinspot` | CEX |
| Delta Exchange | `delta` | CEX |
| Exmo | `exmo` | CEX |
| FTX (Deprecated) | `ftx` | - |
| HitBTC | `hitbtc` | CEX |
| Hollaex | `hollaex` | CEX |
| Idex | `idex` | DEX |
| Independent Reserve | `independentreserve` | CEX |
| Indodax | `indodax` | CEX |
| Latoken | `latoken` | CEX |
| Luno | `luno` | CEX |
| Mercado Bitcoin | `mercado` | CEX |
| NDAX | `ndax` | CEX |
| Novadax | `novadax` | CEX |
| OceanEx | `oceanex` | CEX |
| Okcoin | `okcoin` | CEX |
| P2B | `p2b` | CEX |
| Paribu | `paribujo` | CEX |
| Probit | `probit` | CEX |
| Tidex | `tidex` | CEX |
| TimeX | `timex` | CEX |
| Tokocrypto | `tokocrypto` | CEX |
| WazirX | `wazirx` | CEX |
| Yobit | `yobit` | CEX |
| Zaif | `zaif` | CEX |
| ZB | `zb` | CEX |

</details>

---

## 🌐 Exchange Capabilities

### Trading Features by Exchange

| Feature | Binance | Bybit | OKX | Gate | KuCoin | Hyperliquid |
|---------|:-------:|:-----:|:---:|:----:|:------:|:-----------:|
| Spot Trading | ✅ | ✅ | ✅ | ✅ | ✅ | - |
| USDT Perpetuals | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Coin Perpetuals | ✅ | ✅ | ✅ | ✅ | - | - |
| Options | ✅ | - | ✅ | - | - | - |
| Margin Trading | ✅ | ✅ | ✅ | ✅ | ✅ | - |
| Stop Orders | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trailing Stop | ✅ | ✅ | ✅ | ✅ | - | - |
| OCO Orders | ✅ | - | ✅ | - | - | - |
| Max Leverage | 125x | 100x | 125x | 100x | 100x | 50x |

### API Capabilities

| Capability | Description | Exchanges |
|------------|-------------|-----------|
| `fetchTicker` | Get current price | All |
| `fetchOrderBook` | Get order book | All |
| `fetchOHLCV` | Get candlestick data | All |
| `createOrder` | Place orders | All |
| `createMarketOrder` | Market orders | All |
| `createLimitOrder` | Limit orders | All |
| `createStopOrder` | Stop loss/take profit | Most |
| `fetchBalance` | Get account balance | All |
| `fetchMyTrades` | Get trade history | All |
| `fetchPositions` | Get futures positions | Futures exchanges |
| `watchTicker` | WebSocket price stream | Pro exchanges |
| `watchOrderBook` | WebSocket order book | Pro exchanges |
| `watchMyTrades` | WebSocket trade updates | Pro exchanges |

---

## 🔐 Exchange Setup Instructions

### Binance

1. Log in to [Binance](https://www.binance.com)
2. Go to **Account** → **API Management**
3. Click **Create API**
4. Choose **System generated** (not self-generated)
5. Enable permissions:
   - ✅ Enable Reading
   - ✅ Enable Spot & Margin Trading
   - ✅ Enable Futures (if needed)
6. **Important**: Add IP restrictions (recommended)
7. Copy **API Key** and **Secret Key**

### Bybit

1. Log in to [Bybit](https://www.bybit.com)
2. Go to **Account** → **API**
3. Click **Create New Key**
4. Select **System-generated API Keys**
5. Set permissions:
   - ✅ Read-Write for trading
   - ✅ Contract (for futures)
6. Set IP restriction or leave as "No restriction"
7. Complete 2FA verification
8. Copy **API Key** and **Secret**

### OKX

1. Log in to [OKX](https://www.okx.com)
2. Go to **Profile** → **API**
3. Click **Create API Key**
4. Set a passphrase (required by OKX)
5. Set permissions:
   - ✅ Read
   - ✅ Trade
6. Set IP whitelist (recommended)
7. Copy **API Key**, **Secret**, and **Passphrase**

### Hyperliquid (DEX)

1. Go to [Hyperliquid](https://app.hyperliquid.xyz)
2. Connect your wallet (MetaMask, etc.)
3. Go to **API** tab
4. Click **Generate API Wallet**
5. **Important**: This creates a separate trading wallet
6. Fund the API wallet with USDC
7. Copy the **Private Key** for the API wallet

---

## 📊 Exchange Comparison

### For Copy Trading

| Factor | Best Exchanges | Notes |
|--------|----------------|-------|
| **Liquidity** | Binance, Bybit, OKX | Highest volume, lowest slippage |
| **Futures** | Binance USDM, Bybit, OKX | Best perpetual contracts |
| **Low Fees** | Binance, Gate, MEXC | Maker: 0.02%, Taker: 0.04% |
| **US Users** | Coinbase, Kraken, Gemini | Compliant exchanges |
| **DeFi** | Hyperliquid, dYdX | Non-custodial options |
| **Privacy** | Hyperliquid, dYdX | No KYC required |

### Fee Comparison

| Exchange | Spot Maker | Spot Taker | Futures Maker | Futures Taker |
|----------|:----------:|:----------:|:-------------:|:-------------:|
| Binance | 0.10% | 0.10% | 0.02% | 0.04% |
| Bybit | 0.10% | 0.10% | 0.02% | 0.055% |
| OKX | 0.08% | 0.10% | 0.02% | 0.05% |
| Gate.io | 0.10% | 0.10% | 0.015% | 0.05% |
| KuCoin | 0.10% | 0.10% | 0.02% | 0.06% |
| Hyperliquid | 0.01% | 0.035% | 0.01% | 0.035% |

---

## 🧪 Testnet Support

These exchanges offer testnet/sandbox environments for development:

| Exchange | Testnet URL | Notes |
|----------|-------------|-------|
| Binance Spot | testnet.binance.vision | Separate API keys |
| Binance Futures | testnet.binancefuture.com | USDM & COINM |
| Bybit | api-testnet.bybit.com | Full testnet |
| OKX | Sandbox mode | Via API parameter |
| KuCoin | openapi-sandbox.kucoin.com | Sandbox API |
| BitMEX | testnet.bitmex.com | Testnet environment |
| Phemex | testnet-api.phemex.com | Demo trading |
| Deribit | test.deribit.com | Paper trading |

---

## 🚀 Adding Exchange Support

Obscura can add support for any CCXT-compatible exchange. To request a new exchange:

1. Check if it's in the [CCXT exchange list](https://github.com/ccxt/ccxt#supported-cryptocurrency-exchange-markets)
2. Submit a request to **support@obscura.finance**
3. We'll test and certify within 1-2 weeks

### Self-Integration (Enterprise)

Enterprise B2B clients can integrate custom exchanges:

```python
# Custom exchange adapter example
class CustomExchangeAdapter:
    def __init__(self, api_key: str, secret: str):
        self.exchange = ccxt.customexchange({
            'apiKey': api_key,
            'secret': secret,
        })
    
    async def create_order(self, symbol: str, side: str, amount: float, price: float = None):
        order_type = 'limit' if price else 'market'
        return await self.exchange.create_order(symbol, order_type, side, amount, price)
```

---

## 📞 Exchange-Specific Support

| Issue | Contact |
|-------|---------|
| Exchange integration bugs | support@obscura.finance |
| API rate limits | Check exchange documentation |
| Account/KYC issues | Contact exchange directly |
| New exchange requests | partnerships@obscura.finance |

---

*Last Updated: December 2025*
