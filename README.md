# CryptoQuant — Institutional Signal Research Engine

> **Kripto para piyasalarında kurumsal düzeyde alım-satım sinyalleri üreten, makine öğrenmesi tabanlı araştırma motoru.**

Binance vadeli işlemler piyasasından canlı veri çekerek **10 farklı kripto para birimi** üzerinde derinlemesine analiz yapar ve yüksek olasılıklı ticaret fırsatlarını tespit eder.

---

## 📊 Desteklenen Varlıklar

| BTC/USDT | ETH/USDT | BNB/USDT | SOL/USDT | XRP/USDT |
|----------|----------|----------|----------|----------|
| ADA/USDT | DOGE/USDT | TRX/USDT | AVAX/USDT | LINK/USDT |

---

## ⚙️ Nasıl Çalışır — 6 Aşamalı Pipeline

### Aşama 1: Veri Toplama

Binance Futures API üzerinden aşağıdaki veriler çekilir:

- **OHLCV mum verileri** — 15 dakika, 1 saat, 4 saat (2 yıllık geçmiş)
- **Open Interest** (Açık Pozisyon) geçmişi
- **Funding Rate** (fonlama oranı) geçmişi ve anlık değer
- **Orderbook derinliği** — en iyi 20 alış/satış seviyesi

> Veriler yerel olarak önbelleğe alınır. Tekrar çalıştırıldığında yalnızca eksik mumlar indirilir.

---

### Aşama 2: Özellik Mühendisliği (82 Özellik)

Ham veriden **82 adet** teknik ve türev özellik hesaplanır:

| Kategori | Özellikler |
|---|---|
| **Volatilite** | ATR, Bollinger Bands genişliği, volatilite yüzdelikleri, True Range |
| **Trend** | EMA (8/21/50/200), EMA çaprazları, trend gücü, parabolik yön |
| **Momentum** | RSI, MACD, Stochastic, Williams %R, MFI, CCI, momentum osilatörleri |
| **Order Flow** | CVD (kümülatif hacim delta), VWAP sapması, taker alış/satış oranı |
| **Likidite** | Volume profile düğümleri, sweep skorları, destek/direnç kümeleri, FVG tespiti |
| **Piyasa Rejimi** | Hidden Markov Model (HMM) ile otomatik rejim tespiti (Trend / Ranging / High Vol) |
| **Türev Veriler** | OI değişim oranı, Funding Rate trendi, long/short oranı |
| **Çoklu Zaman Dilimi** | 1 saatlik ve 4 saatlik trend ve momentum onayı |

---

### Aşama 3: Özellik Seçimi (82 → 40)

82 özellikten en etkili **40 tanesi** seçilir. Üç yöntem birlikte kullanılır:

- **SHAP değerleri** — Model açıklanabilirliği
- **Permutation önem** — Özellik karıştırma testi
- **Mutual Information** — İstatistiksel bilgi kazanımı

Bu sayede gürültülü özellikler elenir ve modelin genelleme yeteneği artar.

---

### Aşama 4: Makine Öğrenmesi Ensemblesi

Üç farklı makine öğrenmesi modeli paralel olarak eğitilir:

| Model | Ağırlık | Açıklama |
|---|---|---|
| **XGBoost** | ~%34 | Gradient boosting |
| **LightGBM** | ~%34 | Hafif gradient boost |
| **RandomForest** | ~%32 | Rassal orman |

Her model bağımsız tahmin üretir. Sonuçlar, her modelin out-of-sample AUC skoruna göre **dinamik ağırlıklı ortalama** ile birleştirilir.

**Çıktı:** Her mum için "yukarı yönlü hareket olasılığı" (0.0 – 1.0)

Ardından bir **META MODEL** (ikinci XGBoost) devreye girer:
- Ensemble çıktısını piyasa koşullarıyla birlikte değerlendirir
- Düşük kaliteli sinyalleri filtreler
- Sonuç: Daha az ama **daha güvenilir** sinyal

---

### Aşama 5: Sinyal Üretimi

```
Olasılık > 0.65  →  🟢 LONG  (Al)
Olasılık < 0.35  →  🔴 SHORT (Sat)
0.35 — 0.65      →  ⚪ Sinyal yok (Bekle)
```

> Eşik değerler, eğitim verisinin percentile dağılımından **adaptif olarak** hesaplanır.

Her sinyal için:
- **Stop Loss** — Likidite kümeleri + ATR bazlı hesaplama
- **Take Profit** — Minimum 1.5:1 Risk:Reward oranı
- **Güven skoru** — Model (%40) + Meta (%25) + R:R (%20) + Rejim (%15)
- **Rate limiting** — Günlük sinyal sayısı sınırlı (spam önleme)

---

### Aşama 6: Doğrulama ve Risk Analizi

