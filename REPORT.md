MSSV: 2A202602521

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **D**

> Ghi chú: MSSV có chữ cái nên `run_all.py` không đọc được từ dòng đầu file này (regex chỉ nhận chữ số). Mọi bước đã chạy với `LAB11_MSSV=2A202602521`, đúng cách grader truyền MSSV; `meta.json: _run.mssv = "2A202602521"`. Không bước nào dùng `--quick` (`meta.json: steps.*.quick = false`).

## 1. Phát lại log có trung thực không? (bước A)

Trong `replay.json`: bao nhiêu kịch bản bị bỏ vì người lái gây va chạm, bao nhiêu va chạm khi planner chạy trên log phát lại, bao nhiêu trong thế giới phản ứng, theo từng loại tiếp xúc. Giải thích vì sao phần lớn va chạm của "phát lại" là `rear`, và vì sao đó là va chạm **giả**. Chọn một kịch bản trong `replay_only` và nói chuyện gì đã xảy ra.

| `replay.json` | giá trị |
|---|---|
| `n` / `dropped_human_crash_logs` / `n_used` | 400 / 0 / 400 |
| phát lại: `replay.crashes` | **122** (`rear` 111, `side` 11) |
| phản ứng: `reactive.crashes` | **0** |
| `crash_in_both` | 0 |

Hầu hết va chạm khi phát lại là `rear` vì xe phía sau (chủ yếu xe bám đuôi TG) chạy **đúng quỹ đạo đã ghi** khi người lái thật cầm lái. Planner thận trọng hơn người: nó giảm tốc sớm hơn và mạnh hơn khi tới gần xe buýt và vạch qua đường. Log không biết planner sẽ phanh sớm, nên TG trong log vẫn giữ tốc độ cũ và đâm vào đuôi. Ngoài đời TG sẽ thấy xe ta phanh và phanh theo, đúng như thế giới phản ứng làm (0 va chạm). Vì vậy đây là va chạm **giả**: do cách kiểm thử gây ra, không phải lỗi của planner.

Ví dụ `replay_only[0]` (kịch bản 4, `contact = rear`, `min_ttc_reactive = 3.35` s): ở t = 16,8 s planner đã giảm xuống khoảng 1,0 m/s gần vạch qua đường, trong khi người lái trong log vẫn chạy khoảng 10,6 m/s. TG chạy theo log ở khoảng 5,3 m/s và tông vào đuôi xe ta. Trong thế giới phản ứng, TG giảm tốc theo và không có va chạm, TTC nhỏ nhất là 3,35 s. Ảnh: `evidence/shots/q1_replay.jpg` (chương 9, R-01, lúc 16,8 s).

## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

| Luật | Căn cứ vật lý |
|---|---|
| a_lat = (π/cutDur)²·3,5/2 > μg thì loại | Chuyển làn hình cosin 3,5 m cần gia tốc ngang này. Lốp chỉ cho tối đa μg trước khi trượt, nên chuyển làn quá gấp là không làm được. |
| cutBrake > μg thì loại | Gia tốc phanh không vượt được ma sát (μ = 0,8 khô, khoảng 7,8 m/s²; μ = 0,5 ướt, khoảng 4,9 m/s²). |
| pedV > 2,5 m/s thì loại | Người đi bộ bình thường đi 0,9–1,5 m/s. Trên 2,5 m/s là chạy, không còn thuộc mô hình người qua đường. |
| cutV ∉ (0, 22] m/s thì loại | Xe cắt làn phải đang chạy. 22 m/s (khoảng 80 km/h) là trần hợp lý trên phố đô thị này. |

`hunt.json: <mode>.infeasible`: bus_bypass 3, cut_in 8, crosswalk_block 2, **rear_ended 14** (nhiều nhất). Trong `cloud.json`, cả 27 điểm bất khả thi đều vi phạm luật gia tốc ngang (cutDur quá ngắn). Điểm nguy hiểm của cuộc săn `rear_ended` là TTC của TG tới xe ta. Điểm này tăng khi xe ta phải phanh gấp, và cách dễ nhất để ép xe ta phanh gấp là cho C2 chuyển làn **thật gắt** ngay trước mũi xe. CEM chỉ thấy điểm nguy hiểm, không thấy vật lý, nên bị kéo về vùng cutDur nhỏ, nơi "nguy hiểm nhất" chính vì không thể xảy ra. Nếu **đếm** chúng là va chạm, ước lượng sẽ bị thổi phồng bởi những thất bại không thể có ngoài đời. Tệ hơn, CEM còn tiếp tục khớp q vào vùng đó, nên cả ước lượng IS lẫn danh sách kiểu thất bại đều sai.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k, n, p̂ và khoảng Wilson 95 %. Với p̂ đó, cần bao nhiêu lần chạy để khoảng tin cậy hẹp lại còn ±20 % của p? Có kiểu thất bại nào **chưa hề xuất hiện** trong 3 000 lần chạy không? Con số đúng cần báo cho kiểu đó là gì?

