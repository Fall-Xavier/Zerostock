# ZeroStock Platform — Complete Quant & Bandarmology System
> Senior Quant Analyst + Bandarmology Expert + Product Architect Documentation  
> Version 1.0 | ZeroStock Analytics Engine

---

## ARSITEKTUR GLOBAL

```
ZeroStock Analytics Engine
│
├── Data Layer
│   ├── Market Data (OHLCV)
│   ├── Broker Transaction Data
│   ├── Orderbook Data
│   └── Foreign Flow Data
│
├── Feature Engine
│   ├── Feature 1: Broker Summary
│   ├── Feature 2: Broker Activity
│   ├── Feature 3: Broker Accumulation
│   ├── Feature 4: Foreign Flow
│   ├── Feature 5: Orderbook Analysis
│   ├── Feature 6: Smart Money Analysis
│   ├── Feature 7: Bandarmology Score
│   ├── Feature 8: ChartBit Style Analysis
│   └── Feature 9: Stock Screener
│
└── Master Score Engine
    └── Feature 10: ZeroScore
```

---

# FITUR 1 — BROKER SUMMARY

## 1. Tujuan Fitur
Mengetahui broker mana yang sedang mengakumulasi (beli dominan) atau mendistribusi (jual dominan) suatu saham pada periode tertentu. Fitur ini mengidentifikasi "smart broker" yang gerakannya sering mendahului pergerakan harga.

## 2. Data yang Dibutuhkan
| Field | Tipe | Keterangan |
|---|---|---|
| stock_code | VARCHAR | Kode saham (BBCA, TLKM, dst) |
| broker_code | VARCHAR | Kode broker (BK, YP, AK, ZP, dll) |
| broker_name | VARCHAR | Nama lengkap broker |
| broker_type | ENUM | FOREIGN / LOCAL |
| trade_date | DATE | Tanggal transaksi |
| buy_volume | BIGINT | Total volume beli broker |
| buy_value | BIGINT | Total nilai beli (IDR) |
| buy_lot | BIGINT | Total lot beli |
| buy_freq | INT | Frekuensi transaksi beli |
| sell_volume | BIGINT | Total volume jual broker |
| sell_value | BIGINT | Total nilai jual (IDR) |
| sell_lot | BIGINT | Total lot jual |
| sell_freq | INT | Frekuensi transaksi jual |
| avg_price | DECIMAL | Harga rata-rata saham hari itu |

## 3. Struktur Database

```sql
-- Tabel Master Broker
CREATE TABLE brokers (
    broker_code     VARCHAR(10) PRIMARY KEY,
    broker_name     VARCHAR(100) NOT NULL,
    broker_type     ENUM('FOREIGN', 'LOCAL') NOT NULL,
    is_foreign_flag BOOLEAN DEFAULT FALSE,
    country_origin  VARCHAR(50),
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Tabel Raw Broker Transaction
CREATE TABLE broker_transactions (
    id              BIGSERIAL PRIMARY KEY,
    stock_code      VARCHAR(10) NOT NULL,
    broker_code     VARCHAR(10) NOT NULL,
    trade_date      DATE NOT NULL,
    buy_volume      BIGINT DEFAULT 0,
    buy_value       BIGINT DEFAULT 0,
    buy_lot         BIGINT DEFAULT 0,
    buy_freq        INT DEFAULT 0,
    sell_volume     BIGINT DEFAULT 0,
    sell_value      BIGINT DEFAULT 0,
    sell_lot        BIGINT DEFAULT 0,
    sell_freq       INT DEFAULT 0,
    UNIQUE (stock_code, broker_code, trade_date),
    INDEX idx_stock_date (stock_code, trade_date),
    INDEX idx_broker_date (broker_code, trade_date)
);

-- Tabel Broker Summary (computed/aggregated)
CREATE TABLE broker_summary (
    id                      BIGSERIAL PRIMARY KEY,
    stock_code              VARCHAR(10) NOT NULL,
    trade_date              DATE NOT NULL,
    period_days             INT NOT NULL,            -- 1, 3, 5, 10, 20, 60
    top_buy_broker          VARCHAR(10),
    top_sell_broker         VARCHAR(10),
    total_net_buy_volume    BIGINT,
    total_net_sell_volume   BIGINT,
    total_net_buy_value     BIGINT,
    total_net_sell_value    BIGINT,
    buy_avg_price           DECIMAL(12,2),
    sell_avg_price          DECIMAL(12,2),
    foreign_net_volume      BIGINT,
    foreign_net_value       BIGINT,
    local_net_volume        BIGINT,
    local_net_value         BIGINT,
    concentration_ratio     DECIMAL(5,2),            -- % transaksi dari top 5 broker
    broker_acc_score        DECIMAL(5,2),            -- 0-100
    dominant_broker         VARCHAR(10),
    dominant_type           ENUM('FOREIGN','LOCAL','MIXED'),
    updated_at              TIMESTAMP DEFAULT NOW(),
    UNIQUE (stock_code, trade_date, period_days)
);
```

## 4. Rumus Perhitungan

