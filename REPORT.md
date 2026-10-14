MSSV: 2A202602995

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): C

## 1. Phát lại log có trung thực không? (bước A)

- **Số liệu trích xuất từ `replay.json`:**
  - Tổng số kịch bản ban đầu: `n = 400`
  - Số kịch bản bị bỏ vì chính người lái gây va chạm: `dropped_human_crash_logs = 2` (do log thật của xe tự lái không được chứa tai nạn của chính người lái), số kịch bản thực tế đưa vào thử nghiệm: `n_used = 398`.
  - Va chạm khi planner chạy trên **log phát lại** (`replay.crashes`): **106** va chạm (`rear = 94`, `side = 12`, `front = 0`).
  - Va chạm khi planner chạy trong **thế giới phản ứng** (`reactive.crashes`): **3** va chạm (`front = 2`, `side = 1`, `rear = 0`).
  - Số va chạm xảy ra ở cả hai thế giới: `crash_in_both = 2`.

- **Giải thích:**
  - Phần lớn va chạm của "phát lại" là `rear` (94/106 vụ, chiếm 88.7%), và đây hoàn toàn là **va chạm giả**.
  - **Nguyên nhân:** Trong mô hình phát lại (open-loop replay), quỹ đạo của xe bám đuôi (TG) được giữ nguyên tuyệt đối theo bản ghi chuyến đi của con người. Nhưng planner xe ta lại lái thận trọng hơn (phanh sớm hoặc giảm tốc nhường đường). Xe phía sau do chỉ "nhắm mắt chạy lại log" nên không phản ứng với việc xe ta giảm tốc, dẫn đến việc tông xuyên vào đuôi xe ta. Trong thế giới phản ứng (`reactive`), các xe sau có phản xạ IDM để giữ khoảng cách và phanh theo, do đó số vụ va chạm `rear` giảm về **0**.
  - **Ví dụ kịch bản trong `replay_only`:** Kịch bản `scenario: 13`. Trong phát lại log, xe ta bị tông đuôi (`contact: rear`). Nhưng trong thế giới phản ứng, xe sau nhận diện được khoảng cách và chủ động giữ cự ly an toàn (`min_ttc_reactive = 2.43s`), không hề xảy ra va chạm. Điều này chứng minh phát lại log tĩnh không phản ánh đúng tương tác thực tế giữa các tác nhân.

## 2. Kịch bản bất khả thi (bước B)

- **Các luật trong `is_feasible` và căn cứ vật lý:**
  1. `a_lat = (pi / cutDur)^2 * 3.5 / 2 <= mu * g`: Xe chuyển làn 3.5 m theo quỹ đạo cosin trong thời gian `cutDur` (s) có gia tốc ngang cực đại là $a_{lat} = (\frac{\pi}{\text{cutDur}})^2 \times \frac{3.5}{2}$. Luật vật lý giới hạn gia tốc ngang không được vượt quá lực bám ma sát cực đại của lốp với mặt đường ($\mu g$). Nếu vượt quá, xe sẽ bị trượt văng ngang hoặc lật xe trong thực tế.
  2. `cutBrake <= mu * g`: Gia tốc phanh dọc của xe sau khi nhập làn không được vượt quá gia tốc ma sát tối đa $\mu g$ (xe không thể phanh gấp hơn giới hạn bám của lốp trên mặt đường có hệ số ma sát $\mu$).
  3. `pedV <= 2.5 m/s`: Vận tốc đi bộ của người không vượt quá 2.5 m/s (~9 km/h). Đây là ngưỡng phân định giữa đi bộ rất nhanh và chạy nước rút; kịch bản mô phỏng người đi bộ qua đường thông thường, không phải vận động viên chạy thi.
  4. `0 < cutV <= 22 m/s`: Vận tốc xe chuyển làn phải dương và không vượt quá 22 m/s (~79.2 km/h, tốc độ tối đa hợp lý trong môi trường phố đô thị một làn).

