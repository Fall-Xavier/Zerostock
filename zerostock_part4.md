# ZeroStock Platform — Fitur 11-14 (Ekstensi Baru)

> Pelengkap zerostock_part1.md – part3.md
> Fokus: Seasonality, Technical Pattern, Risk Management, ZeroScore v2
> Semua pseudocode pakai **Python murni** (list, dict, `math`) — tanpa pandas/numpy,
> supaya gampang dibaca dan gampang ditempel langsung ke `broker_summary.py` atau modul lain.

---

## CATATAN METODOLOGI — Kenapa Astrology Di-skip

Astrologi/numerologi **tidak dipakai** sebagai metode analisa di dokumen ini.
Alasan: tidak ada mekanisme yang menjelaskan kenapa posisi planet bisa
memengaruhi order broker atau keputusan investor — dan klaim "akurat" soal itu
biasanya hasil *data-mining* (uji ratusan pola sampai satu kebetulan "nyangkut").
Memakai sinyal seperti itu berisiko membuat keputusan trading terasa meyakinkan
padahal landasannya kosong.

Sebagai gantinya, Fitur 11 memakai **Seasonality / Calendar Effect** — pola
musiman yang *memang* punya penjelasan nyata (window dressing reksadana akhir
bulan/kuartal, januari effect, pre-holiday rally) dan dihitung murni dari data
historis harga saham itu sendiri. Statistik, bukan kepercayaan.

---

# FITUR 11 — SEASONALITY & CALENDAR EFFECT SCORE

## 1. Tujuan Fitur

Mengetahui apakah suatu saham/IHSG punya pola musiman yang konsisten secara
historis: bulan tertentu cenderung naik/turun, hari tertentu dalam seminggu
lebih volatile, atau ada lonjakan menjelang event terjadwal (window dressing,
rebalancing index, RUPS/dividen). Ini BUKAN prediksi masa depan — ini laporan
"secara historis apa yang biasa terjadi", dipakai sebagai konteks tambahan,
bukan sinyal tunggal.

## 2. Data yang Dibutuhkan

| Field        | Tipe    | Keterangan                          |
| ------------ | ------- | ------------------------------------ |
| stock_code   | str     | Kode saham                           |
| trade_date   | str     | Format YYYY-MM-DD                    |
| close        | float   | Harga penutupan                      |
| open         | float   | Harga pembukaan                      |

Minimal data historis 3-5 tahun (lebih panjang = lebih valid secara statistik).

## 3. Struktur Data (Python dict, tanpa SQL dulu — bisa dipetakan ke tabel nanti)

```python
# Satu baris harga harian
price_row = {
    "stock_code": "BBCA",
    "trade_date": "2025-08-15",   # YYYY-MM-DD
    "open": 9800,
    "close": 9875
}

# Tabel opsional kalau mau disimpan permanen
"""
CREATE TABLE seasonality_stats (
    id              BIGSERIAL PRIMARY KEY,
    stock_code      VARCHAR(10) NOT NULL,
    period_type     ENUM('MONTH', 'WEEKDAY', 'PRE_HOLIDAY', 'QUARTER_END'),
    period_label    VARCHAR(20),     -- '1'-'12' utk bulan, 'MON'-'FRI' utk weekday
    win_rate        DECIMAL(5,2),    -- % hari/periode dengan return positif
    avg_return_pct  DECIMAL(6,3),
    sample_size     INT,             -- jumlah observasi historis
    confidence      VARCHAR(10),     -- LOW / MEDIUM / HIGH (berdasar sample_size)
    UNIQUE (stock_code, period_type, period_label)
);
"""
```

## 4. Rumus Perhitungan (Python murni, tanpa pandas)

```python
def daily_return_pct(prev_close: float, today_close: float) -> float:
    """Return harian dalam persen"""
    if prev_close == 0:
        return 0.0
    return (today_close - prev_close) / prev_close * 100


def get_month(date_str: str) -> str:
    return date_str.split("-")[1]  # '01'..'12'


def get_weekday(date_str: str) -> str:
    """Hitung nama hari tanpa library datetime kalau mau full-manual,
    tapi disarankan tetap pakai modul bawaan 'datetime' (bukan pandas/numpy)."""
    import datetime
    y, m, d = map(int, date_str.split("-"))
    weekday_index = datetime.date(y, m, d).weekday()  # 0=Senin .. 6=Minggu
    names = ["MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"]
    return names[weekday_index]


def is_quarter_end_window(date_str: str, window_days: int = 5) -> bool:
    """True jika tanggal berada di window_days terakhir sebelum tutup kuartal
    (window dressing: akhir Mar/Jun/Sep/Des)"""
    import datetime
    y, m, d = map(int, date_str.split("-"))
    quarter_end_months = {3: 31, 6: 30, 9: 30, 12: 31}
    if m not in quarter_end_months:
        return False
    last_day = quarter_end_months[m]
    return (last_day - d) <= window_days
```

## 5. Logika Analisa

```
Untuk setiap kategori periode (bulan, hari-dalam-minggu, window kuartal):
  1. Kumpulkan semua return harian historis yang masuk kategori tsb
  2. win_rate = (jumlah return > 0) / (total observasi) × 100
  3. avg_return_pct = rata-rata semua return di kategori tsb
  4. confidence:
       sample_size < 8   -> LOW    (jangan terlalu dipercaya)
       sample_size 8-20  -> MEDIUM
       sample_size > 20  -> HIGH

Catatan: 8 sample untuk "bulan" = minimal 8 tahun data. Kalau data baru
2-3 tahun, confidence-nya otomatis LOW — engine tidak boleh menyembunyikan ini.
```

## 6. Kondisi "Bullish Seasonality"

- win_rate periode ini > 60% DAN confidence minimal MEDIUM
- avg_return_pct > 0 dan konsisten arahnya dengan win_rate
- Contoh: window dressing Desember historically positif di saham blue-chip

## 7. Kondisi "Bearish Seasonality"