```
Net Buy Volume   = Σ buy_volume - Σ sell_volume  (per broker, per saham)
Net Buy Value    = Σ buy_value  - Σ sell_value
Net Volume       = Σ semua Net Buy Volume broker (total pasar)
Net Value        = Σ semua Net Buy Value broker

Buy Average  = Σ(buy_value) / Σ(buy_volume)
Sell Average = Σ(sell_value) / Σ(sell_volume)

Net Value / Share = Buy Average - Sell Average

Foreign Net = Σ(buy_value foreign) - Σ(sell_value foreign)
Local Net   = Σ(buy_value local)   - Σ(sell_value local)

Broker Concentration Ratio =
    [Σ |net_value| top-5 broker] / [Σ |net_value| semua broker] × 100
```

## 5. Logika Analisa

```
Broker Dominan:
  → Broker dengan |net_value| tertinggi hari ini

Broker Akumulasi:
  → net_buy_value > 0 DAN net_buy_volume > 0 DAN konsisten ≥ 3 hari

Broker Distribusi:
  → net_sell_value > 0 DAN net_sell_volume > 0 DAN konsisten ≥ 3 hari

Broker Asing Dominan:
  → foreign_net_value > local_net_value ATAU
    |foreign_net_value| / total_net_value > 0.6

Broker Lokal Dominan:
  → local_net_value > foreign_net_value

Konsentrasi Transaksi:
  → Jika concentration_ratio > 70%: transaksi sangat terkonsentrasi (waspada manipulasi)
  → Jika concentration_ratio 40-70%: wajar
  → Jika concentration_ratio < 40%: tersebar luas (likuid)
```

## 6. Kondisi Bullish
- Net Buy Value > 0 secara agregat
- Top Buy Broker adalah broker besar / foreign
- Buy Average < Harga sekarang (broker beli di harga lebih rendah)
- Foreign Net positif
- Concentration dari sisi beli > 60% (akumulasi terorganisir)

## 7. Kondisi Bearish
- Net Sell Value dominan
- Top Sell Broker konsisten 3+ hari
- Sell Average > Harga sekarang (distribusi di harga tinggi)
- Foreign Net negatif besar
- Volume jual meningkat dengan harga turun

## 8. Kondisi Netral
- |Net Value| < 5% dari total value transaksi
- Buy & Sell terdistribusi merata di banyak broker
- Tidak ada dominasi satu pihak

## 9. Sistem Skoring — Broker Accumulation Score (0-100)

```
Komponen Skor:

A. Net Value Score (bobot 30%)
   → if net_value > 0:
       raw = min(net_value / median_daily_value * 50, 100)
   → if net_value < 0:
       raw = max(50 - abs(net_value) / median_daily_value * 50, 0)
   score_A = raw * 0.30

B. Broker Consistency Score (bobot 25%)
   → Hitung jumlah hari akumulasi dalam 5 hari terakhir
   → consistency_days / 5 * 100
   score_B = consistency_days_pct * 0.25

C. Foreign Participation Score (bobot 20%)
   → if foreign_net_value > 0:
       raw = min(foreign_net_value / total_value * 200, 100)
   → else raw = 0
   score_C = raw * 0.20

D. Buy Average vs Price Score (bobot 15%)
   → if buy_avg < current_price: raw = 80
   → if buy_avg == current_price: raw = 50
   → if buy_avg > current_price: raw = 20
   score_D = raw * 0.15

E. Concentration Score (bobot 10%)
   → Konsentrasi beli tinggi = akumulasi terorganisir
   → raw = concentration_ratio * 1.0 (langsung %)
   score_E = raw * 0.10

TOTAL = score_A + score_B + score_C + score_D + score_E
```

## 10. Sistem Rating

| Score | Rating |
|---|---|
| 81–100 | Strong Buy |
| 61–80 | Buy |
| 41–60 | Hold |
| 21–40 | Sell |
| 0–20 | Strong Sell |

## 11. Sistem Deteksi Bandar

```
Bandar Terdeteksi jika:
  1. Satu broker menguasai > 30% net buy value DALAM 5 hari
  2. Volume beli broker tersebut naik >200% vs 10 hari rata-rata
  3. Harga sideways / turun perlahan saat akumulasi (suppressed price)
  4. Buy freq tinggi tapi lot kecil (accumulation pattern)
  5. Broker adalah broker sekuritas asing atau institusional besar

Bandar Score:
  bandar_detected = True jika minimal 3 dari 5 kondisi terpenuhi
```

## 12. Sistem Deteksi Distribusi

```
Distribusi Terdeteksi jika:
  1. Net Sell Value meningkat 3 hari berturut-turut
  2. Broker yang sebelumnya beli besar kini jual besar (flip)
  3. Sell Average > Buy Average sebelumnya (profit taking)
  4. Harga naik tapi volume jual meningkat
  5. Foreign Net berubah dari positif ke negatif
```

## 13. Sistem Deteksi Akumulasi

