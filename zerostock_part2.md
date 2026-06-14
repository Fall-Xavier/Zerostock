# ZeroStock Platform — Fitur 4-7

---

# FITUR 4 — FOREIGN FLOW

## 1. Tujuan Fitur
Memantau aliran dana asing secara real-time dan historis. Dana asing sering menjadi "leading indicator" karena investor institusional asing umumnya memiliki riset fundamental lebih dalam dan akses informasi global.

## 2. Data yang Dibutuhkan
| Field | Tipe | Keterangan |
|---|---|---|
| stock_code | VARCHAR | Kode saham |
| trade_date | DATE | Tanggal |
| foreign_buy_volume | BIGINT | Volume beli asing |
| foreign_buy_value | BIGINT | Nilai beli asing |
| foreign_sell_volume | BIGINT | Volume jual asing |
| foreign_sell_value | BIGINT | Nilai jual asing |
| total_market_volume | BIGINT | Total volume pasar |
| total_market_value | BIGINT | Total nilai pasar |
| outstanding_shares | BIGINT | Total saham beredar |
| foreign_ownership_pct | DECIMAL | % kepemilikan asing (dari KSEI) |

## 3. Struktur Database

```sql
CREATE TABLE foreign_flow (
    id                      BIGSERIAL PRIMARY KEY,
    stock_code              VARCHAR(10) NOT NULL,
    trade_date              DATE NOT NULL,
    foreign_buy_volume      BIGINT DEFAULT 0,
    foreign_buy_value       BIGINT DEFAULT 0,
    foreign_sell_volume     BIGINT DEFAULT 0,
    foreign_sell_value      BIGINT DEFAULT 0,
    foreign_net_volume      BIGINT GENERATED ALWAYS AS (foreign_buy_volume - foreign_sell_volume) STORED,
    foreign_net_value       BIGINT GENERATED ALWAYS AS (foreign_buy_value - foreign_sell_value) STORED,
    total_market_volume     BIGINT,
    total_market_value      BIGINT,
    UNIQUE (stock_code, trade_date),
    INDEX idx_stock_date (stock_code, trade_date)
);

CREATE TABLE foreign_flow_aggregated (
    id                  BIGSERIAL PRIMARY KEY,
    stock_code          VARCHAR(10) NOT NULL,
    calc_date           DATE NOT NULL,
    period_days         INT NOT NULL,
    cum_net_value       BIGINT,
    cum_net_volume      BIGINT,
    foreign_ratio       DECIMAL(5,4),
    dominance_score     DECIMAL(5,2),
    trend_direction     ENUM('INFLOW', 'OUTFLOW', 'NEUTRAL'),
    trend_strength      DECIMAL(5,2),
    foreign_strength_score DECIMAL(5,2),
    UNIQUE (stock_code, calc_date, period_days)
);
```

## 4. Rumus Perhitungan

```
Foreign Net Volume = foreign_buy_volume - foreign_sell_volume
Foreign Net Value  = foreign_buy_value  - foreign_sell_value

Foreign Ratio (partisipasi):
  foreign_ratio = (foreign_buy_value + foreign_sell_value) / (total_market_value + 1e-9)

Foreign Dominance:
  dominance = foreign_net_value / (total_market_value + 1e-9)
  → Positif = asing net buyer
  → Negatif = asing net seller

Foreign Buy Ratio:
  buy_ratio = foreign_buy_value / (foreign_buy_value + foreign_sell_value + 1e-9)

Foreign Trend (Moving Average):
  trend_5d  = avg(foreign_net_value) 5 hari
  trend_20d = avg(foreign_net_value) 20 hari
  trend_direction: trend_5d vs trend_20d

Cumulative Foreign Flow:
  cum_net_N = Σ(foreign_net_value) N hari ke belakang

Foreign Ownership Change:
  ownership_change = (today_foreign_net_volume / outstanding_shares) × 100%
```

## 5. Logika Analisa

```
Foreign Inflow:
  → foreign_net_value > 0 DAN positif ≥ 3 dari 5 hari terakhir

Foreign Outflow:
  → foreign_net_value < 0 DAN negatif ≥ 3 dari 5 hari terakhir

Foreign Dominance:
  → foreign_ratio > 0.5 (asing menguasai > 50% transaksi)
  
Foreign Trend (3 level):
  STRONG_INFLOW  → dominance > 0.02 DAN trend_5d > trend_20d
  WEAK_INFLOW    → dominance > 0 DAN trend_5d < trend_20d
  NEUTRAL        → |dominance| < 0.005
  WEAK_OUTFLOW   → dominance < 0 DAN trend_5d > trend_20d
  STRONG_OUTFLOW → dominance < -0.02 DAN trend_5d < trend_20d

Smart Foreign Detection:
  → Asing beli di harga rendah DAN jual di harga tinggi = well-timed
  → Hitung win_rate asing dalam 60 hari
```

## 6. Kondisi Bullish
- Foreign Net positif 5 hari berturut-turut
- Foreign Ratio meningkat (makin aktif)
- Foreign membeli di saat harga koreksi (smart buying)
- Cumulative flow positif dan meningkat
- Foreign buy ratio > 60%

## 7. Kondisi Bearish
- Foreign Net negatif 5 hari berturut-turut
- Foreign Dominance negatif besar
- Asing jual saat harga masih tinggi (distribusi)
- Cumulative flow negatif dan memburuk
- Foreign sell ratio > 60%

## 8. Kondisi Netral
- |Foreign Net| < 1% total nilai transaksi
- Tidak ada tren jelas 5-10 hari
- Buy ratio mendekati 50%

## 9. Foreign Strength Score (0-100)

