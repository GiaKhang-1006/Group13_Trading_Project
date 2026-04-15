# Mean Reversion Strategy on VNF301M

## Abstract
Dự án này kiểm thử chiến lược giao dịch đảo chiều (Mean Reversion) áp dụng trên hợp đồng tương lai chỉ số VN30 (VNF301M) tại khung thời gian 1 giờ (1h). Chiến lược sử dụng Z-Score để đo lường độ lệch của giá so với đường trung bình động (SMA) và tìm kiếm cơ hội khi giá có xu hướng hồi quy về giá trị trung bình. Kết quả kiểm thử cho thấy chiến lược đạt tỷ lệ thắng (Win Rate) trên 50% ở cả tập In-sample và Out-of-sample, tuy nhiên tỷ lệ Risk/Reward chưa tối ưu dẫn đến lợi nhuận tổng vẫn đang âm.

## 0. Introduction
- **Motivation (Tại sao?):** Thị trường phái sinh Việt Nam (VN30F) thường có những biên độ dao động giật (whipsaw) trong phiên hoặc giữa các phiên. Khi giá lệch quá xa so với mức trung bình ngắn hạn, xác suất xảy ra nhịp điều chỉnh (pullback) là rất cao.
- **Method (Như thế nào?):** Dự án sử dụng chỉ báo thống kê Z-Score kết hợp với Simple Moving Average (SMA) để định lượng độ "căng" của giá. Lệnh được kích hoạt khi Z-Score vượt ngưỡng cực đại và đóng lại khi Z-Score quay về vùng an toàn.
- **Goal (Mục tiêu?):** Xây dựng, cài đặt và kiểm thử (backtest) giả thuyết giao dịch này qua quy trình 9 bước của PLUTUS, hướng tới việc tối ưu hóa và đưa vào Paper Trading thực tế.

## 1. Step 1: Trading Hypothesis
Thị trường phái sinh VN30 (VNF301M) trong ngắn hạn (khung 1h) có tính chất đảo chiều về giá trị trung bình (mean-reverting). 

- **Công thức tính Z-Score:** $$Z = \frac{Price - SMA}{\sigma}$$
  *(Trong đó $\sigma$ là độ lệch chuẩn của giá trong chu kỳ tính toán).*

- **Logic giao dịch:** - **Mở vị thế (Entry):** Mở Short khi Z-Score > 2.0 (Quá mua) / Mở Long khi Z-Score < -2.0 (Quá bán).  
  - **Đóng vị thế (Exit):** Đóng lệnh khi Z-Score hồi về mốc 0.5 hoặc -0.5.  
  - **Quản trị rủi ro (Stop Loss):** Cắt lỗ tuyệt đối khi Z-Score tiếp tục phá vỡ mốc 3.0 (xu hướng đi ngược giả thuyết quá mạnh).

## 2. Step 2 & 3: Data

### 2.1 Data Collection
- **Sản phẩm:** Hợp đồng tương lai VN30 (Các kỳ hạn nối tiếp tạo thành chuỗi VNF301M liên tục).
- **Nguồn dữ liệu:** Database official của khóa học Algotrade (Sử dụng credential được cung cấp để query trực tiếp).
- **Định dạng gốc:** Dữ liệu Tick (Tick data) bao gồm thông tin giá và khối lượng khớp lệnh theo thời gian thực.

### 2.2 Data Processing
- **Xử lý:** Dữ liệu Tick được tải về và tổng hợp (resample) thành nến OHLCV ở khung thời gian **1 hour (1h)**.
- **Rollover:** Quá trình chuyển kỳ hạn hợp đồng (rollover) được xử lý tự động để đảm bảo chuỗi giá liên tục không bị đứt gãy giữa các tháng đáo hạn. 

## 3. Implementation (How to Run)
Để chạy lại (reproduce) toàn bộ kết quả của dự án này, vui lòng làm theo các bước sau:

**Môi trường:**
Dự án sử dụng Python thông qua môi trường Anaconda (`plutus_x86`).

1. Kích hoạt môi trường: `conda activate plutus_x86`
2. Cài đặt các thư viện cần thiết (nếu có): `pip install -r requirements.txt`

**Chạy mã nguồn:**
- **Bước 1 (Load & Xử lý data):** Chạy module loader để tải và resample dữ liệu:
  ```bash
  python -m src.data.loader
  ```


