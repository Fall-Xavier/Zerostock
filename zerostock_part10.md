# ZeroStock Platform — Part 10 (FINAL): Ringkasan Lengkap & Fitur Penutup

> Part terakhir dari seri dokumentasi konsep ZeroStock.
> Bagian A: Ringkasan & kesimpulan seluruh Part 1-9 (42 fitur)
> Bagian B: Fitur 43-46 — BSJP Market-Wide Pipeline, Trading Mode Profiles,
>           IHSG Outlook Framework, Trading Plan Generator
> Python murni — `list`, `dict`, `sum()`, `max()`, `min()`, modul bawaan
> `datetime` — konsisten dengan part4-part9, tanpa pandas/numpy.

---

# BAGIAN A — RINGKASAN & KESIMPULAN PART 1-9

## A.1 Peta Lengkap 42 Fitur

| # | Fitur | Part | Kategori |
|---|-------|------|----------|
| 1 | Broker Summary | 1 | Microstructure |
| 2 | Broker Activity | 1 | Microstructure |
| 3 | Broker Accumulation | 1 | Microstructure |
| 4 | Foreign Flow | 2 | Microstructure |
| 5 | Orderbook Analysis | 2 | Microstructure |
| 6 | Smart Money Analysis | 2 | Microstructure |
| 7 | Bandarmology Score | 2 | Master Score (komponen) |
| 8 | ChartBit-style Technical Analysis | 3 | Technical |
| 9 | Stock Screener | 3 | Screening |
| 10 | Liquidity Score | 3 | Risk/Quality |
| 11 | Seasonality & Calendar Effect | 4 | Statistik Historis |
| 12 | Technical Pattern Recognition | 4 | Technical |
| 13 | Risk & Position Sizing Engine | 4 | Risk Management |
| 14 | ZeroScore v2 | 4 | Master Score |
| 15 | BSJP Screener | 5 | Taktikal Intraday |
| 16 | War Opening Screener | 5 | Taktikal Intraday |
| 17 | Top Movers | 5 | Screening |
| 18 | Broker Daily Activity & Distribution | 5 | Microstructure |
| 19 | Foreign-Domestic Activity | 5 | Microstructure |
| 20 | Fundamental Score | 6 | Fundamental |
| 21 | Market Regime / IHSG Context | 6 | Konteks Pasar |
| 22 | Sector Rotation Tracker | 6 | Konteks Pasar |
| 23 | Portfolio Correlation Check | 6 | Risk Management |
| 24 | Backtesting Engine | 6 | Validasi |
| 25 | Corporate Action Calendar | 6 | Konteks/Caution |
| 26 | Alert & Watchlist Monitoring | 6 | Operasional |
| 27 | Data Source Health & Confidence | 7 | Validasi/Infrastruktur |
| 28 | BEI Microstructure Rules (ARA/ARB/UMA/Pasar Nego) | 7 | Konteks/Caution |
| 29 | Broker Classification Accuracy | 7 | Microstructure (koreksi) |
| 30 | Macro-Economic Context | 7 | Konteks Pasar |
| 31 | News & Disclosure Sentiment | 7 | Sentimen |
| 32 | Sector & Index Relative Strength | 7 | Konteks Pasar |
| 33 | Data Quality & Anomaly Detection | 7 | Validasi/Infrastruktur |
| 34 | Alert Delivery & API Layer | 7 | Operasional/Infrastruktur |
| 35 | Compliance & Audit Trail | 7 | Infrastruktur/Etika |
| 36 | Dividend & Total Return Tracking | 8 | Fundamental |
| 37 | Margin Trading & Short Selling Context | 8 | Risk/Leverage |
| 38 | Insider/Pihak Terafiliasi Disclosure | 8 | Sentimen/Sinyal Khusus |
| 39 | Cross-Stock Bandar Footprint | 8 | Microstructure (cross-saham) |
| 40 | Portfolio Construction & Execution Planning | 9 | Portofolio |
| 41 | Orchestration Pipeline | 9 | Infrastruktur |
| 42 | Accumulation-Distribution Phase Detector | 9 | Master Score (turunan) |

## A.2 Evolusi Arsitektur dari Part 1 ke Part 9

```
PART 1-3 (fondasi awal)
  Fokus: microstructure dasar (broker, foreign, orderbook) + technical
  dasar + screener pertama. Pakai pandas/numpy.

PART 4-5 (perluasan analisa & taktikal)
  Fokus: seasonality, technical pattern lengkap, risk management,
  ZeroScore v1->v2, lalu strategi taktikal intraday (BSJP, war opening)
  dan replikasi fitur populer Stockbit (top movers, broker distribution,
  foreign-domestic). Beralih ke Python murni.

PART 6 (konteks & validitas)
  Fokus: mengisi gap besar — fundamental, kondisi market secara umum
  (regime, sector rotation), portfolio-level risk (correlation), dan
  yang paling penting: BACKTESTING (apakah skor ini beneran prediktif?)
  serta corporate action calendar & alert system.

PART 7 (kekhususan BEI & infrastruktur)
  Fokus: hal-hal yang SANGAT spesifik pasar Indonesia (ARA/ARB/UMA/pasar
  nego), koreksi kelemahan struktural (broker classification yang sering
  salah), macro context, sentimen berita, relative strength vs index,
  data quality, infrastruktur API/alert, dan compliance/audit trail.

PART 8 (dimensi yang masih kosong)
  Fokus: total return (dividen, bukan cuma capital gain), leverage pasar
  (margin/short — sebelumnya sama sekali tidak ada), kategori sinyal
  insider yang terpisah dari berita umum, dan koneksi cross-saham
  (broker yang sama bergerak di banyak saham sekaligus).

PART 9 (penutup siklus: dari sinyal ke aksi)
  Fokus: portfolio construction (alokasi & staging entry nyata),
  orchestration pipeline (urutan eksekusi konkret 39 fitur sebelumnya
  + confidence propagation antar lapisan), dan phase detection
  (akumulasi-markup-distribusi-markdown) dengan batasan jujur soal apa
  yang BISA dan TIDAK BISA diklaim dari data semacam ini.
```

## A.3 Lima Prinsip yang Konsisten Dipegang dari Part 1 sampai 9

1. **Transparansi confidence/keterbatasan data.** Hampir semua fitur
   (Seasonality di 11, Data Health di 27, Phase Detector di 42)
   menyertakan label confidence (LOW/MEDIUM/HIGH) dan secara eksplisit
   menolak terlihat lebih yakin dari yang seharusnya berdasarkan jumlah
   sampel/kualitas data yang tersedia.
2. **Skor 0-100 dengan komponen yang bisa diaudit.** Setiap master score
   (Bandarmology, Technical, Fundamental, ZeroScore, dst) dipecah jadi
   komponen dengan bobot eksplisit — bukan angka hitam-kotak.
3. **Narasi otomatis berbasis fakta, bukan template generik.** Tiap
   engine menghasilkan kalimat penjelasan yang menyebut angka konkret,
   bukan sekadar "Bullish" tanpa alasan.