```python
def compute_foreign_strength_score(
    flow_data: pd.DataFrame,   # data 20 hari
    current_price: float
) -> float:
    
    today = flow_data.iloc[-1]
    
    # A. Net Flow Score (30%)
    total_mkt = today['total_market_value']
    dominance = today['foreign_net_value'] / (total_mkt + 1e-9)
    raw_a = 50 + min(max(dominance * 2000, -50), 50)
    score_a = raw_a * 0.30
    
    # B. Consistency Score (25%)
    last_10 = flow_data.tail(10)
    positive_days = (last_10['foreign_net_value'] > 0).sum()
    score_b = (positive_days / 10) * 100 * 0.25
    
    # C. Trend Alignment Score (20%)
    trend_5d = flow_data.tail(5)['foreign_net_value'].mean()
    trend_20d = flow_data.tail(20)['foreign_net_value'].mean()
    if trend_5d > 0 and trend_20d > 0:
        raw_c = 80
    elif trend_5d > 0 and trend_20d < 0:
        raw_c = 60
    elif trend_5d < 0 and trend_20d > 0:
        raw_c = 40
    else:
        raw_c = 20
    score_c = raw_c * 0.20
    
    # D. Participation Score (15%)
    buy_ratio = today['foreign_buy_value'] / (
        today['foreign_buy_value'] + today['foreign_sell_value'] + 1e-9
    )
    score_d = buy_ratio * 100 * 0.15
    
    # E. Magnitude vs Market (10%)
    foreign_ratio = (today['foreign_buy_value'] + today['foreign_sell_value']) / (
        today['total_market_value'] + 1e-9
    )
    raw_e = min(foreign_ratio * 200, 100)
    score_e = raw_e * 0.10
    
    return round(score_a + score_b + score_c + score_d + score_e, 2)
```

## 11. Sistem Deteksi Bandar (Foreign Smart Money)

```
Foreign Smart Money Detected jika:
  1. foreign_net_value > 2x rata-rata harian
  2. Pembelian terjadi saat harga turun (buy on dip)
  3. Foreign ratio meningkat > 30% hari ini vs kemarin
  4. Cumulative 5D positif dan meningkat
  5. Foreign buy avg < harga saat ini
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "trade_date": "2025-08-15",
  "foreign_flow": {
    "today": {
      "buy_volume": 8500000,
      "buy_value": 6715000000,
      "sell_volume": 3200000,
      "sell_value": 2528000000,
      "net_volume": 5300000,
      "net_value": 4187000000,
      "buy_ratio": 0.726,
      "foreign_ratio": 0.148,
      "dominance": 0.0185
    },
    "cumulative": {
      "cum_5d_net_value": 18500000000,
      "cum_10d_net_value": 32000000000,
      "cum_20d_net_value": 45000000000
    },
    "trend": {
      "trend_5d_avg": 3700000000,
      "trend_20d_avg": 2250000000,
      "direction": "STRONG_INFLOW",
      "strength": 0.78
    },
    "signals": {
      "smart_money_detected": true,
      "consecutive_inflow_days": 6,
      "foreign_dominant": true
    },
    "foreign_strength_score": 76.2,
    "rating": "Buy",
    "narrative": "Asing melakukan net buy IDR 4.19M hari ini dengan buy ratio 72.6%. Inflow kumulatif 20 hari mencapai IDR 45M. Tren jangka pendek (5D) lebih kuat dari tren panjang (20D), mengindikasikan akselerasi masuknya dana asing."
  }
}
```

## 17. Pseudocode Python

```python
class ForeignFlowEngine:

    def fetch_flow_data(self, stock_code: str, 
                         date: str, days: int = 20) -> pd.DataFrame:
        query = """
            SELECT * FROM foreign_flow
            WHERE stock_code = %s
              AND trade_date BETWEEN DATE_SUB(%s, INTERVAL %s DAY) AND %s
            ORDER BY trade_date
        """
        return pd.read_sql(query, self.db, params=[stock_code, date, days, date])

    def compute_cumulative(self, df: pd.DataFrame) -> dict:
        return {
            'cum_5d': df.tail(5)['foreign_net_value'].sum(),
            'cum_10d': df.tail(10)['foreign_net_value'].sum(),
            'cum_20d': df.tail(20)['foreign_net_value'].sum()
        }

    def detect_trend(self, df: pd.DataFrame) -> dict:
        trend_5d = df.tail(5)['foreign_net_value'].mean()
        trend_20d = df.tail(20)['foreign_net_value'].mean()

        if trend_5d > 0 and trend_20d > 0:
            direction = 'STRONG_INFLOW' if trend_5d > trend_20d else 'WEAK_INFLOW'
        elif trend_5d < 0 and trend_20d < 0:
            direction = 'STRONG_OUTFLOW' if trend_5d < trend_20d else 'WEAK_OUTFLOW'
        else:
            direction = 'NEUTRAL'

        strength = abs(trend_5d) / (abs(trend_20d) + 1e-9)
        return {
            'trend_5d_avg': trend_5d,
            'trend_20d_avg': trend_20d,
            'direction': direction,
            'strength': round(min(strength, 2.0), 2)
        }

    def detect_smart_foreign(self, df: pd.DataFrame, 
                              today_price: float,
                              yesterday_price: float) -> bool:
        today = df.iloc[-1]
        avg_net = df['foreign_net_value'].mean()
        
        conditions = [
            today['foreign_net_value'] > 2 * avg_net,
            today_price < yesterday_price,  # beli saat harga turun
            today['foreign_buy_value'] / (today['foreign_buy_value'] + 
                today['foreign_sell_value'] + 1e-9) > 0.65,
            df.tail(5)['foreign_net_value'].sum() > 0
        ]
        return sum(conditions) >= 3

    def analyze(self, stock_code: str, date: str,
                current_price: float, yesterday_price: float) -> dict:
        df = self.fetch_flow_data(stock_code, date)
        cumulative = self.compute_cumulative(df)
        trend = self.detect_trend(df)
        smart = self.detect_smart_foreign(df, current_price, yesterday_price)
        score = compute_foreign_strength_score(df, current_price)

        today = df.iloc[-1]
        consecutive = 0
        for _, row in df.iloc[::-1].iterrows():
            if row['foreign_net_value'] > 0:
                consecutive += 1
            else:
                break

        return {
            "stock_code": stock_code,
            "trade_date": date,
            "today_net_value": int(today['foreign_net_value']),
            "cumulative": cumulative,
            "trend": trend,
            "signals": {
                "smart_money_detected": smart,
                "consecutive_inflow_days": consecutive,
                "foreign_dominant": today['foreign_net_value'] > 0
            },
            "foreign_strength_score": score,
            "rating": self._get_rating(score)
        }
```

