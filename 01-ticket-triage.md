# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. Happy path: Enterprise customer báo "admin cannot login, all work is blocked since morning"; kỳ vọng `category = technical`, `urgency = high`, `requires_human = true`, `route_to = technical_support`. Case này bắt lỗi đánh thấp mức độ khẩn khi ticket có tín hiệu chặn công việc.
2. Ambiguous input: Subject là "Need help now", message chỉ ghi "system issue asap"; kỳ vọng không tự tin gán billing/technical cụ thể, cần `category = unknown` hoặc review. Case này bắt hallucination category và confidence quá cao.
3. Missing information: Khách ghi "invoice problem" nhưng không có mã invoice, trạng thái account hay mô tả lỗi; kỳ vọng route về support/billing intake với yêu cầu bổ sung thông tin. Case này bắt lỗi route thẳng sang team xử lý chuyên sâu khi dữ liệu chưa đủ.
4. High-risk / escalation: Enterprise customer ghi "payment failed, account disabled, entire team locked out"; kỳ vọng `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc escalation. Case này bắt lỗi giống mock outcome: đánh medium và không escalate.
5. Regression case: Ticket billing có từ "feature request" trong câu "This is not a feature request, our billing failed"; kỳ vọng không route sang product. Case này bắt lỗi keyword matching nông làm billing ticket bị đẩy sai team.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

> Tôi chọn Unit of Work là: một ticket support mới đi vào, AI đọc subject/message/customer tier rồi trả ra category, urgency, route, cờ cần người xử lý và lý do tóm tắt. Đây là lát cắt đủ nhỏ vì mỗi ticket có thể được eval độc lập bằng input, output và trace routing. Nó vẫn đủ quan trọng vì output này quyết định ticket nằm ở hàng bình thường hay ưu tiên cao, và team nào phải xử lý.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

> Câu hỏi chất lượng chính là: AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý, đặc biệt với khách enterprise và dấu hiệu chặn công việc không? Behavior bắt buộc là phải nâng mức khẩn/cần human khi có locked out, account disabled hoặc blocking work; behavior bị cấm là route billing sang product hoặc tự bịa reason không có trong ticket. Nếu fail, khách có thể bị chậm xử lý, support ops mất trust vào inbox, và các ticket nghiêm trọng bị lẫn vào queue thường.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

> Các field tối thiểu tôi giữ lại gồm:
>
> - `ticket_id`: cần để nối output eval với ticket gốc và trace UI.
> - `category`: enum như `technical`, `billing`, `product_question`, `account_access`, `unknown`; field này quyết định label UI và một phần route.
> - `urgency`: enum `low`, `medium`, `high`, `critical`; field này quyết định hàng đợi bình thường hay ưu tiên.
> - `route_to`: enum như `support_l1`, `technical_support`, `billing_ops`, `product_team`, `human_escalation`; đây là quyết định vận hành quan trọng nhất.
> - `requires_human`: boolean để trigger escalation hoặc bắt nhân viên nhận ticket ngay.
> - `confidence`: số `0-1` để biết case nào cần review khi AI không chắc.
> - `reason_codes`: danh sách mã lý do như `enterprise_customer`, `blocked_work`, `account_disabled`, `payment_failed`; dùng để eval xem AI có dựa trên evidence thật không.
> - `summary_reason`: câu ngắn render lên UI, giúp nhân viên hiểu vì sao AI gợi ý route/urgency đó.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema và allowed enums | Có | Không | Không | Không | Field bắt buộc, kiểu dữ liệu, enum và `confidence` trong `0-1` đều deterministic, code chấm nhanh và ổn định. |
| Business hard rules | Có | Không | Không | Không | Rule như enterprise + high/critical => `requires_human = true`, billing không route product là logic rõ ràng nên giao cho code. |
| Nhận diện category/urgency theo ngữ nghĩa | Không | Có | Có | Không | Ticket có thể dùng nhiều cách diễn đạt như "locked out" hoặc "blocked work"; LLM judge chấm scale, human audit dùng để calibrate. |
| Độ đúng của route/escalation | Có | Có | Có | Không | Một phần là mapping enum deterministic, nhưng phần chọn route đúng cần hiểu ngữ cảnh và hậu quả vận hành. |
| Reason codes và summary_reason có bám evidence | Không | Có | Có | Không | Code khó biết lý do có bị bịa hay không; LLM/human cần đọc input để kiểm tra grounding. |
| Low-confidence/ambiguous handling | Có | Có | Có | Không | Code bắt threshold/review flag, LLM/human đánh giá AI có biết dừng khi thiếu dữ liệu không. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: Output parse được thành object hợp lệ, có đủ `ticket_id`, `category`, `urgency`, `route_to`, `requires_human`, `confidence`, `reason_codes`, `summary_reason`.
  Vì sao nên giao cho code: Đây là kiểm tra schema và required fields, không cần judgment ngữ nghĩa.

- Kiểm tra: `category`, `urgency`, `route_to` chỉ nhận giá trị trong allowed enums.
  Vì sao nên giao cho code: Enum sai sẽ làm UI/routing vỡ ngay, code bắt chính xác hơn LLM.

- Kiểm tra: `confidence` là số trong khoảng `0-1`.
  Vì sao nên giao cho code: Đây là ràng buộc numeric deterministic.

- Kiểm tra: Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical` thì `requires_human` phải bằng `true`.
  Vì sao nên giao cho code: Rule vận hành đã rõ, không cần diễn giải.