`mc.json`: `crashes` k = **1**, `n` = **3000**, `p` = **3,33·10⁻⁴**, Wilson 95 % [`lo` **5,88·10⁻⁵**, `hi` **1,89·10⁻³**]. Vụ duy nhất là `bus_bypass` (`modes`). 10 kịch bản bất khả thi không được chạy (`infeasible = 10`).

Muốn sai số ±20 %: 1,96·√(p(1−p)/n) = 0,2p, suy ra n ≈ 96/p = 96 / 3,33·10⁻⁴ ≈ **288 000 lần chạy**, gần gấp 100 lần số đã chạy.

`cut_in`, `crosswalk_block`, `rear_ended` và `other` **chưa xuất hiện lần nào** (`per_mode.*.k = 0`). Không được báo 0 cho các kiểu này. Con số đúng là cận trên Wilson 95 %: `per_mode.<mode>.hi` = **1,28·10⁻³**. Nghĩa là, với 3 000 lần chạy, ta chỉ nói được mỗi kiểu xảy ra **dưới 1,28·10⁻³**.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

`estimate.json: seeds[i].per.<mode>` (n = 600 mỗi kiểu, mỗi seed):

| kiểu | seed | p | hits | ess | top_share |
|---|---|---|---|---|---|
| bus_bypass | 0 | 9,35·10⁻⁴ | 33 | 142 | 0,35 |
| bus_bypass | 1 | 2,12·10⁻³ | 23 | 122 | **0,62** |
| bus_bypass | 2 | 2,34·10⁻⁴ | 32 | 95 | **0,52** |
| crosswalk_block | 0 | 1,78·10⁻⁶ | 39 | 115 | **0,90** |
| crosswalk_block | 1 | 1,15·10⁻⁷ | 34 | 118 | 0,30 |
| crosswalk_block | 2 | 8,56·10⁻⁸ | 41 | 139 | 0,28 |
| cut_in | 0 / 1 / 2 | 0 (cận `upper_zero_hits` = 0,025) | 0 | 121 / 124 / 112 | 0 |
| rear_ended | 0 / 1 / 2 | 0 (cận `upper_zero_hits` = 0,025) | 0 | 110 / 122 / 112 | 0 |

`total`: 9,37·10⁻⁴ / 2,12·10⁻³ / 2,34·10⁻⁴, **trung vị 9,37·10⁻⁴**. `other` = 0 với cận 1,28·10⁻³, lấy từ `mc.json`.

- **Các seed không đồng ý nhiều.** bus_bypass chênh 9 lần (2,3·10⁻⁴ đến 2,1·10⁻³). crosswalk_block chênh khoảng 20 lần (8,6·10⁻⁸ đến 1,8·10⁻⁶). Với Monte Carlo, bus_bypass khớp về bậc: MC 3,33·10⁻⁴ nằm trong khoảng [5,9·10⁻⁵, 1,9·10⁻³], gần seed 2 và seed 0. crosswalk_block thì IS bảo khoảng 10⁻⁷, trong khi MC chỉ nói được < 1,28·10⁻³. Hai con số không mâu thuẫn, nhưng chênh 20 lần giữa các seed chính là dấu hiệu của **cái bẫy 3**: cuộc săn tìm được *một* cách chặn lối rất hiếm, chưa chắc là cách hay xảy ra nhất. Con số 10⁻⁷ có thể thấp hơn sự thật nhiều.
- **`top_share` > 0,5** ở crosswalk_block seed 0 (0,90), bus_bypass seed 1 (0,62) và seed 2 (0,52). Khi đó một mẫu duy nhất mang hơn nửa giá trị p = mean(w·y). Bỏ mẫu đó đi, ước lượng giảm hơn một nửa. Vì vậy con số thực ra là trọng số w = p/q của một mẫu chia cho n, không phải trung bình của nhiều quan sát. Đó cũng là lý do seed 0 của crosswalk_block cao gấp khoảng 15 lần hai seed kia. ESS khoảng 100–140 trên 600 không phát hiện được điều này, vì ESS đo độ lệch của **mọi** trọng số, còn `top_share` nhìn đúng các mẫu có y = 1.
- **Ngân sách so với MC.** `budget` = **16 800** lần chạy (hunt + IS cho 4 kiểu × 3 seed).
  - bus_bypass seed 0: sai số tương đối 1,96·se/p = 1,96·3,91·10⁻⁴ / 9,35·10⁻⁴ ≈ ±82 %. MC cần n ≈ (1,96/0,82)²/p ≈ 6 100 lần cho cùng độ chính xác. Với kiểu "dễ" này, IS chỉ ngang MC.
  - crosswalk_block seed 1: ±76 %, MC cần n ≈ (1,96/0,76)² / 1,15·10⁻⁷ ≈ **5,8·10⁷** lần, gấp khoảng 3 400 lần toàn bộ `budget`.

  IS chỉ đáng tiền khi xác suất thật sự nhỏ.
