# Group 13 — Multi-Strategy Trading System on VNF301M

## Abstract

Dự án xây dựng hệ thống giao dịch tự động đa chiến lược (Multi-Strategy) trên hợp đồng tương lai chỉ số VN30 (VNF301M). Hệ thống hỗ trợ hai chiến lược chính: **Trend Following (EMA Crossover)** và **Opening Range Breakout (ORB)**. Cả hai đều được tích hợp Stop Loss động dựa trên ATR, bộ lọc RSI và cơ chế thoát lệnh cuối phiên tự động. Việc chuyển đổi giữa các chiến lược được thực hiện chỉ bằng một dòng cấu hình duy nhất trong `main_live.py` và `run_backtest.py`.

This project builds an automated multi-strategy trading system on the VN30 index futures contract (VNF301M). The system supports two main strategies: **Trend Following (EMA Crossover)** and **Opening Range Breakout (ORB)**. Both integrate dynamic ATR-based Stop Loss, RSI filters, and an automatic end-of-session exit mechanism. Switching between strategies requires changing only a single configuration line in `main_live.py` and `run_backtest.py`.

---

## 0. Introduction

**Motivation (Tại sao? / Why?)**

Thị trường phái sinh VN30F có hai trạng thái phổ biến: (1) **xu hướng rõ ràng** sau khi tin tức hoặc dòng tiền lớn xuất hiện, và (2) **biến động mạnh trong buổi sáng** ngay sau khi phiên mở cửa. Mỗi trạng thái phù hợp với một chiến lược riêng biệt.

The VN30F derivatives market commonly exhibits two states: (1) **clear trending** after news or large capital flows, and (2) **strong morning volatility** right after the opening session. Each state suits a different strategy.

**Method (Phương pháp / Approach)**

- **EMA Crossover:** Bắt xu hướng bằng giao cắt EMA 10/30, lọc nhiễu bằng RSI và ATR Stop Loss động.
- **ORB:** Xác định vùng giá High/Low của 30 phút đầu phiên (09:00–09:30) làm ngưỡng breakout. Vào lệnh khi giá phá vỡ vùng này với xác nhận khối lượng.

- **EMA Crossover:** Captures trends via EMA 10/30 crossover, filtered by RSI and dynamic ATR Stop Loss.
- **ORB:** Defines the High/Low range of the first 30 minutes (09:00–09:30) as the breakout threshold. Enters when price breaks out with volume confirmation.

**Goal (Mục tiêu / Goals)**

Xây dựng, kiểm thử và đưa vào Paper Trading một hệ thống giao dịch linh hoạt, có khả năng chuyển đổi chiến lược nhanh chóng theo điều kiện thị trường, tuân theo quy trình 9 bước của PLUTUS.

Build, backtest, and deploy to Paper Trading a flexible trading system capable of rapidly switching strategies based on market conditions, following the PLUTUS 9-step process.

---

## 1. Step 1: Trading Hypotheses

### 1.1 Trend Following — EMA Crossover

Khi thị trường hình thành xu hướng, đường EMA ngắn hạn (10) sẽ cắt qua đường EMA dài hạn (30). Đây là tín hiệu xác nhận momentum và là thời điểm phù hợp để gia nhập vị thế theo xu hướng.

When the market forms a trend, the short-term EMA (10) crosses the long-term EMA (30). This confirms momentum and signals an entry aligned with the trend.

| Sự kiện / Event | Hành động / Action |
| :--- | :--- |
| EMA 10 cắt lên EMA 30 + RSI ∈ [45, 75] | Mở **LONG** / Open **LONG** |
| EMA 10 cắt xuống EMA 30 + RSI ∈ [25, 55] | Mở **SHORT** / Open **SHORT** |
| EMA cắt ngược chiều | Đóng vị thế / Close position |
| Chạm ATR Stop Loss | Cắt lỗ / Stop out |
| 11:25–11:30 hoặc 14:40–14:45 | Force exit cuối phiên / Force exit EOD |

### 1.2 Opening Range Breakout (ORB)

Trong 30 phút đầu phiên (09:00–09:30), giá hình thành một vùng dao động (Opening Range). Khi giá phá vỡ vùng này kèm khối lượng xác nhận, xu hướng trong phiên thường tiếp diễn theo hướng breakout.

In the first 30 minutes (09:00–09:30), price forms an Opening Range. When price breaks out of this range with volume confirmation, the intraday trend tends to continue in the breakout direction.

