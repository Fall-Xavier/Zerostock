# ZeroStock Platform — Fitur 8-10 + Master ZeroScore

---

# FITUR 8 — CHARTBIT STYLE ANALYSIS (Technical Analysis Engine)

## 1. Tujuan Fitur
Analisa teknikal otomatis dengan pendekatan serupa ChartBit Stockbit — mendeteksi tren, pola, dan sinyal dari data OHLCV secara programatik.

## 2. Data yang Dibutuhkan
| Field | Tipe | Keterangan |
|---|---|---|
| stock_code | VARCHAR | Kode saham |
| trade_date | DATE | Tanggal |
| open | DECIMAL | Harga buka |
| high | DECIMAL | Harga tertinggi |
| low | DECIMAL | Harga terendah |
| close | DECIMAL | Harga penutupan |
| volume | BIGINT | Volume transaksi |
| Minimal 200 hari historis | | Untuk MA200 |

## 3. Struktur Database

```sql
CREATE TABLE technical_indicators (
    id              BIGSERIAL PRIMARY KEY,
    stock_code      VARCHAR(10) NOT NULL,
    trade_date      DATE NOT NULL,
    close           DECIMAL(12,2),
    volume          BIGINT,
    
    -- Moving Averages
    ma20            DECIMAL(12,2),
    ma50            DECIMAL(12,2),
    ma200           DECIMAL(12,2),
    ema12           DECIMAL(12,2),
    ema26           DECIMAL(12,2),
    
    -- Oscillators
    rsi_14          DECIMAL(6,2),
    
    -- MACD
    macd_line       DECIMAL(12,4),
    macd_signal     DECIMAL(12,4),
    macd_histogram  DECIMAL(12,4),
    
    -- Volatility
    atr_14          DECIMAL(12,4),
    
    -- Bollinger Bands
    bb_upper        DECIMAL(12,2),
    bb_middle       DECIMAL(12,2),
    bb_lower        DECIMAL(12,2),
    
    -- Volume MA
    vol_ma20        BIGINT,
    vol_ratio       DECIMAL(6,2),
    
    -- Pattern Detection
    trend_direction ENUM('UPTREND','DOWNTREND','SIDEWAYS'),
    pattern_detected VARCHAR(50),
    breakout_signal BOOLEAN,
    breakdown_signal BOOLEAN,
    golden_cross    BOOLEAN,
    death_cross     BOOLEAN,
    
    technical_score DECIMAL(5,2),
    UNIQUE (stock_code, trade_date)
);
```

## 4. Rumus Perhitungan Indikator

```python
import pandas as pd
import numpy as np

class TechnicalIndicatorCalculator:
    
    @staticmethod
    def compute_sma(prices: pd.Series, period: int) -> pd.Series:
        return prices.rolling(window=period, min_periods=period).mean()

    @staticmethod
    def compute_ema(prices: pd.Series, period: int) -> pd.Series:
        return prices.ewm(span=period, adjust=False).mean()

    @staticmethod
    def compute_rsi(prices: pd.Series, period: int = 14) -> pd.Series:
        """
        RSI = 100 - (100 / (1 + RS))
        RS = Avg Gain / Avg Loss
        """
        delta = prices.diff()
        gain = delta.clip(lower=0)
        loss = -delta.clip(upper=0)
        
        avg_gain = gain.ewm(com=period - 1, adjust=False).mean()
        avg_loss = loss.ewm(com=period - 1, adjust=False).mean()
        
        rs = avg_gain / (avg_loss + 1e-9)
        rsi = 100 - (100 / (1 + rs))
        return rsi

    @staticmethod
    def compute_macd(prices: pd.Series, 
                      fast: int = 12, 
                      slow: int = 26, 
                      signal: int = 9) -> pd.DataFrame:
        """
        MACD Line   = EMA(12) - EMA(26)
        Signal Line = EMA(9) of MACD Line
        Histogram   = MACD Line - Signal Line
        """
        ema_fast = prices.ewm(span=fast, adjust=False).mean()
        ema_slow = prices.ewm(span=slow, adjust=False).mean()
        macd_line = ema_fast - ema_slow
        signal_line = macd_line.ewm(span=signal, adjust=False).mean()
        histogram = macd_line - signal_line
        
        return pd.DataFrame({
            'macd': macd_line,
            'signal': signal_line,
            'histogram': histogram
        })

    @staticmethod
    def compute_atr(high: pd.Series, low: pd.Series, 
                     close: pd.Series, period: int = 14) -> pd.Series:
        """
        True Range = max(H-L, |H-PC|, |L-PC|)
        ATR = EMA(TR, period)
        """
        prev_close = close.shift(1)
        tr = pd.concat([
            high - low,
            (high - prev_close).abs(),
            (low - prev_close).abs()
        ], axis=1).max(axis=1)
        return tr.ewm(span=period, adjust=False).mean()

    @staticmethod
    def compute_bollinger_bands(prices: pd.Series, 
                                 period: int = 20, 
                                 std_dev: float = 2.0) -> pd.DataFrame:
        """
        Middle = SMA(20)
        Upper  = SMA(20) + 2 * StdDev(20)
        Lower  = SMA(20) - 2 * StdDev(20)
        """
        middle = prices.rolling(window=period).mean()
        std = prices.rolling(window=period).std()
        return pd.DataFrame({
            'upper': middle + std_dev * std,
            'middle': middle,
            'lower': middle - std_dev * std
        })

    def compute_all(self, df: pd.DataFrame) -> pd.DataFrame:
        """Hitung semua indikator dalam satu DataFrame"""
        result = df.copy()
        
        result['ma20'] = self.compute_sma(df['close'], 20)
        result['ma50'] = self.compute_sma(df['close'], 50)
        result['ma200'] = self.compute_sma(df['close'], 200)
        result['ema12'] = self.compute_ema(df['close'], 12)
        result['ema26'] = self.compute_ema(df['close'], 26)
        result['rsi'] = self.compute_rsi(df['close'])
        
        macd = self.compute_macd(df['close'])
        result['macd'] = macd['macd']
        result['macd_signal'] = macd['signal']
        result['macd_histogram'] = macd['histogram']
        
        result['atr'] = self.compute_atr(df['high'], df['low'], df['close'])
        
        bb = self.compute_bollinger_bands(df['close'])
        result['bb_upper'] = bb['upper']
        result['bb_middle'] = bb['middle']
        result['bb_lower'] = bb['lower']
        
        result['vol_ma20'] = df['volume'].rolling(20).mean()
        result['vol_ratio'] = df['volume'] / (result['vol_ma20'] + 1e-9)
        
        return result
```

