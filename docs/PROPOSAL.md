# System Design Sandbox — Proposal & Plan

**Nhóm:** Nguyễn Thành Tiến – 23127128 · Thái Minh Huy – 23127379 · Nguyễn Tấn Phát – 23127449

**Repository:** https://github.com/tiennt220805/LoadGraph.git

> Các thuật ngữ kỹ thuật (RPS, p95, seed, burst, ...) được giải thích ở cuối file.

## 1. Sản phẩm và người dùng

**Người dùng:** Tiến, thành viên của nhóm, sinh viên năm 3 môn Thiết kế phần mềm. Nhóm đang làm bài tập "thiết kế hệ thống đặt vé chịu được 5.000 RPS" (5.000 request mỗi giây). Tiến đã học về cache, load balancer, queue trên lớp nhưng chưa vận hành hệ thống thật, nên chưa có cảm giác về các con số. Sản phẩm dành cho những sinh viên năm 3 đang ở đúng hoàn cảnh này.

**Vấn đề:** Trước khi nộp bài, Tiến không biết thành phần nào trong thiết kế sẽ quá tải đầu tiên khi tải lên 5.000 RPS, và vì sao.

**Cách làm hiện nay:**

- Tiến vẽ sơ đồ trên Excalidraw.
- Tự ước lượng, ví dụ "DB chịu khoảng 1.000 query/giây nên chắc ổn".
- Dán mô tả vào ChatGPT và hỏi "thiết kế này có ổn không".

Cách này có hai điểm yếu. Câu trả lời của ChatGPT nghe hợp lý nhưng không kiểm chứng được, và hỏi lại có thể ra kết luận khác. Kết quả thật chỉ đến sau khi nộp bài: nhóm Tiến chỉ biết DB là điểm nghẽn khi đọc nhận xét của giảng viên, lúc đó không sửa được nữa.

**Sản phẩm:** Một web app để:

1. Kéo thả các thành phần (Load Balancer, API, Auth, Cache, DB, Queue) thành một sơ đồ.
2. Đặt thông số cho từng thành phần: capacity (số request/giây tối đa xử lý được), latency (thời gian xử lý), queue size (số request tối đa được xếp hàng chờ), tỉ lệ đọc/ghi.
3. Chạy mô phỏng tải bằng engine viết bằng Go. Engine dùng seed cố định, nên cùng một kịch bản luôn cho cùng một kết quả.
4. Xem chỗ nào bị nghẽn, nhờ AI giải thích nguyên nhân và đề xuất cách sửa, rồi chạy lại để kiểm tra. Tất cả diễn ra trước khi nộp bài.

## 2. Tính năng AI và chi phí khi AI sai

**Chỉ có một tính năng AI:** AI chỉ ra node nào đang nghẽn và vì sao, đề xuất một cách sửa kèm dự đoán bằng số. Việc kiểm tra dự đoán đó do code làm, bằng cách chạy mô phỏng lại.

### Cách hoạt động

1. Tiến chạy kịch bản burst (tải tăng đột ngột lên 5.000 RPS). Engine trả về số liệu cho từng node:
   - mức sử dụng (utilization);
   - p95;
   - tỉ lệ lỗi, tách thành lỗi do hàng đợi đầy và lỗi do quá thời gian;
   - số request đang chờ trong hàng đợi;
   - tỉ lệ đọc/ghi thực tế.
2. AI nhận sơ đồ và bản tóm tắt số liệu, rồi trả về ba thứ:
   - **Node nghẽn và nguyên nhân.** Nguyên nhân phải chọn từ một danh sách có sẵn (xem bên dưới), và mỗi nhận định phải trích một số liệu làm bằng chứng.
   - **Một chỉnh sửa nhỏ** lên sơ đồ.
   - **Một dự đoán bằng số**, ví dụ "tỉ lệ lỗi của DB dưới 1%".
3. Backend kiểm tra chỉnh sửa có hợp lệ không (bằng validator), tạo bản so sánh thay đổi (diff), rồi lưu thành **phiên bản mới**. Bản gốc không bị ghi đè.
4. Tiến duyệt và chạy lại đúng kịch bản cũ. **Code** (không phải AI) so dự đoán với số đo thực tế và ghi kết quả: _đúng_, _sai_ hoặc _không cải thiện_.

