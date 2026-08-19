# Báo cáo Lab 19 — GraphRAG vs Flat RAG

**Học viên:** Trần Duy Hoành
**Ngày thực hiện:** 19/08/2026

## 1. Thuyết minh kỹ thuật và failure analysis

### Coreference resolution

Pipeline chỉ thay đại từ khi antecedent xuất hiện rõ trong cùng chunk; trường hợp mơ hồ được giữ nguyên và ghi `unresolved_mentions`. Với tin ngắn có nhiều công ty, “the company” có thể trỏ về chủ thể câu trước hoặc công ty trong mệnh đề phụ. Resolve sai sẽ tạo false edge gắn sự kiện cho sai công ty. Vì false edge nguy hiểm hơn missing edge, lựa chọn an toàn là giữ nguyên cụm từ mơ hồ.

### Entity resolution

Ngưỡng vector là `0.90`, sau đó bắt buộc qua lexical guard `SequenceMatcher >= 0.72`; alias phổ biến được xử lý trước bằng bảng thủ công. Guard chặn các cặp gần nghĩa nhưng khác thực thể như `Apple`/`Apple Music` hoặc hai người cùng họ. Audit ghi `MERGE_MANUAL`, `MERGE_VECTOR`, `REJECT_GUARD`. Lần chạy chỉ tạo 12 node và 6 edge nên chưa đủ candidate để kết luận threshold tối ưu; cần tuning lại trên tập mention lớn hơn.

### Super-node mitigation

Ba node degree cao nhất đều có degree 1; ví dụ `UBC`, `NSPR`, `TD Tawandang Co`. Graph quá thưa nên policy chưa kích hoạt. Thiết kế giới hạn node degree > 100 ở 50 edge mới nhất, toàn context 250 edge/14.000 ký tự. Cách này chặn context explosion nhưng có thể mất bằng chứng lịch sử; truy vấn có thời gian nên lọc time range trước khi cap.

### Benchmark 50 câu

| Metric | Flat RAG | GraphRAG | Delta Graph−Flat |
|---|---:|---:|---:|
| Comprehensiveness | 3.56 | 3.44 | -0.12 |
| Faithfulness | 3.84 | 3.92 | +0.08 |
| Multi-hop reasoning | 3.56 | 3.46 | -0.10 |
| Latency trung bình (s) | 1.980 | 1.822 | -0.158 |
| Token trung bình | 756.04 | 594.30 | -161.74 |

GraphRAG không thắng trong lần chạy này. Graph extraction chỉ có 6 edge, hầu hết câu hỏi không match được seed graph (`graph_supernode_events = 0` toàn bộ), nên nhánh hybrid chủ yếu dùng 4 vector chunks trong khi Flat RAG dùng 6. Điều đó giải thích completeness/multi-hop thấp hơn nhưng token và latency cũng thấp hơn.

Hai failure case:

1. `G5000-03` hỏi chuỗi Aeris–Ericsson và các con số ở báo cáo connectivity sau đó. GraphRAG đạt điểm tổng hợp 1 vì không có edge/event tương ứng; vector top-4 cũng thiếu đủ bằng chứng. Khắc phục: checkpoint triples theo batch, retry riêng batch lỗi và đo edge coverage theo golden entity trước evaluation.
2. `G5000-26` hỏi provider bên ngoài trong đợt mở rộng AI của Amazon và capability đi kèm. Schema không tạo đủ `USES/DEVELOPED/PARTNERED_WITH`, nên traversal không thể bù missing edge. Khắc phục: few-shot extraction và fallback top-8 vector khi graph context rỗng.

### Trade-offs và scale

Flat RAG đơn giản, index nhanh, phù hợp factoid nhưng không biểu diễn đường nối giữa tài liệu. GraphRAG có provenance và traversal cho multi-hop khi graph đủ coverage, đổi lại tốn coreference, extraction, entity resolution, Neo4j và linearization. Lần chạy này minh họa graph kém coverage còn tệ hơn Flat RAG.

Tôi từ chối entity resolution pairwise cosine `O(N²)` vì không thể scale lên khoảng 100.000 bài. Với 350 MB, bottleneck đầu tiên là LLM extraction/rate limit; Groq đã chạm 200.000 token/ngày. Giải pháp: durable queue, checkpoint, idempotent retry, rate limiter, cache theo content hash, HNSW/blocking và partition/community retrieval.

## 2. Mapping bài giảng vào code

| Khái niệm | Hàm/khối code | Quan sát |
|---|---|---|
| Conservative coreference | `resolve_coref_batch()` | Mơ hồ giữ nguyên và log |
| Schema allowlist | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Loại relation ngoài schema |
| Bulk ingestion | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND`, batch 1.000 |
| Entity resolution | `build_resolution_map()`, `UF` | ANN + lexical guard + audit |
| Super-node cap | `retrieve_graph_context()` | degree > 100 → tối đa 50 edge |
| LLM judge | `judge_answer()` | 3 metric, thang 1–5 |

## 3. Debugging và bài học

Lỗi khó nhất là JSON hợp lệ nhưng chứa phần tử `""` thay vì object. Code cũ gọi `.get()` trực tiếp, làm batch extraction thất bại. Fix là kiểm tra `list`/`dict` ở từng cấp và ghi lỗi theo batch. JSON mode không thay thế schema validation; production phải lưu raw response, checkpoint và đo extraction coverage trước benchmark.

Cell streaming còn có thể ghi đè subset sẵn có, khiến thứ tự không khớp golden. Dataset evaluation phải immutable, có checksum/version và downloader phải dùng file đích riêng hoặc xác nhận overwrite.

## 4. Action plan

Áp dụng cho hỏi đáp tài liệu kỹ thuật nội bộ: node `Service`, `Team`, `Person`, `Incident`, `Technology`; relation `OWNS`, `DEPENDS_ON`, `CAUSED_BY`, `RESOLVED_BY`, `USES`. Entity resolution dùng alias registry + HNSW + lexical/type guard. Super-node lọc theo thời gian, relation và quyền truy cập trước cap. Chỉ bật GraphRAG cho câu dependency/incident multi-hop; factoid dùng Flat RAG.

## 5. Tự đánh giá

| Tiêu chí | Điểm | Ghi chú |
|---|---:|---|
| Hiểu GraphRAG | 4/5 | Nắm retrieval, provenance và failure mode |
| Kiểm soát coding agent | 4/5 | Audit số liệu, không che coverage thấp |
| Chất lượng graph | 2/5 | Chỉ 12 node/6 edge, cần extraction lại |
| Phân tích/debug | 4/5 | Tìm malformed JSON và rate-limit bottleneck |
