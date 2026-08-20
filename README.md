# Báo cáo Tổng quan Dự án: Phân tích & Dự phóng Cổ phiếu Ngân hàng

## 1. Ngữ cảnh (Context)
Dự án được khởi xướng xuất phát từ **nhu cầu thực tế của một nhóm nhà đầu tư** quan tâm sâu sắc đến nhóm cổ phiếu ngân hàng (đặc biệt là các ngân hàng có trụ sở hoặc tầm ảnh hưởng lớn tại miền Bắc và các mã trụ cột như VCB, ACB, TCB, VPB, TPB). 

Thị trường chứng khoán luôn biến động và việc ra quyết định phụ thuộc vào quá nhiều biến số phân mảnh. Các nhà đầu tư cần một hệ thống có khả năng:
- Định lượng hóa sức mạnh Mua/Bán của cổ phiếu thay vì cảm tính.
- Cung cấp một cái nhìn dài hạn về tiềm năng giá trong tương lai (đặc biệt là mục tiêu năm 2026).
- Gom toàn bộ các chỉ số kỹ thuật và tài chính phức tạp thành các khuyến nghị hành động đơn giản, trực quan và dễ hiểu nhất.

## 2. Mục Tiêu Dự Án (Project Goals)
*   **Xây dựng Kho dữ liệu (DWH):** Thiết lập cơ sở dữ liệu PostgreSQL chuẩn hóa, quản lý tập trung lịch sử giá cổ phiếu hàng ngày và báo cáo tài chính 5 năm của các ngân hàng.
*   **Mô hình Chấm điểm Định lượng (Scoring Model):** Đánh giá sức khỏe cổ phiếu hàng tháng dựa trên **13 chỉ số đa chiều** (8 chỉ báo kỹ thuật và 5 chỉ số tài chính).
*   **Dự phóng Học máy (Machine Learning Forecasting):** Dự đoán biên độ dao động giá cổ phiếu trong năm 2026 (Giá tối thiểu, Giá trung bình, Giá cao nhất) dựa trên dữ liệu lịch sử nến tháng.
*   **Thiết kế Dashboard Chuyên nghiệp:** Xây dựng hệ thống Power BI Dashboard gồm 3 trang tương tác trực quan với phong cách thiết kế sang trọng, đồng bộ.

## 3. Hướng Phân Tích & Cách Xử Lý Dữ Liệu (Analytical Pipeline & Schema)

### 3.1 Cấu trúc Schemas trong PostgreSQL
Hệ thống chia làm 2 phân vùng dữ liệu chính:
*   **Schema `stg` (Staging):** Chứa các bảng thô lưu trữ dữ liệu nạp trực tiếp từ file CSV/Excel của từng ngân hàng (balance sheet, income statement, daily prices).
*   **Schema `dwh` (Data Warehouse):** Tổ chức dữ liệu theo mô hình hình sao (Star Schema).
    *   *Dimension tables:* `dim_symbol` (mã cổ phiếu), `dim_metrics` (danh mục chỉ báo), `dim_intermediate_metrics`.
    *   *Fact tables:* `fact_price_monthly_end` (lịch sử nến tháng), `fact_yearly_metrics_fin` (lịch sử chỉ số tài chính), `fact_model_evaluation` (kết luận chiến lược tổng hợp), `fact_model_scoring_details` (chi tiết chấm điểm từng chỉ báo).

### 3.2 Quy trình xử lý dữ liệu (Data Pipeline)
Quy trình phân tích dữ liệu trải qua 5 bước :

[Bước 1: ETL & DWH Setup] ──> [Bước 2: Scoring Model] ──> [Bước 3: ML Price Forecast] ──> [Bước 4: Consensus Strategy] ──> [Bước 5: Power BI Visual]

**Bước 1: ETL & Thiết lập Kho dữ liệu:** 
    Chạy `init_db.py`, `load_to_stg.py` để đẩy dữ liệu thô vào schema `stg`. Sau đó chạy `init_dwh.py` và `merge_stg_all.py` để tạo các bảng liên kết. Thực thi stored procedure (`init_dwh_procedures.py`) gộp nến ngày thành nến tháng để tối ưu dung lượng phân tích dài hạn.
    
**Bước 2: Chạy mô hình Chấm điểm Định lượng (Quantitative Scoring):**
    Script `stock_scoring_model.py` tính toán **13 chỉ số sức khỏe** hàng tháng:
    *   *8 Chỉ báo kỹ thuật:* RSI (Động lượng), MA20 & EMA (Xu hướng), MACD (Đảo chiều), Bollinger Bands (Biến động), ATR (Biến động thực tế), Volume & OBV (Dòng tiền lũy kế).
    *   *5 Chỉ số tài chính:* P/E, P/B (Định giá), ROE (Sinh lời), NPL (Nợ xấu), CAR (An toàn vốn).
    *   Đầu ra: Gán tín hiệu **Buy/Sell/Neutral** cho từng chỉ số của từng ngân hàng tại kỳ đánh giá.
    