| Sự kiện / Event | Hành động / Action |
| :--- | :--- |
| Giá phá vỡ ORB High + Volume > 110% MA20 | Mở **LONG** / Open **LONG** |
| Giá phá vỡ ORB Low + Volume > 110% MA20 | Mở **SHORT** / Open **SHORT** |
| Chạm ATR Stop Loss (×3.5) | Cắt lỗ / Stop out |
| Trailing Stop kích hoạt sau lãi ≥ 1×ATR | Bảo vệ lãi / Protect profit |
| Vào giờ nghỉ trưa / cuối phiên | Force exit / Force exit EOD |

**Giới hạn:** Tối đa 1 LONG + 1 SHORT mỗi ngày để tránh over-trading sau khi dính Stop Loss.

**Limit:** Maximum 1 LONG + 1 SHORT per day to avoid over-trading after a stop-out.

---

## 2. Step 2 & 3: Data

### 2.1 Data Collection

| Thuộc tính / Attribute | Chi tiết / Detail |
| :--- | :--- |
| **Sản phẩm / Product** | VN30 Futures — Chuỗi liên tục VNF301M |
| **Nguồn / Source** | Database Algotrade (credentials trong `config.py`) |
| **Định dạng gốc / Raw format** | Tick data (giá + khối lượng khớp theo thời gian thực) |
| **Kho / Repository** | `quote.matched` JOIN `quote.total` |

### 2.2 Data Processing

- **Resample:** Tick data → OHLCV bars theo timeframe cấu hình (`config.STRATEGY["timeframe"]`).
- **Giờ giao dịch / Trading hours:** Chỉ lấy dữ liệu trong khung 09:00–14:45.
- **Rollover:** Tự động xử lý chuyển kỳ hạn theo `ROLL_SCHEDULE` trong `config.py` — xóa duplicate tại điểm roll để đảm bảo chuỗi liên tục.
- **Tick data is resampled** into OHLCV bars at the configured timeframe. Only bars within 09:00–14:45 are kept. Contract rollover is handled automatically via the `ROLL_SCHEDULE` in `config.py`, with duplicates at roll points removed.

---

## 3. Implementation (How to Run)

### 3.1 Môi trường / Environment

```bash
conda activate plutus_x86
pip install -r requirements.txt
```

### 3.2 Đổi chiến lược / Switch Strategy

Chỉ cần sửa **1 dòng** ở đầu mỗi file / Just change **1 line** at the top of each file:

```python
# main_live.py  &  run_backtest.py
ACTIVE_STRATEGY = "ema"    # "ema" | "orb" | "mean"
```

| Giá trị / Value | Chiến lược / Strategy |
| :--- | :--- |
| `"ema"` | Trend Following — EMA 10/30 Crossover |
| `"orb"` | Opening Range Breakout + ATR Trailing Stop |
| `"mean"` | Mean Reversion — Z-Score + Bollinger Bands |

### 3.3 Chạy Backtest / Run Backtest

```bash
# Bước 1: Kiểm tra data loader
python -m src.data.loader

# Bước 2: Chạy backtest (In-sample + Out-of-sample)
python run_backtest.py
```

**Output:**
- `results/insample/backtest_chart.png` — Biểu đồ In-sample
- `results/outsample/backtest_chart.png` — Biểu đồ Out-of-sample
- `results/insample/trades.csv` — Danh sách lệnh In-sample
- `results/outsample/trades.csv` — Danh sách lệnh Out-of-sample

### 3.4 Chạy Live / Run Live Bot

```bash
python main_live.py
```

Bot sẽ tự động kết nối FIX, lấy data realtime, tính tín hiệu và đặt lệnh. Dashboard in ra terminal mỗi nến. Thông báo giao dịch được gửi qua Telegram.

The bot automatically connects via FIX, fetches real-time data, computes signals, and places orders. A dashboard is printed to the terminal on each candle. Trade notifications are sent via Telegram.

---

## 4. Step 4: In-sample Backtesting

- **Giai đoạn / Period:** 2023-01-01 → 2024-12-30 (24 tháng / months)
- **Timeframe:** 15 phút / minutes

### 4.1 EMA Strategy — In-sample Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | — |
| **Win Rate** | — |
| **Total Return** | — |
| **Sharpe Ratio** | — |
| **Max Drawdown** | — |
| **Profit Factor** | — |
| **Avg Win / Avg Loss** | — VND / — VND |