---

# FITUR 5 — ORDERBOOK ANALYSIS

## 1. Tujuan Fitur
Menganalisis supply dan demand secara real-time dari data antrian beli/jual, mendeteksi tekanan pasar, hidden accumulation, dan hidden distribution.

## 2. Data yang Dibutuhkan
| Field | Tipe | Keterangan |
|---|---|---|
| stock_code | VARCHAR | Kode saham |
| snapshot_time | TIMESTAMP | Waktu snapshot |
| bid_price_1..5 | DECIMAL | Harga bid 5 level |
| bid_volume_1..5 | BIGINT | Volume bid per level |
| bid_lot_1..5 | INT | Lot bid per level |
| offer_price_1..5 | DECIMAL | Harga offer 5 level |
| offer_volume_1..5 | BIGINT | Volume offer per level |
| offer_lot_1..5 | INT | Lot offer per level |
| last_price | DECIMAL | Harga terakhir |
| vwap | DECIMAL | VWAP hari ini |

## 3. Struktur Database

```sql
-- Tabel orderbook snapshot (high-frequency, perlu partisi)
CREATE TABLE orderbook_snapshots (
    id              BIGSERIAL,
    stock_code      VARCHAR(10) NOT NULL,
    snapshot_time   TIMESTAMP NOT NULL,
    last_price      DECIMAL(12,2),
    vwap            DECIMAL(12,2),
    
    -- 5 Level Bid
    bid_price_1     DECIMAL(12,2), bid_vol_1 BIGINT, bid_lot_1 INT,
    bid_price_2     DECIMAL(12,2), bid_vol_2 BIGINT, bid_lot_2 INT,
    bid_price_3     DECIMAL(12,2), bid_vol_3 BIGINT, bid_lot_3 INT,
    bid_price_4     DECIMAL(12,2), bid_vol_4 BIGINT, bid_lot_4 INT,
    bid_price_5     DECIMAL(12,2), bid_vol_5 BIGINT, bid_lot_5 INT,
    
    -- 5 Level Offer
    offer_price_1   DECIMAL(12,2), offer_vol_1 BIGINT, offer_lot_1 INT,
    offer_price_2   DECIMAL(12,2), offer_vol_2 BIGINT, offer_lot_2 INT,
    offer_price_3   DECIMAL(12,2), offer_vol_3 BIGINT, offer_lot_3 INT,
    offer_price_4   DECIMAL(12,2), offer_vol_4 BIGINT, offer_lot_4 INT,
    offer_price_5   DECIMAL(12,2), offer_vol_5 BIGINT, offer_lot_5 INT,
    
    -- Computed
    total_bid_vol   BIGINT GENERATED ALWAYS AS (
        COALESCE(bid_vol_1,0) + COALESCE(bid_vol_2,0) + COALESCE(bid_vol_3,0) +
        COALESCE(bid_vol_4,0) + COALESCE(bid_vol_5,0)
    ) STORED,
    total_offer_vol BIGINT GENERATED ALWAYS AS (
        COALESCE(offer_vol_1,0) + COALESCE(offer_vol_2,0) + COALESCE(offer_vol_3,0) +
        COALESCE(offer_vol_4,0) + COALESCE(offer_vol_5,0)
    ) STORED,
    
    PRIMARY KEY (id, snapshot_time)
) PARTITION BY RANGE (snapshot_time);

-- Agregat harian orderbook
CREATE TABLE orderbook_daily (
    id                  BIGSERIAL PRIMARY KEY,
    stock_code          VARCHAR(10) NOT NULL,
    trade_date          DATE NOT NULL,
    avg_bid_offer_ratio DECIMAL(8,4),
    avg_demand_pressure DECIMAL(8,4),
    avg_supply_pressure DECIMAL(8,4),
    max_bid_vol         BIGINT,
    max_offer_vol       BIGINT,
    hidden_acc_signal   BOOLEAN DEFAULT FALSE,
    hidden_dist_signal  BOOLEAN DEFAULT FALSE,
    orderbook_score     DECIMAL(5,2),
    UNIQUE (stock_code, trade_date)
);
```

## 4. Rumus Perhitungan