- win_rate periode ini < 40% DAN confidence minimal MEDIUM
- avg_return_pct negatif

## 8. Kondisi Netral / Tidak Valid

- confidence == LOW (sample terlalu kecil — JANGAN dipakai sebagai sinyal)
- win_rate antara 40-60% (tidak ada pola dominan)

## 9. Seasonality Score (0-100) — Python murni

```python
def compute_seasonality_score(win_rate: float, avg_return_pct: float,
                               sample_size: int) -> dict:
    """
    Skor seasonality. Beda dengan fitur lain: skor ini SELALU dipenalti
    kalau sample kecil, supaya tidak overconfident.
    """
    # A. Win rate score (60%) - basis utama
    raw_a = win_rate  # sudah dalam skala 0-100

    # B. Magnitude score (40%) - seberapa besar rata-rata returnnya
    # clamp avg_return_pct ke range -5%..+5% lalu mapping ke 0-100
    clamped = max(min(avg_return_pct, 5), -5)
    raw_b = (clamped + 5) / 10 * 100

    base_score = raw_a * 0.60 + raw_b * 0.40

    # C. Confidence penalty
    if sample_size < 8:
        confidence = "LOW"
        penalty_factor = 0.5      # skor dipotong setengah
    elif sample_size < 20:
        confidence = "MEDIUM"
        penalty_factor = 0.8
    else:
        confidence = "HIGH"
        penalty_factor = 1.0

    final_score = base_score * penalty_factor

    return {
        "score": round(final_score, 2),
        "confidence": confidence,
        "sample_size": sample_size
    }
```

## 10. Sistem Rating

| Score  | Confidence min. | Rating               |
| ------ | ---------------- | --------------------- |
| 70-100 | MEDIUM            | Seasonal Tailwind      |
| 55-69  | MEDIUM            | Mild Seasonal Tailwind |
| 45-54  | -                 | Netral                |
| 30-44  | MEDIUM            | Mild Seasonal Headwind |
| 0-29   | MEDIUM            | Seasonal Headwind      |
| —      | LOW               | Insufficient Data (selalu ditampilkan apa adanya) |

## 11. Format Output (dict / JSON)

```python
{
    "stock_code": "BBCA",
    "as_of_date": "2025-08-15",
    "seasonality": {
        "current_month": {
            "label": "08",
            "win_rate": 58.3,
            "avg_return_pct": 1.2,
            "sample_size": 12,
            "score": 56.4,
            "confidence": "MEDIUM",
            "rating": "Mild Seasonal Tailwind"
        },
        "quarter_end_window": {
            "in_window": False,
            "historical_win_rate_in_window": 71.0,
            "sample_size": 16,
            "confidence": "MEDIUM"
        },
        "weekday": {
            "label": "FRI",
            "win_rate": 52.1,
            "sample_size": 240,
            "confidence": "HIGH"
        }
    },
    "disclaimer": "Pola musiman adalah statistik historis, bukan jaminan. "
                  "Selalu kombinasikan dengan analisa lain (broker, foreign flow, teknikal)."
}
```

## 12. Pseudocode Python Lengkap (engine, tanpa pandas/numpy)

```python
class SeasonalityEngine:
    """
    Input: list of dict price_history, urut tanggal menaik
    [{"trade_date": "2020-01-02", "close": 6800}, ...]
    """

    def __init__(self, price_history: list):
        # price_history harus sudah urut dari tanggal paling lama -> terbaru
        self.history = price_history

    def _daily_returns(self):
        """Hasilkan list of (date_str, return_pct)"""
        returns = []
        for i in range(1, len(self.history)):
            prev_close = self.history[i - 1]["close"]
            today = self.history[i]
            ret = daily_return_pct(prev_close, today["close"])
            returns.append((today["trade_date"], ret))
        return returns

    def analyze_by_month(self) -> dict:
        returns = self._daily_returns()
        buckets = {}  # '01'..'12' -> list of returns
        for date_str, ret in returns:
            month = get_month(date_str)
            buckets.setdefault(month, []).append(ret)

        result = {}
        for month, rets in buckets.items():
            wins = sum(1 for r in rets if r > 0)
            win_rate = wins / len(rets) * 100 if rets else 0
            avg_ret = sum(rets) / len(rets) if rets else 0
            scored = compute_seasonality_score(win_rate, avg_ret, len(rets))
            result[month] = {
                "win_rate": round(win_rate, 2),
                "avg_return_pct": round(avg_ret, 3),
                "sample_size": len(rets),
                **scored
            }
        return result

    def analyze_by_weekday(self) -> dict:
        returns = self._daily_returns()
        buckets = {}
        for date_str, ret in returns:
            wd = get_weekday(date_str)
            buckets.setdefault(wd, []).append(ret)

        result = {}
        for wd, rets in buckets.items():
            wins = sum(1 for r in rets if r > 0)
            win_rate = wins / len(rets) * 100 if rets else 0
            avg_ret = sum(rets) / len(rets) if rets else 0
            result[wd] = {
                "win_rate": round(win_rate, 2),
                "avg_return_pct": round(avg_ret, 3),
                "sample_size": len(rets)
            }
        return result

    def analyze_quarter_end_window(self, window_days: int = 5) -> dict:
        returns = self._daily_returns()
        in_window_rets = [r for d, r in returns if is_quarter_end_window(d, window_days)]
        wins = sum(1 for r in in_window_rets if r > 0)
        win_rate = wins / len(in_window_rets) * 100 if in_window_rets else 0
        avg_ret = sum(in_window_rets) / len(in_window_rets) if in_window_rets else 0
        return {
            "win_rate": round(win_rate, 2),
            "avg_return_pct": round(avg_ret, 3),
            "sample_size": len(in_window_rets)
        }

    def analyze(self, stock_code: str, as_of_date: str) -> dict:
        by_month = self.analyze_by_month()
        by_weekday = self.analyze_by_weekday()
        qend = self.analyze_quarter_end_window()

        current_month = get_month(as_of_date)

        return {
            "stock_code": stock_code,
            "as_of_date": as_of_date,
            "current_month_stats": by_month.get(current_month, {}),
            "weekday_stats": by_weekday,
            "quarter_end_window_stats": qend,
            "disclaimer": (
                "Pola musiman adalah statistik historis, bukan jaminan. "
                "Selalu kombinasikan dengan analisa lain (broker, foreign flow, teknikal)."
            )
        }
```