4. **Validasi sebelum kesimpulan.** Backtesting (24), Data Quality (33),
   dan Data Source Health (27) ada justru supaya skor-skor lain tidak
   diterima mentah-mentah — selalu ada lapisan "apakah ini bisa
   dipercaya" sebelum lapisan "apa kesimpulannya".
5. **Jujur soal batas metode.** Astrologi ditolak sejak part 4 (diganti
   seasonality berbasis statistik), prediksi tanggal distribusi ditolak
   di part 9, dan prinsip yang sama akan terus dipegang di part 10 ini
   untuk outlook IHSG — proyeksi berbasis skenario, bukan ramalan pasti.

## A.4 Kesimpulan: Apa yang ZeroStock Sudah Bisa Jawab

Dengan 42 fitur di part 1-9, ZeroStock sudah bisa menjawab hampir semua
pertanyaan inti analisa saham BEI:

- **"Saham ini kondisinya bagaimana?"** → Bandarmology, Technical,
  Fundamental, ZeroScore v2 (Fitur 1-3, 7, 8, 12, 20, 14)
- **"Apakah ini sinyal asli atau cuma kebetulan/manipulasi?"** → Data
  Quality, Broker Classification Accuracy, Microstructure Rules (27,
  28, 29, 33)
- **"Apa konteks yang lebih besar di balik saham ini?"** → Market
  Regime, Sector Rotation, Macro Context, Relative Strength (21, 22,
  30, 32)
- **"Bagaimana cara mengeksekusinya dengan aman?"** → Risk Engine,
  Portfolio Correlation, Portfolio Construction (13, 23, 40)
- **"Apakah semua ini beneran berguna?"** → Backtesting Engine (24)
- **"Bagaimana saya tahu kapan harus cek tanpa pantau manual terus?"**
  → Alert & Watchlist, Alert Delivery (26, 34)

Yang BELUM dijawab sebelum part 10 ini: bagaimana menjalankan screening
**market-wide** (semua saham sekaligus, bukan satu-satu) khusus untuk
BSJP secara end-to-end; bagaimana mengkonfigurasi seluruh sistem untuk
**gaya trading berbeda** (swing vs day trade); dan **outlook IHSG**
secara keseluruhan ke depan. Tiga hal inilah yang diisi di Bagian B di
bawah, ditambah satu fitur penutup.

---

# BAGIAN B — FITUR PENUTUP (43-46)

## CATATAN METODOLOGI — Outlook IHSG (Fitur 45)

Sama seperti Fitur 42, fitur outlook IHSG TIDAK memprediksi "IHSG akan
ke level X pada tanggal Y". Yang disajikan adalah **kerangka skenario**:
berdasarkan kombinasi sinyal teknikal dan flow saat ini, skenario mana
(bullish/bearish/sideways continuation) yang paling konsisten dengan
data historis — lengkap dengan level yang membatalkan skenario tersebut
(invalidation level), bukan target waktu.

---

# FITUR 43 — BSJP MARKET-WIDE SCREENING PIPELINE

## 1. Tujuan Fitur

Menyempurnakan Fitur 15 (BSJP Screener) menjadi pipeline end-to-end yang
benar-benar market-wide: mulai dari menarik data SEMUA saham hari itu
sekaligus (OHLC, volume, frequency, foreign flow — gaya snapshot IDX),
melewati filter awal, lalu scoring dua tahap (Technical lengkap dengan
MACD + RSI + candle, dan Bandarmology dengan broker classification) —
bukan lagi BSJP score tunggal seperti di Fitur 15, tapi pipeline
berlapis yang transparan di tiap tahap.

## 2. Tahap 1 — Market-Wide Data Ingestion

```python
def fetch_market_snapshot_structure() -> dict:
    """
    Struktur satu baris snapshot harian SEMUA saham (gaya data IDX
    end-of-day). Ini BUKAN fungsi fetch API sungguhan — definisikan
    struktur field yang dibutuhkan, implementasi pemanggilan API
    (Stockbit/IDX) menyesuaikan dengan kontrak data masing-masing.
    """
    return {
        "stock_code": "BBCA",
        "open": 9800, "high": 9900, "low": 9750, "close": 9875,
        "prev_close": 9800,
        "volume": 12_500_000, "value": 123_500_000_000, "frequency": 4200,
        "foreign_buy_value": 45_000_000_000, "foreign_sell_value": 12_000_000_000,
        "avg_daily_value_20d": 110_000_000_000
    }


def ingest_market_snapshot(raw_rows: list) -> list:
    """
    Tahap pembersihan dasar setelah data masuk — dijalankan SEBELUM
    filter, terhubung ke Fitur 33 (Data Quality) untuk membuang baris
    yang jelas korup (harga 0, volume negatif, dsb).
    """
    clean_rows = []
    for row in raw_rows:
        if row.get("close", 0) <= 0 or row.get("volume", 0) < 0:
            continue  # data invalid, skip (idealnya dicatat ke Fitur 33)
        clean_rows.append(row)
    return clean_rows
```

## 3. Tahap 2 — Filter Awal (Mempersempit Universe Sebelum Scoring Berat)

```python
def apply_initial_filters(market_snapshot: list,
                           min_avg_value: float = 2e9,
                           min_price: float = 50,
                           max_price: float = 20000,
                           exclude_suspended: list = None) -> list:
    """
    Filter awal murah secara komputasi, dijalankan sebelum scoring
    teknikal/bandarmologi yang lebih berat — mempersempit ratusan saham
    jadi puluhan kandidat layak screening lanjutan.
    """
    exclude_suspended = exclude_suspended or []

    filtered = []
    for row in market_snapshot:
        if row["stock_code"] in exclude_suspended:
            continue
        if row["avg_daily_value_20d"] < min_avg_value:
            continue
        if not (min_price <= row["close"] <= max_price):
            continue
        filtered.append(row)

    return filtered
```

## 4. Tahap 3 — Scoring Teknikal Lengkap (Menambahkan MACD yang Belum Ada di Part Sebelumnya)