- **`other`** (va chạm không thuộc kiểu nào) không có điểm nguy hiểm riêng để săn: nó là "mọi thứ khác", nên CEM không biết kéo q về đâu. Phần của nó lấy từ Monte Carlo C1: `estimate.json: other` = k 0 / n 3000, `hi` 1,28·10⁻³, `source` "mc.json".

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

| kiểu | ảnh | chuỗi sự kiện |
|---|---|---|
| **bus_bypass** | `evidence/shots/q5_bus_bypass.jpg` (B-01, 12,5 s), `q5_bus_bypass_2.jpg` (B-02, 12,8 s) | Xe buýt đỗ chắn làn ngoài E2. C2 (xe tím) đang ở làn E2 phía sau xe buýt, phải lách sang làn xe ta. C2 chờ một khe đủ rộng (gap acceptance), thấy khe phía trước mũi xe ta là đủ, rồi chen vào ở tốc độ thấp (x < 112 m, v < 6 m/s). Planner chưa coi C2 là "xe phía trước" vì C2 còn lệch ngang hơn `cutinLat = 2,15` m, nên không phanh kịp và đâm vào hông hoặc đuôi C2. Đường ướt (μ 0,5) làm quãng phanh dài thêm. |
| **crosswalk_block** | `q5_crosswalk_block.jpg` (C-01, 21,4 s), `q5_crosswalk_block_2.jpg` (C-02, 20,9 s) | Người đi bộ P1 bước ra từ sau xe buýt đỗ trước vạch qua đường. Xe buýt che tầm nhìn nên xe ta thấy P1 muộn. Planner phanh gấp và dừng được, không đâm, nhưng thân xe đã nằm trên vạch: P1 không bước vào thân xe nên phải đứng lại (sự kiện `block`). Đường ướt làm xe trượt thêm vài mét. |

**Không tìm thấy:** `cut_in` và `rear_ended`. `hunt.json: cut_in.found = {bus_bypass: 35, crosswalk_block: 1}` và `rear_ended.found = {bus_bypass: 5, crosswalk_block: 9}`: hai cuộc săn này đều trượt sang kiểu khác. `estimate.json` cho cả hai p = 0, và MC thấy 0/3000. Mình tin ở mức vừa phải rằng hai kiểu này **hiếm hơn** bus_bypass trong thị trấn D, chứ không phải không có. Cận đúng để báo là `upper_zero_hits` = 0,025 (IS) hoặc 1,28·10⁻³ (MC Wilson). Điểm nguy hiểm của `cut_in` dừng ở mức −3,4 sau 8 vòng (`log_hunt`), nghĩa là CEM chưa "chạm" được vùng va chạm của nó; nhiều vòng hơn có thể tìm thấy.

**Một cuộc săn chung:** chính các cuộc săn riêng đã cho thấy chuyện gì xảy ra. Cuộc săn `cut_in` tìm ra **35 vụ bus_bypass** và chỉ 1 vụ khác (`hunt.json: cut_in.found`), còn `rear_ended` cũng trôi về bus_bypass và crosswalk_block. Một cuộc săn chung sẽ **co về kiểu dễ nhất** (bus_bypass: `bus_bypass.found = {bus_bypass: 112}`) và lặp mãi ở đó. crosswalk_block, vốn hiếm hơn hàng nghìn lần, sẽ gần như không bao giờ được ước lượng. Vì vậy phải săn riêng từng kiểu, mỗi kiểu với điểm nguy hiểm của nó.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