- **Hiện tượng trong `hunt.json`:**
  - Cuộc săn sinh nhiều kịch bản bất khả thi nhất: **`rear_ended`** với **16** kịch bản bị loại (`infeasible = 16`), kế đến là `crosswalk_block` (**10** kịch bản), `cut_in` (**4** kịch bản), và `bus_bypass` (**1** kịch bản).
  - **Vì sao CEM bị kéo về phía đó:** Thuật toán CEM cập nhật phân phối đề xuất $q$ hướng về nhóm cá thể có điểm nguy hiểm cao nhất (nhóm *elite*). Để kích hoạt nguy cơ bị tông đuôi (`rear_ended`), kịch bản cần ép xe ta phanh gấp nhất có thể. Điều này thôi thúc CEM đề xuất các thông số cực đoan như xe trước tạt đầu trong tích tắc (`cutDur` cực ngắn) hoặc phanh với gia tốc phi lý (`cutBrake` cực lớn), vì các giá trị này mang lại điểm số cao nhất.
  - **Hệ quả nếu đếm kịch bản bất khả thi là va chạm:** Ước lượng xác suất rủi ro sẽ bị **thổi phồng sai lệch** (overestimated). Hệ thống an toàn sẽ bị đánh giá là kém hơn thực tế dựa trên những vụ tai nạn vi phạm các định luật vật lý cơ bản vốn không bao giờ xảy ra ngoài đời thực.

## 3. Monte Carlo cho bạn biết gì (bước C1)

- **Số liệu trích xuất từ `mc.json`:**
  - Số lần thử: `n = 3000`
  - Số vụ va chạm: `crashes (k) = 3` (tất cả đều thuộc kiểu `bus_bypass`: `modes = {"bus_bypass": 3}`)
  - Tỉ lệ va chạm điểm: $\hat{p} = \frac{k}{n} = \frac{3}{3000} = 0.001$ ($1.00 \times 10^{-3}$)
  - Khoảng tin cậy Wilson 95%: `lo = 0.000340` ($3.40 \times 10^{-4}$), `hi = 0.002936` ($2.94 \times 10^{-3}$), tức $[3.40 \times 10^{-4}, 2.94 \times 10^{-3}]$.

- **Số lần chạy cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**
  - Nửa độ rộng mong muốn: $E = 0.20 \times p$.
  - Với khoảng Wilson/Gaussian mức tin cậy 95% ($z = 1.96$):
    $$E = z \sqrt{\frac{p(1 - p)}{n}} \le 0.20 \cdot p \implies n \ge \frac{z^2 (1 - p)}{0.20^2 \cdot p} \approx \frac{1.96^2}{0.04 \cdot p} \approx \frac{96.04}{p}$$
  - Với $p = 0.001$:
    $$n \approx \frac{96.04}{0.001} \approx 96.000 \text{ lần chạy}$$
    *(chính xác: $\frac{1.96^2 \times 0.999}{0.04 \times 0.001} \approx 95.944$ lần chạy).*
  - **Nhận xét:** Để đo chính xác một xác suất $10^{-3}$ với sai số tương đối $\pm 20\%$, Monte Carlo cần gần 96.000 lần chạy mô phỏng — quá tốn kém và bất khả thi cho các sự kiện cực hiếm ($10^{-5}$ cần gần 10 triệu lần chạy).

- **Kiểu thất bại chưa hề xuất hiện trong 3.000 lần chạy:**
  - Các kiểu: `cut_in`, `crosswalk_block`, `rear_ended`, và `other` đều có $k = 0$ vụ va chạm (`per_mode.<mode>.k = 0`).
  - **Con số đúng cần báo:** Tuyệt đối **không được báo bằng 0** (vì chưa quan sát thấy không đồng nghĩa với xác suất bằng 0). Phải báo **cận trên của khoảng tin cậy 95%**:
    - Cận trên Wilson 95% (`mc.json: per_mode.<mode>.hi`): **$1.28 \times 10^{-3}$** ($0.00128$ hay $\le 0.128\%$).
    - Cận theo Rule of Three ($3/n$): $3 / 3000 = 1.00 \times 10^{-3}$ ($0.100\%$).
    - Do đó, với các kiểu này, kết luận đúng là: $p \le 1.28 \times 10^{-3}$ (ở độ tin cậy 95%).

## 4. Importance sampling, ESS và độ suy biến (bước C3)

- **Bảng tổng hợp từ `estimate.json`:**

