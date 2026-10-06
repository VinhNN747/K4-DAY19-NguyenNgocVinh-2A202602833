# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Ngọc Vinh
**MSSV:** 2A202602833
**Ngày chạy benchmark:** 06/10/2026

Thiết lập: `openai:gpt-4o-mini`, embedding `openai:text-embedding-3-small`, `top_k=3`, `chunk_size=800`; 176 chunks; knowledge graph đầy đủ có 201 node và 376 quan hệ. Số liệu dưới đây được chép từ `ket_qua_benchmark_kg.txt` sinh bởi `python bench_kg.py --judge`.

## 1. Chi phí

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     40.0
graph       196     91958     4693   0.00932    115.7

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.33
graph       0.89   1.83     4222       74   0.00067     2.35
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | $0.00112 | $0.00932 | 8.32× |
| Indexing giây | 40.0 | 115.7 | 2.89× |
| Mỗi câu: USD | $0.00013 | $0.00067 | 5.15× |
| Mỗi câu: giây | 1.33 s | 2.35 s | 1.77× |
| Mỗi câu: in_tok | 694 | 4,222 | 6.08× |

Chi phí indexing của GraphRAG tăng chủ yếu vì 20 lần gọi LLM để trích xuất case từ tin tức (và output token JSON), trong khi Flat RAG chỉ embed chunks. Ở thời điểm trả lời, GraphRAG gửi thêm các fact từ Cypher và các khoản luật nên input token tăng mạnh; đổi lại nó nối được tin và luật cho các câu multi-hop.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai lấy được định nghĩa; GraphRAG còn nêu Điều 2 khoản 4. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Đáp án nằm trong một bài báo, graph không tạo lợi thế đáng kể. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Đi từ Lê Minh Thành qua `Crime` tới Điều 251 và khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Truy vấn “tối đa” mở rộng tới khoản 4 Điều 255. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | `Case → Substance(MDMA) → Clause` chọn được khoản 4 Điều 250. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph (một phần) | Graph tổng hợp được hai vụ có tên Thành/Đông nhưng còn bỏ sót Cái Quang Huy ở câu trả lời. |

Quy luật quan sát được: Flat RAG đủ cho dữ kiện nằm trong một đoạn/bài (Q1, Q2), còn GraphRAG rõ ràng tốt hơn khi câu hỏi yêu cầu bắc cầu tin tức–luật hoặc lọc theo nhiều quan hệ (Q3–Q5). Câu tổng hợp Q6 cho thấy graph chỉ hữu ích khi phần trích xuất và prompt trả lời vẫn giữ đủ các node tìm được.

## 3. Phân tích lỗi

### Lỗi E4: Phép đo keyword recall đánh giá thiếu câu trả lời hữu ích

- **Hiện tượng:** Q6 Flat có `recall=0.00` nhưng LLM-as-judge vẫn chấm `1/2`.
- **Bằng chứng:** output benchmark Q6 Flat nêu ba vụ “của Đức”, “của Thành”, “của Đông” cùng lượng/chất MDMA. Tuy nhiên bộ `must_include` đòi chuỗi đầy đủ `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`; do các tên này không xuất hiện nguyên văn nên keyword recall là 0.

```
Q6 flat recall=0.00, judge=1
```

- **Nguyên nhân:** `keyword_recall()` chỉ kiểm substring bắt buộc, không nhận đồng tham chiếu (“Thành”), mô tả tương đương (“vụ của Đức”), hay câu trả lời đúng một phần. Judge linh hoạt hơn nhưng vẫn chỉ là LLM-as-judge.
- **Đề xuất sửa:** bổ sung alias/chấp nhận nhóm từ khoá cho từng ý hoặc chấm theo fact có cấu trúc (case, substance, amount). Đổi lại bộ gold và evaluator sẽ tốn công chuẩn bị hơn; không nên chỉ tăng số keyword vì sẽ làm phép đo dễ bị “nhồi từ”.

### Lỗi E5: LLM bỏ sót fact có sẵn khi tổng hợp

- **Hiện tượng:** Q6 GraphRAG chỉ trả lời hai vụ (Lê Minh Thành và Lê Văn Đông), trong khi gold yêu cầu thêm vụ Cái Quang Huy vận chuyển hơn 9,6kg MDMA.
- **Bằng chứng:** GraphRAG Q5 vừa trả lời đúng “Cái Quang Huy … hơn 9,6kg MDMA … khoản 4 Điều 250”, chứng tỏ graph có entity/quan hệ này. Nhưng câu trả lời GraphRAG Q6 chỉ liệt kê:

```
1. Vụ góp tiền mua ma túy tại Hà Nội — Lê Minh Thành … 5 viên MDMA.
2. Vụ tổ chức sử dụng ma túy tại Sầm Sơn — Lê Văn Đông … 0,686g MDMA.
```

Truy vấn kiểm chứng trực tiếp cho aggregation là:

```cypher
MATCH (k:Case)-[:INVOLVES]->(s:Substance {name:'MDMA'})
OPTIONAL MATCH (p:Person)-[:INVOLVED_IN]->(k)
RETURN k.name, k.summary, collect(DISTINCT p.name) AS people;
```

Nó phải bao gồm cả case có Cái Quang Huy, case Lê Minh Thành và case liên quan Viện Pháp y tâm thần/Sầm Sơn. Vì vậy lỗi nằm ở giai đoạn chọn/trình bày fact cho prompt trả lời, không phải ở đường nối luật của Q5.

- **Nguyên nhân:** `context()` giới hạn fact và còn kèm nhiều khoản luật MDMA. Với câu aggregation, các khoản luật là nhiễu và mô hình có thể chọn một tập con các case trong prompt dài.
- **Đề xuất sửa:** nhận diện intent aggregation (`“những vụ việc nào”`) và trả về riêng một dòng `Case` ngắn cho mọi case khớp chất, không mở rộng sang `Article/Clause`; đồng thời yêu cầu prompt “liệt kê đủ tất cả fact case, không được bỏ dòng nào”. Đánh đổi là phải thêm phân loại intent/nhánh truy vấn và số case có thể làm prompt dài khi dữ liệu lớn.

## 4. Kết luận

Knowledge Graph đáng tiền khi đáp án phải nối dữ kiện phân tán và có đường đi ổn định: Q3–Q5 tăng từ judge `0/0/1` của Flat lên `2/2/2` với GraphRAG. Giá phải trả là indexing đắt hơn 8.32×, query đắt hơn 5.15× và chậm hơn 1.77×. Với định nghĩa luật hoặc một bài tin độc lập (Q1–Q2), Flat RAG đã đạt judge 2 nên là lựa chọn đơn giản, rẻ hơn. Với các câu tổng hợp như Q6, chỉ nên dùng graph khi có truy vấn aggregation và prompt chống bỏ sót fact.

## 5. Tự kiểm

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.19s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 14 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