## 5. Logika Deteksi Pola

```python
class PatternDetector:
    
    def detect_trend(self, df: pd.DataFrame, window: int = 20) -> str:
        """
        Uptrend: close > MA20 > MA50 DAN harga membuat HH (Higher High)
        Downtrend: close < MA20 < MA50 DAN harga membuat LL (Lower Low)
        Sideways: di antara keduanya
        """
        latest = df.iloc[-1]
        
        # MA alignment
        ma_bullish = (latest['close'] > latest['ma20'] and 
                      latest['ma20'] > latest['ma50'])
        ma_bearish = (latest['close'] < latest['ma20'] and 
                      latest['ma20'] < latest['ma50'])
        
        # Higher High / Lower Low
        recent = df.tail(window)
        hh = recent['high'].is_monotonic_increasing
        ll = recent['low'].is_monotonic_decreasing
        
        if ma_bullish or hh:
            return 'UPTREND'
        elif ma_bearish or ll:
            return 'DOWNTREND'
        else:
            return 'SIDEWAYS'

    def detect_breakout(self, df: pd.DataFrame, 
                         resistance_window: int = 20) -> bool:
        """
        Breakout: Close hari ini menembus high tertinggi N hari sebelumnya
        DAN volume > 1.5x rata-rata
        """
        latest = df.iloc[-1]
        prev_N = df.iloc[-(resistance_window+1):-1]
        
        resistance = prev_N['high'].max()
        vol_ok = latest['volume'] > df.tail(20)['volume'].mean() * 1.5
        
        return (latest['close'] > resistance) and vol_ok

    def detect_breakdown(self, df: pd.DataFrame, 
                          support_window: int = 20) -> bool:
        """
        Breakdown: Close menembus low terendah N hari DAN volume tinggi
        """
        latest = df.iloc[-1]
        prev_N = df.iloc[-(support_window+1):-1]
        
        support = prev_N['low'].min()
        vol_ok = latest['volume'] > df.tail(20)['volume'].mean() * 1.5
        
        return (latest['close'] < support) and vol_ok

    def detect_golden_cross(self, df: pd.DataFrame) -> bool:
        """
        Golden Cross: MA50 memotong MA200 dari bawah ke atas
        """
        if len(df) < 2:
            return False
        today = df.iloc[-1]
        yesterday = df.iloc[-2]
        
        return (today['ma50'] > today['ma200'] and 
                yesterday['ma50'] <= yesterday['ma200'])

    def detect_death_cross(self, df: pd.DataFrame) -> bool:
        """
        Death Cross: MA50 memotong MA200 dari atas ke bawah
        """
        if len(df) < 2:
            return False
        today = df.iloc[-1]
        yesterday = df.iloc[-2]
        
        return (today['ma50'] < today['ma200'] and 
                yesterday['ma50'] >= yesterday['ma200'])

    def detect_all_patterns(self, df: pd.DataFrame) -> dict:
        latest = df.iloc[-1]
        
        return {
            "trend": self.detect_trend(df),
            "breakout": self.detect_breakout(df),
            "breakdown": self.detect_breakdown(df),
            "golden_cross": self.detect_golden_cross(df),
            "death_cross": self.detect_death_cross(df),
            "rsi_oversold": latest.get('rsi', 50) < 30,
            "rsi_overbought": latest.get('rsi', 50) > 70,
            "macd_bullish_cross": (latest.get('macd', 0) > latest.get('macd_signal', 0) and 
                                    df.iloc[-2].get('macd', 0) <= df.iloc[-2].get('macd_signal', 0)),
            "macd_bearish_cross": (latest.get('macd', 0) < latest.get('macd_signal', 0) and 
                                    df.iloc[-2].get('macd', 0) >= df.iloc[-2].get('macd_signal', 0)),
            "above_ma200": latest['close'] > latest.get('ma200', 0),
            "volume_spike": latest.get('vol_ratio', 1.0) > 2.0
        }
```

## 6. Technical Score (0-100)

