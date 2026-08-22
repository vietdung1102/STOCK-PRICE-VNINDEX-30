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

### Trang 1: Giới Thiệu Dự Án
Giải đáp bối cảnh thực tế và tổng quan kiến trúc dự án:

*   `Ngữ Cảnh Xây Dựng`: Xuất phát từ nhu cầu thực tế của nhóm nhà đầu tư cá nhân đối với nhóm cổ phiếu ngân hàng trụ cột và quốc doanh/bán lẻ hàng đầu.
    
*   `Đầu Ra Cốt Lõi`: Giải đáp 2 bài toán lớn (Lượng hóa sức mua/bán bằng % Buy Ratio và Dự báo khoảng giá 2026 bằng Hồi quy tuyến tính).
  
*   `Data Flowchart (Luồng Dữ Liệu 4 Bước)`:
    [1. Thu Thập Dữ Liệu] ──► [2. DB & Data Warehouse] ──► [3. Python Scoring & ML] ──► [4. Render UI Web]
    (Daily Price & BCTC)       (PostgreSQL Schema stg/dwh)   (Pandas/Numpy/Audit Log)    (Flask App + Chart.js)

### TRANG 2: Từ Điển Dữ Liệu

Gồm 4 Sub-Tabs Sidebar:
*   `Các Metric (Chỉ số)`: Bảng tra cứu định nghĩa 13 chỉ số, công thức toán học/chỉ báo kỹ thuật (EMA, EWM, Window Functions) và quy tắc gán tín hiệu Buy/Sell/Neutral.
*   `Thuật ngữ`: Định nghĩa các thuật ngữ chuyên môn (*Scoring Model*, *Linear Regression*, *Đồng thuận Tích cực/Tiêu cực*, *Phân kỳ Divergence*, *Vùng Mua Gom*).
*   `Nguồn dữ liệu`: Trình bày chi tiết luồng PostgreSQL Data Warehouse (từ `stg` sang `dwh.fact_price_monthly_end`) và cơ chế **UPSERT** lưu vết lịch sử đánh giá AI vào `dwh.fact_model_evaluation`.
*   `Hệ thống Dashboard`: Hướng dẫn cách đọc và giải thích ý nghĩa 5 loại đồ thị/màn hình phân tích trên giao diện.

### TRANG 3:  Bảng Tổng Hợp Thị Trường 

Đóng vai trò là công cụ quét nhanh toàn thị trường cho 5 mã cổ phiếu ngân hàng mục tiêu (**ACB, VCB, TCB, VPB, TPB**).

    *   `Symbol`: Mã giao dịch của 5 ngân hàng.
    *   `Nhóm C.Số Tài Chính (% Buy Fin)`: Tỷ lệ % Mua trung bình từ 5 chỉ số tài chính (P/E, P/B, ROE, NPL, CAR).
    *   `Nhóm C.Số Kỹ Thuật (% Buy Tech)`: Tỷ lệ % Mua trung bình từ 8 chỉ báo kỹ thuật dài hạn khung Tháng.
    *   `Giá Cuối 2025`: Mốc giá đóng cửa thực tế chốt tại ngày 31/12/2025.
    *   `Giá Dự Đoán Min 2026 & Max 2026`: Biên độ khoảng giá kịch bản xấu nhất (Đỏ) và tốt nhất (Xanh lá) do mô hình ML hồi quy dự báo.
    *   `% Buy (Tổng)`: Tỷ lệ phần trăm Mua tổng hợp từ 13 chỉ tiêu của mô hình Scoring.
    *   `Trend 2026`: Xu hướng biến động giá dự báo năm 2026 (Tăng/Giảm).
    *   `Kết Luận (Action)`: Trạng thái đồng thuận (🟢 Đồng thuận Tích cực, 🔴 Đồng thuận Tiêu cực, 🟡 Phân kỳ) đi kèm **Vùng Mua Gom** (VD: ACB gom mua 21.6k-22.8k) hoặc **Điểm Bán Giảm Tỷ Trọng**.

