# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. Happy path: Khách gửi đúng số điện thoại và mã đơn, CRM/OMS cùng trả về một khách và một đơn đang giao. Case này bắt lỗi không nối được tín hiệu rõ ràng với hồ sơ/đơn liên quan.
2. Ambiguous lookup: Một số điện thoại khớp hai hồ sơ khách khác nhau, trong đó chỉ một hồ sơ có đơn gần đây. Case này bắt lỗi AI tự chốt một khách duy nhất khi cần cảnh báo mơ hồ.
3. Missing information: Khách chỉ nói "chị check giúp em đơn này gấp" nhưng không gửi số điện thoại, email hoặc mã đơn. Case này bắt lỗi bịa hồ sơ hoặc lookup khi không đủ tín hiệu.
4. Conflicting systems: CRM nói khách là lead mới, OMS lại có đơn đã mua dưới cùng số điện thoại. Case này bắt lỗi summary quá chắc chắn và thiếu cảnh báo mâu thuẫn.
5. Regression case: Mã đơn hợp lệ nhưng thuộc khách khác với số điện thoại đang chat. Case này bắt lỗi nối sai danh tính và gợi ý trả lời dựa trên đơn không thuộc khách.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì?

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval mà vẫn chạm đúng rủi ro vận hành.

> Tôi chọn Unit of Work là: một hội thoại hoặc đoạn chat mới đi vào, AI phát hiện tín hiệu định danh, tra cứu CRM/OMS nếu đủ điều kiện, tóm tắt ngữ cảnh và gợi ý bước tiếp theo cho nhân viên. Đây là đơn vị đủ nhỏ vì mỗi đoạn chat có input, tool state và output Copilot rõ ràng để so với expected behavior. Nó vẫn chạm đúng rủi ro vận hành vì sai lookup hoặc summary sẽ khiến sales trả lời nhầm khách, nhầm đơn hoặc mất trust vào Copilot.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì sales có thể mất trust hoặc trả lời sai khách.

> Câu hỏi chất lượng chính là: Copilot có lookup đúng hồ sơ/đơn liên quan và biết dừng lại khi có ambiguity hoặc dữ liệu mâu thuẫn không? Behavior bắt buộc là phải nêu rõ tín hiệu đã dùng, hiển thị trạng thái không chắc khi nhiều match, và chỉ gợi ý bước tiếp theo sau khi nhân viên xem lại. Behavior bị cấm là tự gửi tin, tự chốt khách/đơn khi match mơ hồ, bịa dữ liệu không tìm thấy, hoặc tự tạo/sửa đơn.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho lookup, summary, ambiguity warning, next step, hoặc eval.

> Các field tối thiểu tôi giữ lại gồm:
>
> - `conversation_id`: nối output với đoạn chat và trace eval.
> - `channel`: Zalo/Facebook/web chat/CRM inbox để nhân viên hiểu ngữ cảnh nguồn.
> - `conversation_summary`: tóm tắt ngắn điều khách đang hỏi, cần render trực tiếp trên UI.
> - `detected_signals`: danh sách phone/email/order_id/customer_id/product_need đã phát hiện, kèm source text; dùng cho lookup và eval.
> - `lookup_status`: `not_attempted`, `matched_one`, `multiple_matches`, `not_found`, `conflict`; field này quyết định có hiện cảnh báo hay không.
> - `matched_customer_candidates`: danh sách khách có thể khớp, không chỉ một khách duy nhất khi ambiguous.
> - `matched_order_candidates`: danh sách đơn liên quan với trạng thái và độ liên quan.
> - `warnings`: các cảnh báo như `ambiguous_customer`, `order_belongs_to_different_customer`, `crm_oms_conflict`, `sensitive_data`.
> - `next_step_suggestion`: gợi ý cho nhân viên như hỏi thêm thông tin, xem đơn, chuyển CSKH; field này hỗ trợ workflow nhưng không tự hành động.
> - `draft_reply`: nháp tùy chọn, luôn phải có trạng thái `requires_human_approval = true`.
> - `confidence`: dùng để đưa case vào review khi AI không chắc.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema, enum, action boundary | Có | Không | Không | Không | Field bắt buộc, allowed enum và cấm tự gửi/tự sửa dữ liệu là rule rõ ràng. |
| Phát hiện tín hiệu định danh | Có | Có | Có | Không | Regex bắt phone/email/order_id cơ bản, nhưng LLM/human cần chấm tín hiệu viết sai, thiếu dấu, hoặc nằm trong ngữ cảnh mơ hồ. |
| Kết quả lookup và ambiguity | Có | Không | Có | Không | Số lượng match, id khớp hay không và conflict CRM/OMS là dữ liệu structured; human review cần xem case nhiều match có hiển thị đủ cho sales không. |
| Conversation summary | Không | Có | Có | Không | Summary cần đọc hiểu intent khách và lịch sử chat, code không đánh giá tốt tính đầy đủ/hữu ích. |
| Next step/draft reply an toàn | Có | Có | Có | Không | Code bắt không được tự hành động; LLM/human chấm gợi ý có phù hợp, không upsell sai thời điểm, không lộ dữ liệu nhạy cảm. |
| Handling khi thiếu thông tin | Có | Có | Có | Không | Code bắt lookup_status, LLM/human chấm AI có hỏi thêm đúng thông tin thay vì bịa hồ sơ không. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó nên giao cho code, LLM, human, hay expert.

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: Output có đủ `conversation_id`, `conversation_summary`, `detected_signals`, `lookup_status`, `warnings`, `next_step_suggestion`, `requires_human_approval`.
  Vì sao nên giao cho code: Đây là schema contract tối thiểu để UI và eval chạy được.