```
Total Bid Volume  = Σ(bid_vol_1 to bid_vol_5)
Total Offer Volume = Σ(offer_vol_1 to offer_vol_5)

Bid/Offer Ratio = Total Bid Volume / (Total Offer Volume + 1e-9)
  → > 1.5: demand sangat dominan
  → 1.0-1.5: demand sedikit dominan
  → 0.67-1.0: supply sedikit dominan
  → < 0.67: supply sangat dominan

Demand Pressure = Total Bid Volume / (Total Bid Volume + Total Offer Volume)
Supply Pressure = Total Offer Volume / (Total Bid Volume + Total Offer Volume)

Weighted Bid Value  = Σ(bid_price_i × bid_vol_i)
Weighted Offer Value = Σ(offer_price_i × offer_vol_i)

Value Ratio = Weighted Bid Value / (Weighted Offer Value + 1e-9)

Order Imbalance = (Total Bid Vol - Total Offer Vol) / 
                  (Total Bid Vol + Total Offer Vol)
  → Range: -1 to +1
  → Positif: demand dominan
  → Negatif: supply dominan

Spread = Offer Price 1 - Bid Price 1
Spread % = Spread / ((Offer Price 1 + Bid Price 1) / 2) × 100%
```

## 5. Logika Analisa — Hidden Patterns

```
Hidden Accumulation:
  → Offer volume besar di level 1 TAPI cepat habis (dimakan)
  → Bid muncul besar setelah offer terjual
  → Lot kecil tapi frekuensi tinggi di bid
  → Harga tidak naik meski bid besar (artificial suppression)
  
  Deteksi:
  bid_lot_1 banyak tapi bid_volume_1 kecil per lot
  = banyak antrian kecil = retail atau algo accumulation

Hidden Distribution:
  → Bid besar di level 1 TAPI ketika ditest, tidak benar-benar beli
  → Offer muncul berulang kali di harga yang sama
  → Volume offer selalu muncul kembali setelah dimakan
  → Harga tidak naik meski bid terlihat besar

Order Splitting:
  → Lot kecil merata = algoritma distribusi
  → Lot besar sporadis = institusional agresif

Iceberg Orders:
  → Volume kecil di antrian tapi total transaksi jauh lebih besar
  = ada order tersembunyi
```

## 6-8. Kondisi Bullish / Bearish / Netral

```
Bullish:
  - Bid/Offer Ratio > 1.5 DAN stabil > 3 snapshot
  - Order Imbalance > 0.3
  - Spread menyempit (buyer agresif)
  - Bid volume > 2x rata-rata historis
  - Hidden accumulation terdeteksi

Bearish:
  - Bid/Offer Ratio < 0.67 secara konsisten
  - Order Imbalance < -0.3
  - Spread melebar (panic selling)
  - Offer volume terus muncul kembali setelah dimakan
  - Hidden distribution terdeteksi

Netral:
  - Bid/Offer Ratio 0.8-1.2
  - Order Imbalance mendekati 0
  - Volume stabil di kedua sisi
```

## 9. Orderbook Score (0-100)

```python
def compute_orderbook_score(ob_data: dict, historical_avg: dict) -> float:
    """
    ob_data: snapshot terkini
    historical_avg: rata-rata 10 snapshot terakhir
    """
    total_bid = ob_data['total_bid_vol']
    total_offer = ob_data['total_offer_vol']
    
    # A. Bid/Offer Ratio Score (35%)
    bo_ratio = total_bid / (total_offer + 1e-9)
    # Normalize: ratio 2.0 → skor 100, ratio 0.5 → skor 0
    raw_a = min(max((bo_ratio - 0.5) / 1.5 * 100, 0), 100)
    score_a = raw_a * 0.35
    
    # B. Order Imbalance Score (25%)
    imbalance = (total_bid - total_offer) / (total_bid + total_offer + 1e-9)
    raw_b = (imbalance + 1) / 2 * 100  # Normalize -1..1 ke 0..100
    score_b = raw_b * 0.25
    
    # C. Demand Pressure vs Historical (20%)
    hist_bid = historical_avg.get('avg_bid_vol', total_bid)
    demand_pressure = total_bid / (hist_bid + 1e-9)
    raw_c = min(demand_pressure * 50, 100)
    score_c = raw_c * 0.20
    
    # D. Hidden Signal Bonus (10%)
    hidden_acc = detect_hidden_accumulation(ob_data)
    hidden_dist = detect_hidden_distribution(ob_data)
    if hidden_acc:
        raw_d = 80
    elif hidden_dist:
        raw_d = 20
    else:
        raw_d = 50
    score_d = raw_d * 0.10
    
    # E. Spread Score (10%) - spread kecil = aktif, positif
    spread_pct = ob_data.get('spread_pct', 0.5)
    raw_e = max(100 - spread_pct * 100, 0)
    score_e = raw_e * 0.10
    
    return round(score_a + score_b + score_c + score_d + score_e, 2)


def detect_hidden_accumulation(ob_data: dict) -> bool:
    """
    Deteksi akumulasi tersembunyi dari pola orderbook
    """
    total_bid = ob_data['total_bid_vol']
    total_offer = ob_data['total_offer_vol']
    
    # Bid besar secara volume tapi lot kecil (institusional diam-diam)
    avg_bid_lot = total_bid / (ob_data.get('bid_lot_total', 1) + 1e-9)
    
    conditions = [
        total_bid > total_offer * 1.2,
        ob_data.get('bid_vol_1', 0) > ob_data.get('offer_vol_1', 0) * 1.5,
        avg_bid_lot > 500  # Beli dalam jumlah besar per order
    ]
    return sum(conditions) >= 2


def detect_hidden_distribution(ob_data: dict) -> bool:
    """
    Deteksi distribusi tersembunyi dari pola orderbook  
    """
    total_offer = ob_data['total_offer_vol']
    total_bid = ob_data['total_bid_vol']
    
    conditions = [
        total_offer > total_bid * 1.2,
        ob_data.get('offer_vol_1', 0) > ob_data.get('bid_vol_1', 0) * 1.5,
        ob_data.get('spread_pct', 0) > 0.5  # Spread lebar = seller dominan
    ]
    return sum(conditions) >= 2
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "snapshot_time": "2025-08-15T10:30:00",
  "orderbook": {
    "bid": [
      {"price": 9975, "volume": 825000, "lot": 1650},
      {"price": 9950, "volume": 650000, "lot": 1300},
      {"price": 9925, "volume": 480000, "lot": 960},
      {"price": 9900, "volume": 320000, "lot": 640},
      {"price": 9875, "volume": 215000, "lot": 430}
    ],
    "offer": [
      {"price": 10000, "volume": 290000, "lot": 580},
      {"price": 10025, "volume": 180000, "lot": 360},
      {"price": 10050, "volume": 150000, "lot": 300},
      {"price": 10075, "volume": 125000, "lot": 250},
      {"price": 10100, "volume": 95000,  "lot": 190}
    ]
  },
  "metrics": {
    "total_bid_vol": 2490000,
    "total_offer_vol": 840000,
    "bid_offer_ratio": 2.964,
    "order_imbalance": 0.497,
    "demand_pressure": 0.748,
    "supply_pressure": 0.252,
    "spread": 25,
    "spread_pct": 0.25
  },
  "signals": {
    "hidden_accumulation": true,
    "hidden_distribution": false,
    "dominant_side": "BID",
    "pressure": "STRONG_DEMAND"
  },
  "orderbook_score": 82.4,
  "rating": "Strong Buy",
  "narrative": "Demand sangat kuat dengan bid/offer ratio 2.96x. Order imbalance +0.497 menunjukkan tekanan beli signifikan. Terdeteksi pola hidden accumulation dengan lot besar di antrian beli. Spread sempit 0.25% menandakan aktivitas tinggi."
}
```