```python
def compute_technical_score(df: pd.DataFrame, patterns: dict) -> float:
    """
    Technical Score berdasarkan alignment semua indikator
    """
    latest = df.iloc[-1]
    score_parts = []

    # A. Trend Score (25%)
    trend_map = {'UPTREND': 80, 'SIDEWAYS': 50, 'DOWNTREND': 20}
    score_parts.append(trend_map.get(patterns['trend'], 50) * 0.25)

    # B. MA Score (20%)
    ma_score = 50
    if latest['close'] > latest.get('ma20', 0):
        ma_score += 10
    if latest['close'] > latest.get('ma50', 0):
        ma_score += 10
    if latest['close'] > latest.get('ma200', 0):
        ma_score += 10
    if latest.get('ma20', 0) > latest.get('ma50', 0):
        ma_score += 10
    if latest.get('ma50', 0) > latest.get('ma200', 0):
        ma_score += 10
    score_parts.append(min(ma_score, 100) * 0.20)

    # C. RSI Score (15%)
    rsi = latest.get('rsi', 50)
    if 40 <= rsi <= 60:
        rsi_score = 60
    elif 30 <= rsi < 40 or 60 < rsi <= 70:
        rsi_score = 55
    elif rsi < 30:
        rsi_score = 75  # Oversold = potensi rebound
    else:
        rsi_score = 30  # Overbought = potensi koreksi
    score_parts.append(rsi_score * 0.15)

    # D. MACD Score (20%)
    macd = latest.get('macd', 0)
    histogram = latest.get('macd_histogram', 0)
    prev_hist = df.iloc[-2].get('macd_histogram', 0)
    
    if macd > 0 and histogram > 0 and histogram > prev_hist:
        macd_score = 85
    elif macd > 0 and histogram > 0:
        macd_score = 70
    elif macd > 0 and histogram < 0:
        macd_score = 50
    elif macd < 0 and histogram > 0:
        macd_score = 45
    elif macd < 0 and histogram < 0 and histogram > prev_hist:
        macd_score = 35
    else:
        macd_score = 20
    score_parts.append(macd_score * 0.20)

    # E. Volume Score (10%)
    vol_ratio = latest.get('vol_ratio', 1.0)
    if vol_ratio > 2.0 and patterns['trend'] == 'UPTREND':
        vol_score = 90
    elif vol_ratio > 1.5:
        vol_score = 70
    elif vol_ratio > 1.0:
        vol_score = 55
    else:
        vol_score = 40
    score_parts.append(vol_score * 0.10)

    # F. Pattern Bonus/Penalty (10%)
    pattern_score = 50
    if patterns['breakout']:
        pattern_score = 90
    elif patterns['golden_cross']:
        pattern_score = 85
    elif patterns['breakdown']:
        pattern_score = 10
    elif patterns['death_cross']:
        pattern_score = 15
    score_parts.append(pattern_score * 0.10)

    return round(sum(score_parts), 2)
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "analysis_date": "2025-08-15",
  "price_data": {
    "open": 9925,
    "high": 10050,
    "low": 9900,
    "close": 10000,
    "volume": 45200000
  },
  "indicators": {
    "ma20": 9850.5,
    "ma50": 9620.0,
    "ma200": 9145.0,
    "rsi": 58.4,
    "macd": 125.3,
    "macd_signal": 98.7,
    "macd_histogram": 26.6,
    "atr": 87.5,
    "bb_upper": 10180.0,
    "bb_middle": 9850.5,
    "bb_lower": 9521.0,
    "vol_ratio": 1.85
  },
  "patterns": {
    "trend": "UPTREND",
    "breakout": true,
    "breakdown": false,
    "golden_cross": false,
    "death_cross": false,
    "rsi_oversold": false,
    "rsi_overbought": false,
    "macd_bullish_cross": false,
    "above_ma200": true,
    "volume_spike": false
  },
  "technical_score": 72.8,
  "rating": "Buy",
  "narrative": "BBCA dalam UPTREND kuat. Harga di atas MA20 (9,851), MA50 (9,620), dan MA200 (9,145) — full bullish alignment. RSI 58.4 dalam zona sehat (belum overbought). MACD histogram naik positif. Breakout terdeteksi dengan volume 1.85x rata-rata. Technical Score 72.8/100."
}
```

---

# FITUR 9 — STOCK SCREENER

## 1. Tujuan Fitur
Filter seluruh saham di pasar berdasarkan kombinasi sinyal untuk menemukan peluang terbaik.

## 2. Struktur Screener Engine

