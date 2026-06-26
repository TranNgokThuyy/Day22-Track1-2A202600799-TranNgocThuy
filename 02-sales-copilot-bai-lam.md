# Bài làm Case 2 - Sales Chat Copilot

## Bài làm

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