### TRANG 4,5,6,7,8: Dashboard Phân Tích Cổ Phiếu Chi Tiết (Individual Bank View)

Mỗi trang hiển thị 1 ngân hàng khác nhau, mỗi trang lại bao gốm các khối Panel sau

1.  **Panel 1 - Strategy Recommendation (Khối Khuyến Nghị Chiến Lược):**
    *   *Conclusion Box:* Thể hiện kết luận đồng thuận bằng màu tương ứng (Xanh lá = Đồng thuận Tích cực, Đỏ = Đồng thuận Tiêu cực, Xanh lam = Phân kỳ).
    *   *Action Box:* Cung cấp khuyến nghị hành động cụ thể, tự động tính toán vùng giá mua gom chiết khấu 5-10% hoặc khuyến nghị quản trị rủi ro.
      
2.  **Panel 2 - Scoring Model (Biểu Đồ Doughnut & Bảng Chỉ Số Cụ Thể):**
    *   *Doughnut Chart:* Biểu đồ vành khăn trực quan hóa tỷ lệ phần trăm **% Buy (Xanh bích)** vs **% Sell (Đỏ)**, với con số % Buy chính giữa.
    *   *Metric Table:* Bảng chi tiết 13 chỉ tiêu gồm 5 cột: `Criteria` (Tên chỉ số), `Value` (Giá trị thực tế), `Signal` (Badge Buy/Sell/Neutral), `Strength` (Mức độ phần trăm Mua), và `Description` (Mô tả lý do gán điểm).
      
3.  **Panel 3 - ML Price Prediction 2026 (Khối Dự Báo Giá Học Máy):**
    *   *Bar Chart:* Biểu đồ cột trực quan 3 kịch bản giá cho năm 2026: **Min** (Cột đỏ - Rủi ro), **Avg** (Cột xanh lam - Cơ sở), **Max** (Cột xanh lá - Tích cực).
 
4.  **Panel 4 - Trend Analysis Panel (Hệ Thống Lưới 13 Biểu Đồ Động - Grid Charts):**
    *   *8 Đồ thị Kỹ thuật (Monthly Grid Charts):* Phân tích chuỗi nến Tháng từ **T3/2025 đến T12/2025** của RSI, MA20, EMA20, MACD, BB, ATR, Volume, OBV. Đồ thị đổi màu động theo Signal và đính kèm Signal Badge & Strength Score %.
    *   *5 Đồ thị Tài chính (5-Year Fundamental Trend Grid):* Phân tích đường diễn biến xu hướng 5 năm **(2021-2025)** cho P/E, P/B, ROE, NPL, CAR, cho phép theo dõi đà tăng trưởng cơ bản của ngân hàng.

## 5. Kết quả và insight thu được

Dựa trên việc phân tích như trên, ta có thể đưa ra kết luận về 5 cổ phiếu như sau

🟢 Cơ hội gom mua tối ưu (ACB): ACB là mã duy nhất đạt 🟢 Đồng thuận TÍCH CỰC (Scoring 52% Buy + ML Trend Tăng) nhờ an toàn vốn CAR vững chắc (11.2%), nợ xấu siêu thấp (0.99%) và định giá hợp lý

🟡 Bẫy định giá & Hiện tượng Phân kỳ (VCB, VPB, TPB): Đà tăng kỹ thuật cuối năm 2025 đẩy P/E & P/B của nhóm này vượt trung bình 5 năm lịch sử. Xuất hiện hiện tượng Phân kỳ (Kỹ thuật tăng ngắn hạn nhưng Định giá cơ bản đã đắt), do đó khuyến nghị nắm giữ hoặc đứng ngoài quan sát, không mua đuổi giá cao.

🔴 Cảnh báo rủi ro điều chỉnh (TCB): TCB rơi vào trạng thái 🔴 Đồng thuận TIÊU CỰC (Scoring 48% Buy + ML Trend Giảm, RSI tiệm cận vùng quá mua 66.4) . Cảnh báo áp lực điều chỉnh lớn năm 2026, khuyên canh các nhịp kéo hồi để bán giảm tỷ trọng.