## 13. Ide Pengembangan Lanjutan

- **Event Calendar Overlay**: tandai tanggal RUPS, cum-date dividen, rebalancing
  index IDX30/LQ45 — lihat apakah saham biasanya bergerak menjelang tanggal itu.
- **Sector Seasonality**: agregat seasonality per sektor (consumer biasanya kuat
  jelang lebaran, dst).
- **IHSG Overlay**: bandingkan seasonality saham individual vs seasonality IHSG
  di periode yang sama — kalau saham lebih kuat dari index di bulan tsb, itu
  baru sinyal relative strength musiman.

---

# FITUR 12 — TECHNICAL PATTERN RECOGNITION (SIMPEL)

## 1. Tujuan Fitur

Mendeteksi support/resistance, pola candlestick dasar, dan divergence
sederhana — semua dengan operasi matematika dasar (perbandingan angka,
rata-rata manual) tanpa library teknikal seperti TA-Lib/pandas-ta.

## 2. Data yang Dibutuhkan

| Field  | Tipe  | Keterangan |
| ------ | ----- | ---------- |
| date   | str   | Tanggal    |
| open   | float | Harga buka |
| high   | float | Harga tertinggi |
| low    | float | Harga terendah |
| close  | float | Harga tutup |
| volume | int   | Volume     |

## 3. Struktur Data

```python
ohlcv_row = {
    "date": "2025-08-15",
    "open": 9800, "high": 9900, "low": 9750, "close": 9875, "volume": 12_500_000
}
# ohlcv_history: list of ohlcv_row, urut tanggal menaik
```

## 4. Rumus Perhitungan — Support & Resistance (Local Pivot, tanpa numpy)

```python
def find_local_pivots(ohlcv_history: list, lookback: int = 3) -> dict:
    """
    Pivot high  = titik dengan high tertinggi dibanding 'lookback' bar
                  sebelum dan sesudahnya.
    Pivot low   = sebaliknya.
    Ini definisi support/resistance paling dasar dan paling mudah dipahami.
    """
    pivot_highs = []
    pivot_lows = []
    n = len(ohlcv_history)

    for i in range(lookback, n - lookback):
        window = ohlcv_history[i - lookback: i + lookback + 1]
        current = ohlcv_history[i]

        is_pivot_high = all(current["high"] >= bar["high"] for bar in window)
        is_pivot_low = all(current["low"] <= bar["low"] for bar in window)

        if is_pivot_high:
            pivot_highs.append({"date": current["date"], "price": current["high"]})
        if is_pivot_low:
            pivot_lows.append({"date": current["date"], "price": current["low"]})

    return {"resistance_candidates": pivot_highs, "support_candidates": pivot_lows}


def nearest_support_resistance(current_price: float, pivots: dict) -> dict:
    """Ambil support terdekat di bawah harga & resistance terdekat di atas harga"""
    supports_below = [p["price"] for p in pivots["support_candidates"] if p["price"] < current_price]
    resistances_above = [p["price"] for p in pivots["resistance_candidates"] if p["price"] > current_price]

    nearest_support = max(supports_below) if supports_below else None
    nearest_resistance = min(resistances_above) if resistances_above else None

    return {
        "nearest_support": nearest_support,
        "nearest_resistance": nearest_resistance
    }
```

## 5. Rumus — Simple Moving Average & Momentum (tanpa numpy)

```python
def simple_moving_average(closes: list, period: int) -> float:
    """SMA manual: cukup pakai sum() dan len(), tidak butuh numpy"""
    if len(closes) < period:
        return None
    relevant = closes[-period:]
    return sum(relevant) / period


def momentum_pct(closes: list, period: int = 10) -> float:
    """Perubahan harga vs N hari lalu, dalam persen"""
    if len(closes) < period + 1:
        return 0.0
    past = closes[-(period + 1)]
    now = closes[-1]
    if past == 0:
        return 0.0
    return (now - past) / past * 100


def simple_rsi_like_score(closes: list, period: int = 14) -> float:
    """
    Versi sederhana dari RSI tanpa numpy/pandas.
    RSI asli: rata-rata gain vs rata-rata loss dalam N hari.
    """
    if len(closes) < period + 1:
        return 50.0  # netral kalau data belum cukup

    gains = []
    losses = []
    relevant = closes[-(period + 1):]
    for i in range(1, len(relevant)):
        change = relevant[i] - relevant[i - 1]
        if change > 0:
            gains.append(change)
        else:
            losses.append(abs(change))

    avg_gain = sum(gains) / period if gains else 0
    avg_loss = sum(losses) / period if losses else 0

    if avg_loss == 0:
        return 100.0
    rs = avg_gain / avg_loss
    rsi = 100 - (100 / (1 + rs))
    return round(rsi, 2)
```

## 6. Logika — Deteksi Pola Candlestick Dasar (tanpa library)

