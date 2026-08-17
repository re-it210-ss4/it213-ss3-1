# BÁO CÁO BÀI TẬP LỚN: PHÁT TRIỂN VÀ CẤU HÌNH HỆ THỐNG HYBRID AI - R-LOGISTICS

**Họ và tên:** Nguyễn Thị Yến  
**Mã sinh viên:** B24DTCN180  

---

## BÀI 1: PHÂN TÍCH & LỰA CHỌN — CẤU HÌNH ĐA MÔI TRƯỜNG (PROFILES)

### 1. Phương Án Tối Ưu Nhất: Phương Án B

**Phương án B** là giải pháp tối ưu, chuẩn hóa và đúng đắn nhất theo triết lý thiết kế của Spring Boot và Spring AI. 

#### Lý do lựa chọn & Cơ chế hoạt động:
*   **Chia để trị (Separation of Concerns):** Cấu hình của môi trường nào nằm trọn vẹn và cô lập trong file của môi trường đó (`application-local.properties` cho Development/Ollama và `application-cloud.properties` cho Staging/Production/OpenRouter). 
*   **Cơ chế kích hoạt linh hoạt của Spring Profiles:** File `application.properties` đóng vai trò cấu hình chung và định nghĩa profile mặc định thông qua `spring.profiles.active=local`. Khi chạy trên môi trường Staging/Production, kỹ sư DevOps chỉ cần truyền tham số môi trường lúc khởi động (ví dụ: `-Dspring.profiles.active=cloud` hoặc đặt biến môi trường `SPRING_PROFILES_ACTIVE=cloud`) mà **hoàn toàn không phải sửa đổi bất kỳ dòng mã nguồn Java hay file cấu hình nào trong code**.
*   **Tận dụng cơ chế Tự động cấu hình (Auto-configuration):** Spring AI sử dụng các Annotation điều kiện như `@ConditionalOnProperty` hoặc `@ConditionalOnClass` để quyết định việc khởi tạo Bean. 
    *   Khi active profile `local`, các cấu hình của `spring.ai.openai` hoàn toàn không được nạp $\rightarrow$ Spring Boot chỉ khởi tạo duy nhất Bean `OllamaChatModel`.
    *   Khi active profile `cloud`, các cấu hình của `spring.ai.ollama` bị bỏ qua $\rightarrow$ Spring Boot chỉ khởi tạo duy nhất Bean `OpenAiChatModel`.
    Điều này giúp giải quyết triệt để bài toán hoán đổi môi trường một cách sạch sẽ, tường minh.

---

### 2. Phân Tích Nhược Điểm Và Lỗi Kỹ Thuật Của Các Phương Án Còn Lại

#### Phương án A: Gộp chung cấu hình vào một file duy nhất
Đây là một cấu hình **sai lầm nghiêm trọng** khi làm việc với Spring AI và vi phạm nguyên tắc đóng gói môi trường.

*   **Lỗi xung đột Bean (Conflict ChatModel Bean):** Spring AI dựa vào sự hiện diện của các key cấu hình để auto-configure các Bean tương ứng. Khi gộp chung tất cả vào `application.properties`, Spring Boot sẽ nhận diện cả cấu hình của Ollama và OpenAI cùng một lúc. Hệ quả là IoC Container sẽ cố gắng khởi tạo đồng thời cả `OllamaChatModel` và `OpenAiChatModel`. Khi ứng dụng thực hiện inject `@Autowired private ChatModel chatModel;`, Spring sẽ tung ra ngoại lệ `NoUniqueBeanDefinitionException` do có 2 Bean cùng kiểu mà không có Bean nào được đánh dấu là `@Primary`.
*   **Lỗi thiếu tham số khi khởi động (Missing Property):** Do tất cả cấu hình được nạp đồng thời, nếu lập trình viên ở môi trường local không khai báo biến môi trường `${OPENROUTER_API_KEY}`, Spring Boot sẽ báo lỗi ngay từ giai đoạn startup (`IllegalArgumentException` do không thể resolve placeholder) khiến ứng dụng không thể khởi động thành công tại local.
*   **Vi phạm tính bảo trì:** Mất đi ý nghĩa của việc chia môi trường, tăng rủi ro rò rỉ hoặc ghi đè nhầm thông tin cấu hình Production ở môi trường Local.

#### Phương án C: Tự thiết lập key cấu hình riêng biệt
Phương án này vi phạm nghiêm trọng tính đóng gói của Framework và **khiến ứng dụng không thể chạy được** nếu không viết thêm code Java phức tạp.