```python
def compute_ema(values: list, period: int) -> list:
    """
    Exponential Moving Average manual tanpa numpy/pandas.
    Mengembalikan list EMA sepanjang values (None di awal sebelum cukup data).
    """
    if len(values) < period:
        return [None] * len(values)

    ema_values = [None] * (period - 1)
    multiplier = 2 / (period + 1)

    sma_seed = sum(values[:period]) / period
    ema_values.append(sma_seed)

    for i in range(period, len(values)):
        ema_today = (values[i] - ema_values[-1]) * multiplier + ema_values[-1]
        ema_values.append(ema_today)

    return ema_values


def compute_macd(closes: list, fast_period: int = 12, slow_period: int = 26,
                  signal_period: int = 9) -> dict:
    """
    MACD manual tanpa library teknikal. MACD line = EMA cepat - EMA lambat.
    Signal line = EMA dari MACD line. Histogram = MACD line - Signal line.
    """
    if len(closes) < slow_period + signal_period:
        return {"macd_line": None, "signal_line": None, "histogram": None, "status": "INSUFFICIENT_DATA"}

    ema_fast = compute_ema(closes, fast_period)
    ema_slow = compute_ema(closes, slow_period)

    macd_line_series = []
    for i in range(len(closes)):
        if ema_fast[i] is None or ema_slow[i] is None:
            macd_line_series.append(None)
        else:
            macd_line_series.append(ema_fast[i] - ema_slow[i])

    valid_macd = [v for v in macd_line_series if v is not None]
    signal_series = compute_ema(valid_macd, signal_period)

    current_macd = valid_macd[-1] if valid_macd else None
    current_signal = signal_series[-1] if signal_series else None
    histogram = (current_macd - current_signal) if (current_macd is not None and current_signal is not None) else None

    # Cek crossover dari histogram 2 titik terakhir
    prev_macd = valid_macd[-2] if len(valid_macd) >= 2 else None
    prev_signal = signal_series[-2] if len(signal_series) >= 2 else None

    crossover = "NONE"
    if all(v is not None for v in [current_macd, current_signal, prev_macd, prev_signal]):
        if prev_macd <= prev_signal and current_macd > current_signal:
            crossover = "BULLISH_CROSSOVER"
        elif prev_macd >= prev_signal and current_macd < current_signal:
            crossover = "BEARISH_CROSSOVER"

    return {
        "macd_line": round(current_macd, 2) if current_macd is not None else None,
        "signal_line": round(current_signal, 2) if current_signal is not None else None,
        "histogram": round(histogram, 2) if histogram is not None else None,
        "crossover": crossover,
        "status": "OK"
    }
```

## 5. Tahap 3 (lanjutan) — Gabungan Technical Score untuk BSJP (reuse Fitur 12 + MACD baru)

```python
def compute_bsjp_technical_score(ohlcv_history: list) -> dict:
    """
    Menggabungkan komponen Technical dari Fitur 12 (trend, momentum,
    RSI-like, candle pattern) DENGAN MACD yang baru ditambahkan di sini,
    khusus untuk konteks BSJP (closing strength + momentum lanjutan).
    """
    closes = [bar["close"] for bar in ohlcv_history]

    macd_result = compute_macd(closes)
    # Fungsi-fungsi berikut diasumsikan sudah tersedia dari Fitur 12 part4:
    # simple_rsi_like_score(), detect_candle_pattern(), momentum_pct()
    rsi_like = simple_rsi_like_score(closes, period=14)
    momentum = momentum_pct(closes, period=5)  # 5 hari, lebih relevan untuk BSJP
    last_pattern = detect_candle_pattern(ohlcv_history[-1])

    # MACD Score (35%)
    if macd_result["status"] != "OK":
        raw_macd = 50
    elif macd_result["crossover"] == "BULLISH_CROSSOVER":
        raw_macd = 90
    elif macd_result["histogram"] is not None and macd_result["histogram"] > 0:
        raw_macd = 70
    elif macd_result["crossover"] == "BEARISH_CROSSOVER":
        raw_macd = 15
    else:
        raw_macd = 35
    score_macd = raw_macd * 0.35

    # RSI-like Score (25%) - untuk BSJP, sweet spot sedikit lebih tinggi (momentum kuat)
    if 55 <= rsi_like <= 75:
        raw_rsi = 85
    elif 75 < rsi_like <= 85:
        raw_rsi = 55
    elif rsi_like > 85:
        raw_rsi = 30
    else:
        raw_rsi = 35
    score_rsi = raw_rsi * 0.25

    # Momentum 5 hari Score (25%)
    clamped_mom = max(min(momentum, 8), -8)
    raw_mom = (clamped_mom + 8) / 16 * 100
    score_mom = raw_mom * 0.25

    # Candle Pattern Score (15%)
    bullish_patterns = {"HAMMER", "BULLISH_ENGULFING", "BULLISH_MARUBOZU"}
    raw_candle = 80 if last_pattern in bullish_patterns else 35
    score_candle = raw_candle * 0.15

    total = score_macd + score_rsi + score_mom + score_candle

    return {
        "score": round(total, 2),
        "macd": macd_result,
        "rsi_like": rsi_like,
        "momentum_5d_pct": round(momentum, 2),
        "candle_pattern": last_pattern
    }
```

## 6. Tahap 4 — Scoring Bandarmologi dengan Broker Classification (reuse Fitur 1, 3, 29)

```python
def compute_bsjp_bandarmology_score(broker_rows: list, broker_classification_map: dict,
                                     accumulation_trend_20d: str) -> dict:
    """
    broker_classification_map: hasil dari Fitur 29 (Broker Classification
    Accuracy) — memastikan broker asing/lokal/institusional sudah
    diklasifikasi dengan benar SEBELUM dihitung skornya, bukan asumsi
    nama broker mentah.
    """
    significant_buyers = []
    significant_sellers = []

    for row in broker_rows:
        net = row["buy_value"] - row["sell_value"]
        classification = broker_classification_map.get(row["broker_code"], "UNKNOWN")
        if net > 0 and abs(net) >= 1e9:
            significant_buyers.append({"broker_code": row["broker_code"], "net": net, "classification": classification})
        elif net < 0 and abs(net) >= 1e9:
            significant_sellers.append({"broker_code": row["broker_code"], "net": net, "classification": classification})

    significant_buyers.sort(key=lambda x: x["net"], reverse=True)
    significant_sellers.sort(key=lambda x: x["net"])

    total_buy = sum(b["net"] for b in significant_buyers)
    total_sell = sum(abs(s["net"]) for s in significant_sellers)
    net_significant = total_buy - total_sell

    # A. Net flow direction score (50%)
    if total_buy + total_sell > 0:
        raw_a = 50 + min(max(net_significant / (total_buy + total_sell) * 50, -50), 50)
    else:
        raw_a = 50
    score_a = raw_a * 0.50

    # B. Accumulation trend score (30%) - dari Fitur 3
    trend_map = {"ACCELERATING": 90, "STABLE_POSITIVE": 70, "FLAT": 50, "DECELERATING": 30, "REVERSING": 15}
    raw_b = trend_map.get(accumulation_trend_20d, 50)
    score_b = raw_b * 0.30

    # C. Quality of buyers score (20%) - broker institusional/asing terklasifikasi
    # benar lebih meyakinkan dibanding broker tak teridentifikasi
    institutional_buyers = sum(1 for b in significant_buyers if b["classification"] in ("FOREIGN_INSTITUTIONAL", "LOCAL_INSTITUTIONAL"))
    raw_c = min(institutional_buyers * 25, 100)
    score_c = raw_c * 0.20

    total = score_a + score_b + score_c

    return {
        "score": round(total, 2),
        "top_accumulator_brokers": significant_buyers[:5],
        "top_distributor_brokers": significant_sellers[:5]
    }
```

## 7. Tahap 5 — Gabungan Akhir & Ranking (Reuse Late Session Strength dari Fitur 15)