```python
def detect_candle_pattern(bar: dict) -> str:
    """
    Deteksi pola candle 1-bar paling umum & gampang dipahami.
    Semua hanya pakai operasi matematika dasar.
    """
    o, h, l, c = bar["open"], bar["high"], bar["low"], bar["close"]
    body = abs(c - o)
    full_range = h - l if (h - l) != 0 else 1e-9
    upper_shadow = h - max(o, c)
    lower_shadow = min(o, c) - l

    body_ratio = body / full_range

    # Doji: body sangat kecil dibanding range
    if body_ratio < 0.1:
        return "DOJI"

    # Hammer: body kecil di atas, lower shadow panjang (potensi reversal naik)
    if lower_shadow > body * 2 and upper_shadow < body * 0.5 and c > o:
        return "HAMMER"

    # Shooting Star: body kecil di bawah, upper shadow panjang (potensi reversal turun)
    if upper_shadow > body * 2 and lower_shadow < body * 0.5 and c < o:
        return "SHOOTING_STAR"

    # Bullish Marubozu: body besar, nyaris tanpa shadow, naik
    if body_ratio > 0.9 and c > o:
        return "BULLISH_MARUBOZU"

    # Bearish Marubozu
    if body_ratio > 0.9 and c < o:
        return "BEARISH_MARUBOZU"

    return "NORMAL"


def detect_engulfing(prev_bar: dict, curr_bar: dict) -> str:
    """Bullish/Bearish Engulfing — 2-bar pattern"""
    prev_body_top = max(prev_bar["open"], prev_bar["close"])
    prev_body_bottom = min(prev_bar["open"], prev_bar["close"])
    curr_body_top = max(curr_bar["open"], curr_bar["close"])
    curr_body_bottom = min(curr_bar["open"], curr_bar["close"])

    prev_bearish = prev_bar["close"] < prev_bar["open"]
    curr_bullish = curr_bar["close"] > curr_bar["open"]

    if prev_bearish and curr_bullish and curr_body_top > prev_body_top and curr_body_bottom < prev_body_bottom:
        return "BULLISH_ENGULFING"

    prev_bullish = prev_bar["close"] > prev_bar["open"]
    curr_bearish = curr_bar["close"] < curr_bar["open"]

    if prev_bullish and curr_bearish and curr_body_top > prev_body_top and curr_body_bottom < prev_body_bottom:
        return "BEARISH_ENGULFING"

    return "NONE"
```

## 7. Logika — Divergence Sederhana (harga vs momentum)

```python
def detect_simple_divergence(closes: list, lookback: int = 10) -> str:
    """
    Divergence sederhana: bandingkan arah harga vs arah RSI-like momentum
    dalam 'lookback' hari terakhir.

    Bearish divergence : harga bikin higher-high, tapi momentum melemah
                          (potensi pelemahan tren naik)
    Bullish divergence  : harga bikin lower-low, tapi momentum menguat
                          (potensi pelemahan tren turun)
    """
    if len(closes) < lookback + 15:
        return "INSUFFICIENT_DATA"

    price_now = closes[-1]
    price_before = closes[-lookback]

    rsi_now = simple_rsi_like_score(closes, period=14)
    rsi_before = simple_rsi_like_score(closes[:-lookback], period=14)

    price_higher = price_now > price_before
    rsi_lower = rsi_now < rsi_before

    price_lower = price_now < price_before
    rsi_higher = rsi_now > rsi_before

    if price_higher and rsi_lower and rsi_now > 60:
        return "BEARISH_DIVERGENCE"
    if price_lower and rsi_higher and rsi_now < 40:
        return "BULLISH_DIVERGENCE"
    return "NO_DIVERGENCE"
```

## 8. Kondisi Bullish / Bearish / Netral

```
Bullish Technical:
  - close > SMA20 > SMA50 (uptrend selaras)
  - momentum_pct(10) > 0
  - RSI-like antara 50-70 (momentum sehat, belum overbought ekstrem)
  - pola candle: HAMMER / BULLISH_ENGULFING / BULLISH_MARUBOZU di dekat support
  - tidak ada BEARISH_DIVERGENCE

Bearish Technical:
  - close < SMA20 < SMA50 (downtrend selaras)
  - momentum_pct(10) < 0
  - RSI-like < 50, makin rendah makin lemah
  - pola candle: SHOOTING_STAR / BEARISH_ENGULFING di dekat resistance
  - ada BEARISH_DIVERGENCE saat harga di resistance

Netral:
  - harga di antara SMA20 dan SMA50 tanpa arah jelas
  - RSI-like 45-55
  - tidak ada pola candle signifikan
```

## 9. Technical Score (0-100) — Python murni

```python
def compute_technical_score(ohlcv_history: list) -> dict:
    closes = [bar["close"] for bar in ohlcv_history]
    current_price = closes[-1]

    sma20 = simple_moving_average(closes, 20)
    sma50 = simple_moving_average(closes, 50)
    rsi_like = simple_rsi_like_score(closes, 14)
    mom10 = momentum_pct(closes, 10)

    # A. Trend alignment score (35%)
    if sma20 and sma50:
        if current_price > sma20 > sma50:
            raw_a = 85
        elif current_price > sma20 and sma20 < sma50:
            raw_a = 60
        elif current_price < sma20 < sma50:
            raw_a = 15
        else:
            raw_a = 40
    else:
        raw_a = 50
    score_a = raw_a * 0.35

    # B. Momentum score (30%)
    clamped_mom = max(min(mom10, 10), -10)
    raw_b = (clamped_mom + 10) / 20 * 100
    score_b = raw_b * 0.30

    # C. RSI-like score (20%) - sweet spot 50-70, ekstrem dipenalti
    if 50 <= rsi_like <= 70:
        raw_c = 80
    elif 70 < rsi_like <= 85:
        raw_c = 55          # overbought, mulai waspada
    elif rsi_like > 85:
        raw_c = 30           # sangat overbought
    elif 30 <= rsi_like < 50:
        raw_c = 40
    else:
        raw_c = 20            # oversold ekstrem, ambiguous (bisa rebound bisa lanjut turun)
    score_c = raw_c * 0.20

    # D. Pattern confirmation score (15%)
    last_bar = ohlcv_history[-1]
    pattern = detect_candle_pattern(last_bar)
    bullish_patterns = {"HAMMER", "BULLISH_ENGULFING", "BULLISH_MARUBOZU"}
    bearish_patterns = {"SHOOTING_STAR", "BEARISH_ENGULFING", "BEARISH_MARUBOZU"}
    if pattern in bullish_patterns:
        raw_d = 80
    elif pattern in bearish_patterns:
        raw_d = 20
    else:
        raw_d = 50
    score_d = raw_d * 0.15

    total = score_a + score_b + score_c + score_d

    return {
        "score": round(total, 2),
        "components": {
            "trend_score": round(raw_a, 2),
            "momentum_score": round(raw_b, 2),
            "rsi_like_score": round(raw_c, 2),
            "pattern_score": round(raw_d, 2)
        },
        "raw_values": {
            "sma20": sma20, "sma50": sma50,
            "rsi_like": rsi_like, "momentum_10d_pct": round(mom10, 2),
            "candle_pattern": pattern
        }
    }
```