---

# FITUR 6 — SMART MONEY ANALYSIS

## 1. Tujuan Fitur
Menggabungkan semua sinyal (Broker, Foreign, Orderbook, Volume, Harga) untuk mendeteksi apakah "smart money" sedang masuk, keluar, akumulasi, atau distribusi pada suatu saham.

## 2. Data yang Dibutuhkan
Gabungan output dari Fitur 1-5 + data OHLCV

## 3. Struktur Database

```sql
CREATE TABLE smart_money_signals (
    id                  BIGSERIAL PRIMARY KEY,
    stock_code          VARCHAR(10) NOT NULL,
    signal_date         DATE NOT NULL,
    
    -- Component scores
    broker_acc_score    DECIMAL(5,2),
    foreign_score       DECIMAL(5,2),
    orderbook_score     DECIMAL(5,2),
    volume_score        DECIMAL(5,2),
    price_action_score  DECIMAL(5,2),
    
    -- Signal detection
    sm_entering         BOOLEAN DEFAULT FALSE,
    sm_exiting          BOOLEAN DEFAULT FALSE,
    sm_accumulating     BOOLEAN DEFAULT FALSE,
    sm_distributing     BOOLEAN DEFAULT FALSE,
    sm_confidence       DECIMAL(5,2),           -- 0-100
    
    smart_money_score   DECIMAL(5,2),           -- 0-100
    signal_type         VARCHAR(20),            -- BULLISH, BEARISH, NEUTRAL
    UNIQUE (stock_code, signal_date)
);
```

## 4. Rumus dan Logika Utama