Danh sách nguyên nhân AI được chọn:

| Mã | Ý nghĩa |
| --- | --- |
| `read_load` | Tải đọc quá lớn |
| `write_load` | Tải ghi quá lớn |
| `queue_overflow` | Hàng đợi bị tràn |
| `capacity` | Node không đủ khả năng xử lý |
| `upstream_cascade` | Lỗi lan từ node phía sau |

### Không để AI tự chấm mình đúng

AI **không được tự đặt các giả định** như tỉ lệ cache hit hay tỉ lệ đọc/ghi. Nó chỉ dùng giá trị mặc định của loại node, hoặc giá trị Tiến nhập. Mọi giả định đều hiển thị để Tiến xác nhận.

Khi kiểm tra lại, code chạy thêm một lượt **"xấu hơn"**. Ở lượt này, các tham số mà AI không kiểm soát bị làm tệ đi: hit rate giảm một nửa, tỉ lệ ghi tăng 20 điểm phần trăm, burst tăng 20%. Hệ thống chỉ báo "cải thiện" khi **cả hai lượt chạy** đều tốt hơn mức dao động bình thường giữa các seed khác nhau.

### Vì sao cần AI

Code Go tính sẵn những điều chắc chắn:

- node nào là điểm lỗi duy nhất (SPOF);
- node nào không nối với node nào;
- node nào có capacity thấp hơn lượng tải đổ vào;
- node nào có mức sử dụng cao nhất.

Phần khó viết thành luật cố định là **kết hợp nhiều dấu hiệu từ nhiều node**. Ví dụ: API bị quá thời gian vì DB phía sau đầy hàng đợi; hoặc nghẽn do ghi nhưng đề bài cấm thêm server. AI còn phải chọn cách sửa đúng với nguyên nhân và giải thích theo bối cảnh đề bài. Ở CP6, nhóm so sánh AI với một bộ luật viết tay (rule-based) trên cùng các ca thử.

### Khi AI sai

**Ai bị ảnh hưởng:** Tiến và hai bạn cùng nhóm, vì cả ba nộp chung một thiết kế và nhận cùng một điểm.

**Lỗi dễ phát hiện (thiệt hại nhỏ, sửa lại được).** Cách sửa không giúp gì, ví dụ thêm cache trong khi DB nghẽn vì ghi. Code chạy lại và ghi "không cải thiện". Tiến mất một vòng sửa–chạy lại (khoảng 5 phút) và 1 trong 20 lượt AI mỗi ngày, rồi quay về phiên bản cũ bằng một thao tác.

**Lỗi khó phát hiện (thiệt hại lớn, không sửa lại được).** AI chỉ đúng node nhưng sai nguyên nhân, và cách sửa vẫn có vẻ hiệu quả.

- Ví dụ: DB nghẽn vì hàng đợi chỉ có 50 chỗ nên bị tràn khi burst. AI nói "DB thiếu capacity" và đề xuất thêm replica.
- Gấp đôi capacity cũng khiến hàng đợi bớt tràn và tỉ lệ lỗi giảm, nên dự đoán vẫn được chấm "đúng".
- Tiến tin lời giải thích, viết sai phần phân tích điểm nghẽn trong bài (đúng phần mà nhóm từng bị trừ điểm, xem §1), và chọn cách sửa đắt hơn mức cần thiết. Bài đã nộp thì không sửa được.

_Cách phát hiện:_ vì nguyên nhân nằm trong danh sách cố định, code có thể đối chiếu với số liệu của engine:

- `queue_overflow`: phần lớn lỗi phải là lỗi do hàng đợi đầy.
- `write_load`: tỉ lệ ghi đo được phải vượt ngưỡng.
- Node AI nêu có đúng là node có mức sử dụng cao nhất không.

Nếu có điểm không khớp, giao diện hiện cảnh báo **trước khi** hiển thị lời giải thích của AI.