```python
def compute_bsjp_combined_score(late_session_strength: dict, technical_result: dict,
                                 bandarmology_result: dict) -> dict:
    """
    Skor akhir BSJP versi market-wide pipeline: gabungan 3 dimensi —
    closing strength (dari Fitur 15), technical lengkap (MACD/RSI/candle),
    dan bandarmology dengan broker classification.
    """
    if not late_session_strength.get("valid"):
        return {"score": None, "reason": "Data intraday tidak cukup"}

    # Reuse compute_bsjp_score dari Fitur 15 untuk closing strength saja (60% bobotnya di sini)
    closing_component = (
        max(min(late_session_strength["late_change_pct"], 5), -5) + 5
    ) / 10 * 100 * 0.3 + late_session_strength["close_position_pct"] * 0.3

    technical_component = technical_result["score"] * 0.4 / 100 * 100  # sudah skala 0-100
    bandarmology_component = bandarmology_result["score"] * 0.3 / 100 * 100

    # Normalisasi bobot: closing 30%, technical 40%, bandarmology 30%
    final_score = (closing_component * 0.5) + (technical_component * 0.4) + (bandarmology_component) * 0.3 / 0.3 * 0.3
    # Disederhanakan langsung dengan bobot eksplisit:
    final_score = (
        ((max(min(late_session_strength["late_change_pct"], 5), -5) + 5) / 10 * 100) * 0.15 +
        late_session_strength["close_position_pct"] * 0.15 +
        technical_result["score"] * 0.40 +
        bandarmology_result["score"] * 0.30
    )

    return {"score": round(final_score, 2)}
```

## 8. Pseudocode Pipeline Lengkap (End-to-End)

```python
class BSJPMarketWidePipeline:

    def __init__(self, min_avg_value: float = 2e9):
        self.min_avg_value = min_avg_value

    def run(self, raw_market_snapshot: list, exclude_suspended: list,
            ohlcv_history_map: dict, intraday_bars_map: dict,
            broker_rows_map: dict, broker_classification_map: dict,
            accumulation_trend_map: dict, top_n: int = 20) -> list:
        """
        Pipeline lengkap dari data mentah market-wide sampai ranking akhir.

        ohlcv_history_map: {stock_code: [ohlcv bar harian, ...]}
        intraday_bars_map: {stock_code: [bar intraday hari ini, ...]}
        broker_rows_map: {stock_code: [broker_row hari ini, ...]}
        accumulation_trend_map: {stock_code: "ACCELERATING"/"STABLE_POSITIVE"/dst}
        """
        # Tahap 1: ingest & bersihkan
        clean_snapshot = ingest_market_snapshot(raw_market_snapshot)

        # Tahap 2: filter awal
        candidates = apply_initial_filters(
            clean_snapshot, self.min_avg_value, exclude_suspended=exclude_suspended
        )

        results = []
        for row in candidates:
            stock_code = row["stock_code"]
            if stock_code not in ohlcv_history_map or stock_code not in intraday_bars_map:
                continue

            # Tahap 3: scoring teknikal (reuse compute_late_session_strength dari Fitur 15)
            late_strength = compute_late_session_strength(intraday_bars_map[stock_code], minutes=45)
            technical_result = compute_bsjp_technical_score(ohlcv_history_map[stock_code])

            # Tahap 4: scoring bandarmologi
            bandarmology_result = compute_bsjp_bandarmology_score(
                broker_rows_map.get(stock_code, []),
                broker_classification_map,
                accumulation_trend_map.get(stock_code, "FLAT")
            )

            # Tahap 5: gabungan
            combined = compute_bsjp_combined_score(late_strength, technical_result, bandarmology_result)
            if combined.get("score") is None:
                continue

            results.append({
                "stock_code": stock_code,
                "bsjp_final_score": combined["score"],
                "late_session_strength": late_strength,
                "technical": technical_result,
                "bandarmology": bandarmology_result
            })

        return sorted(results, key=lambda x: x["bsjp_final_score"], reverse=True)[:top_n]
```

## 9. Format Output (Top Hasil Pipeline)

```python
{
    "screening_date": "2025-08-15",
    "universe_size": 850,
    "after_initial_filter": 142,
    "final_ranked_candidates": [
        {
            "stock_code": "BBCA",
            "bsjp_final_score": 81.6,
            "technical": {
                "macd": {"crossover": "BULLISH_CROSSOVER", "histogram": 12.4},
                "rsi_like": 64.2,
                "candle_pattern": "BULLISH_MARUBOZU"
            },
            "bandarmology": {
                "top_accumulator_brokers": [
                    {"broker_code": "BK", "net": 8_675_000_000, "classification": "FOREIGN_INSTITUTIONAL"}
                ]
            },
            "narrative": "BBCA lolos screening BSJP market-wide dengan skor 81.6/100. "
                         "MACD baru saja bullish crossover, candle penutupan Bullish "
                         "Marubozu, didukung akumulasi broker BK (klasifikasi: Foreign "
                         "Institutional) senilai 8.67M."
        }
    ]
}
```

## 10. Ide Pengembangan Lanjutan

- **Adaptive filter threshold**: `min_avg_value` otomatis menyesuaikan
  kondisi likuiditas pasar keseluruhan (dari Fitur 21 Market Regime),
  bukan angka statis.
- **Parallel scoring**: Tahap 3-4 independen per saham, bisa diparalelkan
  untuk mempercepat screening ratusan saham (terhubung ke catatan
  paralelisasi di Fitur 41).
- **Historical pipeline backtesting**: jalankan pipeline ini secara
  historis lewat Fitur 24 untuk memvalidasi apakah bobot 15/15/40/30
  (closing/closing/technical/bandarmology) memang optimal, atau perlu
  disesuaikan dari data riil.

---

# FITUR 44 — TRADING MODE PROFILES (SWING vs DAY TRADE)

## 1. Tujuan Fitur

Mengonfigurasi ulang parameter dan bobot dari fitur-fitur yang SUDAH ADA
(bukan membuat scoring engine baru) sesuai gaya trading yang dipilih
user: **Day Trade** (BSJP/War Opening, horizon jam-hari), **Swing
Harian-Mingguan** (horizon hari-minggu, ZeroScore-driven), dan opsional
**Swing Bulanan** (horizon minggu-bulan, lebih condong fundamental +
seasonality). Ini adalah "lensa konfigurasi", bukan mesin analisa baru.

## 2. Struktur Profile