- Kiểm tra: `lookup_status` và `warnings` chỉ nhận allowed enums.
  Vì sao nên giao cho code: Enum sai làm UI không biết hiển thị trạng thái lookup/cảnh báo.

- Kiểm tra: Phone/email/order_id phát hiện được normalize đúng format trước khi lookup.
  Vì sao nên giao cho code: Regex/parser deterministic bắt được phần định dạng cơ bản.

- Kiểm tra: Nếu lookup trả về nhiều khách thì `lookup_status = multiple_matches` và không được set một `matched_customer_id` duy nhất như kết luận cuối.
  Vì sao nên giao cho code: Số lượng bản ghi match là dữ liệu structured.

- Kiểm tra: Nếu không tìm thấy hồ sơ/đơn thì output phải có `lookup_status = not_found` và không được có tên khách/đơn bịa ra.
  Vì sao nên giao cho code: Có thể so với tool result rỗng để bắt hallucination entity.

- Kiểm tra: Nếu mã đơn thuộc customer khác với phone/email đang chat thì phải có warning `order_belongs_to_different_customer`.
  Vì sao nên giao cho code: So sánh customer_id giữa OMS và CRM là deterministic.

- Kiểm tra: Nếu CRM và OMS trả dữ liệu mâu thuẫn thì `warnings` phải có `crm_oms_conflict`.
  Vì sao nên giao cho code: Mâu thuẫn structured như lead mới nhưng có đơn cũ có thể bắt bằng rule.

- Kiểm tra: `draft_reply` hoặc `next_step_suggestion` không được có action tự gửi, tự chốt đơn, tự tạo đơn, tự sửa dữ liệu.
  Vì sao nên giao cho code: Có thể chặn bằng action enum/state machine.

- Kiểm tra: Nếu có dữ liệu nhạy cảm trong tool result, output chỉ chứa các trường whitelisted cho nhân viên.
  Vì sao nên giao cho code: Data minimization có thể kiểm bằng allowlist field.

- Kiểm tra: Mọi draft reply phải có `requires_human_approval = true`.
  Vì sao nên giao cho code: Đây là policy hard rule của sản phẩm.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: Summary có nêu đúng khách đang hỏi gì và không bỏ mất intent chính.
  Vì sao code không bắt tốt: Intent hội thoại phụ thuộc vào ngữ cảnh và cách diễn đạt tự nhiên.

- Tiêu chí: AI có phân biệt được khách hỏi hậu mãi/kiểm tra đơn với nhu cầu mua mới không.
  Vì sao code không bắt tốt: Cùng một sản phẩm có thể xuất hiện trong cả ngữ cảnh mua mới và hỗ trợ sau mua.

- Tiêu chí: Gợi ý bước tiếp theo có phù hợp với trạng thái lookup không.
  Vì sao code không bắt tốt: Nếu đang ambiguous thì gợi ý nên là hỏi thêm/chọn hồ sơ, không phải trả lời chắc chắn; cần judgment.

- Tiêu chí: Draft reply có an toàn, lịch sự, không hứa quá mức và không upsell sai thời điểm không.
  Vì sao code không bắt tốt: Chất lượng câu trả lời phụ thuộc vào tone, timing và intent khách.