| Yöntem | Açıklama |
|---|---|
| **Per-Asset Backtest** | Her varlık için ayrı geçmiş performans testi. Slippage, komisyon ve spread simülasyonu dahil. |
| **Walk-Forward Validation** | Model hiç görmediği veriler üzerinde test edilir. 4 katlı ilerleyen pencere yöntemi. Overfitting kontrolü sağlar. |
| **Monte Carlo Simülasyonu** | 10.000 iterasyonla trade sonuçları rastgele karıştırılır. En kötü senaryo drawdown ve "Risk of Ruin" hesaplanır. |
| **Strategy Ranking** | Tüm varlıklar Sharpe, Sortino, MaxDD, Profit Factor, Win Rate ve Expectancy metriklerine göre sıralanır. |
| **Confirmation Matrix** | Orderbook Pressure, Market Correlation ve Volatility Safety ile her sinyal bağımsız olarak doğrulanır. |

---

## 🖥️ Terminal Çıktıları

Program çalıştırıldığında aşağıdaki raporlar üretilir:

1. **Market Heatmap** — Alpha sıralaması, rejim, fiyat, meta onay
2. **Live Signals** — Yön, entry, SL, TP, R:R, güven skoru
3. **Confirmation Matrix** — OBP, korelasyon, volatilite güvenliği
4. **Backtest Sonuçları** — Return, Sharpe, MaxDD, trade sayısı
5. **Walk-Forward** — Fold bazında out-of-sample performans
6. **Monte Carlo** — Return dağılımı, DD dağılımı, yıkım riski
7. **Strategy Ranking** — Composite score ile varlık sıralaması
8. **Liquidity Heatmap** — Likidasyon seviyeleri ve sweep durumu
9. **Research Summary** — Tüm önemli metriklerin özeti

**CSV Çıktıları:**
- `signals.csv` — Üretilen tüm sinyaller
- `strategy_ranking.csv` — Varlık sıralaması

---

## 🔧 Teknik Altyapı

| Bileşen | Teknoloji |
|---|---|
| **Dil** | Python 3.10+ |
| **ML Framework** | XGBoost, LightGBM, scikit-learn |
| **Veri** | pandas, numpy, ccxt (Binance API) |
| **İndikatörler** | ta (Technical Analysis Library) |
| **HMM** | hmmlearn |
| **Paralellik** | joblib (loky backend — process-based) |
| **Açıklama** | SHAP (model yorumlanabilirlik) |

---

## 📁 Dosya Yapısı

```
main.py                          # Ana giriş noktası
cryptoquant/
├── config.py                    # Tüm yapılandırma sabitleri
├── data/
│   ├── engine.py                # Veri toplama ve önbellekleme
│   └── cache.py                 # Disk tabanlı veri önbelleği
├── features/
│   ├── pipeline.py              # Özellik mühendisliği pipeline'ı
│   ├── momentum.py              # Momentum indikatörleri
│   ├── liquidity.py             # Likidite özellikleri
│   ├── regime.py                # HMM piyasa rejimi tespiti
│   └── orderflow.py             # Order flow analizi
├── ml/
│   ├── trainer.py               # Ensemble model eğitimi
│   ├── meta.py                  # Meta model (sinyal filtre)
│   └── selector.py              # Özellik seçimi (SHAP/MI/Perm)
├── risk/
│   └── engine.py                # Sinyal motoru ve risk yönetimi
├── liquidity/
│   └── detector.py              # Likidite küme tespiti, SL/TP
├── backtest/
│   └── engine.py                # Event-based backtesting
├── validation/
│   ├── walk_forward.py          # Walk-forward doğrulama
│   └── monte_carlo.py           # Monte Carlo risk analizi
├── ranking/
│   └── ranker.py                # Strateji sıralama modülü
└── terminal/
    └── interface.py             # Terminal UI formatlaması
```

---

## 🚀 Kullanım

```bash
python main.py
```

**Mod Seçenekleri:**

| Mod | Açıklama | Süre |
|---|---|---|
| `1` | Full Portfolio | 10 varlık, 2 yıl veri | ~4 dakika |
| `2` | Single Asset | Tek varlık derinlemesine analiz | — |
| `3` | Fast Test | 3 varlık, 2000 mum, hızlı geliştirme | — |

---

## 📈 Son Sistem Sonuçları

> Aşağıdaki sonuçlar son çalıştırmaya aittir (2026-05-12).

### Genel Durum

- **Piyasa Koşulu:** Yüksek volatilite + düşüş trendi mevcut
- **Aktif Sinyal:** Yok (sermaye koruması — nakit pozisyon)
- **Karar Gerekçesi:** Ensemble modeli, mevcut piyasa koşullarında güven eşiğini (%65) aşan bir olasılık üretemedi. Bu, sistemin kasıtlı bir "bekle" kararıdır.

### Backtest vs Walk-Forward Farkı

- **Backtest:** Geçmiş veriye göre yüksek başarı oranı (eğitim periyodu dahil)
- **Walk-Forward:** Modelin hiç görmediği out-of-sample veride daha muhafazakâr performans
- **Yorum:** Walk-Forward sonuçları, gerçek dünya performansının daha gerçekçi bir göstergesidir. Sistem, beklenen aşırı uyumu (overfitting) minimize etmek için bu validasyon sürecini birincil karar kriteri olarak kullanır.

---

*CryptoQuant — Veriye dayalı, kurumsal düzeyde sinyal araştırması.*