## 10. Sistem Rating

| Score  | Rating      |
| ------ | ----------- |
| 81-100 | Strong Buy  |
| 61-80  | Buy         |
| 41-60  | Hold        |
| 21-40  | Sell        |
| 0-20   | Strong Sell |

## 11. Format Output

```python
{
    "stock_code": "BBCA",
    "date": "2025-08-15",
    "technical": {
        "current_price": 9875,
        "sma20": 9700.5,
        "sma50": 9450.2,
        "support_resistance": {
            "nearest_support": 9650,
            "nearest_resistance": 10000
        },
        "candle_pattern": "BULLISH_MARUBOZU",
        "engulfing": "NONE",
        "divergence": "NO_DIVERGENCE",
        "rsi_like": 64.2,
        "momentum_10d_pct": 3.8,
        "technical_score": 73.4,
        "rating": "Buy"
    }
}
```

## 12. Pseudocode Engine Lengkap

```python
class TechnicalEngine:

    def __init__(self, ohlcv_history: list):
        # urut tanggal menaik, minimal 60 bar untuk SMA50
        self.history = ohlcv_history

    def analyze(self, stock_code: str) -> dict:
        closes = [bar["close"] for bar in self.history]
        current_price = closes[-1]

        pivots = find_local_pivots(self.history, lookback=3)
        sr = nearest_support_resistance(current_price, pivots)

        pattern = detect_candle_pattern(self.history[-1])
        engulf = detect_engulfing(self.history[-2], self.history[-1]) if len(self.history) >= 2 else "NONE"
        divergence = detect_simple_divergence(closes)

        tech_score = compute_technical_score(self.history)

        return {
            "stock_code": stock_code,
            "date": self.history[-1]["date"],
            "technical": {
                "current_price": current_price,
                "sma20": tech_score["raw_values"]["sma20"],
                "sma50": tech_score["raw_values"]["sma50"],
                "support_resistance": sr,
                "candle_pattern": pattern,
                "engulfing": engulf,
                "divergence": divergence,
                "rsi_like": tech_score["raw_values"]["rsi_like"],
                "momentum_10d_pct": tech_score["raw_values"]["momentum_10d_pct"],
                "technical_score": tech_score["score"],
                "rating": self._get_rating(tech_score["score"])
            }
        }

    def _get_rating(self, score: float) -> str:
        if score >= 81: return "Strong Buy"
        elif score >= 61: return "Buy"
        elif score >= 41: return "Hold"
        elif score >= 21: return "Sell"
        else: return "Strong Sell"
```

## 13. Ide Pengembangan Lanjutan

- **Multi-timeframe confirmation**: cek technical score di daily + weekly,
  sinyal lebih kuat kalau dua timeframe selaras.
- **Pattern reliability tracker**: catat win-rate historis tiap pola candle
  per saham (HAMMER di BBCA akurasinya berapa % secara historis?).
- **Auto chart annotation**: tandai otomatis support/resistance + pola candle
  di chart untuk visualisasi.

---

# FITUR 13 — RISK & POSITION SIZING ENGINE

## 1. Tujuan Fitur

Menjawab pertanyaan paling sering diabaikan trader: **"berapa lot yang aman
saya beli?"** dan **"di harga berapa saya harus cut loss?"** — dihitung dari
volatilitas saham dan toleransi risiko portofolio, bukan dari feeling.

## 2. Data yang Dibutuhkan

| Field              | Tipe  | Keterangan                          |
| ------------------- | ----- | ------------------------------------ |
| account_balance     | float | Total modal trading                  |
| risk_per_trade_pct  | float | % modal yang rela hilang per trade (mis. 1-2%) |
| entry_price         | float | Harga rencana beli                   |
| ohlcv_history        | list  | Untuk hitung volatilitas (ATR-like)  |

## 3. Rumus — ATR Sederhana (Average True Range, tanpa numpy)

```python
def true_range(curr_bar: dict, prev_close: float) -> float:
    """True Range = range terbesar antara 3 kemungkinan"""
    high_low = curr_bar["high"] - curr_bar["low"]
    high_prev_close = abs(curr_bar["high"] - prev_close)
    low_prev_close = abs(curr_bar["low"] - prev_close)
    return max(high_low, high_prev_close, low_prev_close)


def simple_atr(ohlcv_history: list, period: int = 14) -> float:
    """ATR manual pakai rata-rata True Range, tanpa numpy/pandas"""
    if len(ohlcv_history) < period + 1:
        return 0.0

    trs = []
    relevant = ohlcv_history[-(period + 1):]
    for i in range(1, len(relevant)):
        tr = true_range(relevant[i], relevant[i - 1]["close"])
        trs.append(tr)

    return sum(trs) / len(trs)
```

## 4. Rumus — Stop Loss Berbasis Volatilitas

```python
def volatility_based_stop_loss(entry_price: float, atr: float,
                                multiplier: float = 2.0) -> dict:
    """
    Stop loss = entry - (ATR x multiplier).
    Logikanya: kasih ruang gerak sesuai volatilitas alami saham,
    bukan persentase tetap yang sama untuk semua saham.
    """
    stop_price = entry_price - (atr * multiplier)
    stop_distance_pct = (entry_price - stop_price) / entry_price * 100

    return {
        "stop_price": round(stop_price, 0),
        "stop_distance_pct": round(stop_distance_pct, 2),
        "atr_used": round(atr, 2)
    }
```