| Seed | Kiểu thất bại | $p$ | hits | $ESS$ | `top_share` |
|:---:|:---|:---:|:---:|:---:|:---:|
| **0** | `bus_bypass` | $1.56 \times 10^{-3}$ | 43 | 113.2 | 0.270 |
| | `cut_in` | 0.0 (cận: 0.025) | 0 | 107.8 | 0.000 |
| | `crosswalk_block` | $4.11 \times 10^{-7}$ | 87 | 101.1 | 0.277 |
| | `rear_ended` | 0.0 (cận: 0.025) | 0 | 129.2 | 0.000 |
| | **Tổng seed 0** | **$1.56 \times 10^{-3}$** | — | — | $[3.52 \times 10^{-4}, 4.06 \times 10^{-3}]$ |
| **1** | `bus_bypass` | $7.64 \times 10^{-4}$ | 41 | 125.6 | 0.285 |
| | `cut_in` | 0.0 (cận: 0.025) | 0 | 117.2 | 0.000 |
| | `crosswalk_block` | $2.29 \times 10^{-7}$ | 77 | 125.0 | 0.199 |
| | `rear_ended` | 0.0 (cận: 0.025) | 0 | 127.3 | 0.000 |
| | **Tổng seed 1** | **$7.64 \times 10^{-4}$** | — | — | $[2.36 \times 10^{-4}, 2.57 \times 10^{-3}]$ |
| **2** | `bus_bypass` | $1.65 \times 10^{-3}$ | 45 | 139.3 | 0.275 |
| | `cut_in` | 0.0 (cận: 0.025) | 0 | 104.2 | 0.000 |
| | `crosswalk_block` | $9.59 \times 10^{-7}$ | 70 | 113.1 | **0.530** |
| | `rear_ended` | 0.0 (cận: 0.025) | 0 | 110.3 | 0.000 |
| | **Tổng seed 2** | **$1.65 \times 10^{-3}$** | — | — | $[4.29 \times 10^{-4}, 4.15 \times 10^{-3}]$ |

- **Đánh giá sự đồng thuận giữa 3 seed và với Monte Carlo:**
  - Cả 3 seed cho ước lượng tổng dao động từ $7.64 \times 10^{-4}$ đến $1.65 \times 10^{-3}$ (chênh nhau khoảng 2.1 lần). Đây là mức dao động bình thường của Importance Sampling khi đo xác suất hiếm ở quy mô ngân sách nhỏ.
  - Ba seed **hoàn toàn đồng thuận với Monte Carlo**: trong `mc.json`, $p = 1.00 \times 10^{-3}$ với khoảng Wilson $[3.40 \times 10^{-4}, 2.94 \times 10^{-3}]$. Tất cả 3 giá trị tổng của IS đều nằm gọn bên trong khoảng tin cậy của Monte Carlo.
- **Vị trí `top_share` lớn (> 0.5) và giải thích:**
  - `top_share` lớn xuất hiện ở **Seed 2, kiểu `crosswalk_block`** (`top_share = 0.530` = 53.0%).
  - **Lý do:** Khi phân phối đề xuất $q$ bị thu hẹp (hoặc lệch khỏi vùng tự nhiên $p$), trọng số $w = p/q$ của một số ít mẫu rơi vào vùng va chạm sẽ rất lớn. Khi `top_share` > 0.5, hơn một nửa giá trị ước lượng $p = \text{mean}(w \cdot y)$ bị chi phối bởi **duy nhất một mẫu nặng ký nhất**. Đây là dấu hiệu của sự suy biến trọng số (weight degeneracy).
- **So sánh ngân sách (`budget`):**
  - Ngân sách của IS (`estimate.json: budget`): **16.800** lần chạy mô phỏng.
  - So với Monte Carlo: Để ước lượng được kiểu siêu hiếm như `crosswalk_block` ($p \approx 4 \times 10^{-7}$) với sai số $\pm 20\%$, Monte Carlo cần tới $n \approx \frac{96}{4 \times 10^{-7}} = 240.000.000$ (240 triệu lần chạy!). IS chỉ cần 16.800 lần chạy đã phát hiện và ước lượng được xác suất ở cấp độ $10^{-7}$.