**Đo độ tin cậy bằng cách nào:** Kết luận đúng/sai do code quyết định. Ở mức toàn hệ thống, nhóm đo trên **20 ca có đáp án** (5 kiến trúc mẫu × 4 biến thể tham số) với các chỉ số:

- tỉ lệ chỉ đúng node;
- tỉ lệ chỉ đúng nguyên nhân;
- tỉ lệ dự đoán đúng khi chạy lại;
- tỉ lệ cảnh báo "nguyên nhân không khớp" được bật.

Giới hạn: việc kiểm tra chỉ đúng **trong phạm vi mô hình mô phỏng**. Giao diện ghi rõ "kết quả mô phỏng, không phải đo trên hạ tầng thật".

## 3. Phạm vi

**Làm trong học kỳ này** (xếp theo thứ tự ưu tiên; nếu trễ thì cắt từ dưới lên):

1. **Engine Go:** traffic constant (tải đều) và burst, số liệu như ở §2, cùng seed cho cùng kết quả, chế độ chạy "xấu hơn".
2. **Tính năng AI ở §2:** chẩn đoán, chỉnh sửa, validator, diff, chạy lại, so dự đoán với số đo, đối chiếu nguyên nhân.
3. **Canvas React Flow:** 6 loại node, cấu hình thông số, lưu/tải file JSON.
4. **20 ca có đáp án và baseline rule-based:** dùng nội bộ để đánh giá AI.
5. **Biểu đồ số liệu** sau mỗi lần chạy, và so sánh trước–sau.
6. **Tài khoản:** đăng nhập; mỗi người sở hữu project, phiên bản và lịch sử chạy của mình; giới hạn 20 lượt AI mỗi ngày cho mỗi người.

**Cố ý không làm:**

- Hiển thị số liệu theo thời gian thực trong lúc đang chạy; các kiểu traffic khác ngoài constant và burst (ramp, Poisson, tự định nghĩa).
- Phân quyền nhiều vai trò, workspace dùng chung, comment/review, giao và chấm bài, nhật ký thao tác (audit log).
- Ước tính chi phí hạ tầng, phát lại lần chạy, đánh giá theo SLO, thư viện kịch bản dựng sẵn.
- AI tự gây sự cố (fault injection) hoặc tự động scale. AI chỉ phân tích kịch bản do người dùng chọn.
- Mô phỏng các cơ chế phân tán thực (consensus, replication lag, network partition); gửi traffic ra ngoài sandbox; chạy code do AI sinh ra.

## 4. Kế hoạch sáu checkpoint

**Phân công:**

- **Tiến:** Frontend/Canvas, đăng nhập, API cho project và phiên bản.
- **Huy:** Engine Go.
- **Phát:** AI, validator, đánh giá AI.

**Quy ước:**

- Ngày ghi ở mỗi checkpoint là thứ Hai của tuần học đó (tuần 1 bắt đầu 28/09/2026).
- Mỗi thứ Năm nhóm demo nội bộ.
- "Owner" là người chịu trách nhiệm chính. Nếu owner báo trễ trước thứ Năm, nhóm cắt phạm vi theo thứ tự ở §3.

### CP1 — Tuần 6 · 02/11

**Owner:** Tiến · **Hỗ trợ:** Huy (schema), Phát (20 ca có đáp án và thử prompt, xem §5)

- Chốt graph schema (cấu trúc dữ liệu của sơ đồ) vào ngày 28/10.
- Repo và Docker Compose chạy được bằng một lệnh.
- Canvas có 6 loại node, cấu hình được thông số, lưu/tải được JSON.

### CP2 — Tuần 7 · 09/11

**Owner:** Huy · **Hỗ trợ:** Phát (baseline rule-based; chạy 20 ca trên engine thật)

- Engine chạy được sơ đồ 5 node, traffic constant, có seed.
- Xuất được số liệu như ở §2.
- Test: chạy 2 lần với cùng seed phải ra cùng kết quả.

### CP3 — Tuần 8 · 16/11

**Owner:** Tiến · **Hỗ trợ:** Huy (API chạy mô phỏng)