```
Akumulasi Terdeteksi jika:
  1. Net Buy Volume positif ≥ 3 dari 5 hari terakhir
  2. Buy Average < harga saat ini (akumulasi di bawah harga pasar)
  3. Volume beli konsisten atau meningkat perlahan
  4. Tidak ada lonjakan harga besar (diam-diam beli)
  5. Foreign + Institusional net positif bersamaan
```

## 14. Sistem Notifikasi

```
TRIGGER NOTIFIKASI:
  🟢 "Akumulasi Terdeteksi" → net_buy_value naik 50%+ hari ini
  🟢 "Broker Asing Masuk" → foreign_net_value > threshold positif
  🔴 "Distribusi Terdeteksi" → net_sell_value naik 50%+ hari ini
  🔴 "Broker Besar Jual" → top broker flip dari beli ke jual
  🟡 "Konsentrasi Tinggi" → concentration_ratio > 70%
  🔵 "Smart Money Signal" → broker score > 75 + volume spike
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "trade_date": "2025-08-15",
  "period": "5D",
  "broker_summary": {
    "top_buy_broker": {
      "code": "BK",
      "name": "J.P. Morgan Securities",
      "type": "FOREIGN",
      "net_buy_volume": 12500000,
      "net_buy_value": 9875000000,
      "buy_avg_price": 790
    },
    "top_sell_broker": {
      "code": "YP",
      "name": "Indo Premier Sekuritas",
      "type": "LOCAL",
      "net_sell_volume": 4200000,
      "net_sell_value": 3318000000,
      "sell_avg_price": 790
    },
    "aggregates": {
      "total_net_buy_value": 12450000000,
      "total_net_sell_value": 8200000000,
      "net_value": 4250000000,
      "buy_avg_price": 786.5,
      "sell_avg_price": 791.2,
      "foreign_net_value": 8300000000,
      "local_net_value": -4050000000,
      "concentration_ratio": 62.4
    },
    "signals": {
      "dominant_broker": "BK",
      "dominant_type": "FOREIGN",
      "is_accumulating": true,
      "is_distributing": false,
      "bandar_detected": true,
      "foreign_dominant": true
    },
    "scores": {
      "broker_acc_score": 74.8,
      "rating": "Buy"
    },
    "narrative": "Broker asing J.P. Morgan mendominasi pembelian dengan net buy IDR 9.87M dalam 5 hari. Foreign net positif IDR 8.3M mengindikasikan akumulasi institusional. Harga rata-rata beli broker (786.5) di bawah harga saat ini menunjukkan posisi broker masih profit."
  }
}
```

## 16. Narasi Otomatis

```python
def generate_broker_summary_narrative(data: dict) -> str:
    top_buy = data["top_buy_broker"]
    agg = data["aggregates"]
    sig = data["signals"]
    score = data["scores"]["broker_acc_score"]

    narrative_parts = []

    # Dominasi broker
    if sig["bandar_detected"]:
        narrative_parts.append(
            f"⚠️ Terdeteksi akumulasi terkonsentrasi oleh {top_buy['name']} "
            f"({top_buy['type']}) dengan net buy IDR "
            f"{agg['total_net_buy_value']/1e9:.2f}M dalam periode ini."
        )
    
    # Foreign vs Local
    if sig["foreign_dominant"]:
        narrative_parts.append(
            f"Dana asing mendominasi dengan net IDR {agg['foreign_net_value']/1e9:.2f}M, "
            f"mengindikasikan kepercayaan investor institusional."
        )
    elif agg["local_net_value"] > 0:
        narrative_parts.append(
            f"Broker lokal melakukan net buy IDR {agg['local_net_value']/1e9:.2f}M."
        )

    # Buy vs Sell Average
    if agg["buy_avg_price"] < agg["sell_avg_price"]:
        narrative_parts.append(
            f"Harga rata-rata beli ({agg['buy_avg_price']:.0f}) "
            f"lebih rendah dari rata-rata jual ({agg['sell_avg_price']:.0f}), "
            f"menunjukkan tekanan beli lebih agresif."
        )

    # Konsentrasi
    if agg["concentration_ratio"] > 60:
        narrative_parts.append(
            f"Transaksi terkonsentrasi di top broker ({agg['concentration_ratio']:.1f}%), "
            f"indikasi akumulasi terorganisir."
        )

    # Rating
    narrative_parts.append(
        f"Broker Accumulation Score: {score:.1f}/100 — Rating: {data['scores']['rating']}"
    )

    return " ".join(narrative_parts)
```

## 17. Pseudocode Python