- **Phần `other` trong tổng lấy từ đâu và vì sao không săn nó:**
  - `other` được lấy từ Monte Carlo C1 (`mc.json: other`, $k = 0, n = 3000$, cận trên Wilson $hi = 0.00128$).
  - **Vì sao không săn nó:** `other` là tập hợp phần dư của mọi kiểu va chạm chưa được định nghĩa trước. Không có cơ chế va chạm hay ranh giới hình học cụ thể nào nên không thể thiết kế hàm điểm nguy hiểm (`score`) định hướng cho CEM. Do các kiểu chia nhau phân hoạch không gian va chạm, phần xác suất của `other` được tính trung thực từ phân phối tự nhiên của Monte Carlo.

## 5. Các kiểu thất bại (bước C2 + 3D)

- **Các kiểu tìm thấy và chuỗi sự kiện:**
  1. **`bus_bypass` (Lách qua xe buýt đỗ):**
     - *Chuỗi sự kiện:* Xe buýt đỗ trước vạch qua đường. Xe láng giềng C2 lách qua sườn xe buýt với vận tốc chậm ($v < 6$ m/s, $x < 112$ m) để nhập làn. Tầm nhìn của xe ta (ego) bị xe buýt cao lớn che khuất; khi C2 nhô đầu ra, cự ly còn lại quá ngắn khiến planner phanh không kịp và đâm vào C2.
     - *Hình ảnh:* `evidence/shots/bus_bypass.jpg` (chụp lúc va chạm tại $x \approx 104$ m).
  2. **`crosswalk_block` (Chặn lối người đi bộ):**
     - *Chuỗi sự kiện:* Người đi bộ P1 bước qua đường ngay sát đầu xe buýt đỗ. Xe ta bị xe buýt che khuất tầm nhìn sang lề đường bên phải. Khi P1 bước vào làn đường của xe ta, xe ta phát hiện muộn và phanh gấp. Xe ta dừng lại được mà không đâm vào P1, nhưng thân xe đã dừng trớn nằm đè lên vạch qua đường, chặn đứng lối đi của người đi bộ (`kind == "block"`).
     - *Hình ảnh:* `evidence/shots/crosswalk_block.jpg` (chụp lúc xe ta dừng trên vạch).

- **Kiểu cuộc săn không tìm thấy và mức độ tin cậy:**
  - Cuộc săn `cut_in` và `rear_ended` không tìm thấy va chạm nào thuộc đúng kiểu của mình (trong `hunt.json`, cuộc săn `cut_in` tìm ra 36 vụ `bus_bypass`, cuộc săn `rear_ended` tìm ra 3 vụ `bus_bypass` và 9 vụ `crosswalk_block`).
  - *Mức độ tin cậy:* Khá cao đối với mô hình này. Trong thế giới phản ứng, xe sau bám đuôi theo mô hình IDM nên luôn chủ động phanh giữ khoảng cách an toàn, trừ khi có biến cố bất khả thi. Với `cut_in`, xe chuyển làn có gap acceptance nên chỉ chuyển làn khi khoảng cách đủ rộng, ít khi dẫn tới va chạm trực tiếp trên đường thẳng so với nút thắt cổ chai ở cạnh xe buýt đỗ.

- **Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu thì điều gì xảy ra:**
  - Nhìn vào `hunt.json: found`, cả 4 cuộc săn đều tìm thấy `bus_bypass`!
  - Nếu chỉ chạy 1 cuộc săn chung với 1 hàm điểm tổng quát, thuật toán CEM sẽ nhanh chóng co cụm (*collapse*) về kiểu va chạm dễ tìm nhất là `bus_bypass`. Nó sẽ săn mãi ở vùng đó và hoàn toàn bỏ sót các kiểu va chạm hiếm hơn nhưng cực kỳ nguy hiểm như `crosswalk_block`. Do đó, bắt buộc phải có các cuộc săn chuyên biệt cho từng kiểu thất bại.

## 6. Sửa planner (bước E + 3D)