**Bước 3: Dự phóng khoảng giá bằng Machine Learning:**
    Script `price_prediction_2026.py` sử dụng thuật toán hồi quy tuyến tính đa biến và phân tích biên độ lịch sử trên chuỗi thời gian nến tháng để tính toán 3 kịch bản giá cho năm 2026: Giá thấp nhất (Min), Giá trung bình (Avg), và Giá cao nhất (Max).
    
**Bước 4: Tổng hợp Chiến lược Đồng thuận (Consensus Strategy):**

    Script `evaluation_strategy.py` kết hợp điểm số định lượng (tổng hợp thành % Tín hiệu Mua) với xu hướng ML dự phóng để đưa ra kết luận chiến lược bằng ngôn ngữ tự nhiên:
    
    *   Scoring > 50% + ML Trend Tăng = **Đồng thuận TÍCH CỰC** (Khuyến nghị: Mua gom ở vùng giá chiết khấu cụ thể).
   
    *   Scoring < 50% + ML Trend Giảm = **Đồng thuận TIÊU CỰC** (Khuyến nghị: Bán giảm tỷ trọng ở các nhịp hồi phục kỹ thuật).
    
    *   Tín hiệu mâu thuẫn = **Phân kỳ** (Khuyến nghị: Đứng ngoài quan sát).
**Bước 5: Tích hợp và trực quan hóa Power BI:**
    Liên kết dữ liệu từ PostgreSQL lên Power BI Desktop thông qua cơ chế DirectQuery/Import để hiển thị báo cáo tương tác tự động.

## 4. Phân Tích Chi Tiết Các Trang Dashboard

### TRANG 1: TỔNG QUAN & KHUYẾN NGHỊ ĐẦU TƯ (INVESTMENT OVERVIEW)
*   **Nội dung & Cách xây dựng:**
    *   **Bộ lọc Slicer:** Cho phép chọn mã cổ phiếu cần phân tích (ACB, VCB, TCB, VPB, TPB), cấu hình **Single Select** để đảm bảo dữ liệu không bị cộng dồn sai lệch.
    *   **Gauge "Tỷ lệ tín hiệu mua":** Hiển thị phần trăm đồng thuận mua (`buy_ratio` từ 0% đến 100%) của 13 chỉ báo. Định dạng màu sắc trực quan (Màu xanh dương sáng nếu >50% - Mua; Màu cam/hồng nhạt nếu <50% - Bán).
    *   **Thẻ kết luận chiến lược & Khuyến nghị hành động:** Hiển thị kết luận bằng ngôn ngữ tự nhiên từ kết quả phân tích DWH (ví dụ: *Đồng thuận Tích cực*, *Đồng thuận Tiêu cực*, *Phân kỳ*) đi kèm vùng giá mua gom chi tiết (ví dụ: *Vùng mua gom: 21.6k - 22.8k*).
    *   **Biểu đồ "Biên độ giá dự báo 2026":** Vẽ biểu đồ thanh ngang so sánh mức giá đóng cửa cuối năm 2025 (dưới dạng đường mốc nét đứt **Constant Line**) với khoảng dao động giá 2026 (Min, Average, Max). Giúp nhà đầu tư định vị nhanh **Upside (biên lợi nhuận)** và **Downside (biên rủi ro)**.

### TRANG 2: PHÂN TÍCH KỸ THUẬT KHUNG THÁNG (TECHNICAL ANALYSIS)
*   **Nội dung & Cách xây dựng:**
    *   **Biểu đồ nến tháng & Bollinger Bands:** Trục X là chuỗi thời gian liên tục (`process_dt`). Vẽ đường giá đóng cửa (`close`) cùng dải trên (`upper_band`) và dải dưới (`lower_band`) của Bollinger Bands để nhận diện vùng biến động giá.
    *   **Bảng Tín hiệu Kỹ thuật:** Bảng danh sách chi tiết chẩn đoán của 8 chỉ báo kỹ thuật (RSI, MA20, EMA, MACD, Bollinger Bands, ATR, Volume, OBV) đi kèm tín hiệu (Buy/Sell/Neutral) và cường độ tín hiệu.
    *   **Biểu đồ Volume & Dòng tiền tích lũy OBV:** Sử dụng Combo Chart (Cột và Đường). Cột biểu diễn khối lượng giao dịch (`volumn`), đường biểu diễn dòng tiền tích lũy (`obv`) theo chuỗi thời gian liên tục để theo dõi hành vi mua gom/xả hàng của dòng tiền lớn.