```python
import pandas as pd
import numpy as np
from dataclasses import dataclass
from typing import Optional

@dataclass
class BrokerSummaryResult:
    stock_code: str
    trade_date: str
    period_days: int
    top_buy_broker: dict
    top_sell_broker: dict
    aggregates: dict
    signals: dict
    scores: dict
    narrative: str

class BrokerSummaryEngine:
    
    FOREIGN_BROKERS = {
        'BK': 'J.P. Morgan', 'ZP': 'Credit Suisse', 'AK': 'UBS',
        'DB': 'Deutsche Bank', 'MS': 'Morgan Stanley', 'CS': 'CLSA',
        'YJ': 'Citigroup', 'RX': 'Macquarie'
    }

    def __init__(self, db_connection):
        self.db = db_connection

    def fetch_transactions(self, stock_code: str, end_date: str, days: int) -> pd.DataFrame:
        """Ambil data transaksi broker dari database"""
        query = """
            SELECT bt.*, b.broker_type, b.broker_name
            FROM broker_transactions bt
            JOIN brokers b ON bt.broker_code = b.broker_code
            WHERE bt.stock_code = %s
              AND bt.trade_date BETWEEN DATE_SUB(%s, INTERVAL %s DAY) AND %s
            ORDER BY bt.trade_date DESC
        """
        return pd.read_sql(query, self.db, params=[stock_code, end_date, days, end_date])

    def compute_broker_metrics(self, df: pd.DataFrame) -> pd.DataFrame:
        """Hitung metrik per broker"""
        grouped = df.groupby('broker_code').agg(
            broker_name=('broker_name', 'first'),
            broker_type=('broker_type', 'first'),
            total_buy_volume=('buy_volume', 'sum'),
            total_buy_value=('buy_value', 'sum'),
            total_sell_volume=('sell_volume', 'sum'),
            total_sell_value=('sell_value', 'sum'),
            buy_freq=('buy_freq', 'sum'),
            sell_freq=('sell_freq', 'sum'),
            active_days=('trade_date', 'nunique')
        ).reset_index()

        grouped['net_volume'] = grouped['total_buy_volume'] - grouped['total_sell_volume']
        grouped['net_value'] = grouped['total_buy_value'] - grouped['total_sell_value']
        grouped['buy_avg'] = np.where(
            grouped['total_buy_volume'] > 0,
            grouped['total_buy_value'] / grouped['total_buy_volume'], 0
        )
        grouped['sell_avg'] = np.where(
            grouped['total_sell_volume'] > 0,
            grouped['total_sell_value'] / grouped['total_sell_volume'], 0
        )
        return grouped

    def get_top_brokers(self, metrics: pd.DataFrame):
        """Identifikasi top buy dan top sell broker"""
        top_buy = metrics.nlargest(1, 'net_value').iloc[0]
        top_sell = metrics.nsmallest(1, 'net_value').iloc[0]
        return top_buy, top_sell

    def compute_aggregates(self, metrics: pd.DataFrame) -> dict:
        """Hitung agregat keseluruhan"""
        total_buy_value = metrics['total_buy_value'].sum()
        total_sell_value = metrics['total_sell_value'].sum()
        total_buy_vol = metrics['total_buy_volume'].sum()
        total_sell_vol = metrics['total_sell_volume'].sum()

        foreign = metrics[metrics['broker_type'] == 'FOREIGN']
        local = metrics[metrics['broker_type'] == 'LOCAL']

        # Concentration ratio (top 5 broker by |net_value|)
        top5 = metrics.nlargest(5, 'net_value')
        concentration = (
            top5['net_value'].abs().sum() / 
            (metrics['net_value'].abs().sum() + 1e-9)
        ) * 100

        return {
            'total_net_buy_value': total_buy_value - total_sell_value,
            'total_net_buy_volume': total_buy_vol - total_sell_vol,
            'buy_avg_price': total_buy_value / (total_buy_vol + 1e-9),
            'sell_avg_price': total_sell_value / (total_sell_vol + 1e-9),
            'foreign_net_value': foreign['net_value'].sum(),
            'local_net_value': local['net_value'].sum(),
            'concentration_ratio': round(concentration, 2)
        }

    def compute_broker_acc_score(self, metrics: pd.DataFrame, 
                                  aggregates: dict,
                                  current_price: float,
                                  median_daily_value: float) -> float:
        """Hitung Broker Accumulation Score 0-100"""
        net_value = aggregates['total_net_buy_value']
        
        # A. Net Value Score (30%)
        if net_value > 0:
            raw_a = min(net_value / (median_daily_value + 1e-9) * 50, 100)
        else:
            raw_a = max(50 - abs(net_value) / (median_daily_value + 1e-9) * 50, 0)
        score_a = raw_a * 0.30

        # B. Consistency Score (25%)
        # Hitung dari data harian (perlu daily breakdown)
        # Placeholder: pakai active_days / period_days ratio
        max_active = metrics['active_days'].max()
        raw_b = min(max_active / 5 * 100, 100)
        score_b = raw_b * 0.25

        # C. Foreign Participation Score (20%)
        foreign_net = aggregates['foreign_net_value']
        total_val = abs(net_value) + 1e-9
        if foreign_net > 0:
            raw_c = min(foreign_net / total_val * 100, 100)
        else:
            raw_c = 0
        score_c = raw_c * 0.20

        # D. Buy Average vs Price Score (15%)
        buy_avg = aggregates['buy_avg_price']
        if buy_avg < current_price:
            raw_d = 80
        elif buy_avg == current_price:
            raw_d = 50
        else:
            raw_d = 20
        score_d = raw_d * 0.15

        # E. Concentration Score (10%)
        conc = aggregates['concentration_ratio']
        raw_e = min(conc, 100)
        score_e = raw_e * 0.10

        total_score = score_a + score_b + score_c + score_d + score_e
        return round(total_score, 2)

    def detect_bandar(self, metrics: pd.DataFrame, 
                       aggregates: dict,
                       period_days: int) -> bool:
        """Deteksi pola bandar"""
        conditions_met = 0

        # 1. Single broker dominasi > 30%
        total_abs = metrics['net_value'].abs().sum()
        if total_abs > 0:
            max_broker_share = metrics['net_value'].max() / total_abs
            if max_broker_share > 0.30:
                conditions_met += 1

        # 2. Foreign masuk (proxy bandar institusional)
        if aggregates['foreign_net_value'] > 0:
            conditions_met += 1

        # 3. Konsentrasi tinggi (terorganisir)
        if aggregates['concentration_ratio'] > 60:
            conditions_met += 1

        # 4. Buy avg < sell avg (beli di bawah, jual di atas)
        if aggregates['buy_avg_price'] < aggregates['sell_avg_price']:
            conditions_met += 1

        return conditions_met >= 3

    def get_rating(self, score: float) -> str:
        if score >= 81: return "Strong Buy"
        elif score >= 61: return "Buy"
        elif score >= 41: return "Hold"
        elif score >= 21: return "Sell"
        else: return "Strong Sell"

    def analyze(self, stock_code: str, trade_date: str, 
                period_days: int = 5,
                current_price: float = 0,
                median_daily_value: float = 0) -> BrokerSummaryResult:
        """Main analyzer - entry point"""
        
        # 1. Ambil data
        df = self.fetch_transactions(stock_code, trade_date, period_days)
        if df.empty:
            raise ValueError(f"No data for {stock_code} on {trade_date}")

        # 2. Hitung metrik broker
        metrics = self.compute_broker_metrics(df)

        # 3. Top brokers
        top_buy, top_sell = self.get_top_brokers(metrics)

        # 4. Agregat
        aggregates = self.compute_aggregates(metrics)

        # 5. Skor
        acc_score = self.compute_broker_acc_score(
            metrics, aggregates, current_price, median_daily_value
        )
        rating = self.get_rating(acc_score)

        # 6. Sinyal
        signals = {
            'dominant_broker': top_buy['broker_code'],
            'dominant_type': top_buy['broker_type'],
            'is_accumulating': aggregates['total_net_buy_value'] > 0,
            'is_distributing': aggregates['total_net_buy_value'] < 0,
            'bandar_detected': self.detect_bandar(metrics, aggregates, period_days),
            'foreign_dominant': aggregates['foreign_net_value'] > abs(aggregates['local_net_value'])
        }

        # 7. Buat result
        result_data = {
            "top_buy_broker": top_buy.to_dict(),
            "top_sell_broker": top_sell.to_dict(),
            "aggregates": aggregates,
            "signals": signals,
            "scores": {"broker_acc_score": acc_score, "rating": rating}
        }

        narrative = generate_broker_summary_narrative(result_data)

        return BrokerSummaryResult(
            stock_code=stock_code,
            trade_date=trade_date,
            period_days=period_days,
            top_buy_broker=top_buy.to_dict(),
            top_sell_broker=top_sell.to_dict(),
            aggregates=aggregates,
            signals=signals,
            scores={"broker_acc_score": acc_score, "rating": rating},
            narrative=narrative
        )
```