```python
TRADING_MODE_PROFILES = {
    "DAY_TRADE": {
        "primary_engines": ["bsjp_pipeline", "war_opening", "orderbook", "smart_money"],
        "zeroscore_weights": {
            "bandarmology": 0.30, "technical": 0.45, "foreign_flow": 0.15,
            "seasonality": 0.0, "risk_quality": 0.10
        },
        "technical_lookback_days": 10,
        "holding_period_label": "Jam s.d. 1 hari",
        "risk_per_trade_pct_default": 0.5,
        "min_liquidity_avg_value": 5e9,    # day trade butuh likuiditas lebih ketat
        "relevant_features": [1, 2, 3, 5, 6, 12, 13, 15, 16, 43]
    },
    "SWING_HARIAN_MINGGUAN": {
        "primary_engines": ["zeroscore", "technical", "bandarmology"],
        "zeroscore_weights": {
            "bandarmology": 0.40, "technical": 0.30, "foreign_flow": 0.15,
            "seasonality": 0.10, "risk_quality": 0.05
        },
        "technical_lookback_days": 60,
        "holding_period_label": "Beberapa hari s.d. beberapa minggu",
        "risk_per_trade_pct_default": 1.5,
        "min_liquidity_avg_value": 2e9,
        "relevant_features": [1, 3, 4, 7, 8, 11, 12, 13, 14, 17, 20, 25, 39, 40, 42]
    },
    "SWING_BULANAN": {
        "primary_engines": ["fundamental", "zeroscore", "sector_rotation", "dividend"],
        "zeroscore_weights": {
            "bandarmology": 0.25, "technical": 0.20, "foreign_flow": 0.10,
            "seasonality": 0.15, "risk_quality": 0.05, "fundamental": 0.25
        },
        "technical_lookback_days": 200,
        "holding_period_label": "Beberapa minggu s.d. beberapa bulan",
        "risk_per_trade_pct_default": 2.0,
        "min_liquidity_avg_value": 1e9,
        "relevant_features": [3, 7, 14, 20, 22, 25, 30, 32, 36, 40, 42]
    }
}
```

## 3. Rumus — Profile-Aware ZeroScore (reuse compute_zeroscore_v2 dari Fitur 14)

```python
def apply_trading_mode_to_zeroscore(component_scores: dict, mode: str) -> dict:
    """
    Membungkus compute_zeroscore_v2() (sudah ada di Fitur 14) dengan bobot
    yang disesuaikan mode trading — TIDAK mengubah cara skor komponen
    dihitung, hanya bobot penggabungannya.
    """
    if mode not in TRADING_MODE_PROFILES:
        mode = "SWING_HARIAN_MINGGUAN"  # default aman

    weights = TRADING_MODE_PROFILES[mode]["zeroscore_weights"]
    # compute_zeroscore_v2 diasumsikan sudah tersedia dari Fitur 14 (part4)
    return compute_zeroscore_v2(component_scores, weights)
```

## 4. Rumus — Profile-Aware Risk Sizing (reuse Fitur 13)

```python
def apply_trading_mode_to_risk(mode: str, account_balance: float,
                                custom_risk_pct: float = None) -> dict:
    """
    Mengambil risk_per_trade_pct default sesuai mode, kecuali user
    override manual. Day trade defaultnya lebih kecil (lebih sering
    transaksi, butuh risiko per transaksi lebih ketat).
    """
    profile = TRADING_MODE_PROFILES.get(mode, TRADING_MODE_PROFILES["SWING_HARIAN_MINGGUAN"])
    risk_pct = custom_risk_pct if custom_risk_pct is not None else profile["risk_per_trade_pct_default"]

    return {
        "mode": mode,
        "risk_per_trade_pct": risk_pct,
        "holding_period_label": profile["holding_period_label"],
        "min_liquidity_required": profile["min_liquidity_avg_value"]
    }
```

## 5. Logika — Memilih Engine yang Relevan per Mode

```python
def filter_relevant_engines_output(mode: str, full_analysis_result: dict) -> dict:
    """
    Dari hasil analisa lengkap (semua 42 fitur dijalankan via Orchestration
    Pipeline di Fitur 41), saring & highlight hanya bagian yang relevan
    untuk mode yang dipilih user — supaya tampilan tidak overwhelming.
    """
    profile = TRADING_MODE_PROFILES.get(mode, TRADING_MODE_PROFILES["SWING_HARIAN_MINGGUAN"])
    relevant_keys = profile["relevant_features"]

    filtered = {
        "mode": mode,
        "holding_period_label": profile["holding_period_label"],
        "highlighted_sections": {}
    }

    for feature_num in relevant_keys:
        key = f"feature_{feature_num}"
        if key in full_analysis_result:
            filtered["highlighted_sections"][key] = full_analysis_result[key]

    return filtered
```

## 6. Format Output

```python
{
    "stock_code": "BBCA",
    "mode": "DAY_TRADE",
    "holding_period_label": "Jam s.d. 1 hari",
    "zeroscore_adjusted": {
        "final_score": 79.1,
        "effective_weights": {"bandarmology": 0.30, "technical": 0.45, "foreign_flow": 0.15, "risk_quality": 0.10}
    },
    "risk_sizing": {
        "risk_per_trade_pct": 0.5,
        "min_liquidity_required": 5_000_000_000
    },
    "highlighted_engines": ["BSJP Pipeline (43)", "War Opening (16)", "Orderbook (5)", "Smart Money (6)"],
    "narrative": "Mode DAY_TRADE dipilih — bobot Technical dinaikkan ke 45% (dari "
                 "default 30%) karena horizon sangat pendek lebih bergantung pada "
                 "momentum & microstructure dibanding fundamental/seasonality. Risk "
                 "per trade dibatasi 0.5% mengingat frekuensi transaksi lebih tinggi."
}
```

## 7. Pseudocode Engine Lengkap

```python
class TradingModeEngine:

    def __init__(self):
        self.profiles = TRADING_MODE_PROFILES

    def apply_mode(self, stock_code: str, mode: str, component_scores: dict,
                    account_balance: float, custom_risk_pct: float = None) -> dict:

        if mode not in self.profiles:
            mode = "SWING_HARIAN_MINGGUAN"

        profile = self.profiles[mode]

        zeroscore_result = apply_trading_mode_to_zeroscore(component_scores, mode)
        risk_result = apply_trading_mode_to_risk(mode, account_balance, custom_risk_pct)

        return {
            "stock_code": stock_code,
            "mode": mode,
            "holding_period_label": profile["holding_period_label"],
            "zeroscore_adjusted": zeroscore_result,
            "risk_sizing": risk_result,
            "highlighted_engines": profile["primary_engines"],
            "narrative": self._generate_narrative(mode, zeroscore_result, risk_result)
        }

    def _generate_narrative(self, mode: str, zeroscore_result: dict, risk_result: dict) -> str:
        mode_label = {
            "DAY_TRADE": "DAY_TRADE — horizon sangat pendek, bobot Technical & microstructure dinaikkan",
            "SWING_HARIAN_MINGGUAN": "SWING_HARIAN_MINGGUAN — bobot seimbang antara Bandarmology dan Technical",
            "SWING_BULANAN": "SWING_BULANAN — bobot Fundamental & Seasonality dinaikkan, horizon lebih panjang"
        }
        return (
            f"Mode {mode_label.get(mode, mode)}. ZeroScore final {zeroscore_result.get('score')} "
            f"dengan risk per trade {risk_result['risk_per_trade_pct']}%."
        )
```

## 8. Ide Pengembangan Lanjutan

- **Hybrid mode**: user bisa custom bobot sendiri di luar 3 preset
  (mis. 50% Technical, 30% Bandarmology, 20% Fundamental) — preset di
  atas sebagai starting point, bukan satu-satunya opsi.