*   **Sai Key cấu hình chuẩn của Spring AI:** Spring AI không tự động nhận diện các key tự chế như `spring.ai.ollama.url` hay `spring.ai.openai.url`. Các key chuẩn của starter phải là `spring.ai.ollama.base-url` và `spring.ai.openai.base-url`. Do đó, Spring AI Starter sẽ hoàn toàn bỏ qua các cấu hình này và không tự động tạo ra bất kỳ một `ChatModel` nào.
*   **Phá vỡ nguyên tắc "Không sửa đổi mã nguồn":** Để key tự chế `spring.ai.active-model-type=ollama` hoạt động, lập trình viên bắt buộc phải viết thêm một lớp `@Configuration` thủ công, sử dụng `@Value` để đọc key, rồi dùng các câu lệnh rẽ nhánh `if-else` hoặc `switch-case` để khởi tạo Bean. Điều này vi phạm trực tiếp yêu cầu đề bài: *"không cần sửa đổi bất kỳ dòng mã nguồn Java nào"*.
*   **Thiếu thông tin cấu hình bắt buộc:** Phương án này bỏ sót hoàn toàn các tham số quan trọng để mô hình có thể kết nối và chạy được như `model` (định danh model `qwen2.5-coder:7b` hoặc `google/gemini-2.5-flash`) và `api-key` cho OpenAI.

---

### 3. Kết Luận So Sánh

| Tiêu chí đánh giá | Phương án A | Phương án B (Tối ưu nhất) | Phương án C |
| :--- | :--- | :--- | :--- |
| **Tính đúng đắn kỹ thuật** | ❌ Sai (Gây lỗi trùng/xung đột Bean) |  **Đúng (Chuẩn auto-config)** | ❌ Sai (Sai key chuẩn, thiếu tham số) |
| **Yêu cầu không sửa code** | ❌ Phải dùng `@Qualifier` hoặc chỉnh code |  **Đảm bảo 100% không sửa code** | ❌ Bắt buộc viết thêm code khởi tạo |
| **Tính cô lập môi trường** | ❌ Kém (Trộn lẫn local và cloud) |  **Xuất sắc (Tách biệt hoàn toàn)** | ❌ Kém (Trộn lẫn cấu hình thô) |

---
---

## BÀI 2: TÍNH TOÁN — CƠ CHẾ TOKENIZATION & CONTEXT WINDOW

### 1. Cơ Chế Tokenization Và Đặc Thù Xử Lý Tiếng Việt

#### Định nghĩa cơ chế Tokenization
Tokenization (Mã hóa từ) là quá trình chia nhỏ một văn bản thô thành các đơn vị thông tin nhỏ hơn gọi là **Tokens** (có thể là từ nguyên vẹn, cụm từ, ký tự hoặc các chuỗi ký tự con - subwords). Hệ thống LLM cần Tokenizer để chuyển đổi các chuỗi ký tự này thành các ID số (Token IDs) trước khi đưa vào mạng nơ-ron xử lý.

#### Vì sao Tiếng Việt tiêu tốn nhiều token hơn Tiếng Anh?
Hầu hết các LLM hiện đại sử dụng thuật toán mã hóa subword như **Byte-Pair Encoding (BPE)** hoặc **WordPiece**. Các bộ Tokenizer này gặp bất lợi lớn với Tiếng Việt vì những lý do sau:
*   **Dữ liệu huấn luyện thiên lệch:** Các bộ Tokenizer được tối ưu hóa dựa trên tần suất xuất hiện của từ ngữ trong tập dữ liệu huấn luyện. Tiếng Anh chiếm tỷ trọng áp đảo, giúp các từ tiếng Anh nguyên vẹn được gộp thành $1$ token duy nhất. Trái lại, Tiếng Việt chiếm tỷ lệ rất nhỏ, khiến Tokenizer không thể "học" được các từ nguyên vẹn mà phải bẻ nhỏ từ Tiếng Việt thành các mảnh subword ký tự hoặc byte, làm số lượng token tăng vọt.
*   **Đặc điểm chữ quốc ngữ có dấu:** Tiếng Việt sử dụng các ký tự có dấu phức tạp (như `á`, `ở`, `ệ`, `đ`). Trên bảng mã UTF-8, các ký tự có dấu này thường được biểu diễn bằng nhiều byte. Khi gặp các từ này, Tokenizer không nhận diện được trong từ điển (Vocabulary) nên buộc phải tách chúng thành từng byte riêng lẻ. 
    *   *Ví dụ:* Từ tiếng Anh `student` chỉ tốn $1$ token. Từ tiếng Việt `sinh viên` có thể tốn từ $2$ đến $3$ tokens. Từ `học` có dấu nặng và chữ `ọ` có thể bị phân rã thành nhiều tokens nhỏ hơn. Trung bình $1$ từ tiếng Anh $\approx$ $0.75$ token, trong khi $1$ từ tiếng Việt $\approx$ $1.5$ đến $2$ tokens. Do đó, 8,000 từ tiếng Việt dễ dàng bị đẩy lên thành 12,000 tokens.