- **Số liệu trích xuất từ `fix.json`:**
  - **Policy được chọn:** `{"cutinLat": 3.0}` (tăng ngưỡng nhận diện xe nhập làn từ 2.15 m lên 3.0 m).
  - **Trên tập TRAIN:**
    - `fail = 4.40e-07`, `fail_hi = 9.41e-07`, `unseen = ['bus_bypass']`
    - Quãng đường: `progress = 207.24 m` (giữ được 100% so với baseline trên TRAIN: 207.25 m).
  - **Trên tập TEST:**
    - **Bản sửa (`chosen_test`):** `fail = 5.89e-07`, `fail_hi_seen = 9.15e-07`, `fail_hi = 0.0388` ($3.88 \times 10^{-2}$), `unseen = ['bus_bypass']`, quãng đường `progress = 210.93 m`, số frame phanh gấp `harsh = 8.14`.
    - **Policy gốc (`baseline_test`):** `fail = 2.48e-03` ($2.48 \times 10^{-3}$), `fail_hi_seen = 4.98e-03`, `fail_hi = 6.26e-03`, `unseen = []`, quãng đường `progress = 211.09 m`, số frame phanh gấp `harsh = 8.76`.

- **Vì sao `fail_hi` của bản sửa ($3.88 \times 10^{-2}$) lại lớn hơn `fail` của policy gốc ($2.48 \times 10^{-3}$) dù bản sửa tốt hơn thật?**
  - Trên tập TEST, bản sửa không gặp bất kỳ vụ va chạm `bus_bypass` nào (`hits = 0`, kiểu này trở thành `unseen`).
  - Theo nguyên lý thống kê của lab, khi một sự kiện có 0 lần xuất hiện trong $n = 400$ mẫu từ hỗn hợp phòng thủ ($\beta = 0.2$), ta **không thể khẳng định xác suất của nó bằng 0**. Cận trên hợp lệ 95% theo quy tắc toán học là `upper_zero_hits = 3.0 / (0.2 * 400) = 0.0375` (khoảng 15/400).
  - Con số `fail_hi` (valid bound) bắt buộc phải cộng thêm cận trên 0.0375 này, nên trông nó lớn hơn. Đây là bài học thực tế: với ngân sách thử nghiệm $n = 400$, ta chưa đủ mẫu để chứng nhận một kiểu va chạm đã biến mất dưới $10^{-2}$ (muốn chứng minh $p < 10^{-3}$ cần tới cỡ 15.000 lần chạy).
  - Tuy nhiên, nếu xét trên các kiểu quan sát thấy có va chạm (`fail_hi_seen`), bản sửa chỉ có $9.15 \times 10^{-7}$, thấp hơn hàng nghìn lần so với policy gốc ($4.98 \times 10^{-3}$), đồng thời giảm số lần phanh gấp từ 8.76 xuống 8.14 frame mà không hề suy giảm quãng đường (210.93 m so với 211.09 m).

- **Núm chỉnh "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại:**
  - Xét núm chỉnh `{"T": 2.0}`: Tăng khoảng thời gian an toàn theo sau từ 1.6 s lên 2.0 s (nghe rất an toàn vì xe giữ cự ly xa hơn với xe phía trước).
  - Thực tế trên TRAIN: Xác suất thất bại `fail` tăng vọt lên **$2.66 \times 10^{-3}$** (gấp 3.3 lần so với baseline $7.91 \times 10^{-4}$)!
  - **Giải thích bằng phản ứng của thế giới:** Khi xe ta tăng khoảng cách $T$, xe ta chủ động lùi lại và để lộ một khoảng trống lớn phía trước. Trong môi trường tương tác phản ứng, khoảng trống này lại kích thích xe hàng xóm C2 vượt lên và lách qua xe buýt ở những góc tiếp cận nguy hiểm hơn, đồng thời khiến xe bám đuôi phía sau bị dồn toa phanh gấp, làm gia tăng tổng số va chạm trong toàn bộ kịch bản.

- **Hình ảnh 3D minh hoạ:** `evidence/shots/fix_comparison.jpg` (kịch bản lách xe buýt: bóng mờ ghost car của policy gốc bị va chạm, trong khi bản sửa với `cutinLat = 3.0` phát hiện xe C2 sớm hơn và dừng xe an toàn).

## 7. Từ mô phỏng ra đường thật