- **Bước 2 (Chạy Backtest):** Thực thi lệnh sau để chạy toàn bộ quá trình backtest cho cả In-sample và Out-of-sample:
  ```bash
  python -m run_backtest
  ```
  *(Cấu hình tham số chiến lược được đặt sẵn trong file `run_backtest.py`)*

- **Đầu ra (Output):** Các biểu đồ (`backtest_chart.png`) và danh sách lệnh (`trades.csv`) sẽ được lưu tự động vào các thư mục `results/insample/` và `results/outsample/`.

## 4. Step 4: In-sample Backtesting
- **Giai đoạn:** 2023-01-01 → 2024-06-30 (18 tháng)
- **Cấu hình tham số:** Window = 20, Entry = 2.0, Exit = 0.5, Stop Loss = 3.0.
- **Dữ liệu đầu vào:** 19 hợp đồng VN30F (từ 2301 đến 2407), tổng cộng 1,817 bars (khung 1h).

### 4.1 Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | 61 |
| **Win Rate** | 52.46% |
| **Total Return** | -2.48% |
| **Sharpe Ratio** | -0.241 |
| **Max Drawdown** | -4.02% |
| **Profit Factor** | 0.867 |
| **Avg Win / Avg Loss** | 1,595,625 VND / -2,031,897 VND |

## 5. Step 5: Optimization
Hiện tại, chiến lược đang sử dụng các tham số heuristic (kinh nghiệm). Quá trình tối ưu hóa (Optimization) để tìm ra bộ tham số (Window, Entry/Exit Threshold) tốt nhất cho Profit Factor sẽ được tiến hành và báo cáo chi tiết trong giai đoạn chuẩn bị cho Paper Trading.

## 6. Step 6: Out-of-sample Backtesting
- **Giai đoạn:** 2024-07-01 → 2024-12-31 (6 tháng)
- **Cấu hình tham số:** Tương tự như In-sample để đánh giá độ ổn định.
- **Dữ liệu đầu vào:** 6 hợp đồng VN30F (từ 2407 đến 2412), tổng cộng 610 bars (khung 1h).

### 6.1 Result

| Metric | Value |
| :--- | :--- |
| **Total Trades** | 23 |
| **Win Rate** | 56.52% |
| **Total Return** | -1.66% |
| **Sharpe Ratio** | -0.581 |
| **Max Drawdown** | -3.26% |
| **Profit Factor** | 0.671 |
| **Avg Win / Avg Loss** | 1,066,538 VND / -2,066,000 VND |

> **Ghi chú:** Số lượng trade ở OOS là 23 (chưa đạt mốc >=30) do thời gian test khá ngắn (6 tháng) trên khung thời gian lớn (1h).

## 7. Conclusion
Chiến lược Mean Reversion trên VNF301M thể hiện khả năng dự báo điểm đảo chiều khá tốt với tỷ lệ thắng (Win Rate) duy trì trên 50% ổn định từ In-sample sang Out-of-sample. Dù vậy, điểm yếu cố hữu nằm ở mức Cắt lỗ (Stop Loss) hiện tại quá rộng, dẫn đến lỗ trung bình lớn hơn lãi trung bình và tổng lợi nhuận âm. Dự án sẽ tiếp tục tinh chỉnh quản trị rủi ro ở Step 7 (Paper Trading).





# Mean Reversion Strategy on VNF301M
## Abstract
This project tests the Mean Reversion trading strategy applied to the VN30 index futures contract (VNF301M) at a 1-hour (1h) timeframe. The strategy uses Z-Score to measure the deviation of the price from the simple moving average (SMA) and seeks opportunities when the price tends to revert to the average value. Test results show that the strategy achieves a win rate of over 50% in both the In-sample and Out-of-sample sets. However, the Risk/Reward ratio is not yet optimized, resulting in negative overall profits.
## 0. Introduction
- **Motivation (Why?):** The Vietnamese derivatives market (VN30F) often experiences whipsaw fluctuations during or between sessions. When prices deviate too far from the short-term average, the probability of a pullback is very high.
- **Method (How?):** The project uses the Z-Score statistical indicator combined with Simple Moving Average (SMA) to quantify price “tension.” Orders are triggered when the Z-Score exceeds the maximum threshold and closed when the Z-Score returns to the safe zone.
- **Goal:** Build, set up, and backtest this trading hypothesis through PLUTUS's 9-step process, aiming to optimize and implement it in real Paper Trading.
## 1. Step 1: Trading Hypothesis
The VN30 derivatives market (VNF301M) in the short term (1-hour timeframe) has mean-reverting properties.
- **Z-Score calculation formula:** $$Z = \frac{Price - SMA}{\sigma}$$
  