- **Mode-aware alert (terhubung Fitur 26)**: threshold alert otomatis
  beda per mode (Day Trade butuh alert real-time/intraday, Swing Bulanan
  cukup harian).
- **Mode performance comparison**: lewat Backtesting Engine (Fitur 24),
  bandingkan hasil historis tiap mode untuk gaya screening yang sama,
  supaya user tahu mode mana yang historically lebih cocok dengan gaya
  saham yang mereka screening.

---

# FITUR 45 — IHSG OUTLOOK FRAMEWORK (Technical & Flow-Based Scenario)

## 1. Tujuan Fitur

Memberi kerangka skenario untuk arah IHSG ke depan berdasarkan kombinasi
analisa teknikal (level kunci, struktur trend) dan flow (foreign/
domestic, breadth) — **disajikan sebagai skenario berbobot probabilitas
kualitatif dengan level invalidasi yang jelas, BUKAN target harga/tanggal
pasti.**

## 2. Data yang Dibutuhkan

OHLCV historis IHSG (sudah dipakai di Fitur 21), data foreign flow
market-wide harian (agregasi dari Fitur 4/19 di level seluruh pasar),
dan breadth (sudah ada di Fitur 21).

## 3. Rumus — Key Technical Levels (reuse Fitur 12 pivot logic)

```python
def identify_ihsg_key_levels(ihsg_history: list, lookback: int = 120) -> dict:
    """
    Reuse find_local_pivots() dari Fitur 12, diterapkan ke IHSG untuk
    cari support/resistance jangka menengah yang relevan sebagai level
    invalidasi skenario.
    """
    relevant = ihsg_history[-lookback:] if len(ihsg_history) >= lookback else ihsg_history
    pivots = find_local_pivots(relevant, lookback=5)

    current_price = ihsg_history[-1]["close"]
    sr = nearest_support_resistance(current_price, pivots)

    # All-time/period high-low sebagai level psikologis tambahan
    period_high = max(bar["high"] for bar in relevant)
    period_low = min(bar["low"] for bar in relevant)

    return {
        "nearest_support": sr["nearest_support"],
        "nearest_resistance": sr["nearest_resistance"],
        "period_high": period_high,
        "period_low": period_low,
        "current_price": current_price
    }
```

## 4. Rumus — Flow Strength Indicator (Market-Wide)

```python
def compute_market_flow_strength(daily_foreign_net_history: list, days: int = 10) -> dict:
    """
    daily_foreign_net_history: list net foreign value market-wide harian
    (agregasi seluruh saham, bukan per saham), urut tanggal menaik.
    """
    recent = daily_foreign_net_history[-days:] if len(daily_foreign_net_history) >= days else daily_foreign_net_history
    if not recent:
        return {"trend": "NO_DATA"}

    cumulative_net = sum(recent)
    positive_days = sum(1 for v in recent if v > 0)
    consistency_pct = positive_days / len(recent) * 100

    if cumulative_net > 0 and consistency_pct >= 60:
        trend = "SUSTAINED_INFLOW"
    elif cumulative_net < 0 and consistency_pct <= 40:
        trend = "SUSTAINED_OUTFLOW"
    else:
        trend = "MIXED_FLOW"

    return {
        "trend": trend,
        "cumulative_net_value": cumulative_net,
        "consistency_pct": round(consistency_pct, 2),
        "days_observed": len(recent)
    }
```

## 5. Logika — Membentuk Skenario (Bukan Prediksi Tunggal)

```
Skenario disusun SELALU sebagai 3 opsi paralel, bukan 1 jawaban pasti:

SKENARIO BULLISH CONTINUATION berlaku jika kondisi saat ini:
  - Market Regime (Fitur 21) = BULL_MARKET
  - Flow Strength = SUSTAINED_INFLOW
  - Breadth = BROAD_BASED_STRENGTH
  Invalidation level: breakdown di bawah nearest_support dengan volume
  tinggi -> skenario ini batal, re-evaluasi ke skenario lain.

SKENARIO BEARISH CONTINUATION berlaku jika kondisi saat ini:
  - Market Regime = BEAR_MARKET
  - Flow Strength = SUSTAINED_OUTFLOW
  - Breadth = BROAD_BASED_WEAKNESS
  Invalidation level: breakout di atas nearest_resistance dengan volume
  tinggi -> skenario ini batal.

SKENARIO SIDEWAYS/TRANSISI berlaku jika kondisi campuran:
  - Market Regime = TRANSITIONAL, ATAU
  - Flow Strength = MIXED_FLOW, ATAU
  - Sinyal teknikal dan flow saling bertentangan arah
  Level kunci: range antara nearest_support dan nearest_resistance —
  breakout dari salah satu sisi (disertai volume & flow konfirmasi)
  yang akan menentukan skenario mana yang berlaku berikutnya.

PENTING: skenario yang "paling konsisten dengan kondisi saat ini" BUKAN
berarti skenario itu pasti terjadi — hanya skenario yang paling didukung
oleh data SAAT INI. Kondisi bisa berubah kapan saja, terutama oleh faktor
makro/global yang sifatnya tidak bisa diprediksi dari data historis IHSG semata.
```

## 6. Pseudocode Engine Lengkap

```python
class IHSGOutlookEngine:

    def analyze(self, date: str, ihsg_history: list,
                daily_foreign_net_history: list,
                market_regime_result: dict, breadth_result: dict) -> dict:

        key_levels = identify_ihsg_key_levels(ihsg_history)
        flow_strength = compute_market_flow_strength(daily_foreign_net_history)

        scenario = self._determine_scenario(market_regime_result, flow_strength, breadth_result)

        return {
            "date": date,
            "key_levels": key_levels,
            "flow_strength": flow_strength,
            "primary_scenario": scenario,
            "narrative": self._generate_narrative(scenario, key_levels, flow_strength),
            "disclaimer": (
                "Ini adalah kerangka skenario berdasarkan kondisi teknikal & flow "
                "SAAT INI, bukan prediksi target harga atau tanggal. Skenario dapat "
                "berubah sewaktu-waktu, terutama akibat faktor makro/global yang "
                "tidak tercermin dari data historis IHSG semata."
            )
        }

    def _determine_scenario(self, market_regime_result: dict, flow_strength: dict,
                             breadth_result: dict) -> dict:
        trend = market_regime_result.get("market_regime", {}).get("trend", "TRANSITIONAL")
        flow_trend = flow_strength.get("trend", "MIXED_FLOW")
        breadth_signal = breadth_result.get("breadth_signal", "MIXED")

        if trend == "BULL_MARKET" and flow_trend == "SUSTAINED_INFLOW" and breadth_signal == "BROAD_BASED_STRENGTH":
            return {
                "scenario": "BULLISH_CONTINUATION",
                "confidence": "MEDIUM-HIGH",
                "supporting_factors": ["Market Regime BULL_MARKET", "Foreign flow sustained inflow", "Breadth broad-based strength"]
            }
        elif trend == "BEAR_MARKET" and flow_trend == "SUSTAINED_OUTFLOW" and breadth_signal == "BROAD_BASED_WEAKNESS":
            return {
                "scenario": "BEARISH_CONTINUATION",
                "confidence": "MEDIUM-HIGH",
                "supporting_factors": ["Market Regime BEAR_MARKET", "Foreign flow sustained outflow", "Breadth broad-based weakness"]
            }
        else:
            return {
                "scenario": "SIDEWAYS_TRANSISI",
                "confidence": "LOW-MEDIUM",
                "supporting_factors": [f"Trend: {trend}", f"Flow: {flow_trend}", f"Breadth: {breadth_signal}"],
                "note": "Sinyal teknikal dan flow tidak sepenuhnya selaras — kondisi pasar sedang mencari arah."
            }

    def _generate_narrative(self, scenario: dict, key_levels: dict, flow_strength: dict) -> str:
        base = (
            f"Skenario yang paling konsisten dengan kondisi saat ini: {scenario['scenario']} "
            f"(confidence {scenario['confidence']}). Faktor pendukung: {'; '.join(scenario['supporting_factors'])}."
        )

        if scenario["scenario"] == "BULLISH_CONTINUATION":
            base += (
                f" Level yang membatalkan skenario ini: breakdown di bawah support "
                f"{key_levels['nearest_support']} dengan volume tinggi."
            )
        elif scenario["scenario"] == "BEARISH_CONTINUATION":
            base += (
                f" Level yang membatalkan skenario ini: breakout di atas resistance "
                f"{key_levels['nearest_resistance']} dengan volume tinggi."
            )
        else:
            base += (
                f" Range kunci untuk dipantau: {key_levels['nearest_support']} - "
                f"{key_levels['nearest_resistance']}. Breakout dari salah satu sisi "
                f"(disertai konfirmasi volume & flow) akan menentukan arah skenario berikutnya."
            )

        return base
```