- Kiểm tra: Nếu `category = billing` thì `route_to` không được là `product_team`.
  Vì sao nên giao cho code: Mapping cấm rõ ràng, code bắt được mọi regression.

- Kiểm tra: Nếu subject/message chứa tín hiệu trực tiếp như `locked out`, `account disabled`, `blocking my work` thì `urgency` không được là `low`.
  Vì sao nên giao cho code: Đây là hard keyword guardrail để không bỏ sót tín hiệu high-risk đã biết.

- Kiểm tra: Nếu `requires_human = true` thì `route_to` phải là queue có người xử lý được như `technical_support`, `billing_ops`, hoặc `human_escalation`, không được là queue chỉ để phân loại.
  Vì sao nên giao cho code: Quan hệ giữa flag escalation và route có thể kiểm tra bằng mapping.

- Kiểm tra: `reason_codes` không được rỗng với case high/critical và không được chứa code ngoài danh sách cho phép.
  Vì sao nên giao cho code: Code kiểm tra được độ đầy đủ tối thiểu và enum của reason.

- Kiểm tra: Với low-info ticket như "Help", nếu `confidence > 0.8` và category không phải `unknown` thì đưa vào fail hoặc review.
  Vì sao nên giao cho code: Threshold cho ambiguity có thể biến thành assertion để bắt overconfidence.

- Kiểm tra: Không có field hành động tự động gửi khách hàng hoặc tự đóng ticket trong output.
  Vì sao nên giao cho code: Bài toán chỉ gợi ý triage nội bộ, action vượt scope là vi phạm contract rõ ràng.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: Category có phản ánh đúng vấn đề chính của ticket không, nhất là khi có nhiều tín hiệu như billing + account access.
  Vì sao code không bắt tốt: Người dùng có thể diễn đạt bằng nhiều ngôn ngữ và không dùng đúng keyword.

- Tiêu chí: Urgency có tương xứng với hậu quả vận hành được mô tả không.
  Vì sao code không bắt tốt: "Gấp" không phải lúc nào cũng critical, còn "không vào được tài khoản admin" có thể rất nghiêm trọng dù không viết chữ urgent.

- Tiêu chí: Route có đưa ticket tới team có khả năng xử lý nguyên nhân chính không.
  Vì sao code không bắt tốt: Route đúng phụ thuộc vào hiểu nội dung ticket, tier khách hàng và loại lỗi.

- Tiêu chí: Summary_reason có ngắn, đúng, bám evidence và không thêm sự thật ngoài input không.
  Vì sao code không bắt tốt: Cần đọc hiểu câu tóm tắt để biết nó có grounded hay hallucinated.