### TRANG 3: SỨC KHỎE TÀI CHÍNH 5 NĂM (FINANCIAL HEALTH & SLOPES)
*   **Nội dung & Cách xây dựng:**
    *   **Biểu đồ Đường kép Định giá (P/E & P/B):** Trục X là `year_key` (2021-2025). Trục Y chính hiển thị giá trị P/E, trục Y phụ hiển thị giá trị P/B lịch sử giúp nhận diện cổ phiếu đang đắt hay rẻ so với chính nó trong quá khứ.
    *   **Biểu đồ Đường Hiệu suất & Sức khỏe (ROE, NPL, CAR):** Vẽ 3 đường biểu diễn tỷ suất sinh lời ROE, tỷ lệ nợ xấu NPL và tỷ lệ an toàn vốn CAR qua các năm để đánh giá chất lượng tài sản và hiệu quả hoạt động dài hạn.
    *   **Bảng Chẩn đoán Tài chính & Độ dốc Xu hướng (Slope):** Hiển thị chi tiết giá trị hiện tại của các chỉ số tài chính kèm theo chẩn đoán xu hướng dài hạn (ví dụ: *Xu hướng 5 năm (Tăng +4.8%/năm)*) được tính toán tự động bằng thuật toán hồi quy tuyến tính trong Python.

## 5. Kết Quả & Insight Rút Ra (Key Insights)

### 5.1 Bức tranh toàn cảnh ngành ngân hàng (Giai đoạn 2021 - 2025)
Từ chuỗi dữ liệu lịch sử được tích hợp trong kho dữ liệu, có 3 xu hướng vĩ mô lớn của hệ thống ngân hàng thương mại Việt Nam lộ diện:
*   **Sự phân hóa mạnh mẽ về chất lượng tài sản (NPL):** Tỷ lệ nợ xấu (NPL) toàn hệ thống có xu hướng gia tăng từ 2021 đến 2025 do tác động của thị trường bất động sản đóng băng và kinh tế phục hồi chậm. Tuy nhiên, mức độ kiểm soát rất khác nhau. Nhóm ngân hàng quốc doanh (VCB) và bán lẻ thận trọng (ACB) kiểm soát NPL cực tốt (VCB < 0.9%, ACB < 1.3%), trong khi nhóm ngân hàng cho vay tiêu dùng (VPB) chịu áp lực nợ xấu cao hơn hẳn (NPL chạm mức 2.45% năm 2025).
*   **Độ dày bộ đệm vốn (CAR) được gia cố:** Dưới sự giám sát của Ngân hàng Nhà nước, hầu hết các ngân hàng đều tăng trưởng hệ số an toàn vốn CAR qua các năm (ACB tăng từ 13.3% lên 13.9%, VPB đạt đỉnh 15.8% nhờ bán vốn cho cổ đông chiến lược SMBC).
*   **Bẫy định giá ngắn hạn:** Giai đoạn cuối năm 2025 chứng kiến sự phục hồi mạnh về giá của cổ phiếu ngân hàng. Việc giá tăng nhanh khiến định giá định lượng P/E và P/B của nhiều mã vượt qua trung bình lịch sử 5 năm, kích hoạt tín hiệu "Sell" (Bán) ở góc độ định giá cơ bản, tạo ra hiện tượng **Phân kỳ** giữa xu hướng kỹ thuật ngắn hạn (đang tăng) và định giá cơ bản (đang đắt).

  ## 5.2 Chi tiết phân tích & khuyến nghị cụ thể cho 5 ngân hàng

| Mã CK | Tỷ lệ Mua | Biên độ dự phóng 2026 | Kết luận của mô hình | Khuyến nghị chiến lược |
| :---: | :---: | :---: | :--- | :--- |
| **ACB** | **55%** | **20.9k - 29.1k** (Avg: 24.3k) | 🟢 **Đồng thuận TÍCH CỰC** | Chờ mua gom ở vùng giá chiết khấu **21.6k - 22.8k**. |
| **VCB** | **54%** | **67.5k - 81.9k** (Avg: 74.4k) | 🟡 Phân kỳ (Giá đắt - Xu hướng Tăng) | Nắm giữ nếu có sẵn, hạn chế mua đuổi giá cao. |
| **TPB** | **54%** | **12.4k - 18.8k** (Avg: 15.2k) | 🟡 Phân kỳ (Giá đắt - Xu hướng Tăng) | Nắm giữ nếu có sẵn, hạn chế mua đuổi giá cao. |
| **VPB** | **53%** | **18.4k - 27.4k** (Avg: 22.2k) | 🟡 Phân kỳ (Giá đắt - Xu hướng Tăng) | Nắm giữ nếu có sẵn, hạn chế mua đuổi giá cao. |
| **TCB** | **48%** | **21.0k - 34.3k** (Avg: 27.7k) | 🟡 Phân kỳ (Giá đắt - Xu hướng Tăng) | Nắm giữ nếu có sẵn, hạn chế mua đuổi giá cao. |