## 18. Ide Pengembangan Lanjutan
- **Broker Fingerprinting**: Identifikasi "gaya" trading setiap broker (agresif vs konservatif)
- **Broker Network Graph**: Visualisasi hubungan antar broker (cluster analysis)
- **Broker Momentum Shift Alert**: Notifikasi saat broker besar tiba-tiba flip posisi
- **Historical Win Rate**: Tracking akurasi prediksi broker masa lalu vs pergerakan harga
- **Broker ML Model**: Machine learning untuk prediksi arah harga berdasarkan pola broker

---

# FITUR 2 — BROKER ACTIVITY

## 1. Tujuan Fitur
Melihat aktivitas satu broker di SEMUA saham — broker ini sedang mengoleksi apa saja, membuang apa saja, dan bagaimana perubahan perilakunya dibanding hari-hari sebelumnya.

## 2. Data yang Dibutuhkan
Sama dengan Fitur 1, tapi query dibalik: dari broker_code → semua stock_code

## 3. Struktur Database

```sql
-- View untuk mempercepat query broker activity
CREATE MATERIALIZED VIEW broker_daily_activity AS
SELECT 
    broker_code,
    trade_date,
    COUNT(DISTINCT stock_code)              AS stocks_traded,
    SUM(buy_value)                          AS total_buy_value,
    SUM(sell_value)                         AS total_sell_value,
    SUM(buy_value - sell_value)             AS net_value,
    SUM(buy_volume - sell_volume)           AS net_volume,
    SUM(buy_value + sell_value)             AS total_turnover,
    -- Top stocks being bought
    STRING_AGG(
        CASE WHEN (buy_value - sell_value) > 0 
             THEN stock_code END, ','
        ORDER BY (buy_value - sell_value) DESC
    ) FILTER (WHERE (buy_value - sell_value) > 0) AS top_buy_stocks,
    STRING_AGG(
        CASE WHEN (buy_value - sell_value) < 0 
             THEN stock_code END, ','
        ORDER BY (buy_value - sell_value) ASC
    ) FILTER (WHERE (buy_value - sell_value) < 0) AS top_sell_stocks
FROM broker_transactions
GROUP BY broker_code, trade_date;

-- Tabel broker strength score history
CREATE TABLE broker_strength_scores (
    id              BIGSERIAL PRIMARY KEY,
    broker_code     VARCHAR(10) NOT NULL,
    score_date      DATE NOT NULL,
    strength_score  DECIMAL(5,2),
    total_buy_value BIGINT,
    total_sell_value BIGINT,
    net_value       BIGINT,
    stocks_count    INT,
    score_change_5d DECIMAL(5,2),
    UNIQUE (broker_code, score_date)
);
```