- **Ba giả định ảnh hưởng nhiều nhất trong `space.py`:**
  1. **Tỉ lệ người đi bộ ngó nhìn qua xe buýt (`pedLook = 0.80` trong Town C):**
     - *Dữ liệu thật thay thế:* Dữ liệu video giám sát hành vi người đi bộ qua đường tại các trạm dừng đỗ xe buýt công cộng (trích xuất từ camera đô thị hoặc các tập dữ liệu mở như JAAD / PIE Pedestrian Action Dataset, Waymo Open Dataset).
     - *Nếu sai lệch:* Nếu ngoài đời tỉ lệ người đi bộ quan sát cẩn thận thấp hơn (ví dụ chỉ 40-50% người ngó nhìn, còn lại mất tập trung nhìn điện thoại hoặc vội vã băng qua), thì nguy cơ xe ta chặn lối hoặc va chạm với người đi bộ (`crosswalk_block`) sẽ tăng vọt; con số trong mô phỏng hiện tại đang bị **thấp hơn thực tế (underestimated)**.
  2. **Khe chấp nhận chuyển làn của xe láng giềng (`cut_gap_median = 10.0m` trong Town C):**
     - *Dữ liệu thật thay thế:* Dữ liệu quỹ đạo phương tiện từ trên cao hoặc radar/lidar (như tập dữ liệu NGSIM, highD, CitySim) thống kê hàm phân phối khe thời gian (time headway/gap) mà tài xế người thật chấp nhận khi vượt xe hoặc lách qua chướng ngại vật trong đô thị.
     - *Nếu sai lệch:* Nếu tài xế ngoài đời thận trọng hơn (chỉ vượt khi khe hở $> 15$m), va chạm `bus_bypass` sẽ giảm mạnh, tức con số của mô phỏng đang bị **cao hơn thực tế (overestimated)**. Ngược lại, nếu ở các thành phố có thói quen lái xe chen lấn liều lĩnh hơn (khe $< 8$m), con số của ta lại bị **thấp hơn thực tế**.
  3. **Hệ số ma sát đường ướt (`mu = 0.5`, tỉ lệ đường ướt 20%):**
     - *Dữ liệu thật thay thế:* Dữ liệu khí tượng thủy văn địa phương về số ngày mưa/năm kết hợp với số liệu đo lực cản ma sát bánh xe trên đường nhựa/bê tông theo chuẩn đo lường ASTM E1337 / E274.
     - *Nếu sai lệch:* Mô phỏng gán cứng ma sát đường ướt là 0.5. Ngoài đời khi mưa to có thể xảy ra hiện tượng trượt nước (*hydroplaning*) khiến $\mu$ giảm sâu xuống $0.2 - 0.3$, làm quãng đường phanh dài hơn đáng kể. Do đó con số an toàn của planner trong lab là **lạc quan quá mức so với điều kiện thời tiết xấu thực tế**.

## 8. Giới hạn

- **Ba điều mà xác suất báo cáo KHÔNG nói lên được:**
  1. *Hình học và bối cảnh giao thông quá đơn giản:* Thế giới mô phỏng chỉ là một đoạn phố thẳng 1 làn với 1 xe buýt đỗ và 1 vạch qua đường. Xác suất này không phản ánh được rủi ro ở ngã tư, vòng xuyến, đường có nhiều làn xe chuyển làn phức tạp, đường dốc, góc cua khuất tầm nhìn, hay sự xuất hiện của các phương tiện đa dạng như xe máy, xe đạp, xe tải lớn.
  2. *Mô hình hành vi con người bị rập khuôn (IDM):* Các tác nhân khác chỉ chạy theo thuật toán IDM một chiều dọc. Ngoài đời con người có phản xạ đa dạng và linh hoạt hơn nhiều (bấm còi cảnh báo, nháy đèn, đánh lái né tránh, chuyển làn né chướng ngại vật) hoặc ngược lại có thể mắc lỗi chủ quan do mất tập trung, dùng chất kích thích.
  3. *Hệ thống nhận thức và cảm biến lý tưởng hóa:* Mô phỏng giả định cảm biến biết chính xác tọa độ, vận tốc và kích thước của các vật thể ngay khi chúng ra khỏi vùng che khuất hình học của xe buýt. Thực tế, hệ thống perception có độ trễ nhận diện (latency ~100-300ms), sai số bounding box, nhiễu cảm biến do thời tiết/ánh sáng ngược, và nguy cơ nhận diện sót (*false negative*).