**Policy được chọn** (`fix.json: chosen.policy`): `{cutinLat: 2.6}`, tức coi một xe đang lấn sang là "xe phía trước" từ khi nó còn cách 2,6 m ngang (gốc 2,15 m).

| | fail | fail_hi_seen | fail_hi | unseen | progress (m) | harsh (khung) |
|---|---|---|---|---|---|---|
| TRAIN gốc (`baseline_train`) | 7,29·10⁻⁴ | 1,97·10⁻³ | 1,97·10⁻³ | – | 215,7 | 8,35 |
| TRAIN bản sửa (`chosen`) | 1,84·10⁻⁷ | 3,45·10⁻⁷ | 3,45·10⁻⁷ | bus_bypass | 215,7 | 7,99 |
| **TEST bản sửa** (`chosen_test`) | **3,70·10⁻⁷** | **6,24·10⁻⁷** | **3,88·10⁻²** | bus_bypass | 217,4 | 7,60 |
| **TEST gốc** (`baseline_test`) | **5,55·10⁻⁵** | 9,65·10⁻⁵ | **1,38·10⁻³** | – | 217,4 | 7,76 |

TRAIN chỉ dùng để sàng; con số báo cáo là TEST. Trên TEST, bản sửa không còn vụ bus_bypass nào (0 hits trong cuộc săn mới nhắm vào chính nó). `fail` giảm từ 5,55·10⁻⁵ xuống 3,70·10⁻⁷, phần còn lại là crosswalk_block, giống hệt policy gốc. Quãng đường gần như không đổi (217,39 so với 217,40 m) và số khung phanh gấp giảm nhẹ (7,60 so với 7,76).

**Vì sao `fail_hi` của bản sửa (3,88·10⁻²) lớn hơn cả `fail` của policy gốc:** bus_bypass trở thành `unseen`, tức 0 vụ. 0 vụ trong n = 400 không chứng minh được p = 0, nên cận hợp lệ phải cộng `upper_zero_hits` = 3/(0,2·400) = 0,0375. Cộng thêm cận của các va chạm khác trên tập tự nhiên (`other_crashes.hi` = 1,28·10⁻³) thì ra khoảng 3,88·10⁻². Đây là giới hạn của ngân sách, không phải bản sửa tệ hơn. Muốn chứng nhận bus_bypass < 10⁻³ phải chạy cỡ 15 000 lần trên đúng vùng đó. `fail_hi_seen` = 6,24·10⁻⁷ là cái dữ liệu thực sự cho thấy.

**Ảnh bóng mờ:** `evidence/shots/q6_ghost_fix.jpg` (B-03, 12,8 s). Xe xanh đặc là policy gốc, đâm vào C2 đang lách qua xe buýt. Bóng mờ xanh phía sau là bản sửa `cutinLat = 2,6`: nó nhận C2 là xe phía trước sớm hơn, giảm tốc và đứng sau, không va chạm. (Với các kịch bản săn được dưới policy gốc, trình xem chạy xe chính theo policy gốc và vẽ bản sửa thành bóng mờ.)

**Núm "nghe thì an toàn hơn" nhưng không an toàn hơn:**
- `T = 2,0` (giữ khoảng cách xa hơn, gốc 1,6 s): TRAIN `fail` 7,40·10⁻⁴ (hi 1,84·10⁻³), không tốt hơn gốc (7,29·10⁻⁴), và mất quãng đường (204,9 so với 215,7 m). Lý do là thế giới phản ứng lại: C2 lách xe buýt bằng gap acceptance, chờ khe đủ rộng. Xe ta để khoảng trống phía trước lớn hơn thì **khe đó xuất hiện thường hơn**, nên C2 chen vào ngay trước mũi xe ta nhiều hơn. Khoảng cách an toàn với xe trước bị "tiêu" bởi chính xe chen vào.
- `cautionV` không đơn điệu: 6,0 m/s cho `fail` 2,39·10⁻⁴, **cao hơn** 6,5 m/s (1,90·10⁻⁴). Chạy chậm hơn làm xe ta tới xe buýt muộn hơn, trùng đúng lúc C2 hoặc người đi bộ bắt đầu đi. Ngược lại, `cautionV` 5,0 giảm `fail` (4,03·10⁻⁵) nhưng mất 20 % quãng đường (172,1 m), nên bị `choose_policy` loại.