---

### 2. Hiện Tượng Xảy Ra Khi Gửi Request & Phân Tích Lỗi

#### Hiện tượng xảy ra
Khi gửi request chứa 12,000 tokens (vượt quá Context Window mặc định là 8,192 tokens của Qwen2.5-Coder:7B trên Ollama), **Hiện tượng Tràn ngữ cảnh (Context Window Overflow/Truncation)** sẽ xảy ra.

#### Phân tích nguyên nhân và Hậu quả đối với chất lượng tóm tắt
*   **Cơ chế cắt gọt dữ liệu (Truncation):** Context Window là giới hạn cứng về mặt vật lý của bộ nhớ RAM/VRAM và kiến trúc Attention mạng nơ-ron. Khi số lượng token đầu vào lớn hơn giới hạn (12,000 > 8,192), Ollama sẽ tự động thực hiện cơ chế cắt gọt dữ liệu đầu vào theo thuật toán định sẵn (thường là cắt bỏ các token đầu tiên hoặc cắt bỏ toàn bộ phần đuôi vượt giới hạn).
*   **Hậu quả đối với chất lượng tóm tắt:**
    *   **Mất mát thông tin nghiêm trọng:** Đoạn văn bản bị cắt (khoảng gần 4,000 tokens phía sau) sẽ hoàn toàn "tàng hình" đối với mô hình. Chatbot sẽ chỉ thực hiện tóm tắt dựa trên ~70-80% nội dung đầu tiên của tài liệu kỹ thuật. Phần cuối tài liệu (thường chứa các kết luận, hướng dẫn vận hành hoặc lưu ý quan trọng) sẽ bị bỏ qua hoàn toàn.
    *   **Phản hồi sai lệch hoặc ảo tưởng (Hallucination):** Do không đọc được trọn vẹn tài liệu nhưng vẫn nhận được Prompt yêu cầu "tóm tắt toàn bộ tài liệu", mô hình có thể tự suy diễn, vá víu thông tin logic dựa trên phần dữ liệu thiếu hụt dẫn tới việc tạo ra các hướng dẫn kỹ thuật sai sót, gây nguy hiểm cho học viên khi thực hành.

---

### 3. Đề Xuất Giải Pháp Kỹ Thuật Khắc Phục

#### Giải pháp 1: Áp dụng chiến lược Chunking & MapReduce
Đây là giải pháp xử lý ở tầng logic mã nguồn ứng dụng (ví dụ: viết bằng Java/Python hoặc sử dụng các framework như LangChain/Spring AI).
*   **Bước 1 (Split):** Chia nhỏ tài liệu 12,000 tokens thành các đoạn nhỏ (Chunks), ví dụ mỗi đoạn 3,000 tokens. Đảm bảo có một khoảng chồng lấp (**Chunk Overlap** khoảng 200 tokens) giữa các đoạn để không làm mất ngữ cảnh ở ranh giới cắt.
*   **Bước 2 (Map):** Gửi từng đoạn Chunk này kèm theo prompt tóm tắt độc lập đến Ollama. Lúc này, mỗi request chỉ khoảng ~3,200 tokens, nằm an toàn trong giới hạn 8,192. Kết quả thu về là $4$ đoạn văn bản tóm tắt ngắn tương ứng với $4$ chunk.
*   **Bước 3 (Reduce):** Gộp $4$ đoạn tóm tắt ngắn đó lại thành một văn bản mới (tổng dung lượng lúc này chỉ còn khoảng 1,500 - 2,000 tokens). Tiếp tục gửi văn bản gộp này kèm theo prompt *"Hãy tổng hợp các bản tóm tắt thành một bài tóm tắt tài liệu kỹ thuật hoàn chỉnh"* đến Ollama để nhận về kết quả cuối cùng.

#### Giải pháp 2: Cấu hình tăng Context Window thông qua Modelfile của Ollama
Dòng mô hình Qwen2.5 hỗ trợ chiều dài ngữ cảnh mở rộng tối đa lên tới **128K tokens** nhờ kỹ thuật nội suy RoPE. Do đó, giới hạn 8,192 chỉ là cấu hình mặc định của Ollama và có thể tăng lên được.
*   **Cách triển khai:** Tạo một file có tên là `Modelfile` với nội dung cấu hình lại tham số `num_ctx` hệ thống:
```dockerfile
# Chỉ định mô hình gốc
FROM qwen2.5-coder:7b

# Cấu hình tăng Context Window lên 16,384 tokens (đủ chứa 12,000 tokens tài liệu)
PARAMETER num_ctx 16384