- **Nếu được thêm một tuần, bạn sẽ đo gì tiếp:**
  - Mô phỏng và đo lường ảnh hưởng của **độ trễ nhận thức (perception latency)** từ 50ms đến 300ms lên xác suất va chạm khi xe lách qua xe buýt.
  - Đưa thêm các tác nhân đặc thù như **xe máy luồn lách từ phía sau xe buýt** hoặc tạt đầu bất ngờ ở cự ly gần để kiểm tra khả năng phản ứng của planner trong môi trường giao thông hỗn hợp.
  - Mở rộng thêm khả năng **đánh lái tránh va chạm khẩn cấp (evasive steering)** thay vì chỉ thử nghiệm duy nhất hành vi phanh khẩn cấp dọc làn đường.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:**
  - *Tình huống:* Người đi bộ bất ngờ băng ngang đường ngoài vạch kẻ (Jaywalking Pedestrian — `type: "ped"`), xuất hiện tại $x = 200$ m khi xe ta đã qua khỏi khu vực xe buýt và đang chạy với tốc độ hành trình $v_0 \approx 12$ m/s.
  - *Bối cảnh đời thực:* Tương tự người dân bất ngờ băng qua đường ở khu phố dân cư hoặc từ vỉa hè bước xuống đón xe buýt / bắt taxi ở vị trí không có vạch kẻ đường.
- **Căn cứ:**
  - Xe ta đang di chuyển ở tốc độ cao (12.5 m/s), nếu người đi bộ bước ra ở cự ly kích hoạt gần (`trig` ngắn) và mặt đường ướt ($\mu = 0.5$), quãng đường phanh cần thiết tăng lên ($> 14$ m). Nếu planner chỉ dựa vào phanh dọc thông thường mà không có độ dự đoán ý định sớm, nguy cơ không dừng kịp là rất lớn.
- **Giả định (`hazard.json: why`):**
  - Không gian tham số (`space`): Vị trí $x \in [160, 240]$ m, vận tốc người đi bộ $v \in [1.2, 2.0]$ m/s (chuẩn đi bộ tự nhiên theo HCM/TRB), khoảng cách kích hoạt `trig` $\in [10, 24]$ m (tương ứng thời gian phản xạ $0.8 - 2.0$ s).
  - Tần suất xuất hiện (`exposure`): `0.02` (trung bình 1 lần mỗi 50 chuyến đi trong đô thị).
  - Căn cứ: Báo cáo an toàn NHTSA chỉ ra khoảng 70% tai nạn người đi bộ xảy ra ngoài vạch qua đường.