## 5. Rumus — Position Sizing

```python
def calculate_position_size(account_balance: float, risk_per_trade_pct: float,
                             entry_price: float, stop_price: float,
                             lot_size: int = 100) -> dict:
    """
    Position sizing standar manajemen risiko:
    risk_amount = berapa rupiah rela hilang kalau stop loss kena
    shares = risk_amount / (jarak entry ke stop loss per saham)
    """
    risk_amount = account_balance * (risk_per_trade_pct / 100)
    risk_per_share = entry_price - stop_price

    if risk_per_share <= 0:
        return {"error": "Stop price harus di bawah entry price"}

    raw_shares = risk_amount / risk_per_share
    # Bulatkan ke kelipatan lot_size (1 lot = 100 lembar di BEI)
    lots = int(raw_shares // lot_size)
    shares = lots * lot_size

    total_cost = shares * entry_price
    actual_risk = shares * risk_per_share

    return {
        "recommended_shares": shares,
        "recommended_lots": lots,
        "total_cost": round(total_cost, 0),
        "actual_risk_amount": round(actual_risk, 0),
        "risk_pct_of_account": round(actual_risk / account_balance * 100, 2) if account_balance else 0
    }
```

## 6. Rumus — Risk/Reward Ratio

```python
def risk_reward_ratio(entry_price: float, stop_price: float,
                       target_price: float) -> dict:
    risk = entry_price - stop_price
    reward = target_price - entry_price

    if risk <= 0:
        return {"error": "Risk harus positif"}

    rr_ratio = reward / risk

    return {
        "risk_per_share": round(risk, 0),
        "reward_per_share": round(reward, 0),
        "risk_reward_ratio": round(rr_ratio, 2),
        "is_favorable": rr_ratio >= 2.0   # aturan umum: minimal 1:2
    }
```

## 7. Rumus — Portfolio Heat (total risiko semua posisi terbuka)

```python
def portfolio_heat(open_positions: list, account_balance: float) -> dict:
    """
    open_positions: list of dict
    [{"stock_code": "BBCA", "risk_amount": 2_000_000}, ...]

    Portfolio heat = total risiko semua posisi terbuka / total modal.
    Aturan umum: jangan biarkan total heat > 6-8% dari modal sekaligus.
    """
    total_risk = sum(pos["risk_amount"] for pos in open_positions)
    heat_pct = (total_risk / account_balance * 100) if account_balance else 0

    if heat_pct > 8:
        status = "OVEREXPOSED"
    elif heat_pct > 5:
        status = "ELEVATED"
    else:
        status = "SAFE"

    return {
        "total_risk_amount": round(total_risk, 0),
        "heat_pct": round(heat_pct, 2),
        "status": status,
        "open_positions_count": len(open_positions)
    }
```

## 8. Logika Analisa — Kapan Sizing "Sehat"

```
Sehat:
  - risk_per_trade_pct antara 0.5% - 2% dari total modal
  - risk_reward_ratio >= 2.0 (potensi untung minimal 2x potensi rugi)
  - portfolio heat < 6%

Berisiko:
  - risk_per_trade_pct > 3%
  - risk_reward_ratio < 1.5
  - portfolio heat > 8% (terlalu banyak posisi terbuka sekaligus)
```

## 9. Format Output Gabungan

```python
{
    "stock_code": "BBCA",
    "plan": {
        "entry_price": 9875,
        "atr_14": 145.3,
        "stop_loss": {
            "stop_price": 9584,
            "stop_distance_pct": 2.95
        },
        "target_price": 10450,
        "risk_reward": {
            "risk_per_share": 291,
            "reward_per_share": 575,
            "risk_reward_ratio": 1.98,
            "is_favorable": False
        },
        "position_sizing": {
            "account_balance": 50_000_000,
            "risk_per_trade_pct": 1.5,
            "recommended_lots": 25,
            "recommended_shares": 2500,
            "total_cost": 24_687_500,
            "actual_risk_amount": 727_500,
            "risk_pct_of_account": 1.46
        }
    },
    "narrative": "Dengan ATR 14-hari sebesar 145, stop loss disarankan di 9584 "
                 "(jarak 2.95% dari entry). Risk/Reward 1.98 — sedikit di bawah "
                 "target minimal 2.0, pertimbangkan target price lebih tinggi atau "
                 "entry lebih rendah. Ukuran posisi 25 lot menjaga risiko di 1.46% modal."
}
```

## 10. Pseudocode Engine Lengkap

```python
class RiskEngine:

    def __init__(self, account_balance: float, risk_per_trade_pct: float = 1.5):
        self.account_balance = account_balance
        self.risk_per_trade_pct = risk_per_trade_pct

    def plan_trade(self, stock_code: str, ohlcv_history: list,
                    entry_price: float, target_price: float,
                    atr_multiplier: float = 2.0) -> dict:

        atr = simple_atr(ohlcv_history, period=14)
        sl = volatility_based_stop_loss(entry_price, atr, atr_multiplier)
        rr = risk_reward_ratio(entry_price, sl["stop_price"], target_price)
        sizing = calculate_position_size(
            self.account_balance, self.risk_per_trade_pct,
            entry_price, sl["stop_price"]
        )

        narrative = self._generate_narrative(entry_price, atr, sl, rr, sizing)

        return {
            "stock_code": stock_code,
            "plan": {
                "entry_price": entry_price,
                "atr_14": round(atr, 2),
                "stop_loss": sl,
                "target_price": target_price,
                "risk_reward": rr,
                "position_sizing": sizing
            },
            "narrative": narrative
        }

    def _generate_narrative(self, entry, atr, sl, rr, sizing) -> str:
        parts = [
            f"Dengan ATR 14-hari sebesar {atr:.0f}, stop loss disarankan di "
            f"{sl['stop_price']:.0f} (jarak {sl['stop_distance_pct']}% dari entry)."
        ]
        if rr.get("is_favorable"):
            parts.append(f"Risk/Reward {rr['risk_reward_ratio']} — rasio sehat (>= 2.0).")
        else:
            parts.append(
                f"Risk/Reward {rr.get('risk_reward_ratio', 0)} — di bawah target "
                f"minimal 2.0, pertimbangkan ulang target atau entry."
            )
        if "recommended_lots" in sizing:
            parts.append(
                f"Ukuran posisi {sizing['recommended_lots']} lot menjaga risiko "
                f"di {sizing['risk_pct_of_account']}% dari modal."
            )
        return " ".join(parts)
```