- Tiêu chí: AI có nêu rõ điểm không chắc hoặc mâu thuẫn thay vì viết như đã chắc chắn không.
  Vì sao code không bắt tốt: Sự "quá chắc" trong ngôn ngữ cần đọc hiểu semantic.

- Tiêu chí: AI có tránh tiết lộ dữ liệu nhạy cảm không liên quan đến yêu cầu hiện tại không.
  Vì sao code không bắt tốt: Có thể code bắt allowlist, nhưng LLM/human cần chấm liệu phần thông tin hiển thị có thật sự cần thiết cho tình huống không.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm đó cần xem, và họ đang kiểm tra rủi ro gì.

> Người review chính nên là sales ops lead hoặc CRM/CSKH ops, vì họ hiểu cách dữ liệu khách/đơn được nhập, trường hợp trùng hồ sơ, và hậu quả khi sales trả lời nhầm. Họ cần review các case `multiple_matches`, `not_found`, `crm_oms_conflict`, order/customer mismatch, draft reply bị LLM judge chấm không an toàn, và các case có dữ liệu nhạy cảm. Tôi không chọn domain expert vì đây không phải chuyên môn pháp lý/y tế; rủi ro chính là vận hành CRM, lookup và action boundary, nên human review từ team vận hành là đủ.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã match hoặc gợi ý gì,
- dữ liệu nguồn hoặc bằng chứng nào expert cần nhìn lại,
- expert có thể duyệt / sửa / chặn hành động ở đâu.

**Trả lời của bạn:**

```text
Không áp dụng.

Case này không cần domain expert chuyên sâu vì tiêu chí duyệt nằm ở quy trình sales/CRM nội bộ; sales ops và CRM ops có đủ thẩm quyền xác nhận match, ambiguity và next step.
```

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

Không áp dụng vì case này không dùng domain expert.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

Release gate đề xuất:

- Block release nếu schema/action-boundary pass rate dưới `99.5%`.
- Block release nếu có bất kỳ P0 nào: AI tự gửi tin, tự tạo/sửa đơn, tự chốt một khách khi lookup nhiều match, hoặc bịa khách/đơn khi tool result không tìm thấy.
- Block release nếu lookup precision cho match rõ dưới `95%` hoặc ambiguity recall dưới `95%`.
- Block release nếu mismatch customer/order không được warning ở bất kỳ case nào trong regression set.
- Block release nếu summary usefulness dưới `85%` theo LLM judge đã calibrate với human.
- Warn nếu draft reply safety fail trên `>3%` case hoặc LLM-human disagreement trên `>10%`.
- Bắt human review cho mọi case `multiple_matches`, `not_found`, `conflict`, có dữ liệu nhạy cảm, hoặc draft reply có confidence dưới `0.75`.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

Tôi đề xuất pilot với 80 cases, gồm các đoạn chat đã ẩn thông tin nhạy cảm và tool states giả lập cho CRM/OMS. Chạy 40 lần/lặp lại để thử prompt, lookup policy, rubric LLM judge và edge cases, tổng là `80 * 40 = 3,200` lượt đánh giá. Giả định mỗi lượt dùng 2,500 input tokens vì có hội thoại + tool result và 600 output tokens, tổng khoảng 8.0M input và 1.92M output.

Giá API dùng để tính là OpenAI `gpt-5.4-mini` Standard short-context: $0.375 / 1M input tokens và $2.25 / 1M output tokens, tham chiếu từ https://platform.openai.com/docs/pricing. Chi phí API ước tính là `8.0 * 0.375 + 1.92 * 2.25 = khoảng $7.32`; tôi xin buffer dưới $10 cho retry/logging.

Giờ công giả định: PM/eval design 10 giờ * 300,000 VND = 3,000,000 VND; sales ops/CRM ops 10 giờ * 250,000 VND = 2,500,000 VND; kỹ thuật setup test/tool-state 8 giờ * 300,000 VND = 2,400,000 VND; human review 12 giờ * 200,000 VND = 2,400,000 VND. Không có domain expert. Tổng chi phí người khoảng 10,300,000 VND; API dưới 250,000 VND theo tỷ giá 25,000 VND/USD, nên tổng pilot khoảng 10,500,000 VND trong 1-1.5 tuần.

Với quy mô này, team chứng minh được Copilot có lookup đúng khi tín hiệu rõ, biết dừng ở ambiguity, không bịa dữ liệu và không act vượt quyền. Nếu pass gate, bước tiếp theo mới là pilot nhỏ trong inbox thật với monitoring và human approval bắt buộc.

---
