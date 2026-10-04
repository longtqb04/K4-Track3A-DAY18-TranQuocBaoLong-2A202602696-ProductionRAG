# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Trần Quốc Bảo Long  
**Khóa:** K4 - Track 3A  
**Ngày hoàn thành:** 04/10/2026

---

## Phần 1: Mapping bài giảng (Lecture Mapping)
Map từng concept trong lecture vào code bạn vừa viết trong lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| Semantic chunking | M1 | `chunk_semantic()` | Chia văn bản thành câu, mã hóa bằng `all-MiniLM-L6-v2`, rồi so sánh cosine similarity giữa các câu liền kề. Với threshold 0.85, các câu có độ tương đồng thấp hơn ngưỡng sẽ bắt đầu chunk mới. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Tạo parent tối đa 2048 ký tự và child tối đa 256 ký tự; mỗi child giữ `parent_id` để nối về ngữ cảnh cha. Tìm kiếm child có thể tăng độ chính xác, còn parent cung cấp ngữ cảnh rộng hơn để trả lời. |
| BM25 + Dense fusion | M2 | `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | BM25 tìm khớp từ khóa sau khi tách từ tiếng Việt; Dense Search biểu diễn nội dung bằng embedding BGE-M3 để tìm ý nghĩa tương tự. RRF hợp nhất theo thứ hạng thay vì cộng trực tiếp hai loại điểm vốn khác thang đo. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Cross-encoder chấm từng cặp (query, document), nên có thể xếp lại candidate theo mức liên quan chi tiết hơn bước retrieval. Cấu hình pipeline lấy top 20 từ các retriever rồi giữ top 3 để tạo context. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Bốn metric quan sát các khía cạnh khác nhau: faithfulness (câu trả lời bám context), answer relevancy (đúng trọng tâm câu hỏi), context precision (context lấy được có liên quan), context recall (context có đủ thông tin ground truth). |
| Contextual enrichment | M5 | `_enrich_single_call()`, `enrich_chunks()` | Chế độ combined yêu cầu LLM trả summary, câu hỏi giả thuyết, câu mô tả vị trí chunk và metadata trong một lần gọi. Contextual prepend giúp đoạn được index có thêm thông tin tài liệu/chủ đề, qua đó tăng khả năng retrieval khi câu hỏi thiếu từ khóa trùng khớp. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

- **Lỗi kỹ thuật gặp phải (Exact error message):**
  - M4/RAGAS: `No module named 'langchain_community.chat_models.vertexai'`; trong lần chạy trước đó, hàm đánh giá bắt lỗi nhưng trả sai kiểu dữ liệu, dẫn tới `TypeError: argument of type 'builtin_function_or_method' is not iterable`.
  - M5/enrichment: `Expecting value: line 1 column 1 (char 0)`, sau đó `AttributeError: 'NoneType' object has no attribute 'get'` tại `result.get("summary", "")`.
  - M2/dense embedding: `ValueError: Unrecognized processing class in BAAI/bge-m3. Can't instantiate a processor, a tokenizer, an image processor, a video processor or a feature extractor for this model.`