## 11. Ide Pengembangan Lanjutan

- **Trailing stop dinamis**: stop loss naik mengikuti ATR seiring harga naik.
- **Correlation check**: cegah portfolio heat tinggi karena beli banyak saham
  yang sebenarnya bergerak searah (mis. semua saham bank).
- **Kelly Criterion (versi sederhana)**: sizing berbasis win-rate historis
  strategi, sebagai pelengkap (bukan pengganti) risk_per_trade_pct.

---

# FITUR 14 — ZEROSCORE v2 (Master Score, Versi Diperluas)

## 1. Tujuan Fitur

Menggabungkan semua skor (Bandarmology, Technical, Seasonality, Risk Quality)
jadi satu angka akhir 0-100, tapi tetap **modular** — tiap komponen bisa
dimatikan/dinyalakan, dan bobot bisa diubah sesuai gaya analisa kamu sendiri.

## 2. Komponen dan Bobot Default

| Komponen             | Bobot Default | Sumber             |
| --------------------- | -------------- | ------------------- |
| Bandarmology Score     | 40%            | Fitur 7              |
| Technical Score        | 30%            | Fitur 12 (baru)      |
| Foreign Flow Score     | 15%            | Fitur 4 (sudah ada, tapi dipisah dari Bandarmology agar tidak double-count berlebihan) |
| Seasonality Score      | 10%            | Fitur 11 (baru)      |
| Risk Quality Score     | 5%             | Fitur 13 (baru)      |

> Catatan: Bandarmology Score (Fitur 7) sudah memasukkan Foreign Flow di
> dalamnya dengan bobot 25%. Di ZeroScore v2 ini, Foreign Flow diberi sedikit
> bobot tambahan terpisah karena sifatnya sebagai leading indicator — tapi
> totalnya tetap dijaga supaya tidak overweight. Kamu bebas set bobot foreign
> jadi 0% kalau menganggap ini redundant.

## 3. Rumus Risk Quality Score (komponen baru, sederhana)

```python
def compute_risk_quality_score(risk_reward_ratio_value: float,
                                portfolio_heat_pct: float) -> float:
    """
    Skor ini BUKAN soal arah saham (bullish/bearish), tapi soal
    apakah trading plan-nya 'sehat' secara manajemen risiko.
    """
    # A. RR Score (70%)
    if risk_reward_ratio_value >= 3:
        raw_a = 100
    elif risk_reward_ratio_value >= 2:
        raw_a = 80
    elif risk_reward_ratio_value >= 1.5:
        raw_a = 55
    elif risk_reward_ratio_value >= 1:
        raw_a = 30
    else:
        raw_a = 10
    score_a = raw_a * 0.70

    # B. Portfolio heat score (30%) - makin rendah heat makin baik
    if portfolio_heat_pct <= 4:
        raw_b = 100
    elif portfolio_heat_pct <= 6:
        raw_b = 70
    elif portfolio_heat_pct <= 8:
        raw_b = 40
    else:
        raw_b = 10
    score_b = raw_b * 0.30

    return round(score_a + score_b, 2)
```

## 4. Rumus ZeroScore v2 — Modular

```python
def compute_zeroscore_v2(component_scores: dict,
                          weights: dict = None) -> dict:
    """
    component_scores contoh:
    {
        "bandarmology": 75.6,
        "technical": 73.4,
        "foreign_flow": 76.2,
        "seasonality": 56.4,   # boleh None kalau confidence LOW / data kurang
        "risk_quality": 84.0   # boleh None kalau belum ada trade plan
    }

    weights contoh (default kalau None):
    {
        "bandarmology": 0.40,
        "technical": 0.30,
        "foreign_flow": 0.15,
        "seasonality": 0.10,
        "risk_quality": 0.05
    }
    """
    default_weights = {
        "bandarmology": 0.40,
        "technical": 0.30,
        "foreign_flow": 0.15,
        "seasonality": 0.10,
        "risk_quality": 0.05
    }
    weights = weights or default_weights

    # Hanya hitung komponen yang datanya tersedia (None = skip),
    # lalu re-normalize bobot supaya totalnya tetap 100%
    available = {k: v for k, v in component_scores.items() if v is not None}
    if not available:
        return {"score": None, "note": "Tidak ada komponen data tersedia"}

    used_weights = {k: weights.get(k, 0) for k in available}
    total_weight = sum(used_weights.values())

    if total_weight == 0:
        return {"score": None, "note": "Total bobot komponen tersedia = 0"}

    weighted_sum = sum(available[k] * used_weights[k] for k in available)
    final_score = weighted_sum / total_weight  # re-normalisasi

    return {
        "score": round(final_score, 2),
        "components_used": list(available.keys()),
        "effective_weights": {k: round(used_weights[k] / total_weight, 3) for k in available}
    }
```

## 5. Sistem Rating (sama seperti Fitur 7, dipertahankan agar konsisten)