- Tiêu chí: AI có biết giảm tự tin hoặc đề xuất review khi input mơ hồ/thiếu thông tin không.
  Vì sao code không bắt tốt: Ambiguity là hiện tượng semantic, không chỉ là độ dài input.

- Tiêu chí: AI có phát hiện đúng tín hiệu escalation dù khách viết bằng tiếng Việt, tiếng Anh, hoặc tiếng Việt không dấu không.
  Vì sao code không bắt tốt: Cần hiểu paraphrase và ngữ cảnh, không thể chỉ dựa vào một danh sách keyword cố định.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> Người review chính nên là support ops lead hoặc senior support agent, vì họ hiểu queue, SLA và hậu quả khi route sai. Họ cần review các case high/critical, case `confidence < 0.7`, case category/route bị LLM judge chấm fail, và các ticket low-info nhưng AI vẫn tự tin. Tôi không chọn domain expert chuyên sâu cho case này vì rủi ro là vận hành support và chính sách routing nội bộ; human review từ team support là đủ để xác nhận ticket có bị đi sai hàng hoặc thiếu escalation hay không.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

```text
Không áp dụng.

Case này không cần domain expert chuyên sâu vì không có quyết định pháp lý/y tế/tài chính cần chuyên môn ngoài support ops; senior support/human reviewer đã đủ thẩm quyền duyệt route và escalation.
```

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

Không áp dụng vì case này không dùng domain expert.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

Release gate đề xuất:

- Block release nếu schema pass rate dưới `99.5%`, vì output vỡ schema sẽ làm UI/routing không đáng tin.
- Block release nếu có bất kỳ P0 nào: enterprise + high/critical nhưng `requires_human = false`, ticket có `account disabled/locked out/blocking work` bị đánh `low`, hoặc billing ticket route sang `product_team`.
- Block release nếu escalation recall trên tập high-risk dưới `95%`.
- Block release nếu route accuracy tổng dưới `90%` hoặc category accuracy dưới `88%` trên 75 case pilot.
- Warn nếu LLM judge và human reviewer bất đồng trên `>10%` case, vì rubric chưa ổn định.
- Bắt human review trước khi ship cho tất cả case high/critical, case confidence dưới `0.7`, và case LLM judge đánh fail phần route/escalation.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

Tôi đề xuất pilot với 75 cases, gồm khoảng 50 case chuẩn từ seed/edge coverage và 25 case lấy từ ticket thật đã ẩn thông tin nhạy cảm. Chạy 35 lần/lặp lại để thử prompt/model/rubric, tổng là `75 * 35 = 2,625` lượt đánh giá. Giả định mỗi lượt dùng khoảng 2,000 input tokens và 500 output tokens cho model/judge, tổng token khoảng 5.25M input và 1.31M output.

Giá API dùng để tính là OpenAI `gpt-5.4-mini` Standard short-context: $0.375 / 1M input tokens và $2.25 / 1M output tokens, tham chiếu từ https://platform.openai.com/docs/pricing. Chi phí API ước tính là `5.25 * 0.375 + 1.31 * 2.25 = khoảng $4.92`; tôi xin buffer thành dưới $8 để tính retry và logging.

Giờ công giả định: PM/eval design 8 giờ * 300,000 VND = 2,400,000 VND; kỹ thuật/ops setup 8 giờ * 300,000 VND = 2,400,000 VND; human review 10 giờ * 200,000 VND = 2,000,000 VND. Không có domain expert. Tổng chi phí người là khoảng 6,800,000 VND; API dưới 200,000 VND theo tỷ giá nhẩm 25,000 VND/USD, nên tổng pilot khoảng 7,000,000 VND trong 1 tuần làm việc.

Với quy mô này, team chứng minh được triage có giữ đúng schema, có bắt được escalation enterprise/high-risk, và có đủ chính xác để xin bước pilot production nhỏ hay chưa. Đây chưa phải release production đầy đủ, nhưng đủ để ra quyết định có tiếp tục đầu tư eval runner và monitoring không.

---