```python
class SmartMoneyEngine:
    
    # Bobot komponen Smart Money Score
    WEIGHTS = {
        'broker': 0.30,
        'foreign': 0.25,
        'orderbook': 0.20,
        'volume': 0.15,
        'price_action': 0.10
    }

    def compute_volume_score(self, ohlcv: pd.DataFrame) -> float:
        """
        Volume Score: Apakah volume hari ini abnormal positif?
        """
        today = ohlcv.iloc[-1]
        avg_20d = ohlcv.tail(21).iloc[:-1]['volume'].mean()
        std_20d = ohlcv.tail(21).iloc[:-1]['volume'].std()
        
        volume_zscore = (today['volume'] - avg_20d) / (std_20d + 1e-9)
        
        # Volume spike saat naik = bullish, saat turun = bearish
        if today['close'] > today['open']:
            raw = 50 + min(volume_zscore * 15, 50)
        else:
            raw = 50 - min(volume_zscore * 15, 50)
        
        return min(max(raw, 0), 100)

    def compute_price_action_score(self, ohlcv: pd.DataFrame) -> float:
        """
        Price Action Score berdasarkan candlestick dan posisi harga
        """
        today = ohlcv.iloc[-1]
        high, low, open_, close = today['high'], today['low'], today['open'], today['close']
        
        # 1. Body size
        body = abs(close - open_)
        shadow = high - low
        body_ratio = body / (shadow + 1e-9)
        
        # 2. Candlestick bias
        if close > open_:  # Bullish candle
            upper_shadow = high - close
            lower_shadow = open_ - low
            if lower_shadow > upper_shadow:  # Hammer
                raw = 75
            else:
                raw = 60 + body_ratio * 30
        else:  # Bearish candle
            upper_shadow = high - open_
            lower_shadow = close - low
            if upper_shadow > lower_shadow:  # Shooting star
                raw = 25
            else:
                raw = 40 - body_ratio * 30
        
        # 3. Gap factor
        yesterday = ohlcv.iloc[-2]
        if close > yesterday['close']:
            raw += 5
        else:
            raw -= 5
        
        return min(max(raw, 0), 100)

    def detect_smart_money_entering(self, components: dict, 
                                     ohlcv: pd.DataFrame) -> bool:
        """
        Smart Money Masuk:
        - Volume spike
        - Harga naik atau stabil
        - Broker net buy
        - Foreign net positif
        - Orderbook bid dominan
        """
        today = ohlcv.iloc[-1]
        avg_vol = ohlcv.tail(20)['volume'].mean()
        
        conditions = [
            today['volume'] > avg_vol * 1.5,                    # Volume spike
            today['close'] >= today['open'],                    # Price bullish
            components['broker_acc_score'] > 60,                # Broker beli
            components['foreign_score'] > 55,                   # Foreign masuk
            components['orderbook_score'] > 60                  # Demand dominan
        ]
        return sum(conditions) >= 3

    def detect_smart_money_exiting(self, components: dict,
                                    ohlcv: pd.DataFrame) -> bool:
        """
        Smart Money Keluar:
        - Volume tinggi tapi harga tidak naik / turun
        - Broker distribusi
        - Foreign net negatif
        - Orderbook offer dominan
        """
        today = ohlcv.iloc[-1]
        avg_vol = ohlcv.tail(20)['volume'].mean()
        
        conditions = [
            today['volume'] > avg_vol * 1.3,                   # Volume masih tinggi
            today['close'] <= today['open'],                    # Price bearish
            components['broker_acc_score'] < 40,               # Broker jual
            components['foreign_score'] < 45,                  # Foreign keluar
            components['orderbook_score'] < 40                 # Supply dominan
        ]
        return sum(conditions) >= 3

    def detect_accumulation(self, components: dict,
                             broker_acc_5d: float,
                             foreign_cum_5d: float) -> bool:
        """
        Smart Money Akumulasi:
        - Beli konsisten, harga tidak naik besar
        - Net buy volume positif 3+ hari
        - Price sideways atau turun perlahan
        """
        conditions = [
            components['broker_acc_score'] > 65,
            broker_acc_5d > 0,
            foreign_cum_5d > 0,
            components['volume_score'] > 55
        ]
        return sum(conditions) >= 3

    def detect_distribution(self, components: dict,
                             broker_acc_5d: float,
                             foreign_cum_5d: float) -> bool:
        """
        Smart Money Distribusi:
        - Jual konsisten, harga masih relatif tinggi
        - Net sell volume positif 3+ hari
        - Harga naik tapi volume jual meningkat
        """
        conditions = [
            components['broker_acc_score'] < 40,
            broker_acc_5d < 0,
            foreign_cum_5d < 0,
            components['orderbook_score'] < 45
        ]
        return sum(conditions) >= 3

    def compute_smart_money_score(self, components: dict) -> float:
        score = (
            components['broker_acc_score'] * self.WEIGHTS['broker'] +
            components['foreign_score'] * self.WEIGHTS['foreign'] +
            components['orderbook_score'] * self.WEIGHTS['orderbook'] +
            components['volume_score'] * self.WEIGHTS['volume'] +
            components['price_action_score'] * self.WEIGHTS['price_action']
        )
        return round(score, 2)

    def analyze(self, stock_code: str, date: str, 
                broker_result: dict, foreign_result: dict,
                ob_result: dict, ohlcv: pd.DataFrame) -> dict:
        
        components = {
            'broker_acc_score': broker_result['scores']['broker_acc_score'],
            'foreign_score': foreign_result['foreign_strength_score'],
            'orderbook_score': ob_result['orderbook_score'],
            'volume_score': self.compute_volume_score(ohlcv),
            'price_action_score': self.compute_price_action_score(ohlcv)
        }

        sm_score = self.compute_smart_money_score(components)
        
        broker_acc_5d = broker_result['aggregates'].get('total_net_buy_value', 0)
        foreign_cum_5d = foreign_result['cumulative'].get('cum_5d', 0)

        signals = {
            'sm_entering': self.detect_smart_money_entering(components, ohlcv),
            'sm_exiting': self.detect_smart_money_exiting(components, ohlcv),
            'sm_accumulating': self.detect_accumulation(
                components, broker_acc_5d, foreign_cum_5d),
            'sm_distributing': self.detect_distribution(
                components, broker_acc_5d, foreign_cum_5d)
        }

        # Confidence berdasarkan konsensus
        bullish_signals = sum([
            signals['sm_entering'],
            signals['sm_accumulating']
        ])
        bearish_signals = sum([
            signals['sm_exiting'],
            signals['sm_distributing']
        ])
        confidence = abs(bullish_signals - bearish_signals) / 2 * 100

        return {
            "stock_code": stock_code,
            "signal_date": date,
            "component_scores": components,
            "smart_money_score": sm_score,
            "signals": signals,
            "confidence": confidence,
            "signal_type": "BULLISH" if sm_score > 60 else 
                           "BEARISH" if sm_score < 40 else "NEUTRAL",
            "rating": self._get_rating(sm_score)
        }
```

## 15. Format Output JSON

```json
{
  "stock_code": "BBCA",
  "signal_date": "2025-08-15",
  "component_scores": {
    "broker_acc_score": 74.8,
    "foreign_score": 76.2,
    "orderbook_score": 82.4,
    "volume_score": 68.5,
    "price_action_score": 71.0
  },
  "smart_money_score": 75.5,
  "signals": {
    "sm_entering": true,
    "sm_exiting": false,
    "sm_accumulating": true,
    "sm_distributing": false
  },
  "confidence": 100.0,
  "signal_type": "BULLISH",
  "rating": "Buy",
  "narrative": "Smart Money terdeteksi sedang MASUK dan MENGAKUMULASI BBCA. Broker asing net buy dengan volume 1.5x rata-rata. Foreign inflow konsisten 6 hari berturut-turut. Orderbook bid/offer ratio sangat bullish di 2.96x. Seluruh komponen selaras menunjukkan sinyal beli kuat."
}
```

---

# FITUR 7 — BANDARMOLOGY SCORE

## 1. Tujuan Fitur
Menghasilkan skor tunggal 0-100 yang merangkum seluruh analisis bandarmologi untuk menilai apakah ada "bandar" yang sedang menggerakkan saham tersebut dan ke arah mana.