## 7. Format Output

```python
{
    "date": "2025-08-15",
    "key_levels": {
        "nearest_support": 6980, "nearest_resistance": 7250,
        "period_high": 7310, "period_low": 6750, "current_price": 7150
    },
    "flow_strength": {
        "trend": "SUSTAINED_INFLOW", "cumulative_net_value": 1_850_000_000_000,
        "consistency_pct": 70.0, "days_observed": 10
    },
    "primary_scenario": {
        "scenario": "BULLISH_CONTINUATION",
        "confidence": "MEDIUM-HIGH",
        "supporting_factors": ["Market Regime BULL_MARKET", "Foreign flow sustained inflow", "Breadth broad-based strength"]
    },
    "narrative": "Skenario yang paling konsisten dengan kondisi saat ini: "
                 "BULLISH_CONTINUATION (confidence MEDIUM-HIGH)... Level yang "
                 "membatalkan skenario ini: breakdown di bawah support 6980 dengan volume tinggi.",
    "disclaimer": "Ini adalah kerangka skenario berdasarkan kondisi teknikal & flow "
                  "SAAT INI, bukan prediksi target harga atau tanggal."
}
```

## 8. Ide Pengembangan Lanjutan

- **Historical scenario accuracy**: lewat Backtesting Engine (Fitur 24),
  hitung secara historis berapa sering skenario "BULLISH_CONTINUATION"
  (dengan kombinasi kondisi yang sama) memang berlanjut vs gagal — untuk
  memberi confidence yang benar-benar berbasis data, bukan asumsi.
- **Global market overlay**: tambahkan konteks index regional/global
  (lewat Macro Context, Fitur 30) sebagai faktor tambahan pembentuk skenario.
- **Multi-horizon scenario**: pisahkan skenario jangka pendek (1-2 minggu)
  vs menengah (1-3 bulan), karena faktor pendukungnya bisa berbeda.

---

# FITUR 46 — TRADING PLAN GENERATOR (Penutup)

## 1. Tujuan Fitur

Fitur penutup yang menyatukan SEMUA hasil — ZeroScore, Trading Mode
(Fitur 44), Phase Detector (42), Risk Engine (13), dan Portfolio
Construction (40) — menjadi SATU output akhir per saham yang benar-benar
actionable: bukan lagi kumpulan skor terpisah, tapi rencana trading yang
siap dibaca dan dieksekusi.

## 2. Rumus — Sintesis Akhir (murni menggabungkan output engine lain, tanpa skor baru)

```python
def generate_trading_plan(stock_code: str, mode: str,
                           zeroscore_result: dict, phase_result: dict,
                           technical_result: dict, risk_plan: dict,
                           corporate_action_context: dict,
                           confidence_multiplier: float) -> dict:
    """
    Tidak menghitung skor baru — murni mensintesis output dari Fitur 14,
    42, 12, 13, 25, dan 41 (confidence) menjadi satu kesimpulan akhir.
    """
    final_score = zeroscore_result.get("score") or zeroscore_result.get("final_score")
    phase = phase_result.get("phase", {}).get("phase", "TIDAK_DIKETAHUI")

    # Tentukan aksi utama
    if final_score is None:
        action = "TIDAK_CUKUP_DATA"
    elif phase == "DISTRIBUSI" and phase_result.get("distribution_proximity", {}).get("score", 0) >= 70:
        action = "HINDARI_ENTRY_BARU - ciri distribusi kuat terdeteksi"
    elif final_score >= 70 and phase in ("AKUMULASI", "MARKUP"):
        action = "LAYAK_ENTRY"
    elif final_score >= 70 and phase == "TIDAK_DIKETAHUI":
        action = "LAYAK_ENTRY_DENGAN_KEHATI_HATIAN - fase belum jelas"
    elif 50 <= final_score < 70:
        action = "PANTAU - belum cukup kuat untuk entry penuh"
    else:
        action = "HINDARI"

    # Caution flags dari corporate action
    cautions = []
    if corporate_action_context.get("signal_caution"):
        cautions.append(corporate_action_context["signal_caution"])

    if confidence_multiplier < 0.7:
        cautions.append("Confidence keseluruhan rendah — pertimbangkan ukuran posisi lebih kecil dari biasanya.")

    return {
        "stock_code": stock_code,
        "mode": mode,
        "action": action,
        "final_score": final_score,
        "phase": phase,
        "confidence_multiplier": confidence_multiplier,
        "cautions": cautions
    }
```

## 3. Pseudocode Engine Lengkap (Orkestrator Final)