*Cái bẫy TRAIN:* `cutinLat = 2,6` thắng TRAIN một phần vì bus_bypass là `unseen` (mẫu TRAIN rút ở nơi **policy gốc** thất bại). TEST xác nhận bus_bypass thực sự giảm, nhưng không chứng nhận được nó đã biến mất (`fail_hi` 3,88·10⁻²).

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

| Giả định | Dữ liệu thật thay nó | Nếu sai thì con số sai thế nào |
|---|---|---|
| **Khe chấp nhận của xe chuyển làn** (`cutGap`), quyết định C2 có chen vào khi lách xe buýt không. Đây là động cơ của bus_bypass, kiểu chiếm gần hết `total`. | Quỹ đạo xe thật đã gán nhãn chuyển làn: highD / inD / exiD (drone, Đức), NGSIM (Mỹ), hoặc video drone tại các điểm dừng xe buýt ở Hà Nội / TP.HCM. Đo phân phối khe (m và s) mà người lái thực sự chấp nhận khi bị chắn làn. | Người thật (nhất là xe máy, xe con ở Việt Nam) chấp nhận khe **nhỏ hơn** giả định, nên bus_bypass xảy ra nhiều hơn và P(bus_bypass) ≈ 10⁻³ đang **thấp hơn** sự thật. Nếu ngược lại thì con số đang cao. |
| **Tỉ lệ người đi bộ ngó qua xe buýt** (`pedLook`), quyết định P1 có thấy xe ta trước khi bước ra không. Ảnh hưởng crosswalk_block. | Đếm bằng video tại các vạch qua đường có điểm dừng xe buýt: bao nhiêu người dừng lại nhìn trước khi bước ra khỏi chỗ bị che. Có thể dùng dữ liệu hành vi người đi bộ trong các bộ dữ liệu như PIE / JAAD (video có nhãn ý định qua đường). | Nếu ngoài đời ít người ngó hơn giả định thì crosswalk_block (≈ 10⁻⁷) bị **đánh giá thấp**; IS vốn đã có xu hướng thấp (cái bẫy 2, 3), nên sai số cộng dồn. |
| **Ma sát và tỉ lệ đường ướt** (`mu` 0,8 / 0,5 và xác suất ướt). Hầu hết va chạm chụp được đều trên đường ướt. | Số ngày mưa theo giờ cao điểm (trạm khí tượng, ví dụ dữ liệu mưa của Trung tâm Dự báo KTTV quốc gia), đo hệ số bám mặt đường (thiết bị đo ma sát, hoặc dữ liệu ABS/ESP từ đội xe). | Nếu đường ướt thường xuyên hơn hoặc μ ướt thấp hơn 0,5 (mặt đường nhẵn, có dầu) thì quãng phanh dài hơn, mọi xác suất đều **thấp hơn** sự thật. Nếu thành phố khô hơn thì ngược lại. |

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

1. **Một con phố, 24 s, một xe buýt.** Mọi con số là "trên 24 giây đi qua đúng đoạn phố có một xe buýt đỗ trước vạch", không phải "mỗi km" hay "mỗi chuyến" thật. Không có giao lộ, vòng xuyến, xe máy, hay nhiều xe buýt.
2. **Các tác nhân khác chỉ là IDM cộng gap acceptance và vài luật nhường.** Không có người lái hung hăng, xe vượt ẩu, người đi bộ chạy ngang giữa dòng xe (trừ rắc rối ở câu 9), và không ai mất tập trung. Va chạm do hành vi ngoài mô hình đều không được đếm.
3. **Cảm biến hoàn hảo trừ phần bị xe buýt che, và xe ta chỉ phanh, không đánh lái.** Không có nhiễu, trễ hay nhận nhầm. Trong thực tế, trễ nhận thức 0,2–0,5 s sẽ biến một phần va chạm "tránh được" thành không tránh được.