```python
class ZeroStockScreener:
    
    def __init__(self, db, broker_engine, foreign_engine, 
                  tech_engine, bandar_engine):
        self.db = db
        self.broker = broker_engine
        self.foreign = foreign_engine
        self.tech = tech_engine
        self.bandar = bandar_engine

    # =========================================================
    # SCREENER 1: BANDAR ACCUMULATION SCREENER
    # =========================================================
    def screen_bandar_accumulation(self, date: str, 
                                    min_score: float = 70.0,
                                    min_market_cap: float = 1e12) -> list:
        """
        Kriteria:
        - Broker Acc Score > 70
        - Net Buy Value positif 3+ hari berturut
        - Volume meningkat
        - Harga sideways atau sedikit naik
        """
        query = """
            SELECT DISTINCT bt.stock_code
            FROM broker_summary bs
            JOIN stock_master sm ON bs.stock_code = sm.stock_code
            WHERE bs.trade_date = %s
              AND bs.broker_acc_score >= %s
              AND sm.market_cap >= %s
              AND bs.total_net_buy_value > 0
        """
        candidates = pd.read_sql(query, self.db, params=[date, min_score, min_market_cap])
        
        results = []
        for stock_code in candidates['stock_code']:
            score = self._get_bandar_score(stock_code, date)
            if score >= min_score:
                results.append({
                    'stock_code': stock_code,
                    'bandar_score': score,
                    'signal': 'BANDAR_ACCUMULATION'
                })
        
        return sorted(results, key=lambda x: x['bandar_score'], reverse=True)

    # =========================================================
    # SCREENER 2: FOREIGN BUY SCREENER
    # =========================================================
    def screen_foreign_buy(self, date: str,
                            min_consecutive_days: int = 3,
                            min_net_value: float = 5e9) -> list:
        """
        Kriteria:
        - Foreign net buy positif min 3 hari berturut
        - Total net value > 5M IDR
        - Foreign strength score > 65
        """
        query = """
            SELECT stock_code, foreign_net_value, 
                   consecutive_inflow_days, foreign_strength_score
            FROM foreign_flow_aggregated
            WHERE calc_date = %s
              AND period_days = 5
              AND cum_net_value >= %s
              AND dominance_score > 0
            ORDER BY foreign_strength_score DESC
        """
        return pd.read_sql(query, self.db, params=[date, min_net_value]).to_dict('records')

    # =========================================================
    # SCREENER 3: VOLUME SPIKE SCREENER
    # =========================================================
    def screen_volume_spike(self, date: str,
                             min_vol_ratio: float = 3.0,
                             min_price_change: float = 0.0) -> list:
        """
        Kriteria:
        - Volume hari ini > 3x rata-rata 20 hari
        - Harga tidak turun (flat atau naik)
        - Didukung broker net buy
        """
        query = """
            SELECT ti.stock_code, ti.vol_ratio, 
                   ti.close, ti.close / LAG(ti.close) OVER 
                   (PARTITION BY ti.stock_code ORDER BY ti.trade_date) - 1 AS price_change
            FROM technical_indicators ti
            WHERE ti.trade_date = %s
              AND ti.vol_ratio >= %s
            HAVING price_change >= %s
            ORDER BY ti.vol_ratio DESC
        """
        return pd.read_sql(query, self.db, params=[date, min_vol_ratio, min_price_change]).to_dict('records')

    # =========================================================
    # SCREENER 4: BREAKOUT SCREENER
    # =========================================================
    def screen_breakout(self, date: str, 
                         resistance_window: int = 20,
                         min_vol_ratio: float = 1.5) -> list:
        """
        Kriteria:
        - Close > max(high) 20 hari sebelumnya
        - Volume > 1.5x average
        - RSI 50-70 (tidak terlalu overbought)
        """
        query = """
            SELECT ti.stock_code, ti.close, ti.vol_ratio, ti.rsi_14,
                   (SELECT MAX(high) FROM technical_indicators 
                    WHERE stock_code = ti.stock_code 
                      AND trade_date BETWEEN DATE_SUB(%s, INTERVAL %s DAY) 
                                        AND DATE_SUB(%s, INTERVAL 1 DAY)
                   ) AS resistance
            FROM technical_indicators ti
            WHERE ti.trade_date = %s
              AND ti.vol_ratio >= %s
              AND ti.rsi_14 BETWEEN 45 AND 75
        """
        df = pd.read_sql(query, self.db, 
                         params=[date, resistance_window, date, date, min_vol_ratio])
        return df[df['close'] > df['resistance']].to_dict('records')

    # =========================================================
    # SCREENER 5: SMART MONEY SCREENER
    # =========================================================
    def screen_smart_money(self, date: str, min_sm_score: float = 70.0) -> list:
        """
        Kriteria:
        - Smart Money Score > 70
        - sm_entering = True ATAU sm_accumulating = True
        - Bandarmology Score > 65
        """
        query = """
            SELECT sms.stock_code, sms.smart_money_score, 
                   sms.sm_entering, sms.sm_accumulating,
                   bs.bandar_score
            FROM smart_money_signals sms
            JOIN bandarmology_scores bs ON sms.stock_code = bs.stock_code 
                                       AND sms.signal_date = bs.signal_date
            WHERE sms.signal_date = %s
              AND sms.smart_money_score >= %s
              AND (sms.sm_entering = TRUE OR sms.sm_accumulating = TRUE)
              AND bs.bandar_score >= 65
            ORDER BY sms.smart_money_score DESC
        """
        return pd.read_sql(query, self.db, params=[date, min_sm_score]).to_dict('records')

    # =========================================================
    # SCREENER 6: MOMENTUM SCREENER
    # =========================================================
    def screen_momentum(self, date: str,
                         min_rsi: float = 50,
                         max_rsi: float = 70,
                         min_macd_hist: float = 0) -> list:
        """
        Kriteria:
        - RSI 50-70 (dalam momentum, belum overbought)
        - MACD histogram positif dan meningkat
        - Harga > MA20 > MA50
        - Volume di atas rata-rata
        """
        query = """
            SELECT stock_code, close, rsi_14, macd_histogram, vol_ratio,
                   ma20, ma50, ma200
            FROM technical_indicators
            WHERE trade_date = %s
              AND rsi_14 BETWEEN %s AND %s
              AND macd_histogram >= %s
              AND close > ma20
              AND ma20 > ma50
              AND vol_ratio > 1.0
            ORDER BY rsi_14 DESC
        """
        return pd.read_sql(query, self.db, 
                           params=[date, min_rsi, max_rsi, min_macd_hist]).to_dict('records')

    # =========================================================
    # SCREENER 7: TREND FOLLOWING SCREENER
    # =========================================================
    def screen_trend_following(self, date: str) -> list:
        """
        Kriteria:
        - Harga > MA200 (long-term bull)
        - MA20 > MA50 > MA200 (perfect alignment)
        - RSI 45-65
        - Volume stabil
        """
        query = """
            SELECT stock_code, close, rsi_14, ma20, ma50, ma200, vol_ratio
            FROM technical_indicators
            WHERE trade_date = %s
              AND close > ma200
              AND ma20 > ma50
              AND ma50 > ma200
              AND rsi_14 BETWEEN 45 AND 65
            ORDER BY (close - ma200) / ma200 DESC
        """
        return pd.read_sql(query, self.db, params=[date]).to_dict('records')

    # =========================================================
    # SCREENER 8: TURNAROUND SCREENER
    # =========================================================
    def screen_turnaround(self, date: str,
                           max_rsi: float = 35,
                           min_broker_score: float = 60) -> list:
        """
        Kriteria:
        - RSI < 35 (oversold)
        - MACD histogram mulai naik dari negatif
        - Broker mulai akumulasi (score > 60)
        - Foreign net mulai membaik
        - Harga test support penting
        """
        query = """
            SELECT ti.stock_code, ti.close, ti.rsi_14, 
                   ti.macd_histogram,
                   bs.broker_acc_score
            FROM technical_indicators ti
            LEFT JOIN broker_summary bs ON ti.stock_code = bs.stock_code 
                                        AND ti.trade_date = bs.trade_date
            WHERE ti.trade_date = %s
              AND ti.rsi_14 <= %s
              AND ti.macd_histogram > 0
              AND bs.broker_acc_score >= %s
            ORDER BY ti.rsi_14 ASC
        """
        return pd.read_sql(query, self.db, 
                           params=[date, max_rsi, min_broker_score]).to_dict('records')

    def run_all_screeners(self, date: str) -> dict:
        """Jalankan semua screener sekaligus"""
        return {
            "date": date,
            "bandar_accumulation": self.screen_bandar_accumulation(date),
            "foreign_buy": self.screen_foreign_buy(date),
            "volume_spike": self.screen_volume_spike(date),
            "breakout": self.screen_breakout(date),
            "smart_money": self.screen_smart_money(date),
            "momentum": self.screen_momentum(date),
            "trend_following": self.screen_trend_following(date),
            "turnaround": self.screen_turnaround(date)
        }
```