*(Where $\sigma$ is the standard deviation of the price in the calculation period).*
- **Trading Logic:** - **Open Position (Entry):** Open Short when Z-Score > 2.0 (Overbought) / Open Long when Z-Score < -2.0 (Oversold).
    
- **Close position (Exit):** Close the order when Z-Score returns to 0.5 or -0.5.  
- **Risk management (Stop Loss):** Cut losses immediately when Z-Score continues to break through 3.0 (the countertrend hypothesis is too strong).
## 2. Step 2 & 3: Data
### 2.1 Data Collection
- **Product:** VN30 futures contract (Successive maturities form a continuous VNF301M series).
- **Data source:** Official database of the Algotrade course (Use the provided credentials to query directly).
- **Original format:** Tick data including real-time price and order volume information.
### 2.2 Data Processing
- **Processing:** Tick data is downloaded and resampled into OHLCV candles at the **1-hour (1h)** timeframe.
- **Rollover:** The contract rollover process is handled automatically to ensure a continuous price series without gaps between expiration months.
## 3. Implementation (How to Run)
To reproduce the entire results of this project, please follow these steps:
**Environment:**
The project uses Python through the Anaconda environment (`plutus_x86`).
1. Activate the environment: `conda activate plutus_x86`
2. Install the necessary libraries (if any): `pip install -r requirements.txt`
**Run the source code:**
- **Step 1 (Load & Process data):** Run the loader module to load and resample the data:
    ```bash
    python -m src.data.loader
    ```

- **Step 2 (Run Backtest):** Execute the following command to run the entire backtest process for both In-sample and Out-of-sample:
    ```bash
    python -m run_backtest
    ```
  
*(Strategy parameter configuration is pre-set in the `run_backtest.py` file)*
- **Output:** Charts (`backtest_chart.png`) and order lists (`trades.csv`) will be automatically saved to the `results/insample/` and `results/outsample/` directories.
## 4. Step 4: In-sample Backtesting
- **Period:** 2023-01-01 → 2024-06-30 (18 months)
- **Parameter configuration:** Window = 20, Entry = 2.0, Exit = 0.5, Stop Loss = 3.0.
- **Input data:** 19 VN30F contracts (from 2301 to 2407), totaling 1,817 bars (1h timeframe).
### 4.1 Result
| Metric | Value |
| :--- | :--- |
| **Total Trades** | 61 |
| **Win Rate** | 52.46% |
| **Total Return** | -2.48% |
| **Sharpe Ratio** | -0.241 |
| **Max Drawdown** | -4.02% |
| **Profit Factor** | 0.867 |
| **Avg Win / Avg Loss** | 1,595,625 VND / -2,031,897 VND |
## 5. Step 5: Optimization
Currently, the strategy uses heuristic (experience-based) parameters. The optimization process to find the best set of parameters (Window, Entry/Exit Threshold) for Profit Factor will be conducted and reported in detail during the Paper Trading preparation phase.
## 6. Step 6: Out-of-sample Backtesting
- **Period:** July 1, 2024 → December 31, 2024 (6 months)
- **Parameter configuration:** Similar to In-sample to assess stability.
- **Input data:** 6 VN30F contracts (from 2407 to 2412), totaling 610 bars (1h timeframe).
### 6.1 Result
| Metric | Value |
| :--- | :--- |
| **Total Trades** | 23 |
| **Win Rate** | 56.52% |
| **Total Return** | -1.66% |
| **Sharpe Ratio** | -0.581 |
| **Max Drawdown** | -3.26% |
| **Profit Factor** | 0.671 |
| **Avg Win / Avg Loss** | 1,066,538 VND / -2,066,000 VND |
> **Note:** The number of trades in OOS is 23 (not reaching the threshold of >=30) due to the relatively short testing period (6 months) on a large timeframe (1h).
## 7. Conclusion
The Mean Reversion strategy on VNF301M demonstrates a fairly good ability to predict reversal points with a Win Rate that remains above 50% consistently from In-sample to Out-of-sample. However, the inherent weakness lies in the current Stop Loss level being too wide, leading to average losses greater than average profits and a negative total profit. The project will continue to refine risk management in Step 7 (Paper Trading).