```python
def get_zeroscore_rating(score: float) -> dict:
    if score >= 81:
        return {"rating": "Sangat Bullish", "action": "Strong Buy", "emoji": "🚀"}
    elif score >= 61:
        return {"rating": "Bullish", "action": "Buy", "emoji": "📈"}
    elif score >= 41:
        return {"rating": "Netral", "action": "Hold / Watch", "emoji": "⚖️"}
    elif score >= 21:
        return {"rating": "Bearish", "action": "Sell", "emoji": "📉"}
    else:
        return {"rating": "Sangat Bearish", "action": "Strong Sell", "emoji": "💀"}
```

## 6. Format Output Gabungan

```python
{
    "stock_code": "BBCA",
    "date": "2025-08-15",
    "zeroscore_v2": {
        "components": {
            "bandarmology": 75.6,
            "technical": 73.4,
            "foreign_flow": 76.2,
            "seasonality": 56.4,
            "risk_quality": 84.0
        },
        "effective_weights": {
            "bandarmology": 0.40,
            "technical": 0.30,
            "foreign_flow": 0.15,
            "seasonality": 0.10,
            "risk_quality": 0.05
        },
        "final_score": 73.8,
        "rating": {
            "label": "Bullish",
            "action": "Buy",
            "emoji": "📈"
        },
        "narrative": "ZeroScore BBCA 73.8/100 — Rating BULLISH. Didukung "
                     "Bandarmology kuat (75.6) dan Technical solid (73.4) dengan "
                     "tren naik selaras SMA20>SMA50. Foreign flow turut positif "
                     "(76.2). Seasonality bulan ini netral-positif (56.4, confidence "
                     "MEDIUM — jangan jadi sinyal utama). Risk plan sehat (84.0) "
                     "dengan RR ratio mendukung entry saat ini."
    }
}
```

## 7. Pseudocode Engine Orkestrator (menggabungkan semua fitur)

```python
class ZeroScoreV2Engine:
    """
    Orkestrator yang memanggil semua engine lain (Fitur 7, 11, 12, 13)
    dan menghasilkan satu skor akhir.
    """

    def __init__(self, bandarmology_engine, technical_engine,
                 seasonality_engine, risk_engine, weights: dict = None):
        self.bandarmology = bandarmology_engine
        self.technical = technical_engine
        self.seasonality = seasonality_engine
        self.risk = risk_engine
        self.weights = weights

    def analyze(self, stock_code: str, date: str,
                ohlcv_history: list,
                trade_plan: dict = None) -> dict:

        # 1. Bandarmology (sudah termasuk foreign flow di dalamnya)
        bandar_result = self.bandarmology.analyze(stock_code, date, ohlcv_history)
        bandar_score = bandar_result["bandarmology"]["bandar_score"]
        foreign_score = bandar_result["bandarmology"]["components"]["foreign_score"]

        # 2. Technical
        tech_result = self.technical.analyze(stock_code)
        tech_score = tech_result["technical"]["technical_score"]

        # 3. Seasonality (boleh None kalau confidence LOW)
        season_result = self.seasonality.analyze(stock_code, date)
        season_stats = season_result.get("current_month_stats", {})
        season_score = season_stats.get("score") if season_stats.get("confidence") != "LOW" else None

        # 4. Risk quality (hanya ada kalau user sudah submit trade plan)
        risk_score = None
        if trade_plan:
            plan_result = self.risk.plan_trade(
                stock_code, ohlcv_history,
                trade_plan["entry_price"], trade_plan["target_price"]
            )
            rr = plan_result["plan"]["risk_reward"]["risk_reward_ratio"]
            heat = trade_plan.get("portfolio_heat_pct", 0)
            risk_score = compute_risk_quality_score(rr, heat)

        components = {
            "bandarmology": bandar_score,
            "technical": tech_score,
            "foreign_flow": foreign_score,
            "seasonality": season_score,
            "risk_quality": risk_score
        }

        result = compute_zeroscore_v2(components, self.weights)
        rating = get_zeroscore_rating(result["score"]) if result["score"] is not None else None

        return {
            "stock_code": stock_code,
            "date": date,
            "zeroscore_v2": {
                "components": components,
                "effective_weights": result.get("effective_weights"),
                "final_score": result["score"],
                "rating": rating
            }
        }
```

## 8. Ide Pengembangan Lanjutan

- **Custom weight profile per gaya trading**: profile "Swing Trader" (bobot
  technical lebih besar) vs "Bandarmology Hunter" (bobot bandarmology lebih
  besar) vs "Conservative" (bobot risk_quality lebih besar).
- **Score history & backtesting**: simpan ZeroScore harian per saham, lalu
  cek apakah skor tinggi historically diikuti return positif N hari ke depan
  (ini cara *valid* untuk memvalidasi apakah skoringnya beneran berguna,
  bukan asumsi).
- **Score divergence alert**: notifikasi kalau Bandarmology tinggi tapi
  Technical rendah (atau sebaliknya) — sinyal konflik yang perlu investigasi
  manual sebelum entry.

---

# RINGKASAN PENAMBAHAN

| Fitur | Nama | Menjawab Pertanyaan |
| ----- | ---- | --------------------- |
| 11 | Seasonality & Calendar Effect | "Apakah saham ini punya pola musiman yang terbukti secara historis?" |
| 12 | Technical Pattern Recognition | "Di mana support/resistance, dan apa kata candle/momentum?" |
| 13 | Risk & Position Sizing | "Berapa lot yang aman, dan di mana stop loss saya?" |
| 14 | ZeroScore v2 | "Kalau digabung semua, skor akhirnya berapa — dan apakah trading plan-nya sehat?" |

Semua engine di atas:
- Pakai **Python murni** — `list`, `dict`, `sum()`, `max()`, `min()`, modul
  bawaan `datetime`. Tidak ada pandas/numpy supaya gampang ditempel ke
  `broker_summary.py` atau modul manapun tanpa dependency tambahan.
- Bisa dipanggil independen (analisa technical saja) atau digabung lewat
  `ZeroScoreV2Engine`.
- Selalu transparan soal **confidence/keterbatasan data** — terutama di
  Seasonality, supaya tidak overconfident pada sample kecil.