---

# FITUR 10 — MASTER SCORE ZEROSTOCK (ZeroScore)

## 1. Tujuan Fitur
Menghasilkan satu skor komprehensif (0-100) yang menggabungkan SEMUA dimensi analisa untuk menghasilkan rating akhir yang akurat dan dapat ditindaklanjuti.

## 2. Komponen, Bobot, dan Alasan

| # | Komponen | Bobot | Alasan Pemilihan |
|---|---|---|---|
| 1 | Technical Score | 25% | Analisa teknikal adalah dasar timing entry/exit yang paling universal |
| 2 | Bandarmology Score | 30% | Di BEI, aktivitas bandar/institusi adalah driver utama harga jangka pendek-menengah |
| 3 | Foreign Score | 20% | Dana asing sering menjadi "informed smart money" dengan riset lebih dalam |
| 4 | Orderbook Score | 10% | Konfirmasi demand/supply real-time yang tidak bisa dipalsukan dalam jangka panjang |
| 5 | Momentum Score | 8% | Momentum tren adalah filter tambahan untuk kualitas sinyal |
| 6 | Liquidity Score | 4% | Pastikan saham cukup likuid untuk keluar-masuk dengan mudah |
| 7 | Risk Score | 3% | Penalti untuk saham dengan risiko tinggi (volatilitas, fundamental buruk) |

**Total Bobot: 100%**

**Alasan Memilih Bandarmology sebagai Bobot Tertinggi:**
Di Bursa Efek Indonesia (BEI), pasar masih didominasi "bandar" (institusi besar yang menggerakkan harga). Retail investor yang mengikuti jejak bandar secara historis menghasilkan return lebih baik daripada pure technical trader. Foreign flow sebagai bobot kedua karena asing sering menjadi "precursor" sebelum kenaikan harga signifikan.

## 3. Rumus Matematis Lengkap