**Thêm một tuần:** (a) săn tiếp `cut_in` và `rear_ended` với nhiều vòng hơn và kiểm tra lại crosswalk_block bằng Monte Carlo tập trung (khoảng 15 000 lần trong vùng q) để biết con số 10⁻⁷ thấp đến đâu; (b) thêm trễ cảm biến và nhiễu vị trí rồi đo lại toàn bộ; (c) thay ba giả định ở câu 7 bằng dữ liệu drone tại một điểm dừng xe buýt thật.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** bạn dựng gì, và ngoài đời nó giống tình huống nào.
- **Căn cứ:** vì sao bạn nghĩ planner sẽ xử lý kém.
- **Giả định:** `space` và `exposure` lấy từ đâu (`hazard.json: why`).
- **Kết quả**, theo `mine.json`:
  - số va chạm `avoidable` (tránh được bằng phanh hết cỡ, biết trước), `unavoidable`, `unattributed`, và `low_speed`, với policy gốc và với policy bạn chọn; `accepted` (bao nhiêu phần `space` của bạn là khả thi);
  - P(lỗi planner | rắc rối) kèm khoảng Wilson;
  - tần suất mỗi chuyến, đặt cạnh tần suất lách xe buýt của bạn.
- **Hình:** một ảnh chương 9 hoặc 10 ở lúc va chạm.
- **Bạn tự lái thì sao:** nếu bạn đã thử lái, bạn tránh được không, và bạn làm gì khác planner?
- **Cách sửa:** núm chỉnh nào sửa được rắc rối này? Nếu không núm nào sửa được, planner cần thêm **logic gì** (mô tả, không cần code)?

**Rắc rối.** Một thẻ `ped` (H0, từ lề nam, bên phải xe ta): **hành khách vừa xuống xe buýt đỗ ở điểm dừng rồi băng qua đường ngay trước hoặc sau thân xe buýt, ngoài vạch qua đường** (x = 96–118 m; vạch ở x = 122). Ngoài đời đây là tình huống kinh điển ở điểm dừng xe buýt: người vội đi tắt qua đầu hoặc đuôi xe buýt, bị thân xe buýt che, bước ra khi xe bên cạnh đang tới.

**Căn cứ.** Planner chỉ chạy chậm thận trọng (`cautionV`, `cautionRange`) quanh **vạch qua đường bị che**. Logic dự đoán người đi bộ (`pedBuffer`) chỉ phản ứng khi đã thấy người. Người bước ra từ thân xe buýt, ngoài vạch, nằm đúng giữa hai cơ chế đó: bị che nên thấy muộn, và không phải chỗ planner chờ sẵn người.

**Giả định (`hazard.json: why`).**
- `space`:
  - `h0_x` [96, 118]: dọc thân xe buýt đỗ (tâm 106 m, dài khoảng 12 m) cộng vài mét hai đầu.
  - `h0_v` [0,9, 2,5] m/s: 0,91 m/s là tốc độ thiết kế cho người già, khoảng 1,2–1,45 m/s cho người trẻ (Knoblauch, Pietrucha & Nitzburg 1996, TRR 1538); 2,5 m/s là đi vội.
  - `h0_trig` [10, 50] m: người bước ra khi xe ta còn cách 1–4 s.
- `exposure` = **0,05 mỗi chuyến**: mỗi chuyến trong thị trấn này qua đúng 1 xe buýt đỗ, và **giả định** 5 % số lần đó có hành khách băng ngang quanh xe buýt khi xe ta tới gần. Mình chưa tìm được nghiên cứu đo trực tiếp tỉ lệ này. Hai nghiên cứu sau xác nhận hành vi này phổ biến và nguy hiểm: video ở điểm dừng giữa đoạn tại Kerala (NITK) cho thấy xe buýt đỗ che tầm nhìn là nguyên nhân chính; nghiên cứu ca-chứng ở Lima (Injury Prevention 2015) cho OR 2,36 với điểm dừng giữa đoạn. Con số 5 % là chỗ yếu nhất và cần đếm bằng video thực địa.

**Kết quả** (`mine.json`, `world` = "town", giao thông của thị trấn D):

- `exact.planner` = `[["ego", "H0", 17.35]]`: cảnh dựng va chạm người băng ngang ở 17,35 s.
- `accepted` = **0,994**: 1 491 / 1 500 khả thi, 9 bị loại vì `is_feasible` của thế giới.