- **Nguyên nhân gốc rễ & Cách debug:**
  - Ở M4, evaluator phụ thuộc các tích hợp LLM và thư viện bên ngoài. Thiếu module khiến RAGAS không thể tính metric. Giá trị 0 do fallback khi đó biểu thị lần đánh giá không chạy thành công, không nên diễn giải như kết quả benchmark.
  - Ở M5, API phản hồi nội dung rỗng hoặc không phải JSON theo schema mong đợi. `json.loads()` thất bại; hàm `_enrich_single_call()` không trả object hợp lệ ở nhánh lỗi, nhưng `enrich_chunks()` vẫn giả định kết quả luôn là dict và gọi `.get()`. Debug theo traceback đã chỉ ra chuỗi lỗi từ API response → JSON parse → giá trị `None` → pipeline dừng. Cách sửa là luôn trả dữ liệu fallback có cấu trúc và kiểm tra kiểu/field trước khi dùng.
  - Ở M2, lỗi xuất hiện lúc Sentence Transformers nạp tokenizer/processor của BGE-M3, sau khi weights đã tải. `requirements.txt` trước đó chỉ đặt phiên bản tối thiểu nên có thể cài tổ hợp thư viện mới hơn dự kiến; cache model thiếu tokenizer cũng là khả năng cần kiểm tra. Mình đã giới hạn tương thích `sentence-transformers<5` và `transformers<5`, đồng thời thêm thông báo hướng dẫn nếu model vẫn không load được. Cần cài lại dependencies và chạy lại indexing để xác nhận.
  - Các test hiện có không tái hiện đầy đủ những lỗi production này: test M5 dùng `methods=["contextual"]`, không đi qua nhánh `combined` mà pipeline gọi mặc định; test M2 chủ yếu kiểm tra BM25 và RRF, không tải embedding BGE-M3; test M4 kiểm tra kiểu output chứ không xác nhận evaluator backend thật đã hoạt động. Vì vậy test pass chưa đủ chứng minh toàn pipeline chạy được với API và model thật.
- **Kiến thức còn thiếu & Cách khắc phục:**
  - Cần hiểu rõ ranh giới lỗi giữa retrieval, enrichment và evaluation; schema hợp đồng giữa các module; cách quản lý phiên bản phụ thuộc/cache model; và ý nghĩa của từng RAGAS metric. Một lỗi hạ tầng ở evaluator hoặc model loader phải được phân biệt với chất lượng câu trả lời.
  - Bổ sung kiểm tra tích hợp dùng API mock trả về JSON hợp lệ, JSON sai định dạng, nội dung rỗng và exception; kiểm tra luồng `combined`; kiểm tra kích thước vector bằng embedding thật khi môi trường sẵn sàng. Ghi rõ trạng thái đánh giá thành công/thất bại để không trình bày fallback 0 như điểm thật.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

Dựa trên những kỹ thuật đã học và thực hành, lập kế hoạch cụ thể áp dụng vào project của bạn:

### Project: SafeRoute AI

#### 1. Hiện trạng
- **Pipeline hiện tại:** SafeRoute AI hiện **chưa có pipeline RAG hoạt động**. Trong source code chưa có bước nạp/chia tài liệu, tạo embeddings, vector store hoặc truy xuất chunks; Chroma chỉ xuất hiện ở cấu hình mẫu/tài liệu hướng dẫn và dependency còn tắt trong `requirements.txt`.
- **Vấn đề / Bottlenecks đang gặp:** Chưa có retrieval nên chưa đo được retrieval precision/recall hay chất lượng trả lời dựa trên tài liệu. Agent giải thích tuyến có lượt GPT thật khoảng 2 giây, vượt ngân sách timeout 1,5 giây; prompt hiện tại được rút gọn, nhưng phần này thuộc latency của LLM chứ không phải latency của RAG.