```python
class ZeroScoreEngine:
    """
    Master Score Engine untuk ZeroStock Platform
    
    ZeroScore = weighted sum dari seluruh komponen
    dengan adjustment untuk divergensi dan penalti risiko
    """
    
    # === BOBOT KOMPONEN ===
    WEIGHTS = {
        'technical':     0.25,
        'bandarmology':  0.30,
        'foreign':       0.20,
        'orderbook':     0.10,
        'momentum':      0.08,
        'liquidity':     0.04,
        'risk':          0.03
    }
    
    def compute_momentum_score(self, df: pd.DataFrame) -> float:
        """
        Momentum Score dari rate of change dan trend strength
        
        Komponen:
        - ROC (Rate of Change) 5 hari
        - ROC 20 hari
        - Price vs 52-week high
        - Consecutive up days
        """
        latest = df.iloc[-1]
        price_5d_ago = df.iloc[-6]['close'] if len(df) >= 6 else latest['close']
        price_20d_ago = df.iloc[-21]['close'] if len(df) >= 21 else latest['close']
        high_52w = df.tail(252)['high'].max() if len(df) >= 252 else df['high'].max()
        
        roc_5d = (latest['close'] - price_5d_ago) / (price_5d_ago + 1e-9) * 100
        roc_20d = (latest['close'] - price_20d_ago) / (price_20d_ago + 1e-9) * 100
        
        # Normalize ROC to 0-100
        # ROC +10% → score ~80, ROC -10% → score ~20
        roc5_score = min(max(50 + roc_5d * 3, 0), 100)
        roc20_score = min(max(50 + roc_20d * 1.5, 0), 100)
        
        # Distance from 52w high
        dist_from_high = (latest['close'] - high_52w) / (high_52w + 1e-9) * 100
        high_score = min(max(80 + dist_from_high, 0), 100)
        
        # Consecutive up/down days
        closes = df['close'].tail(5).values
        up_days = sum(1 for i in range(1, len(closes)) if closes[i] > closes[i-1])
        consec_score = up_days / 4 * 100
        
        return round(
            roc5_score * 0.35 + roc20_score * 0.35 + 
            high_score * 0.20 + consec_score * 0.10, 2
        )

    def compute_liquidity_score(self, df: pd.DataFrame, 
                                 avg_daily_value: float) -> float:
        """
        Liquidity Score berdasarkan kemudahan jual beli
        
        Saham tidak likuid = risiko stuck di posisi
        """
        # Benchmark: > 50M IDR/hari = sangat likuid
        if avg_daily_value >= 50e9:     return 100.0
        elif avg_daily_value >= 10e9:   return 85.0
        elif avg_daily_value >= 1e9:    return 65.0
        elif avg_daily_value >= 100e6:  return 40.0
        else:                           return 15.0

    def compute_risk_score(self, df: pd.DataFrame,
                            atr: float, 
                            fundamental_risk: float = 50.0) -> float:
        """
        Risk Score — SEMAKIN TINGGI = SEMAKIN BERISIKO (inverse untuk ZeroScore)
        
        Komponen:
        - Volatilitas (ATR%)
        - Drawdown maksimal 20 hari
        - Fundamental risk (opsional, dari data fundamental)
        
        Output: 0-100, dimana 100 = sangat berisiko
        Nanti diinverse: (100 - risk_score) untuk meningkatkan ZeroScore
        """
        latest = df.iloc[-1]
        atr_pct = atr / (latest['close'] + 1e-9) * 100
        
        # ATR% to risk: < 1% = rendah, > 5% = sangat tinggi
        if atr_pct < 1.0:   vol_risk = 20
        elif atr_pct < 2.0: vol_risk = 35
        elif atr_pct < 3.0: vol_risk = 50
        elif atr_pct < 5.0: vol_risk = 70
        else:               vol_risk = 90
        
        # Max drawdown 20 hari
        recent_20 = df.tail(20)
        peak = recent_20['close'].max()
        trough = recent_20['close'].min()
        drawdown = (peak - trough) / (peak + 1e-9) * 100
        dd_risk = min(drawdown * 5, 100)
        
        total_risk = vol_risk * 0.50 + dd_risk * 0.30 + fundamental_risk * 0.20
        return round(min(total_risk, 100), 2)

    def compute_zero_score(
        self,
        technical_score: float,
        bandarmology_score: float,
        foreign_score: float,
        orderbook_score: float,
        momentum_score: float,
        liquidity_score: float,
        risk_score: float         # 0-100, semakin tinggi semakin buruk
    ) -> dict:
        """
        RUMUS UTAMA ZEROSCORE
        
        ZeroScore = Σ(score_i × weight_i) × liquidity_factor × (1 - risk_penalty)
        
        Dimana:
        - liquidity_factor = liquidity_score / 100 (0-1)
        - risk_penalty = risk_score / 100 * 0.15  (max 15% pengurangan)
        """
        
        # === BASE SCORE (weighted average) ===
        base_score = (
            technical_score     * self.WEIGHTS['technical'] +
            bandarmology_score  * self.WEIGHTS['bandarmology'] +
            foreign_score       * self.WEIGHTS['foreign'] +
            orderbook_score     * self.WEIGHTS['orderbook'] +
            momentum_score      * self.WEIGHTS['momentum']
        )
        
        # === LIQUIDITY ADJUSTMENT ===
        # Saham tidak likuid max score 70 (tidak boleh dapat "Strong Buy")
        liq_cap = 70 if liquidity_score < 40 else 100
        
        # === RISK PENALTY ===
        # Risk tinggi mengurangi skor maks 15%
        risk_penalty = (risk_score / 100) * 0.15
        adjusted_score = base_score * (1 - risk_penalty)
        
        # === CONVERGENCE BONUS ===
        # Jika semua komponen utama selaras (semua > 70), berikan bonus 5 poin
        main_scores = [technical_score, bandarmology_score, foreign_score]
        if all(s > 70 for s in main_scores):
            adjusted_score = min(adjusted_score + 5.0, 100.0)
        
        # === DIVERGENCE PENALTY ===
        # Jika ada komponen sangat kontradiktif (teknikal > 80 tapi bandar < 30)
        if abs(technical_score - bandarmology_score) > 40:
            adjusted_score = max(adjusted_score - 5.0, 0.0)
        
        final_score = min(min(adjusted_score, liq_cap), 100.0)
        
        return {
            "base_score": round(base_score, 2),
            "risk_adjusted_score": round(adjusted_score, 2),
            "final_zero_score": round(final_score, 2),
            "liquidity_cap_applied": liq_cap < 100,
            "risk_penalty_pct": round(risk_penalty * 100, 2),
            "convergence_bonus": all(s > 70 for s in main_scores)
        }

    def get_rating(self, zero_score: float) -> dict:
        if zero_score >= 81:
            return {
                "rating": "Strong Buy",
                "color": "#00A651",
                "action": "Beli agresif, ini adalah peluang terbaik",
                "risk_level": "Rendah-Sedang",
                "holding_suggestion": "Short-Medium Term"
            }
        elif zero_score >= 61:
            return {
                "rating": "Buy",
                "color": "#6ABF69",
                "action": "Beli bertahap, fundamental dan teknikal mendukung",
                "risk_level": "Sedang",
                "holding_suggestion": "Short-Medium Term"
            }
        elif zero_score >= 41:
            return {
                "rating": "Hold",
                "color": "#FFC107",
                "action": "Pertahankan jika sudah punya, tunggu konfirmasi jika belum",
                "risk_level": "Sedang",
                "holding_suggestion": "Watch & Wait"
            }
        elif zero_score >= 21:
            return {
                "rating": "Sell",
                "color": "#FF7043",
                "action": "Pertimbangkan untuk mengurangi posisi",
                "risk_level": "Tinggi",
                "holding_suggestion": "Reduce position"
            }
        else:
            return {
                "rating": "Strong Sell",
                "color": "#D32F2F",
                "action": "Keluar dari posisi atau hindari sepenuhnya",
                "risk_level": "Sangat Tinggi",
                "holding_suggestion": "Exit immediately"
            }

    def generate_master_narrative(
        self,
        stock_code: str,
        zero_score_result: dict,
        components: dict,
        rating: dict,
        broker_detail: dict,
        foreign_detail: dict,
        tech_detail: dict
    ) -> str:
        """
        Narasi otomatis yang menjelaskan MENGAPA saham mendapat skor tersebut
        """
        score = zero_score_result['final_zero_score']
        parts = []
        
        # === HEADER ===
        parts.append(
            f"━━━ ZeroScore {stock_code}: {score:.1f}/100 ━━━\n"
            f"Rating: {rating['rating']} {self._get_emoji(rating['rating'])}\n"
            f"{rating['action']}\n"
        )
        
        # === ANALISA TEKNIKAL ===
        trend = tech_detail.get('patterns', {}).get('trend', 'N/A')
        rsi = tech_detail.get('indicators', {}).get('rsi', 50)
        parts.append(
            f"\n📊 TEKNIKAL (skor: {components['technical_score']:.1f})\n"
            f"  Tren: {trend} | RSI: {rsi:.1f}\n"
            f"  {'✅ Harga di atas MA200 — bullish jangka panjang.' if tech_detail.get('indicators',{}).get('close',0) > tech_detail.get('indicators',{}).get('ma200',0) else '⚠️ Harga di bawah MA200 — hati-hati.'}"
        )
        
        # === BANDARMOLOGI ===
        bandar_score = components['bandarmology_score']
        bandar_level = broker_detail.get('bandar_detection', {}).get('level', 0)
        parts.append(
            f"\n🎯 BANDARMOLOGI (skor: {bandar_score:.1f})\n"
            f"  Bandar Level: {['No Signal','Possible','Probable','Confirmed'][bandar_level]}\n"
            f"  Broker Net: {'BELI' if broker_detail.get('aggregates',{}).get('total_net_buy_value',0) > 0 else 'JUAL'}"
        )
        
        # === FOREIGN FLOW ===
        foreign_score = components['foreign_score']
        foreign_dir = foreign_detail.get('trend', {}).get('direction', 'NEUTRAL')
        cons_days = foreign_detail.get('signals', {}).get('consecutive_inflow_days', 0)
        parts.append(
            f"\n🌍 FOREIGN FLOW (skor: {foreign_score:.1f})\n"
            f"  Arah: {foreign_dir}\n"
            f"  Inflow berturut: {cons_days} hari"
        )
        
        # === ORDERBOOK ===
        ob_score = components['orderbook_score']
        parts.append(
            f"\n📋 ORDERBOOK (skor: {ob_score:.1f})\n"
            f"  {'Demand dominan 💚' if ob_score > 60 else 'Supply dominan 🔴' if ob_score < 40 else 'Seimbang ⚖️'}"
        )
        
        # === RISK ===
        risk_score = components['risk_score']
        parts.append(
            f"\n⚠️ RISIKO (skor: {risk_score:.1f} — semakin rendah semakin aman)\n"
            f"  Risk Penalty diterapkan: -{zero_score_result['risk_penalty_pct']:.1f}%"
        )
        
        # === KESIMPULAN ===
        convergence = zero_score_result.get('convergence_bonus', False)
        if convergence:
            parts.append(
                "\n✨ BONUS KONVERGENSI: Semua sinyal utama selaras ke atas! "
                "+5 poin bonus diberikan."
            )
        
        parts.append(
            f"\n{'━' * 35}\n"
            f"Saran: {rating['action']}\n"
            f"Holding: {rating['holding_suggestion']}\n"
            f"Risk Level: {rating['risk_level']}"
        )
        
        return "\n".join(parts)

    def _get_emoji(self, rating: str) -> str:
        return {
            "Strong Buy": "🚀",
            "Buy": "📈",
            "Hold": "⚖️",
            "Sell": "📉",
            "Strong Sell": "💀"
        }.get(rating, "")

    def analyze(
        self,
        stock_code: str,
        date: str,
        ohlcv: pd.DataFrame,
        broker_result: dict,
        foreign_result: dict,
        orderbook_result: dict,
        tech_result: dict,
        avg_daily_value: float = 10e9,
        fundamental_risk: float = 50.0
    ) -> dict:
        """
        MASTER ANALYZER — Menggabungkan semua fitur menjadi ZeroScore
        """
        
        # === Compute Momentum ===
        momentum_score = self.compute_momentum_score(ohlcv)
        
        # === Compute Liquidity ===
        liquidity_score = self.compute_liquidity_score(ohlcv, avg_daily_value)
        
        # === Compute Risk ===
        atr = tech_result['indicators'].get('atr', 0)
        risk_score = self.compute_risk_score(ohlcv, atr, fundamental_risk)
        
        # === Gather Component Scores ===
        components = {
            'technical_score': tech_result['technical_score'],
            'bandarmology_score': broker_result.get('bandarmology', {}).get(
                'bandar_score', 
                broker_result.get('scores', {}).get('broker_acc_score', 50)
            ),
            'foreign_score': foreign_result['foreign_strength_score'],
            'orderbook_score': orderbook_result['orderbook_score'],
            'momentum_score': momentum_score,
            'liquidity_score': liquidity_score,
            'risk_score': risk_score
        }
        
        # === Compute ZeroScore ===
        zero_score_result = self.compute_zero_score(
            technical_score=components['technical_score'],
            bandarmology_score=components['bandarmology_score'],
            foreign_score=components['foreign_score'],
            orderbook_score=components['orderbook_score'],
            momentum_score=components['momentum_score'],
            liquidity_score=components['liquidity_score'],
            risk_score=components['risk_score']
        )
        
        rating = self.get_rating(zero_score_result['final_zero_score'])
        
        narrative = self.generate_master_narrative(
            stock_code=stock_code,
            zero_score_result=zero_score_result,
            components=components,
            rating=rating,
            broker_detail=broker_result,
            foreign_detail=foreign_result,
            tech_detail=tech_result
        )
        
        return {
            "stock_code": stock_code,
            "analysis_date": date,
            "zero_score": zero_score_result,
            "components": components,
            "rating": rating,
            "narrative": narrative
        }
```

