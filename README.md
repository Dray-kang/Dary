# Dary

import ccxt
import pandas as pd
import time
from ta.volatility import BollingerBands, AverageTrueRange
from ta.trend import EMAIndicator, ADXIndicator, MACD
from ta.momentum import RSIIndicator

# Binance Testnet 연결 (API 키 설정)
API_KEY = "your_testnet_api_key"
API_SECRET = "your_testnet_secret_key"

exchange = ccxt.binance({
    "apiKey": API_KEY,
    "secret": API_SECRET,
    "options": {"defaultType": "future"},
    "urls": {"api": "https://testnet.binancefuture.com"}
})

# 거래 설정
SYMBOLS = ["BTC/USDT", "ETH/USDT", "BNB/USDT", "SOL/USDT", "XRP/USDT", "DOGE/USDT"]  # 거래량이 가장 많은 6개 심볼
LEVERAGE = 15  # 레버리지 고정
RISK_PERCENTAGE = 2  # 계좌 리스크 비율
TIMEFRAME = "1m"  # 기본 시간 프레임
POSITION_SIZE = 0.001  # 기본 포지션 크기 (예: BTC)

# 바이낸스 레버리지 설정
def set_leverage(symbol, leverage):
    try:
        exchange.fapiPrivate_post_leverage({'symbol': symbol.replace("/", ""), 'leverage': leverage})
        print(f"Leverage set to {leverage}x for {symbol}")
    except Exception as e:
        print(f"Error setting leverage for {symbol}: {e}")

# Helper Function: OHLCV 데이터 가져오기
def fetch_data(symbol, timeframe, limit=500):
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    df = pd.DataFrame(ohlcv, columns=["timestamp", "open", "high", "low", "close", "volume"])
    df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms")
    return df

# Helper Function: 주문 실행
def place_order(symbol, side, quantity, order_type="MARKET"):
    try:
        order = exchange.create_order(symbol, order_type, side, quantity)
        print(f"Order placed: {order}")
        return order
    except Exception as e:
        print(f"Error placing order: {e}")

# Strategy 1: Squeeze Zone Strategy
class SqueezeZoneStrategy:
    def __init__(self, data):
        self.data = data
        self.bb = BollingerBands(close=data["close"], window=20, window_dev=2)
        self.ema20 = EMAIndicator(close=data["close"], window=20)
        self.ema100 = EMAIndicator(close=data["close"], window=100)
        self.rsi = RSIIndicator(close=data["close"], window=14)

    def apply_indicators(self):
        self.data["bbw"] = self.bb.bollinger_wband()
        self.data["upper_band"] = self.bb.bollinger_hband()
        self.data["lower_band"] = self.bb.bollinger_lband()
        self.data["ema20"] = self.ema20.ema_indicator()
        self.data["ema100"] = self.ema100.ema_indicator()
        self.data["rsi"] = self.rsi.rsi()

    def generate_signals(self):
        self.data["squeeze"] = self.data["bbw"] < self.data["bbw"].rolling(100).min()
        self.data["breakout"] = (self.data["close"] > self.data["upper_band"] * 1.005) | \
                                (self.data["close"] < self.data["lower_band"] * 0.995)
        self.data["trend"] = self.data["ema20"] > self.data["ema100"]
        self.data["rsi_filter"] = (self.data["rsi"] > 30) & (self.data["rsi"] < 70)
        self.data["entry_signal"] = self.data["squeeze"] & self.data["breakout"] & \
                                    self.data["trend"] & self.data["rsi_filter"]
        return self.data[self.data["entry_signal"]]

# Strategy 2: Hyper Dynamic Reversal Momentum Strategy
class HyperDynamicReversalMomentum:
    def __init__(self, data):
        self.data = data
        self.atr = AverageTrueRange(high=data["high"], low=data["low"], close=data["close"], window=14)
        self.rsi = RSIIndicator(close=data["close"], window=14)

    def apply_indicators(self):
        self.data["atr"] = self.atr.average_true_range()
        self.data["rsi"] = self.rsi.rsi()
        self.data["delta_volume"] = self.data["volume"] / self.data["volume"].rolling(20).mean()

    def generate_signals(self):
        self.data["reversal_candle"] = (self.data["close"] > self.data["open"]) & \
                                       (self.data["low"].shift(1) > self.data["close"]) & \
                                       (self.data["rsi"] < 30)
        self.data["entry_signal"] = self.data["reversal_candle"] & (self.data["delta_volume"] > 2)
        return self.data[self.data["entry_signal"]]

# 자동 매매봇 실행
def run_bot():
    print("자동 매매봇 실행 중...")
    
    # 각 심볼에 대해 레버리지 설정
    for symbol in SYMBOLS:
        set_leverage(symbol, LEVERAGE)
    
    while True:
        try:
            for symbol in SYMBOLS:
                print(f"Fetching data for {symbol}...")
                data = fetch_data(symbol, TIMEFRAME)

                # Squeeze Zone Strategy 실행
                squeeze_strategy = SqueezeZoneStrategy(data.copy())
                squeeze_strategy.apply_indicators()
                squeeze_signals = squeeze_strategy.generate_signals()
                
                if not squeeze_signals.empty:
                    print(f"Squeeze Zone Entry Signal for {symbol}: {squeeze_signals.iloc[-1]}")
                    place_order(symbol, "buy", POSITION_SIZE)

                # Hyper Dynamic Reversal Momentum Strategy 실행
                hdrm_strategy = HyperDynamicReversalMomentum(data.copy())
                hdrm_strategy.apply_indicators()
                hdrm_signals = hdrm_strategy.generate_signals()
                
                if not hdrm_signals.empty:
                    print(f"Hyper Dynamic Reversal Entry Signal for {symbol}: {hdrm_signals.iloc[-1]}")
                    place_order(symbol, "buy", POSITION_SIZE)

                # 추가 전략 실행 가능...

            # 1분 대기 후 반복
            time.sleep(60)
        except Exception as e:
            print(f"Error in bot execution: {e}")
            time.sleep(60)

# 실행
if __name__ == "__main__":
    run_bot()