#### 2. Kế hoạch cải tiến
1. **Chunking strategy:** Chọn structure-aware chunking theo cấu trúc PDF: giữ nguyên tiêu đề/chương/mục, đoạn văn, danh sách bước, bảng và chú thích liên quan. Bảng quy trình hoặc ngưỡng an toàn được giữ thành một đơn vị hoàn chỉnh; không cắt giữa điều kiện và hành động tương ứng.
2. **Search retrieval:** So sánh ba cấu hình trên cùng test set: BM25 thuần, dense thuần và hybrid BM25+dense. Hybrid là cấu hình ứng viên vì câu hỏi có cả thuật ngữ chính xác/mã như `STAIR_E`, `60°C`, tên phòng và câu hỏi diễn đạt tự nhiên bằng tiếng Việt.
3. **Reranking:** Lấy top 20 từ hybrid rồi rerank xuống top 5 bằng cross-encoder đa ngôn ngữ **`BAAI/bge-reranker-v2-m3`** làm ứng viên đầu tiên. Model card công bố hỗ trợ multilingual và có ví dụ dùng Sentence Transformers; tuy nhiên phải đánh giá trên câu hỏi tiếng Việt của đội trước khi chốt model.
4. **Evaluation:** Tạo **40 câu hỏi gắn ground truth** từ tài liệu đã xác nhận: tra cứu ngưỡng/quy định, câu diễn đạt lại, câu hỏi cần hai mục tài liệu, và câu ngoài phạm vi/không có câu trả lời. Mỗi câu ghi đáp án chuẩn, tài liệu/mục hỗ trợ, loại rủi ro và hành vi mong muốn nếu thiếu nguồn. Chia tập tune và holdout; giữ holdout cố định khi so sánh chunking/retrieval. Dùng bốn metric RAGAS: **Context Precision, Context Recall, Faithfulness và Answer/Response Relevancy** để đánh giá thứ tự/chất lượng context và câu trả lời. RAGAS cung cấp các metric retrieval-augmented generation này; điểm LLM-as-judge chỉ là một phần đánh giá, không thay thế kiểm tra ground truth của đội
5. **Enrichment:** Trích xuất metadata xác định khi ingest: `source_id`, tên tài liệu, phiên bản/ngày hiệu lực, chương/mục/trang, tầng/khu vực, loại nội dung (quy trình/ngưỡng/FAQ) và ngôn ngữ. Metadata lấy từ nguồn/tên section có kiểm tra; không để LLM tự đặt phiên bản hoặc ngưỡng an toàn. Thêm **contextual prepend** ngắn vào text dùng embedding, ví dụ “Tài liệu: Quy trình PCCC; Mục 3.2; áp dụng tầng 2; phiên bản …”. Giữ nguyên nội dung gốc riêng để hiển thị trích dẫn. Không prepend metadata chưa xác minh.

#### 3. Timeline triển khai
- **Tuần 1:**
+ Chốt nguồn tài liệu được phép dùng, bản hiệu lực và chủ sở hữu; kiểm kê PDF scan/bảng/ngưỡng cần OCR hoặc rà soát thủ công.
+ Viết manifest tài liệu và metadata schema; tạo 40 câu hỏi ground truth, chia tune/holdout.
+ Cài prototype ingest offline; làm structure-aware parent/child chunking, kiểm tra trích dẫn page/section và dedupe theo document version.
+ Tạo dense baseline; đo Recall@5, MRR, p50/p95 và lỗi parser. Chưa nối dữ liệu RAG vào quyết định tuyến.
+ Bàn giao cuối tuần: corpus versioned, sample chunks được review, test set và bảng baseline.
- **Tuần 2:**
+ Thêm BM25 và RRF; so sánh BM25/dense/hybrid trên tune, chọn cấu hình theo retrieval metrics.
+ Chạy reranker ứng viên trên top 20; giữ lại nếu cải thiện chất lượng đủ bù latency.
+ Thêm contextual prepend/metadata filter; chạy HyQA offline như ablation, không bật mặc định nếu không có lợi ích đo được.
+ Nối vào nhánh hỏi đáp tài liệu riêng: chỉ trả lời từ top 5 context, bắt buộc dẫn nguồn/mục/trang; abstain khi thiếu căn cứ. Không cho RAG thay thế route engine/sensor/HITL.
+ Chạy holdout và kiểm tra thủ công toàn bộ ca an toàn; ghi metric thực đo, latency, lỗi OCR và ví dụ thất bại vào report. Nếu có hướng dẫn nguy hiểm hoặc sai phiên bản, chưa bật cho người dùng.
+ Bàn giao cuối tuần: pipeline thử nghiệm, benchmark so sánh, report có số liệu thực, danh sách việc chưa đạt.