## 4. Format Output JSON ZeroScore

```json
{
  "stock_code": "BBCA",
  "analysis_date": "2025-08-15",
  "zero_score": {
    "base_score": 73.4,
    "risk_adjusted_score": 71.8,
    "final_zero_score": 76.8,
    "liquidity_cap_applied": false,
    "risk_penalty_pct": 2.2,
    "convergence_bonus": true
  },
  "components": {
    "technical_score": 72.8,
    "bandarmology_score": 75.6,
    "foreign_score": 76.2,
    "orderbook_score": 82.4,
    "momentum_score": 68.5,
    "liquidity_score": 100.0,
    "risk_score": 22.5
  },
  "rating": {
    "rating": "Buy",
    "color": "#6ABF69",
    "action": "Beli bertahap, fundamental dan teknikal mendukung",
    "risk_level": "Sedang",
    "holding_suggestion": "Short-Medium Term"
  },
  "narrative": "━━━ ZeroScore BBCA: 76.8/100 ━━━\nRating: Buy 📈\nBeli bertahap, fundamental dan teknikal mendukung\n\n📊 TEKNIKAL (skor: 72.8)\n  Tren: UPTREND | RSI: 58.4\n  ✅ Harga di atas MA200 — bullish jangka panjang.\n\n🎯 BANDARMOLOGI (skor: 75.6)\n  Bandar Level: Probable\n  Broker Net: BELI\n\n🌍 FOREIGN FLOW (skor: 76.2)\n  Arah: STRONG_INFLOW\n  Inflow berturut: 6 hari\n\n📋 ORDERBOOK (skor: 82.4)\n  Demand dominan 💚\n\n⚠️ RISIKO (skor: 22.5 — semakin rendah semakin aman)\n  Risk Penalty diterapkan: -2.2%\n\n✨ BONUS KONVERGENSI: Semua sinyal utama selaras ke atas! +5 poin bonus diberikan.\n\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\nSaran: Beli bertahap, fundamental dan teknikal mendukung\nHolding: Short-Medium Term\nRisk Level: Sedang"
}
```