## 4. Rumus Perhitungan

```
Broker Daily Net = Σ(buy_value) - Σ(sell_value) semua saham hari ini
Broker Turnover  = Σ(buy_value) + Σ(sell_value) semua saham
Buy Ratio        = Σ(buy_value) / Broker Turnover
Sell Ratio       = Σ(sell_value) / Broker Turnover

Change vs N days = (today_net - avg_net_N_days) / (avg_net_N_days + 1e-9) × 100%

Concentration per stock = stock_net_value / total_net_value × 100%
```

## 5. Logika Analisa

```
Saham yang dikoleksi broker:
  → Urutkan saham berdasarkan net_buy_value DESC
  → Ambil top 10 dengan net_buy_value > 0

Saham yang dibuang broker:
  → Urutkan saham berdasarkan net_sell_value DESC
  → Ambil top 10 dengan net_sell_value > 0 (jual lebih besar)

Perubahan perilaku:
  → Bandingkan net_value hari ini vs avg 5/10/20 hari
  → Jika naik > 50%: peningkatan signifikan
  → Jika turun > 50%: penurunan signifikan
  → Deteksi flip: dari net_sell kemarin → net_buy hari ini
```

## 6-8. Kondisi Bullish / Bearish / Netral

```
Bullish:
  - Buy Ratio > 65%
  - Net Value positif dan naik vs rata-rata 5 hari
  - Jumlah saham yang dibeli meningkat
  - Broker besar (market cap) masuk

Bearish:
  - Sell Ratio > 65%
  - Net Value negatif dan turun
  - Broker besar keluar dari posisi
  - Penurunan drastis vs rata-rata historis

Netral:
  - Buy Ratio 45-55%
  - Net Value mendekati 0
  - Tidak ada perubahan signifikan
```

## 9. Broker Strength Score (0-100)

```
A. Net Value Ratio Score (35%)
   → buy_ratio * 100 * 0.35

B. Momentum Score (25%)
   → change_vs_5d dalam range -100% to +100%, normalize ke 0-100
   → score = (change_vs_5d + 100) / 2 * 0.25

C. Activity Breadth Score (20%)
   → Banyaknya saham yang dibeli vs dijual
   → raw = (stocks_bought / (stocks_bought + stocks_sold)) * 100
   → score = raw * 0.20

D. Consistency Score (20%)
   → Jumlah hari net positif dalam 10 hari terakhir
   → raw = (positive_days / 10) * 100
   → score = raw * 0.20

TOTAL = A + B + C + D
```

## 15. Format Output JSON

```json
{
  "broker_code": "BK",
  "broker_name": "J.P. Morgan Securities",
  "broker_type": "FOREIGN",
  "date": "2025-08-15",
  "activity": {
    "total_buy_value": 45000000000,
    "total_sell_value": 28000000000,
    "net_value": 17000000000,
    "net_volume": 85000000,
    "buy_ratio": 0.616,
    "stocks_traded": 45,
    "top_buy_stocks": ["BBCA", "TLKM", "ASII", "BMRI", "UNVR"],
    "top_sell_stocks": ["GOTO", "BUKA", "ARTO"]
  },
  "behavioral_change": {
    "vs_5d_avg_pct": 42.5,
    "vs_10d_avg_pct": 28.3,
    "vs_20d_avg_pct": 15.7,
    "trend": "INCREASING",
    "flip_detected": false
  },
  "strength_score": 71.4,
  "rating": "Buy"
}
```

## 17. Pseudocode Python