## 2. Komponen dan Bobot

| Komponen | Bobot | Alasan |
|---|---|---|
| Broker Accumulation Score | 30% | Sinyal utama aksi bandar |
| Foreign Flow Score | 25% | Smart money institusional asing |
| Orderbook Score | 20% | Supply/demand real-time |
| Volume Spike Score | 15% | Konfirmasi aktivitas |
| Price Action Score | 10% | Konfirmasi pergerakan |

## 3. Rumus Matematis

```python
def compute_bandarmology_score(
    broker_acc_score: float,    # 0-100
    foreign_score: float,       # 0-100
    orderbook_score: float,     # 0-100
    volume_score: float,        # 0-100
    price_action_score: float   # 0-100
) -> float:
    """
    Bandarmology Score dengan bobot berimbang
    
    Bobot berbasis penelitian:
    - Broker (30%): aktivitas broker adalah sinyal terkuat bandar di BEI
    - Foreign (25%): dana asing sering menjadi smart money leader
    - Orderbook (20%): menangkap demand/supply real-time
    - Volume (15%): konfirmator, volume spike tanpa broker beli = noise
    - Price Action (10%): konfirmator akhir setelah semua sinyal lain
    """
    
    WEIGHTS = {
        'broker': 0.30,
        'foreign': 0.25,
        'orderbook': 0.20,
        'volume': 0.15,
        'price_action': 0.10
    }
    
    raw_score = (
        broker_acc_score * WEIGHTS['broker'] +
        foreign_score * WEIGHTS['foreign'] +
        orderbook_score * WEIGHTS['orderbook'] +
        volume_score * WEIGHTS['volume'] +
        price_action_score * WEIGHTS['price_action']
    )
    
    # Penalty jika komponen utama sangat divergen
    # (misalnya broker beli tapi foreign jual besar = konflik sinyal)
    max_comp = max(broker_acc_score, foreign_score)
    min_comp = min(broker_acc_score, foreign_score)
    divergence = max_comp - min_comp
    
    # Jika divergensi > 40 poin, berikan penalti
    if divergence > 40:
        penalty = (divergence - 40) * 0.5
        raw_score = raw_score - penalty
    
    return round(min(max(raw_score, 0), 100), 2)
```

## 4. Sistem Rating

```python
def get_bandarmology_rating(score: float) -> dict:
    """
    Rating Bandarmology Score
    """
    if score >= 81:
        return {
            "rating": "Sangat Bullish",
            "action": "Strong Buy",
            "confidence": "Sangat Tinggi",
            "emoji": "🚀"
        }
    elif score >= 61:
        return {
            "rating": "Bullish",
            "action": "Buy",
            "confidence": "Tinggi",
            "emoji": "📈"
        }
    elif score >= 41:
        return {
            "rating": "Netral",
            "action": "Hold / Watch",
            "confidence": "Sedang",
            "emoji": "⚖️"
        }
    elif score >= 21:
        return {
            "rating": "Bearish",
            "action": "Sell",
            "confidence": "Tinggi",
            "emoji": "📉"
        }
    else:
        return {
            "rating": "Sangat Bearish",
            "action": "Strong Sell",
            "confidence": "Sangat Tinggi",
            "emoji": "💀"
        }
```

## 5. Sistem Deteksi Bandar Multi-Level

```python
class BandarDetector:
    
    def detect_bandar_level(self, bandar_score: float, 
                             components: dict,
                             price_data: dict) -> dict:
        """
        Level 1: Possible Bandar (skor 60-70)
        Level 2: Probable Bandar (skor 70-80)  
        Level 3: Confirmed Bandar (skor 80+)
        """
        
        broker = components['broker_acc_score']
        foreign = components['foreign_score']
        volume = components['volume_score']
        
        # Level 1: Possible
        if bandar_score >= 60:
            level = 1
            evidence = []
            if broker > 65: evidence.append("broker_accumulation")
            if foreign > 60: evidence.append("foreign_inflow")
            if volume > 60: evidence.append("volume_above_average")
        
        # Level 2: Probable
        if bandar_score >= 70:
            level = 2
            if broker > 70 and foreign > 65:
                evidence.append("coordinated_buying")
            if price_data.get('is_sideways') and broker > 70:
                evidence.append("stealth_accumulation")
        
        # Level 3: Confirmed
        if bandar_score >= 80:
            level = 3
            if broker > 80 and foreign > 75 and volume > 70:
                evidence.append("all_signals_aligned")
        
        if bandar_score < 60:
            level = 0
            evidence = ["no_strong_signal"]
        
        return {
            "level": level,
            "description": ["No Signal", "Possible", "Probable", "Confirmed"][level],
            "evidence": evidence,
            "action_suggestion": self._get_action(level, bandar_score)
        }
    
    def _get_action(self, level: int, score: float) -> str:
        actions = {
            0: "Tidak ada sinyal bandar. Hindari atau tunggu konfirmasi.",
            1: "Ada indikasi awal. Pantau lebih lanjut sebelum masuk.",
            2: "Kemungkinan besar ada aksi bandar. Pertimbangkan beli kecil.",
            3: "Bandar terkonfirmasi aktif. Ikuti dengan manajemen risiko."
        }
        return actions.get(level, "")
```

## 15. Format Output JSON — Bandarmology Score