- Đăng nhập, project CRUD, phiên bản, lịch sử chạy, giới hạn lượt AI.
- Test: người dùng B sửa project của người dùng A thì bị lỗi 403; lượt AI thứ 21 trong ngày bị lỗi 429.

### CP4 — Tuần 9 · 23/11

**Owner:** Huy · **Hỗ trợ:** Tiến (biểu đồ), Phát (validator, định dạng trả về của AI, diff)

- Engine thêm traffic burst.
- Engine ghi nhận được dấu hiệu của từng nguyên nhân: lỗi do hàng đợi đầy hay quá thời gian, tỉ lệ ghi.
- Chế độ chạy "xấu hơn".
- Đo mức dao động của kết quả khi chạy 20 seed khác nhau.
- Biểu đồ hiển thị sau mỗi lần chạy.
- Validator, tool schema (định dạng AI phải trả về) và diff.

### CP5 — Tuần 10 · 30/11

**Owner:** Phát · **Hỗ trợ:** Tiến (giao diện duyệt diff và cảnh báo), Huy (tích hợp engine)

- AI chạy trọn một vòng: chẩn đoán, chỉnh sửa, dự đoán, tạo phiên bản mới, chạy lại (thường và "xấu hơn"), so dự đoán, đối chiếu nguyên nhân.
- Script đánh giá chạy được.
- **Đóng băng tính năng:** từ đây không thêm tính năng mới.

### CP6 — Tuần 11 · 07/12

**Owner:** Phát · **Hỗ trợ:** Huy (số liệu engine), Tiến (demo, báo cáo)

- Chạy script đánh giá AI và baseline trên 20 ca.
- Demo, báo cáo, SELF_ASSESSMENT_REPORT.

Script đánh giá được viết xong ở CP5, nên tuần CP6 chỉ cần chạy và viết báo cáo. Tuần này cũng là khoảng đệm nếu CP4–CP5 bị trễ.

## 5. Hai rủi ro lớn nhất

### Rủi ro 1: Kết quả mô phỏng không lặp lại được

**Vấn đề:** Nếu engine chạy nhiều goroutine theo thời gian thực, kết quả sẽ dao động tùy máy đang bận hay rảnh. Khi đó, so sánh trước–sau vô nghĩa, và AI có thể trông như "cải thiện" chỉ nhờ may mắn.

**Cách xử lý** (Huy, xong trước 28/10): làm một bản thử của engine với các đặc điểm:

- chạy theo sự kiện, đơn luồng;
- dùng bộ sinh số ngẫu nhiên có seed;
- dùng đồng hồ ảo (thời gian do chương trình tự tính, không phụ thuộc đồng hồ máy).

Kiểm tra: cùng seed chạy 20 lần phải cho kết quả giống hệt nhau. Sau đó chạy 20 seed khác nhau để đo p95 dao động bao nhiêu.

**Phương án dự phòng:** Nếu đến 09/11 kết quả vẫn chưa ổn định, chuyển sang tính bằng công thức toán của lý thuyết hàng đợi cho từng node. Cách này luôn cho cùng kết quả. Đổi lại, nhóm mất đường cong số request chờ theo thời gian, nhưng vẫn so sánh được trước–sau.

### Rủi ro 2: AI chẩn đoán sai nguyên nhân hoặc tạo ra sơ đồ không hợp lệ

**Vấn đề:** Toàn bộ giá trị của sản phẩm nằm ở phần này.

**Cách xử lý** (Phát, xong trước 28/10, chưa cần engine):

- Dựng 20 ca có đáp án (node nghẽn và nguyên nhân), kèm số liệu tự tính tay từ tải và capacity.
- Thử prompt trên các ca này và đo tỉ lệ đúng.
- Từ CP2, chạy lại bằng số liệu thật của engine.
- AI chỉ trả về chỉnh sửa nhỏ; validator chặn sơ đồ sai trước khi hiển thị.

**Phương án dự phòng:** Nếu sau khi chỉnh prompt vẫn có dưới 16/20 ca đúng nguyên nhân, AI chỉ được chọn trong danh sách cách sửa gắn sẵn với từng nguyên nhân (cache, replica, queue, tăng queue size).

## 6. Công nghệ