```python
class BrokerActivityEngine:
    
    def fetch_broker_activity(self, broker_code: str, 
                               date: str, lookback: int = 20) -> pd.DataFrame:
        query = """
            SELECT bt.stock_code, bt.trade_date,
                   bt.buy_value, bt.sell_value,
                   bt.buy_volume, bt.sell_volume
            FROM broker_transactions bt
            WHERE bt.broker_code = %s
              AND bt.trade_date BETWEEN DATE_SUB(%s, INTERVAL %s DAY) AND %s
        """
        return pd.read_sql(query, self.db, params=[broker_code, date, lookback, date])

    def compute_daily_summary(self, df: pd.DataFrame) -> pd.DataFrame:
        daily = df.groupby('trade_date').agg(
            total_buy_value=('buy_value', 'sum'),
            total_sell_value=('sell_value', 'sum'),
            total_buy_volume=('buy_volume', 'sum'),
            total_sell_volume=('sell_volume', 'sum'),
            stocks_count=('stock_code', 'nunique')
        ).reset_index()
        
        daily['net_value'] = daily['total_buy_value'] - daily['total_sell_value']
        daily['buy_ratio'] = daily['total_buy_value'] / (
            daily['total_buy_value'] + daily['total_sell_value'] + 1e-9
        )
        return daily.sort_values('trade_date')

    def compute_behavioral_change(self, daily: pd.DataFrame) -> dict:
        today = daily.iloc[-1]
        
        def pct_change_vs_avg(n):
            if len(daily) < n + 1:
                return 0
            hist_avg = daily.iloc[-(n+1):-1]['net_value'].mean()
            if hist_avg == 0:
                return 0
            return (today['net_value'] - hist_avg) / abs(hist_avg) * 100

        yesterday = daily.iloc[-2] if len(daily) >= 2 else None
        flip_detected = (
            yesterday is not None and
            np.sign(today['net_value']) != np.sign(yesterday['net_value']) and
            abs(today['net_value']) > 1e9
        )

        return {
            'vs_5d_avg_pct': round(pct_change_vs_avg(5), 2),
            'vs_10d_avg_pct': round(pct_change_vs_avg(10), 2),
            'vs_20d_avg_pct': round(pct_change_vs_avg(20), 2),
            'trend': 'INCREASING' if pct_change_vs_avg(5) > 10 else
                     'DECREASING' if pct_change_vs_avg(5) < -10 else 'STABLE',
            'flip_detected': flip_detected
        }

    def get_top_stocks(self, df: pd.DataFrame, date: str, n: int = 10) -> dict:
        today_df = df[df['trade_date'] == date].copy()
        today_df['net_value'] = today_df['buy_value'] - today_df['sell_value']
        
        top_buy = today_df.nlargest(n, 'net_value')[['stock_code', 'net_value']]
        top_sell = today_df.nsmallest(n, 'net_value')[['stock_code', 'net_value']]
        
        return {
            'top_buy_stocks': top_buy.to_dict('records'),
            'top_sell_stocks': top_sell[top_sell['net_value'] < 0].to_dict('records')
        }

    def compute_strength_score(self, daily: pd.DataFrame, 
                                behavioral: dict) -> float:
        if daily.empty:
            return 0

        today = daily.iloc[-1]
        
        # A. Net Value Ratio
        score_a = today['buy_ratio'] * 100 * 0.35

        # B. Momentum
        momentum_raw = (behavioral['vs_5d_avg_pct'] + 100) / 2
        score_b = min(max(momentum_raw, 0), 100) * 0.25

        # C. Activity Breadth (simplified)
        stocks_ratio = min(today['stocks_count'] / 50, 1.0) * 100
        score_c = stocks_ratio * 0.20

        # D. Consistency
        recent_20 = daily.tail(20)
        positive_days = (recent_20['net_value'] > 0).sum()
        score_d = (positive_days / min(len(recent_20), 20)) * 100 * 0.20

        return round(score_a + score_b + score_c + score_d, 2)

    def analyze(self, broker_code: str, date: str) -> dict:
        df = self.fetch_broker_activity(broker_code, date)
        daily = self.compute_daily_summary(df)
        behavioral = self.compute_behavioral_change(daily)
        top_stocks = self.get_top_stocks(df, date)
        strength_score = self.compute_strength_score(daily, behavioral)
        
        today = daily[daily['trade_date'] == date].iloc[0]

        return {
            "broker_code": broker_code,
            "date": date,
            "activity": {
                "total_buy_value": int(today['total_buy_value']),
                "total_sell_value": int(today['total_sell_value']),
                "net_value": int(today['net_value']),
                "buy_ratio": round(today['buy_ratio'], 3),
                "stocks_traded": int(today['stocks_count']),
                **top_stocks
            },
            "behavioral_change": behavioral,
            "strength_score": strength_score,
            "rating": self._get_rating(strength_score)
        }

    def _get_rating(self, score: float) -> str:
        if score >= 81: return "Strong Buy"
        elif score >= 61: return "Buy"
        elif score >= 41: return "Hold"
        elif score >= 21: return "Sell"
        else: return "Strong Sell"
```

---

# FITUR 3 — BROKER ACCUMULATION

## 1. Tujuan Fitur
Mengidentifikasi broker yang konsisten melakukan akumulasi dalam berbagai timeframe (1, 3, 5, 10, 20, 60 hari) dan mendeteksi tren, kecepatan, serta konsistensi akumulasi.

## 2. Data yang Dibutuhkan
Data historis transaksi broker minimal 60 hari ke belakang.

## 3. Struktur Database