## 5. Diagram Alur ZeroScore

```
DATA INPUT
    │
    ├─── OHLCV (200 hari)
    ├─── Broker Transactions (60 hari)
    ├─── Foreign Flow (60 hari)
    └─── Orderbook Snapshots (real-time)
         │
         ▼
FEATURE ENGINES
    │
    ├─[1] BrokerSummaryEngine      → broker_acc_score (0-100)
    ├─[2] ForeignFlowEngine        → foreign_strength_score (0-100)
    ├─[3] OrderbookEngine          → orderbook_score (0-100)
    ├─[4] TechnicalEngine          → technical_score (0-100)
    ├─[5] BandarmologyEngine       → bandar_score (0-100)
    ├─[6] MomentumEngine           → momentum_score (0-100)
    ├─[7] LiquidityEngine          → liquidity_score (0-100)
    └─[8] RiskEngine               → risk_score (0-100)
         │
         ▼
ZEROSCORE ENGINE
    │
    ├─ Weighted Sum (base_score)
    ├─ Risk Adjustment (-max 15%)
    ├─ Liquidity Cap (max 70 jika tidak likuid)
    ├─ Convergence Bonus (+5 jika semua > 70)
    └─ Divergence Penalty (-5 jika kontradiksi)
         │
         ▼
OUTPUT
    ├─ ZeroScore (0-100)
    ├─ Rating (Strong Buy → Strong Sell)
    ├─ Narrative (teks otomatis)
    └─ Component Breakdown
```

## 6. Ide Pengembangan Lanjutan ZeroScore

### 6.1 ZeroScore Backtesting
```
- Uji ZeroScore historis vs return aktual saham
- Hitung win rate per rating bracket
- Optimasi bobot berdasarkan data historis
- Target: Strong Buy memiliki win rate > 65% dalam 10 hari
```

### 6.2 ZeroScore ML Enhancement
```
- Input: semua komponen skor + market condition
- Output: ZeroScore yang lebih akurat via XGBoost/LightGBM
- Training data: 5 tahun historis BEI
- Fitur tambahan: IHSG trend, sector rotation, sentiment
```

### 6.3 ZeroScore Alerts
```
- ZeroScore cross 60 (masuk zona Buy) = alert notifikasi
- ZeroScore cross 40 (turun ke Hold) = warning
- ZeroScore drop drastis > 20 poin dalam 1 hari = URGENT alert
```

### 6.4 Portfolio ZeroScore
```
- Hitung rata-rata ZeroScore seluruh portofolio
- Identifikasi saham portofolio yang perlu di-review
- Rebalancing suggestion berbasis ZeroScore
```

### 6.5 ZeroScore Sector Relative
```
- Bandingkan ZeroScore vs rata-rata sektornya
- Saham dengan ZeroScore 70 tapi sektor rata-rata 85 = relatif underperform
- Relative ZeroScore = ZeroScore - SectorAvgScore
```