| Thành phần | Lựa chọn | Lý do |
| --- | --- | --- |
| Frontend | React + TypeScript + React Flow | React Flow có sẵn canvas node/edge, nên nhóm dành thời gian cho engine và AI |
| Biểu đồ | Recharts | Chỉ cần biểu đồ đường cho p95 và số request chờ theo thời gian, kèm so sánh trước–sau |
| Backend & Engine | Go | Dùng một ngôn ngữ cho cả API và engine. Engine đơn luồng vẫn đủ nhanh để chạy hàng trăm lượt mô phỏng khi đánh giá ở CP6 |
| Đăng nhập | JWT + bcrypt, middleware Go | Chỉ cần biết ai là chủ project và đếm lượt AI của từng người, không cần dịch vụ đăng nhập bên ngoài |
| Database | PostgreSQL | Sơ đồ lưu dạng JSONB; phiên bản và lịch sử chạy cần truy vấn quan hệ |
| Triển khai | Docker Compose | Dựng frontend, API và Postgres bằng một lệnh, đúng tiêu chí CP1 |
| LLM | Claude Haiku 5.5 qua Anthropic API, dùng tool use với `strict: true` | Nguyên nhân là một giá trị trong danh sách cố định, nên `strict: true` bắt AI trả đúng định dạng để code đối chiếu. Haiku đủ rẻ để chạy hàng trăm lần. Sonnet 5.5 chỉ dùng ở CP6 để so sánh |

### Chi phí LLM

Giá của Anthropic (USD cho 1 triệu token), tỉ giá khoảng 26.000 VND/USD:

| Model | Token vào | Token ra |
| --- | --- | --- |
| Haiku 5.5 | $0,10 | $0,50 |
| Sonnet 5.5 | $2 | $10 |

Mỗi lượt gọi AI (tối đa 4 vòng tool call) ước tính dùng khoảng 12.000 token vào và 3.000 token ra, tính cả phần AI suy nghĩ. Với Haiku, mỗi lượt tốn khoảng **$0,0027 (~70 VND)**.

| Khoản | Cách tính | Chi phí |
| --- | --- | --- |
| Phát triển | 2.000 lượt Haiku | ≈ $5,4 |
| So sánh ở CP6 | 20 ca × 5 lượt × $0,054 (Sonnet) | ≈ $5,4 |
| **Tổng** | | **≈ $10,8 (~280.000 VND)** |

Ngoài ra:

- Giới hạn 20 lượt AI mỗi ngày cho mỗi người dùng.
- Số token ước tính ở trên sẽ được chốt lại ở CP1, dựa trên trường `usage` mà API trả về.
- Phần gọi LLM nằm sau một interface tên `LLMClient`, nên có thể đổi nhà cung cấp mà không phải sửa phần còn lại.

## Giải thích thuật ngữ

| Thuật ngữ | Nghĩa |
| --- | --- |
| RPS | Số request mỗi giây |
| Burst | Kiểu tải tăng đột ngột lên mức cao trong thời gian ngắn |
| Seed | Số khởi tạo cho bộ sinh số ngẫu nhiên. Cùng seed thì kết quả "ngẫu nhiên" giống hệt nhau |
| p95 | Thời gian phản hồi mà 95% request nhanh hơn. Dùng để đo độ chậm trong trường hợp xấu |
| Utilization | Mức sử dụng của một node: tải đang nhận chia cho capacity |
| Capacity | Số request mỗi giây tối đa một node xử lý được |
| Queue (hàng đợi) | Nơi request chờ khi node đang bận. Đầy thì request bị từ chối |
| SPOF | Điểm lỗi duy nhất: nếu nó hỏng thì cả hệ thống ngừng chạy |
| Validator | Đoạn code kiểm tra chỉnh sửa của AI có hợp lệ không trước khi áp dụng |
| Diff | Bản so sánh hiển thị những gì thay đổi giữa hai phiên bản sơ đồ |
| Rule-based | Cách chẩn đoán bằng các luật viết tay cố định, không dùng AI. Dùng làm mốc so sánh |
| CP | Checkpoint, mốc tiến độ |