```sql
CREATE TABLE broker_accumulation (
    id              BIGSERIAL PRIMARY KEY,
    stock_code      VARCHAR(10) NOT NULL,
    broker_code     VARCHAR(10) NOT NULL,
    calc_date       DATE NOT NULL,
    
    -- Akumulasi per periode
    acc_1d          BIGINT,    -- net value 1 hari
    acc_3d          BIGINT,    -- net value 3 hari
    acc_5d          BIGINT,    -- net value 5 hari
    acc_10d         BIGINT,    -- net value 10 hari
    acc_20d         BIGINT,    -- net value 20 hari
    acc_60d         BIGINT,    -- net value 60 hari
    
    -- Trend metrics
    acc_trend       ENUM('ACCELERATING', 'DECELERATING', 'STABLE', 'REVERSING'),
    acc_speed       DECIMAL(10,4),   -- rate of change per day
    acc_consistency DECIMAL(5,2),    -- % hari positif dalam 20 hari
    
    accumulation_score DECIMAL(5,2),
    UNIQUE (stock_code, broker_code, calc_date)
);
```

## 4. Rumus Perhitungan

```
Akumulasi N hari = Σ(net_buy_value dari D-N hingga D)

Trend Akumulasi:
  → Slope linear dari [acc_1d, acc_3d, acc_5d, acc_10d, acc_20d]
  → Positif & meningkat = ACCELERATING
  → Positif & menurun = DECELERATING
  → Bolak-balik tanda = REVERSING
  → Stabil = STABLE

Kecepatan Akumulasi (per hari):
  speed = (acc_5d - acc_1d) / 4  (rata-rata tambahan per hari)

Konsistensi Akumulasi:
  consistency = (hari dengan net_buy > 0) / total_hari × 100%

Normalized Accumulation:
  norm_acc = acc_Nd / (avg_daily_volume × avg_price × N)
  Ini membandingkan akumulasi vs ukuran normal pasar
```

## 5. Logika Analisa

```
Fase Akumulasi:
  EARLY    → acc_3d > 0 tapi acc_10d ≈ 0 (baru mulai)
  ACTIVE   → acc_5d > 0, acc_10d > 0, consistency > 60%
  HEAVY    → acc_20d besar, speed meningkat, semua periode positif
  LATE     → acc_60d sangat besar, speed mulai turun

Akumulasi Diam-diam (Stealth):
  → Volume beli tidak jauh di atas average
  → Harga tidak naik signifikan
  → Tapi acc_20d terus bertambah

Akumulasi Agresif:
  → Volume beli jauh di atas average
  → Speed tinggi
  → Terjadi dalam window pendek
```

## 9. Accumulation Score (0-100)

```python
def compute_accumulation_score(acc_data: dict, avg_daily_value: float) -> float:
    """
    acc_data: {'acc_1d', 'acc_3d', 'acc_5d', 'acc_10d', 'acc_20d', 'acc_60d'}
    """
    
    # A. Magnitude Score (35%) - seberapa besar akumulasi
    acc_20d_norm = acc_data['acc_20d'] / (avg_daily_value * 20 + 1e-9)
    raw_a = min(acc_20d_norm * 100, 100)
    score_a = max(raw_a, 0) * 0.35
    
    # B. Consistency Score (25%) - seberapa konsisten
    raw_b = acc_data.get('consistency_pct', 50)
    score_b = raw_b * 0.25
    
    # C. Trend Score (20%) - apakah akumulasi meningkat
    periods = [
        acc_data['acc_1d'], acc_data['acc_3d'],
        acc_data['acc_5d'], acc_data['acc_10d']
    ]
    # Linear slope normalized
    x = [1, 3, 5, 10]
    slope = np.polyfit(x, periods, 1)[0]
    raw_c = 50 + min(max(slope / (avg_daily_value + 1e-9) * 500, -50), 50)
    score_c = raw_c * 0.20
    
    # D. Multi-timeframe Alignment (20%)
    # Berapa banyak periode yang positif
    positive_periods = sum(1 for v in [
        acc_data['acc_1d'], acc_data['acc_3d'], acc_data['acc_5d'],
        acc_data['acc_10d'], acc_data['acc_20d'], acc_data['acc_60d']
    ] if v > 0)
    raw_d = (positive_periods / 6) * 100
    score_d = raw_d * 0.20
    
    return round(score_a + score_b + score_c + score_d, 2)
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "broker_code": "BK",
  "calc_date": "2025-08-15",
  "accumulation": {
    "acc_1d":  2500000000,
    "acc_3d":  7800000000,
    "acc_5d":  11200000000,
    "acc_10d": 19500000000,
    "acc_20d": 31000000000,
    "acc_60d": 58000000000
  },
  "analytics": {
    "trend": "ACCELERATING",
    "speed": 850000000,
    "consistency_pct": 75.0,
    "phase": "ACTIVE",
    "stealth": false
  },
  "accumulation_score": 78.5,
  "rating": "Buy",
  "narrative": "J.P. Morgan menunjukkan pola akumulasi aktif dengan konsistensi 75% dalam 20 hari. Kecepatan akumulasi meningkat secara konsisten. Total akumulasi IDR 31M dalam 20 hari terakhir."
}
```