- **Kết quả thực nghiệm theo `mine.json`:**
  - Tỉ lệ kịch bản khả thi: `accepted = 99.53%` (1.493/1.500 kịch bản hợp lệ về mặt vật lý, vượt xa yêu cầu $\ge 50\%$).
  - **Với Policy gốc (`baseline`):**
    - Tổng va chạm: `5` vụ (trong đó `avoidable = 0`, `unavoidable = 0`, `unattributed = 5`, `low_speed = 0`).
    - Xác suất lỗi planner: $P(\text{lỗi planner} \mid \text{rắc rối}) = 0.00$ (khoảng Wilson 95%: $[0.00, 2.57 \times 10^{-3}]$).
    - Tần suất lỗi mỗi chuyến đi: $0.00$ (cận trên $5.13 \times 10^{-5}$ mỗi chuyến).
  - **Với Policy được chọn (`chosen` - `cutinLat = 3.0`):**
    - Tổng va chạm: `0` vụ (`avoidable = 0`, `unavoidable = 0`, `unattributed = 0`, `low_speed = 0`).
    - Xác suất lỗi planner: $P(\text{lỗi planner} \mid \text{rắc rối}) = 0.00$ (khoảng Wilson 95%: $[0.00, 2.57 \times 10^{-3}]$).
    - Tần suất lỗi mỗi chuyến đi: $0.00$ (cận trên $5.13 \times 10^{-5}$ mỗi chuyến).
  - **Phân biệt ba loại va chạm:**
    - `avoidable` (tránh được): Va chạm mà xe ta có thể tránh được hoàn toàn nếu phanh hết cỡ ($\mu g$) ngay từ khoảnh khắc cảm biến có đường ngắm nhìn thấy rắc rối. Chỉ va chạm này mới tính là lỗi của planner.
    - `unavoidable` (không tránh được): Va chạm mà dù có phanh khẩn cấp tối đa ngay khi nhìn thấy vẫn đâm do cự ly quá gần, giới hạn vật lý không cho phép dừng kịp (không phải lỗi của planner).
    - `unattributed` (không quy kết được): Va chạm xảy ra với tác nhân khác (như xe C2 vượt ẩu) trước khi rắc rối bắt đầu hoặc ở vị trí hoàn toàn độc lập với rắc rối. 5 vụ va chạm của policy gốc đều thuộc loại này (va chạm giao thông nền của thị trấn).
  - **Vì sao "tránh được" là chuẩn lạc quan:**
    1. *Theo chiều đếm nhiều hơn:* Chuẩn `avoidable` giả định xe phanh tối đa tức thời với gia tốc $\mu g$ không hề có độ trễ nhận thức (perception latency) hay thời gian trễ cơ cấu phanh, cũng như bỏ qua sự thoải mái (jerk limit). Vì vậy nó đại diện cho năng lực phanh hoàn hảo lý tưởng, đóng vai trò là cận trên của lỗi planner.
    2. *Theo chiều đếm ít đi:* Chuẩn này chỉ thử nghiệm duy nhất hành động phanh khẩn cấp trên làn đường thẳng, không thử đánh lái né tránh (*evasive steering*). Thực tế ngoài đời tài xế có thể đánh lái nhẹ sang làn bên cạnh để tránh người đi bộ, nên việc không thử đánh lái làm cho một số trường hợp có thể tránh được lại bị xếp vào không tránh được.
  - **So sánh tần suất mỗi chuyến với lách xe buýt:**
    - Tần suất rủi ro của rắc rối này là $\le 5.13 \times 10^{-5}$ mỗi chuyến.
    - Trong khi đó, tần suất va chạm lách xe buýt (`bus_bypass`) của policy gốc trong Thị trấn C lên tới $1.56 \times 10^{-3} - 2.48 \times 10^{-3}$ mỗi chuyến (nguy hiểm gấp hơn 30 - 50 lần).
    - *Giải thích vì sao planner xử lý tốt:* Ở đoạn đường $x \in [160, 240]$ m, không gian thoáng đãng không có xe buýt che khuất, cảm biến xe ta có tầm nhìn thẳng 120 m. Nhờ núm `pedBuffer = 2.5` m giữ cự ly dự đoán với người đi bộ, planner nhận diện được từ sớm và phanh êm ái trước khi xảy ra va chạm. Đặc biệt, bản sửa `cutinLat = 3.0` đã giải quyết triệt để 5 vụ va chạm nền với xe C2 mà policy gốc gặp phải!
- **Hình ảnh minh họa:** `evidence/shots/hazard_studio.jpg` (thiết kế rắc rối người đi bộ trong Chương 10 Studio).
- **Bạn tự lái thì sao:**
  - Khi tự lái thử trong Chương 10, người lái dễ dàng tránh được người đi bộ. Khác với planner (thường duy trì tốc độ rồi phanh gấp khi khoảng cách đến ngưỡng), con người nhả chân ga từ xa (*coasting*), giảm tốc độ vừa phải để người đi bộ băng qua đường trước, sau đó tiếp tục hành trình một cách mượt mà mà không cần dừng hẳn xe.
- **Cách sửa và đề xuất:**
  - *Núm chỉnh có sẵn:* Tăng `pedBuffer` từ 2.5 lên 3.5 m để xe ta duy trì biên khoảng cách an toàn rộng hơn khi dự đoán quỹ đạo người đi bộ.
  - *Logic mới cần bổ sung cho Planner:* Bổ sung module **nhận biết cử chỉ và hướng quay đầu (Pedestrian Intent Prediction / Head Pose Tracking)**. Nếu người đi bộ đang nhìn về phía lòng đường hoặc có xu hướng bước xuống vỉa hè, planner cần chủ động giảm ga từ khoảng cách 40 m trước khi người đó thực sự bước vào làn xe, thay vì chờ đến khi người đó cắt ngang qua tâm đường mới phản ứng.