```python
class TradingPlanGenerator:
    """
    Dipanggil PALING AKHIR dalam keseluruhan pipeline (setelah Fitur 41
    Layer 4-5 selesai) — menyatukan semua hasil jadi satu kartu ringkas
    per saham.
    """

    def generate(self, stock_code: str, mode: str, layer4_result: dict,
                 risk_engine, account_balance: float,
                 corporate_action_engine, as_of_date: str,
                 entry_price: float, target_price: float) -> dict:

        zeroscore_result = layer4_result["zeroscore"]["zeroscore_v2"]
        phase_result = layer4_result["phase"]
        technical_result = layer4_result["technical"]
        confidence = layer4_result["confidence_applied"]["confidence_multiplier"]

        ca_context = corporate_action_engine.get_context(stock_code, as_of_date)

        risk_plan = risk_engine.plan_trade(
            stock_code, layer4_result.get("ohlcv_history", []),
            entry_price, target_price
        )

        synthesis = generate_trading_plan(
            stock_code, mode, zeroscore_result, phase_result,
            technical_result, risk_plan, ca_context, confidence
        )

        return {
            **synthesis,
            "technical_snapshot": technical_result.get("technical", {}),
            "phase_snapshot": phase_result.get("phase", {}),
            "risk_plan": risk_plan.get("plan", {}),
            "corporate_action_notes": ca_context.get("corporate_action_context", {}),
            "narrative": self._generate_final_narrative(synthesis, risk_plan)
        }

    def _generate_final_narrative(self, synthesis: dict, risk_plan: dict) -> str:
        action_text = {
            "LAYAK_ENTRY": "Layak dipertimbangkan untuk entry baru.",
            "LAYAK_ENTRY_DENGAN_KEHATI_HATIAN - fase belum jelas": "Skor mendukung entry, namun fase pergerakan belum jelas — pertimbangkan ukuran posisi lebih kecil.",
            "PANTAU": "Belum cukup kuat untuk entry penuh — masuk watchlist untuk dipantau.",
            "HINDARI": "Tidak disarankan untuk entry baru saat ini.",
            "HINDARI_ENTRY_BARU - ciri distribusi kuat terdeteksi": "Tidak disarankan entry baru — ciri-ciri fase distribusi cukup kuat terdeteksi.",
            "TIDAK_CUKUP_DATA": "Data tidak cukup untuk memberi rekomendasi yang dapat dipertanggungjawabkan."
        }

        base = f"{synthesis['stock_code']} (Mode {synthesis['mode']}): {action_text.get(synthesis['action'], synthesis['action'])}"
        base += f" ZeroScore {synthesis['final_score']}, fase saat ini: {synthesis['phase']}."

        if synthesis["cautions"]:
            base += " Catatan: " + " ".join(synthesis["cautions"])

        plan = risk_plan.get("plan", {})
        if plan.get("position_sizing", {}).get("recommended_lots"):
            base += (
                f" Jika entry, ukuran posisi disarankan {plan['position_sizing']['recommended_lots']} lot "
                f"dengan stop loss di {plan.get('stop_loss', {}).get('stop_price')}."
            )

        return base
```

## 4. Format Output Final (Contoh Kartu Trading Plan)

```python
{
    "stock_code": "BBCA",
    "mode": "SWING_HARIAN_MINGGUAN",
    "action": "LAYAK_ENTRY",
    "final_score": 78.4,
    "phase": "MARKUP",
    "confidence_multiplier": 0.95,
    "cautions": [],
    "technical_snapshot": {
        "candle_pattern": "BULLISH_MARUBOZU",
        "rsi_like": 64.2,
        "support_resistance": {"nearest_support": 9650, "nearest_resistance": 10000}
    },
    "phase_snapshot": {"phase": "MARKUP", "confidence": "MEDIUM"},
    "risk_plan": {
        "entry_price": 9875,
        "stop_loss": {"stop_price": 9584},
        "position_sizing": {"recommended_lots": 25, "risk_pct_of_account": 1.46}
    },
    "corporate_action_notes": {"upcoming_events": []},
    "narrative": "BBCA (Mode SWING_HARIAN_MINGGUAN): Layak dipertimbangkan untuk "
                 "entry baru. ZeroScore 78.4, fase saat ini: MARKUP. Jika entry, "
                 "ukuran posisi disarankan 25 lot dengan stop loss di 9584."
}
```

## 5. Ide Pengembangan Lanjutan

- **Multi-stock plan board**: jalankan generator ini untuk seluruh
  watchlist sekaligus, hasilkan "papan rencana" harian, bukan satu-satu.
- **Plan versioning**: simpan histori trading plan per saham dari waktu
  ke waktu untuk evaluasi konsistensi rekomendasi (terhubung ke Fitur 35
  Audit Trail).
- **Feedback loop ke Backtesting**: setiap action yang dihasilkan (LAYAK_
  ENTRY, HINDARI, dst) dicatat dan divalidasi forward return-nya lewat
  Fitur 24 — siklus penuh dari rekomendasi sampai validasi hasilnya.

---

# PENUTUP — KESIMPULAN AKHIR ZEROSTOCK (46 FITUR, 10 PART)

## Ringkasan Perjalanan

ZeroStock berkembang dari 3 dokumen awal (broker summary, foreign flow,
technical & screener dasar) menjadi 46 fitur yang mencakup seluruh
siklus analisa saham BEI: dari **validasi data mentah**, **microstructure
broker & asing**, **technical & fundamental**, **konteks pasar &
makro**, **kekhususan aturan BEI**, **dimensi yang sebelumnya kosong**
(dividen, leverage, insider, cross-saham), sampai **portofolio nyata
dan rencana eksekusi** yang siap dipakai sehari-hari — lengkap dengan
mode trading yang bisa disesuaikan gaya masing-masing trader.

## Prinsip yang Dipegang Konsisten Sampai Akhir

1. **Tidak pernah mengklaim lebih dari yang bisa dibuktikan data** —
   astrologi ditolak (part 4), prediksi tanggal distribusi ditolak (part
   9), prediksi target/tanggal IHSG ditolak (part 10). Semua diganti
   dengan kerangka berbasis statistik & skenario yang jujur soal batasnya.
2. **Setiap skor bisa diaudit** — tidak ada angka 0-100 yang keluar dari
   "kotak hitam"; semua dipecah jadi komponen dengan bobot eksplisit.
3. **Python murni dari part 4 ke atas** — tidak ada dependency pandas/
   numpy, supaya mudah dipahami dan ditempel langsung ke codebase yang
   sudah berjalan.
4. **Validasi sebelum kesimpulan** — Data Quality, Data Health,
   Backtesting selalu jadi lapisan terpisah yang menjaga supaya skor
   lain tidak diterima mentah-mentah.

## Untuk Pengembangan Selanjutnya

Dokumen part 1-10 ini adalah **konsep & blueprint**, bukan kode produksi.
Langkah wajar berikutnya secara teknis (di luar cakupan dokumentasi
konsep ini):

- Implementasi nyata tiap engine sebagai modul `.py` terpisah, diuji unit
  test per fungsi
- Integrasi pemanggilan API riil (Stockbit/IDX) menggantikan struktur
  data contoh di tiap fitur
- Setup database sesuai schema yang sudah didefinisikan di part 1-7
  untuk fitur yang membutuhkan riwayat data
- Eksekusi nyata Backtesting Engine (Fitur 24) dengan data historis riil
  untuk akhirnya memvalidasi — bukan berasumsi — bobot-bobot skor yang
  sudah didesain di seluruh dokumen ini

Selamat mengembangkan ZeroStock. Semoga blueprint 46 fitur ini menjadi
fondasi yang kokoh untuk platform analisa saham yang benar-benar
membantu pengambilan keputusan investasi di pasar modal Indonesia secara
jujur dan bertanggung jawab.