| | va chạm | avoidable | unavoidable | unattributed | low_speed | avoidable_foreknown |
|---|---|---|---|---|---|---|
| policy gốc (`baseline`) | 17 | **16** | 0 | 1 | 0 | 16 |
| bản sửa `cutinLat 2.6` (`chosen`) | 16 | **16** | 0 | 0 | 0 | 16 |

- P(lỗi planner | rắc rối) = 16/1 491 = **1,07·10⁻²**, Wilson 95 % [**6,62·10⁻³**, **1,74·10⁻²**]. Giống hệt cho cả hai policy.
- Tần suất mỗi chuyến = 0,05 × 1,07·10⁻² = **5,37·10⁻⁴** (`per_trip`; cận trên `per_trip_hi` 8,68·10⁻⁴).
- So với lách xe buýt của mình: P(bus_bypass) ≈ 9,4·10⁻⁴ (trung vị `estimate.json`), hoặc 3,33·10⁻⁴ theo `mc.json`. **Cùng bậc độ lớn**, tức rắc rối này nguy hiểm ngang kiểu thất bại chính của thị trấn, *nếu* exposure 5 % đúng.
- **Bản sửa ở bước E không cứu được** (16 → 16): `cutinLat` chỉ đổi cách planner nhìn xe chen làn, không đụng tới người đi bộ.
- `unattributed` = 1 ở policy gốc là một vụ bus_bypass bình thường của thị trấn, xảy ra trước khi H0 bắt đầu đi. Bản sửa tránh được vụ đó, đúng như câu 6.

**Vì sao "tránh được" là chuẩn lạc quan, theo cả hai hướng.**
- **Đếm nhiều hơn thực tế:** người phanh "oracle" biết đây là rắc rối ngay khung hình đầu tiên nó vừa bắt đầu vừa nhìn thấy được, và phanh tối đa μg ngay lập tức, không trễ phản xạ, không do dự. Một planner thật cần thời gian xác nhận đó là người sắp băng qua. Vì vậy `avoidable` là **cận trên** của số va chạm mà ta có quyền đòi planner tránh.
- **Đếm ít đi:** chỉ thử phanh, không thử đánh lái. Vài vụ `unavoidable` (ở đây là 0) có thể tránh được bằng đánh lái nhẹ.

`avoidable_foreknown` = `avoidable` = 16: trong mọi vụ, phanh ngay từ lúc người bắt đầu đi (dù còn bị che) cũng cứu được, đúng như phanh từ lúc thấy. Ví dụ đầu trong `examples`: người bắt đầu đi ở 13,15 s, xe ta thấy được từ 14,5 s, va chạm ở 15,85 s. Kể cả khi bị xe buýt che 1,35 s, vẫn còn 1,35 s từ lúc thấy để phanh, đủ để dừng. Vậy đây là **lỗi của planner** (phản ứng quá nhẹ, quá muộn), không phải do bị che không thể biết.

**Hình:** `evidence/shots/q9_hazard_bus_passenger.jpg` (chương 9, GB-01, 15,9 s: xe ta đâm người băng ngang ngay trước vạch, cạnh xe buýt) và `q9_hazard_bus_passenger_2.jpg` (GB-02, 13,5 s).

**Bạn tự lái thì sao:** mình chưa thử lái cảnh này ở chương 10 (nút "Bạn"), nên không có số so sánh.

**Cách sửa.**
- **Núm chỉnh:** `cautionRange` và `cautionV` chỉ tác dụng quanh vạch, nên muốn cứu phải hạ `cautionV` mạnh (ví dụ 5 m/s). Nhưng ở câu 6, `cautionV` 5,0 đã mất 20 % quãng đường (172,1 m), vượt ngưỡng 5 %. Tăng `pedBuffer` chỉ có ích sau khi đã thấy người. Đây là đề xuất chưa được đo trên `mine`.
- **Logic mới:** coi **mọi xe buýt đang đỗ ở điểm dừng là một vùng che có thể sinh ra người đi bộ**, không chỉ vạch qua đường. Khi đi ngang xe buýt đỗ:
  1. giới hạn tốc độ sao cho quãng phanh từ mép thân xe buýt nhỏ hơn khoảng cách tới đó (logic "điểm mù sinh người ảo", phantom pedestrian);
  2. khi một người đi bộ vừa xuất hiện từ mép vùng che, phanh tới mức tối đa ngay thay vì phanh "thoải mái" `b` = 2 m/s².

  Logic (2) chính là thứ tách planner khỏi oracle trong 16 vụ `avoidable`.