```json
{
  "stock_code": "BBCA",
  "signal_date": "2025-08-15",
  "bandarmology": {
    "components": {
      "broker_acc_score": 74.8,
      "foreign_score": 76.2,
      "orderbook_score": 82.4,
      "volume_score": 68.5,
      "price_action_score": 71.0
    },
    "bandar_score": 75.6,
    "rating": {
      "label": "Bullish",
      "action": "Buy",
      "confidence": "Tinggi",
      "emoji": "📈"
    },
    "bandar_detection": {
      "level": 2,
      "description": "Probable",
      "evidence": [
        "broker_accumulation",
        "foreign_inflow",
        "coordinated_buying",
        "stealth_accumulation"
      ],
      "action_suggestion": "Kemungkinan besar ada aksi bandar. Pertimbangkan beli kecil."
    },
    "signals": {
      "accumulation_detected": true,
      "distribution_detected": false,
      "foreign_dominant": true,
      "volume_spike": false,
      "price_suppressed": true
    },
    "narrative": "📈 BBCA mendapat Bandarmology Score 75.6/100 — Rating BULLISH. Broker asing J.P. Morgan mendominasi akumulasi dengan skor 74.8. Aliran dana asing konsisten positif 6 hari (skor 76.2). Orderbook sangat bullish dengan bid/offer ratio 2.96x (skor 82.4). Harga masih sideways meski tekanan beli kuat — ciri khas stealth accumulation bandar besar. Bandar terkategori PROBABLE (Level 2). Saran: Pantau breakout resistance sebagai konfirmasi."
  }
}
```

## 17. Pseudocode Python Lengkap

```python
class BandarmologyScoreEngine:
    
    def __init__(self, broker_engine, foreign_engine, 
                  orderbook_engine, smart_money_engine):
        self.broker = broker_engine
        self.foreign = foreign_engine
        self.orderbook = orderbook_engine
        self.sm = smart_money_engine
        self.detector = BandarDetector()

    def analyze(self, stock_code: str, date: str, 
                ohlcv: pd.DataFrame) -> dict:
        """Master analyzer untuk Bandarmology Score"""
        
        current_price = ohlcv.iloc[-1]['close']
        yesterday_price = ohlcv.iloc[-2]['close']
        
        # 1. Broker Analysis
        broker_result = self.broker.analyze(
            stock_code, date, period_days=5,
            current_price=current_price
        )
        
        # 2. Foreign Flow
        foreign_result = self.foreign.analyze(
            stock_code, date, current_price, yesterday_price
        )
        
        # 3. Orderbook
        ob_result = self.orderbook.get_latest_analysis(stock_code)
        
        # 4. Volume Score
        volume_score = self.sm.compute_volume_score(ohlcv)
        
        # 5. Price Action
        price_action_score = self.sm.compute_price_action_score(ohlcv)
        
        # 6. Compute Bandarmology Score
        components = {
            'broker_acc_score': broker_result['scores']['broker_acc_score'],
            'foreign_score': foreign_result['foreign_strength_score'],
            'orderbook_score': ob_result['orderbook_score'],
            'volume_score': volume_score,
            'price_action_score': price_action_score
        }
        
        bandar_score = compute_bandarmology_score(**components)
        rating = get_bandarmology_rating(bandar_score)
        
        # 7. Bandar detection
        price_context = {
            'is_sideways': self._is_sideways(ohlcv),
            'current_price': current_price
        }
        bandar_detection = self.detector.detect_bandar_level(
            bandar_score, components, price_context
        )
        
        # 8. Generate narrative
        narrative = self.generate_narrative(
            stock_code, bandar_score, rating, components, 
            broker_result, foreign_result, bandar_detection
        )
        
        return {
            "stock_code": stock_code,
            "signal_date": date,
            "bandarmology": {
                "components": components,
                "bandar_score": bandar_score,
                "rating": rating,
                "bandar_detection": bandar_detection,
                "narrative": narrative
            }
        }

    def _is_sideways(self, ohlcv: pd.DataFrame, 
                      window: int = 10, threshold: float = 0.03) -> bool:
        """Harga sideways = max-min dalam window < 3%"""
        recent = ohlcv.tail(window)
        price_range = (recent['high'].max() - recent['low'].min()) / recent['close'].mean()
        return price_range < threshold

    def generate_narrative(self, stock_code, score, rating, 
                           components, broker_result, 
                           foreign_result, bandar_detection) -> str:
        
        parts = [f"{rating['emoji']} {stock_code} mendapat Bandarmology Score "
                 f"{score:.1f}/100 — Rating {rating['rating'].upper()}."]
        
        # Broker
        top_buy = broker_result.get('top_buy_broker', {})
        if top_buy and components['broker_acc_score'] > 60:
            parts.append(
                f"Broker {top_buy.get('broker_name', 'asing')} mendominasi "
                f"akumulasi (skor {components['broker_acc_score']:.1f})."
            )
        
        # Foreign
        cons_days = foreign_result.get('signals', {}).get('consecutive_inflow_days', 0)
        if cons_days >= 3:
            parts.append(
                f"Aliran dana asing konsisten positif {cons_days} hari "
                f"(skor {components['foreign_score']:.1f})."
            )
        
        # Orderbook
        parts.append(
            f"Orderbook skor {components['orderbook_score']:.1f}."
        )
        
        # Bandar level
        parts.append(
            f"Status bandar: {bandar_detection['description']} (Level {bandar_detection['level']})."
        )
        parts.append(bandar_detection['action_suggestion'])
        
        return " ".join(parts)
```

## 18. Ide Pengembangan Lanjutan
- **Bandar Pattern Library**: Database pola historis bandar per saham
- **Bandar Cycle Detector**: Identifikasi fase akumulasi → markup → distribusi → markdown
- **Bandar Prediction Model**: ML model prediksi target harga berdasarkan volume akumulasi
- **Multi-Stock Bandar Correlation**: Bandar yang sama bergerak di saham mana saja secara bersamaan
- **Alert Timing Optimizer**: Notifikasi di waktu optimal berdasarkan pola historis