### 4.2 ORB Strategy — In-sample Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | — |
| **Win Rate** | — |
| **Total Return** | — |
| **Sharpe Ratio** | — |
| **Max Drawdown** | — |
| **Profit Factor** | — |
| **Avg Win / Avg Loss** | — VND / — VND |

---

## 5. Step 5: Optimization

Các tham số có thể tối ưu hóa cho từng chiến lược / Parameters available for optimization per strategy:

**EMA:**
- `ema_fast` / `ema_slow` spans (hiện tại: 10/30)
- RSI filter band (hiện tại: Long ∈ [45,75] / Short ∈ [25,55])
- `atr_multiplier` cho Stop Loss (hiện tại: 2.0)

**ORB:**
- `atr_mult_sl` — SL ban đầu (hiện tại: 3.5×)
- `atr_mult_trail` — Trailing stop (hiện tại: 3.0×)
- `atr_activate` — Ngưỡng kích hoạt trailing (hiện tại: 1.0×)
- `vol_confirm` — Ngưỡng khối lượng xác nhận (hiện tại: 1.1×)

---

## 6. Step 6: Out-of-sample Backtesting

- **Giai đoạn / Period:** 2025-01-01 → 2025-12-31 (12 tháng / months)
- **Cấu hình:** Tương tự In-sample để đánh giá độ ổn định / Same parameters as In-sample to assess stability.

### 6.1 EMA Strategy — Out-of-sample Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | — |
| **Win Rate** | — |
| **Total Return** | — |
| **Sharpe Ratio** | — |
| **Max Drawdown** | — |
| **Profit Factor** | — |
| **Avg Win / Avg Loss** | — VND / — VND |

### 6.2 ORB Strategy — Out-of-sample Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | — |
| **Win Rate** | — |
| **Total Return** | — |
| **Sharpe Ratio** | — |
| **Max Drawdown** | — |
| **Profit Factor** | — |
| **Avg Win / Avg Loss** | — VND / — VND |

---

## 7. Conclusion

**EMA Crossover** phù hợp với các phiên có xu hướng rõ ràng, đặc biệt là khi thị trường chịu tác động của tin tức hoặc dòng tiền lớn. Điểm mạnh là logic đơn giản, dễ kiểm soát rủi ro. Điểm yếu là dễ bị "whipsaw" trong thị trường sideway.

**ORB** khai thác tốt biến động đầu phiên — thời điểm có thanh khoản cao và breakout thường có chất lượng tốt. Trailing Stop giúp bảo vệ lãi trong các sóng dài. Điểm yếu là số lượng lệnh ít (tối đa 2 lệnh/ngày) nên cần thời gian backtest đủ dài để có kết quả có ý nghĩa thống kê.

Hệ thống được thiết kế để chuyển đổi linh hoạt giữa các chiến lược chỉ bằng một dòng code, cho phép nhóm phản ứng nhanh với từng giai đoạn thị trường trong Paper Trading.

---

**EMA Crossover** suits sessions with a clear trend direction, especially when driven by news or large capital flows. Its strength is simplicity and controllable risk. Its weakness is susceptibility to whipsaws in sideways markets.

**ORB** effectively exploits early-session volatility — a period of high liquidity where breakouts tend to have higher quality. The Trailing Stop protects profits during extended moves. Its weakness is a low trade count (max 2 per day), requiring a sufficiently long backtest period for statistically meaningful results.

The system is designed to switch between strategies with a single line of code, allowing the team to respond quickly to different market phases during Paper Trading.

---

## 8. Project Structure

```
.
├── config/
│   └── config.py              # Toàn bộ cấu hình (STRATEGY, BACKTEST, DB, FIX)
├── src/
│   ├── data/
│   │   └── loader.py          # Load & resample tick data → OHLCV
│   ├── features/
│   │   └── indicators.py      # Tính toán EMA, RSI, ATR, Z-Score, Bollinger Bands
│   ├── strategy/
│   │   ├── trend_following.py # EMA Crossover strategy
│   │   ├── orb_strategy.py    # Opening Range Breakout strategy
│   │   └── mean_reversion.py  # Mean Reversion (Z-Score) strategy
│   └── backtest/
│       ├── engine.py          # Event-driven backtest engine
│       └── metrics.py         # Tính Sharpe, Drawdown, Win Rate, v.v.
├── main_live.py               # 🔴 Live bot — đổi ACTIVE_STRATEGY để switch
├── run_backtest.py            # 📊 Backtest runner — đổi ACTIVE_STRATEGY để switch
└── results/
    ├── insample/
    └── outsample/
```