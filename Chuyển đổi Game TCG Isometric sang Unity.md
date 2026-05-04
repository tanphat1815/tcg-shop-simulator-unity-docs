# **Báo Cáo Phân Tích Kiến Trúc Kỹ Thuật Và Lộ Trình Phát Triển Trò Chơi TCG Card Shop Simulator Trên Nền Tảng Unity Thông Qua Trí Tuệ Nhân Tạo**

## **Mở Đầu Và Tầm Nhìn Chuyển Đổi Nền Tảng Kỹ Thuật**

Việc chuyển đổi một dự án phần mềm mô phỏng quản lý cửa hàng thẻ bài (TCG Card Shop Simulator) từ nền tảng công nghệ web dựa trên Phaser và Vue sang bộ máy phát triển trò chơi chuyên nghiệp Unity là một chiến lược tái cấu trúc kỹ thuật mang tính bước ngoặt. Nền tảng Phaser, mặc dù ưu việt trong các ứng dụng web 2D tuyến tính, lại bộc lộ những hạn chế kiến trúc sâu sắc khi phải đối mặt với không gian hình học ba chiều giả lập (Isometric).1 Đặc biệt, việc quản lý thứ tự hiển thị chiều sâu (depth sorting) và xử lý va chạm trên không gian lưới chéo của Phaser thường đòi hỏi những can thiệp thủ công phức tạp, gây suy giảm hiệu suất máy tính nghiêm trọng. Trong khi đó, Unity sở hữu một hệ sinh thái Isometric Tilemap được tích hợp sâu vào quy trình dựng hình (rendering pipeline), cho phép kiểm soát tự động các thuật toán xếp chồng trục Z (Z-as-Y sorting) một cách tối ưu.2

Quá trình dịch chuyển mã nguồn này không đơn thuần là việc biên dịch lại (recompiling) từ JavaScript/TypeScript sang C\#, mà là một sự thay đổi hoàn toàn về hệ tư tưởng lập trình. Hệ thống quản lý trạng thái phản ứng (reactive state management) của Vue phải được tái thiết kế thành mô hình hướng thành phần (Component-based) và hướng đối tượng đa hình của Unity.4 Để đảm bảo quá trình chuyển dịch khổng lồ này diễn ra với độ chính xác tuyệt đối, không dung thứ cho bất kỳ sai sót logic nào (zero-bug tolerance), việc ứng dụng các mô hình Ngôn ngữ Lớn (LLMs) như Claude, Gemini thông qua môi trường phát triển tích hợp (IDE) Cursor là giải pháp tối ưu. Tuy nhiên, AI chỉ có thể lập trình chính xác nếu nó được đặt trong một khuôn khổ quy tắc khắt khe, được định hướng bởi các siêu câu lệnh (prompts) chi tiết và được kiểm chứng tự động qua từng khâu phát triển.5

## **Cấu Trúc Hệ Thống Lõi Của Trợ Lý Trí Tuệ Nhân Tạo**

Nghiên cứu từ các kho dữ liệu mã nguồn mở như Claude Code Game Studios (CCGS) và Everything Claude Code (ECC) chỉ ra rằng việc coi AI là một thực thể lập trình duy nhất thường dẫn đến hiện tượng suy diễn sai lầm (hallucination) và phá vỡ cấu trúc tổng thể của dự án.5 Thay vào đó, AI cần được cấu hình như một studio phát triển trò chơi toàn diện với hệ thống phân cấp vai trò minh bạch và các giao thức cộng tác nghiêm ngặt.

### **Phân Cấp Môi Trường Làm Việc (Tiered Agent Coordination)**

Hệ thống điều phối của CCGS chia trợ lý AI thành 49 đặc vụ chuyên biệt, hoạt động trên ba cấp độ phân quyền logic 5:

1. **Định hướng Tầm nhìn (Directors \- Cấp 1):** Bao gồm Giám đốc Sáng tạo và Giám đốc Kỹ thuật. Ở cấp độ này, AI chịu trách nhiệm phân tích tài liệu đặc tả (SPEC) từ dự án Vue/Phaser cũ để quyết định các tiêu chuẩn kiến trúc (Architecture Decision Records) cho dự án Unity mới.  
2. **Quản lý Chuyên môn (Department Leads \- Cấp 2):** Các đặc vụ quản lý miền giới hạn, chịu trách nhiệm cho các hệ thống như Thiết kế Gameplay, Lập trình Trí tuệ Nhân tạo, và Thiết kế Giao diện Người dùng.  
3. **Thực thi Mã nguồn (Specialists \- Cấp 3):** Các đặc vụ thực hành trực tiếp như Chuyên viên lập trình Gameplay, Chuyên viên xử lý Hiệu năng, những người sẽ trực tiếp gõ mã C\# vào các tập tin .cs trong Unity.5

Sự phân cấp này đảm bảo rằng khi một đặc vụ thực thi hệ thống di chuyển cho NPC, nó không được phép tự ý thay đổi cấu trúc dữ liệu lưu trữ thẻ bài của hệ thống kho đồ trừ khi có sự phê duyệt theo trục dọc.

### **Thiết Lập Quy Tắc Hệ Thống (The.cursorrules Protocol)**

Để AI Cursor hoạt động chuẩn xác trong môi trường Unity, toàn bộ không gian làm việc phải được áp đặt một hệ thống quy tắc cốt lõi thông qua tập tin .cursorrules đặt tại thư mục gốc của dự án.8 Tập tin này nhúng kiến thức chuyên ngành sâu sắc vào bộ nhớ ngữ cảnh của AI, ép buộc nó suy nghĩ như một kỹ sư Unity cao cấp.

| Hạng mục Quy tắc | Nội dung Kỹ thuật Chuyên sâu | Mục đích và Tác động |
| :---- | :---- | :---- |
| **Kiến trúc Lõi** | Bắt buộc sử dụng Mô hình Dependency Injection hoặc Singleton Pattern an toàn. Không bao giờ sử dụng GameObject.Find() hoặc GetComponent() trong hàm Update().8 | Tối ưu hóa hiệu năng phần cứng, tránh rò rỉ bộ nhớ (memory leaks) và giảm tải cho bộ thu gom rác (Garbage Collector). |
| **Giao thức Cộng tác** | Bắt buộc áp dụng nguyên tắc "Plan-First" (Lên kế hoạch trước). AI phải giải trình thuật toán bằng mã giả (pseudo-code) và chờ người dùng phê duyệt trước khi sửa đổi file.5 | Ngăn chặn hiện tượng mã nguồn bị thay đổi ngoài ý muốn, đảm bảo con người nắm toàn quyền kiểm soát (Human-in-the-Loop). |
| **Xử lý Đầu vào** | Yêu cầu 100% tích hợp Unity New Input System. Mọi sự kiện tương tác phải hỗ trợ đồng thời cả PointerClick (Chuột) và Touch (Màn hình cảm ứng).9 | Đảm bảo tính khả thi cho việc phát hành đa nền tảng (Cross-platform) giữa Desktop và Mobile. |
| **Quản lý Dữ liệu** | Mọi dữ liệu tĩnh về thẻ bài, chỉ số, giá cả phải được lưu trữ dưới dạng ScriptableObjects. Tuyệt đối không hardcode dữ liệu vào các lớp hành vi (MonoBehaviour).10 | Giảm thiểu dung lượng RAM bị chiếm dụng, hỗ trợ việc thiết kế cân bằng game độc lập với logic lập trình. |

## **Giải Quyết Bài Toán Đa Nền Tảng (Desktop Và Mobile)**

Trăn trở về khả năng tương thích của trò chơi trong tương lai trên cả hai nền tảng máy tính để bàn (Desktop) và thiết bị di động (Mobile) là một vấn đề cốt lõi cần được giải quyết ngay từ khâu thiết kế kiến trúc. Phân tích dữ liệu cho thấy việc phát triển một trò chơi duy nhất đáp ứng cả hai môi trường này hoàn toàn khả thi nếu hệ thống đáp ứng được ba trụ cột kỹ thuật: Khả năng phản ứng của giao diện (Responsive UI), Xử lý luồng nhập liệu đồng nhất (Unified Input Handling), và Khung quản lý hiệu năng phần cứng (Performance Profiling).12

### **Kiến Trúc Giao Diện Phản Ứng (Responsive UI Layout)**

Trên môi trường di động, không gian hiển thị bị giới hạn nghiêm trọng và thường xuyên thay đổi tỷ lệ khung hình (Aspect Ratio) từ 16:9 đến 19.5:9 hoặc dài hơn. Trong khi đó, màn hình Desktop lại cung cấp không gian rộng rãi nhưng yêu cầu sự sắp xếp thông tin dày đặc hơn. Để xử lý nghịch lý này, hệ thống Canvas của Unity phải được áp dụng các nguyên tắc thiết kế Mobile-First.12

Tất cả các thành phần giao diện (UI Components) bắt buộc phải nằm dưới sự quản lý của thành phần Canvas Scaler. Chế độ Scale With Screen Size được kích hoạt với độ phân giải cơ sở được đặt ở mức 1920x1080.14 Điểm mấu chốt nằm ở việc định vị các điểm neo (Anchoring). Thay vì sử dụng tọa độ tĩnh (Absolute Positioning), mọi phần tử UI phải được neo vào các góc cụ thể của màn hình hoặc kéo giãn theo tỷ lệ phần trăm (Percentage-based stretching). Chẳng hạn, khu vực hiển thị tài chính của cửa hàng luôn được neo vào góc trên bên trái, trong khi thanh công cụ xây dựng luôn bám sát cạnh dưới của màn hình. Hơn nữa, kích thước của các khu vực có thể nhấp chuột (Touch Targets) trên nền tảng di động phải được bảo đảm tối thiểu ở mức 44x44 pixel để thỏa mãn tiêu chuẩn công thái học của thao tác chạm, ngăn ngừa việc người chơi chạm nhầm khi đang thao tác quản lý thẻ bài chi tiết.13

### **Hệ Thống Nhập Liệu Đồng Nhất Và Tối Ưu Hóa**

Việc phát triển riêng rẽ hệ thống chuột cho Desktop và hệ thống cảm ứng cho Mobile sẽ làm phình to mã nguồn và gây ra các lỗi không nhất quán. Unity New Input System cung cấp giải pháp trừu tượng hóa (abstraction layer) toàn diện, nơi một hành động duy nhất như Interact có thể được liên kết đồng thời với thao tác nhấp chuột trái hoặc thao tác chạm ngón tay.9 Khi người chơi thực hiện thao tác tương tác trên màn hình, hệ thống sẽ phát ra một tia chiếu (Raycast) xuyên qua màn hình vật lý vào không gian 2D Isometric. Các lớp (Layers) va chạm sẽ được cấu hình cẩn thận để hệ thống luôn phân biệt rõ ràng giữa việc người chơi đang cố gắng chạm vào nút UI hay đang cố gắng chọn một kệ hàng nằm ở không gian thế giới bên dưới.

Về mặt hiệu suất, các thiết bị di động có giới hạn về RAM và chu kỳ xử lý của CPU. Các tính năng tối ưu hóa bắt buộc bao gồm việc gộp các lệnh vẽ (Draw Call Batching), sử dụng cơ chế gom ảnh (Sprite Atlas) để giảm bớt số lượng tài nguyên đồ họa rời rạc phải tải vào bộ nhớ, và áp dụng cơ chế Object Pooling cho các đối tượng xuất hiện thường xuyên như thẻ bài, khách hàng, hoặc các ký hiệu cảnh báo.16

## **Tiêu Chuẩn Kỹ Thuật Hình Ảnh Hình Học Isometric**

Sự thành công của ảo giác không gian 3D trên bề mặt 2D phụ thuộc tuyệt đối vào các thông số toán học của từng điểm ảnh. Để dự án đạt được mức độ đồ họa sắc nét tương tự hình ảnh tham chiếu, người phát triển cần cung cấp cho các họa sĩ thiết kế (hoặc bộ phận tìm kiếm tài nguyên) các thông số kỹ thuật (Image Specifications) khắt khe.2

| Thuộc Tính Đồ Họa | Thông Số Bắt Buộc | Giải Thích Nguyên Lý Hoạt Động |
| :---- | :---- | :---- |
| **Kích thước Lõi (Tile Size)** | 64x32 pixels (hoặc tỷ lệ 2:1) | Đây là tỷ lệ vàng của hình chiếu dimetric projection (thường được gọi lỏng lẻo là isometric trong game). Tỷ lệ này tạo ra góc nhìn từ trên xuống ở khoảng 30 độ, hoàn hảo cho thể loại mô phỏng.2 |
| **Pixels Per Unit (PPU)** | Luôn bằng chiều rộng của ô lưới (ví dụ: 64\) | Unity quy ước không gian bằng đơn vị đo (Units). Đặt PPU bằng 64 đồng nghĩa với việc 1 ô lưới trên màn hình sẽ chứa chính xác 1 bức ảnh gạch sàn, loại trừ hoàn toàn các đường nứt (seams) khó chịu giữa các ô.2 |
| **Trọng Tâm Ảnh (Pivot Point)** | Đáy trung tâm của vật thể (0.5X, Y tùy chỉnh) | Đối với gạch lát sàn 64x32, Pivot thường ở 0.5, 0.5. Đối với vật thể có chiều cao như kệ tủ (ví dụ kích thước 64x128), Pivot phải đặt ở vị trí chân kệ chạm sàn (ví dụ 0.5X, 0.1Y hoặc cụ thể hơn là 32, 16 theo tọa độ điểm ảnh) để tính toán độ sâu Z-sorting chính xác.2 |
| **Cấu hình Lưới (Grid Component)** | Cell Size (1, 0.5, 1\) | Lưới Isometric chuẩn yêu cầu kích thước Y bằng một nửa X để định hình hình kim cương (diamond shape) của không gian đồ họa.2 |
| **Mesh Type** | Tight | Giảm thiểu việc dựng hình (rendering) các khu vực trong suốt (alpha pixels) ở các góc của hình thoi, giúp tiết kiệm băng thông GPU một cách đáng kể.2 |

## **Giai Đoạn 1: Tiền Xử Lý Phân Tích Tính Năng Từ Mã Nguồn Cũ**

Trước khi ra lệnh cho AI viết các dòng mã mới trên Unity, một bước trung gian cực kỳ quan trọng là buộc AI phải nghiên cứu lại toàn bộ kiến trúc của dự án cũ (Phaser/Vue) thông qua kho mã nguồn tcg-cards-shop-simulator. Việc này giúp trích xuất chính xác các luồng logic (logic flows), các thuật toán (algorithms), và cấu trúc trạng thái (state shapes) đã được phát triển trước đó để đảm bảo tính toàn vẹn của các chức năng.

Dưới đây là Siêu lệnh (Prompt) trích xuất dữ liệu, được gửi riêng cho Claude (hoặc Gemini) có khả năng đọc tập tin diện rộng:

---

Hãy đóng vai trò là một Kiến trúc sư Hệ thống phần mềm. Vui lòng phân tích toàn diện mã nguồn từ kho lưu trữ https://github.com/tanphat1815/tcg-cards-shop-simulator được viết bằng Phaser và Vue. Nhiệm vụ của bạn không phải là viết code mới, mà là lập tài liệu đặc tả toàn bộ các tính năng cốt lõi (Feature Extraction Specification) để chuẩn bị cho việc chuyển đổi sang C\# Unity.

---

Dự án cũ quản lý trạng thái trò chơi qua Vue (Vuex/Pinia) và dựng hình trò chơi qua Phaser. Tôi cần bạn bóc tách phần logic thuật toán ra khỏi phần hiển thị đồ họa.

---

Hãy cung cấp một bản báo cáo chi tiết dưới dạng Markdown cho từng tính năng sau đây. Với mỗi tính năng, phải mô tả rõ: (1) Mục đích tính năng, (2) Cấu trúc dữ liệu hiện tại (JSON/State), (3) Thuật toán/Logic cốt lõi (ví dụ: công thức xác suất, cơ chế vòng lặp).

1. **Hệ Thống Dữ Liệu Thẻ Bài & Các Gói (Cards & Packs Data):** Khám phá cấu trúc dữ liệu mô tả các lá bài (ID, tên, độ hiếm, giá trị). Đặc biệt chú trọng vào cơ chế Thuật toán Mở Gói (Gacha RNG): Làm thế nào hệ thống tính toán tỷ lệ rơi (Drop rates) cho các lá bài hiếm?  
2. **Hệ Thống Kho Đồ & Trưng Bày (Inventory & Display):** Logic nào quản lý việc chuyển thẻ bài từ kho lưu trữ của người chơi sang các kệ trưng bày tại cửa hàng?  
3. **Hệ Thống Lưới & Xây Dựng Cửa Hàng (Grid Placement):** Trong Phaser, người chơi mua và đặt kệ hàng như thế nào? Thuật toán nào kiểm tra giới hạn không gian hoặc tính hợp lệ của vị trí đặt?  
4. **Trí Tuệ Nhân Tạo Khách Hàng (Customer AI Logic):** Máy trạng thái (FSM) của khách hàng bao gồm những chu kỳ nào? (Xuất hiện, di chuyển, chọn hàng, quyết định mua dựa trên giá, xếp hàng thanh toán). Phân tích công thức kinh tế đánh giá sự chấp nhận mức giá của khách hàng.  
5. **Nền Kinh Tế & Thời Gian (Economy & Time Systems):** Cơ chế dòng tiền tệ hoạt động ra sao? Chu kỳ ngày/đêm tác động thế nào đến lượng khách hàng?

---

Chỉ cung cấp mô tả kiến trúc logic. Tuyệt đối không sinh ra mã C\# ở bước này. Tài liệu này sẽ làm nền tảng đầu vào cho các đặc vụ lập trình C\# ở giai đoạn tiếp theo.

Bản đặc tả được tạo ra từ câu lệnh trên sẽ là tài liệu tham chiếu (Context Reference) bất biến, được đính kèm vào tất cả các Prompt tiếp theo, đảm bảo AI không tự sáng tác ra các quy luật trò chơi nằm ngoài chủ đích ban đầu của nhà phát triển.

## **Lộ Trình Phát Triển Tích Hợp Bằng Cursor: Từng Bước Không Có Lỗi (Zero-Bug Roadmap)**

Lộ trình dưới đây chia nhỏ quá trình di chuyển nền tảng thành các cột mốc logic độc lập. Yêu cầu tiên quyết cho mỗi bước: **Không bao giờ chuyển sang bước tiếp theo nếu bước hiện tại chưa vượt qua khâu kiểm chứng thực tế (Playtesting) và không phát sinh bất kỳ ngoại lệ (Exception) nào trên Unity Console**.

Các siêu lệnh này tuân thủ chặt chẽ nguyên lý Prompting tối ưu cho Cursor bao gồm: Xác định mục tiêu, Cung cấp bối cảnh, Áp đặt ràng buộc, Yêu cầu lập kế hoạch trước (Plan-first), và Xác định chuẩn đầu ra.7

### **Bước 1: Thiết Lập Ma Trận Không Gian Hình Học Z-as-Y**

Kiến trúc đầu tiên cần thiết lập là bộ não không gian của trò chơi, giải quyết dứt điểm rắc rối về chiều sâu mà nền tảng Phaser cũ gặp phải.3

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Xây dựng kết cấu nền tảng Unity 2D Isometric Z-as-Y Tilemap, thiết lập hệ thống máy ảnh trực giao (Orthographic Camera) có khả năng phản ứng đa đầu vào (chuột và cảm ứng), và bộ quản lý trung tâm (GameManager).

---

Đây là bước đầu tiên trong dự án TCG Card Shop Simulator. Hệ thống cần không gian lưới chéo, trong đó các nhân vật và kệ hàng có thể che khuất lẫn nhau dựa trên vị trí của chúng ở trục Y và một phần trục Z.

1. ---

   Thiết lập Transparency Sort Mode thành Custom Axis. Cấu hình Transparency Sort Axis thành mảng Vector3 (0, 1, \-0.26) để giải quyết hiện tượng layer soup (vật thể xếp chồng lộn xộn) thường thấy ở không gian lưới chéo.3  
2. Viết CameraController.cs sử dụng Unity New Input System. Hỗ trợ thao tác nhấn chuột giữa/kéo (Pan) và cuộn chuột (Zoom) cho Desktop; vuốt một ngón (Pan) và chụm hai ngón (Pinch) cho Mobile. Cần áp đặt các hằng số toán học Mathf.Clamp để giới hạn vùng di chuyển của camera không trôi khỏi bản đồ cửa hàng.  
3. Cấu trúc GameManager theo mô hình Singleton để điều phối toàn bộ trạng thái sống còn của trò chơi.

---

Hãy mô tả bằng văn bản danh sách các GameObject cần tạo trên Hierarchy và giải thích cách bạn xử lý thuật toán Pinch-to-Zoom trước khi viết mã nguồn. Dừng lại chờ tôi xác nhận.

---

Hãy cung cấp mã nguồn CameraController.cs và GameManager.cs.

*Quy trình Debug yêu cầu:* Tôi sẽ tạo Tilemap và chạy dự án. Khi thao tác chuột và kéo vuốt trên màn hình cảm ứng mô phỏng, camera phải lướt nhẹ nhàng (dùng Vector3.Lerp để nội suy tạo độ mượt). Console phải ghi nhận "GameManager Ready" mà không có dòng lỗi đỏ (NullReferenceException).

### **Bước 2: Chuyển Đối Cấu Trúc Dữ Liệu Lõi Sang Khung Tĩnh (ScriptableObjects)**

Việc quản lý hàng trăm thẻ bài Pokemon/Yugioh đòi hỏi một kiến trúc không tiêu tốn tài nguyên bộ nhớ cho các thực thể trùng lặp.10 Hệ thống State của Vue phải được ánh xạ hoàn hảo sang các ScriptableObjects của Unity.

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Tái thiết kế kiến trúc lưu trữ dữ liệu tính năng thẻ bài, gói thẻ, và tủ kệ dựa trên đặc tả tính năng đã phân tích từ dự án Phaser cũ. Tích hợp thuật toán ngẫu nhiên (Gacha RNG) phục vụ việc mở hộp thẻ bài.

---

Trong TCG Shop Simulator, kho đồ (Inventory) chứa các gói thẻ (Pack). Mỗi gói chứa danh sách tỷ lệ rơi xác suất cho các thẻ bài (Card) bên trong.10

1. ---

   Thiết lập các lớp C\# kế thừa ScriptableObject:  
   * CardData: chứa ID, Name, Sprite, Rarity (enum: Common, Rare, Epic, Legendary), MarketValue.  
   * PackData: chứa ID, Cost, List\<CardDropTable\> (bao gồm tham chiếu CardData và tỷ lệ rớt tính theo phần trăm Float).  
   * ShelfData: chứa ID, Dimensions Vector2Int, Sprite lưới chéo.  
2. Viết InventoryManager.cs lưu trữ kho đồ người chơi sử dụng cấu trúc dữ liệu Dictionary\<string, int\> để tìm kiếm độ phức tạp O(1).  
3. Thuật toán Mở Gói (RNG): Hàm OpenPack(PackData) phải thực hiện đổ xúc xắc ngẫu nhiên dựa trên bảng xác suất tích lũy (Cumulative Weighted Probability) để trả về mảng 5 thẻ bài một cách minh bạch.10

---

Khái quát logic toán học của hàm quay xác suất tích lũy. Nếu tổng phần trăm rơi thẻ không đạt 100%, thuật toán của bạn xử lý ra sao? Hãy giải trình.

---

Cung cấp mã nguồn. Để Debug: Viết đoạn mã kiểm thử cục bộ trong hàm Start() của InventoryManager, gọi hàm mở gói 10 lần liên tục và xuất báo cáo tỷ lệ thống kê ra màn hình Console (ví dụ: "Đã mở 10 gói. Nhận được: 30 Common, 15 Rare, 5 Legendary"). Chắn chắn RNG không bị thiên lệch.

### **Bước 3: Toán Học Xây Dựng Và Hệ Thống Xếp Hàng Trên Lưới (Grid Placement)**

Đây là tính năng tương tác phức tạp bậc nhất, yêu cầu sự chính xác toán học tuyệt đối khi quy đổi sự kiện chạm/click trên không gian hai chiều của màn hình vật lý sang không gian tọa độ thế giới giả 3D của Unity.22

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Phát triển mô-đun xây dựng (Grid Placement System), cho phép người chơi chọn các kệ hàng (ShelfData) và sắp xếp chúng vào không gian lưới lưới chéo một cách trực quan, có cảnh báo va chạm.23

---

Người dùng cần thấy một bóng ma (Ghost Object) của chiếc kệ lơ lửng bám dính vào các ô lưới khi họ di chuột hoặc di ngón tay. Bóng ma đổi màu xanh lá nếu vị trí hợp lệ, và màu đỏ nếu có kệ khác đang chiếm dụng ô đó.22

1. ---

   Viết GridPlacementManager.cs. Thu thập tọa độ chuột/cảm ứng, sử dụng API Grid.WorldToCell để khóa chặt vật thể lơ lửng vào tâm của lưới tọa độ.22  
2. Cấu trúc lưu trữ Mạng lưới Không gian: Sử dụng một mảng 2D hoặc Dictionary Dictionary\<Vector2Int, GridNode\> để lưu trạng thái của từng ô (X, Y). Các thuộc tính gồm isOccupied và placedObjectRef.26  
3. Cung cấp cơ chế Xoay (Rotation): Nhấn phím R hoặc một nút trên UI di động sẽ xoay bóng ma 90 độ, thay đổi chiều không gian chiếm dụng của đối tượng (từ chiều ngang sang chiều dọc của lưới).  
4. Khi nhấn chuột trái/thả tay hợp lệ, khởi tạo (Instantiate) Prefab thực sự của kệ tủ, gán tọa độ chính xác, cập nhật ma trận GridNode thành isOccupied \= true.

---

Hãy giải thích chi tiết cơ chế cập nhật trạng thái ô lưới, đặc biệt đối với các kệ tủ có kích thước lớn hơn 1 ô lưới (ví dụ kệ tủ dài 2x1). Bạn cập nhật 2 ô (X, Y) và (X+1, Y) cùng lúc như thế nào để tránh lỗi va chạm cục bộ?

---

Khởi chạy Unity. Quá trình di chuyển chuột liên tục sẽ khiến bóng ma lưới chéo di chuyển nhịp nhàng bám theo các ô sàn. Khi click xác nhận, kệ hàng xuất hiện vĩnh viễn và bộ phân loại chiều sâu tự động xử lý hình ảnh kệ hàng che khuất các phần phía sau. Click chồng lên vị trí đó lần thứ hai phải hoàn toàn vô tác dụng và hiển thị lỗi trên Console: "Vị trí không hợp lệ, lưới bị chiếm dụng".

### **Bước 4: Trí Tuệ Không Gian \- Thuật Toán A\* Cho Đám Đông Khách Hàng**

Thay vì sử dụng tính năng NavMesh thông thường vốn được tối ưu cho thế giới 3D, sự di chuyển trong môi trường lưới Isometric yêu cầu một thuật toán giải quyết vấn đề toán học trên đồ thị (Graph) hoàn toàn thủ công để đảm bảo sự liền mạch.27 Các kệ hàng được đặt ở Bước 3 chính là các rào cản động (Dynamic Obstacles) cho quá trình tìm đường này.

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Tích hợp động cơ tìm đường A\* (A-Star Pathfinding Algorithm) được tối ưu hóa riêng biệt cho cấu trúc lưới của trò chơi cửa hàng thẻ bài, đảm bảo Khách hàng NPC tự động né tránh các kệ hàng lơ lửng.26

---

Khi khách hàng xuất hiện, họ cần điều hướng đến các tủ kính chứa bài. Nếu một kệ được người chơi đặt xuống, hệ thống lưới sẽ cập nhật và bất kỳ khách hàng nào đang di chuyển về hướng đó phải tính toán lại đường đi của mình.

1. ---

   Viết PathfindingCore.cs. Hệ thống phải đọc dữ liệu từ Dictionary\<Vector2Int, GridNode\> tạo ra từ Bước 3\. Bất kỳ ô nào có isOccupied \== true sẽ được gán cờ isWalkable \= false.26  
2. Hàm ước lượng (Heuristic Function): Bắt buộc sử dụng khoảng cách Manhattan (Manhattan Distance) do mạng lưới khách hàng chỉ di chuyển theo hệ thống 4 hướng hoặc 8 hướng.27  
3. Cấu trúc danh sách mở (Open List) phải sử dụng Cấu trúc dữ liệu Min-Heap (Binary Heap) thay vì List thông thường để giảm thiểu độ phức tạp tìm kiếm Nút có F-Cost thấp nhất từ ![][image1] xuống ![][image2], duy trì FPS ổn định khi có 50 khách hàng cùng di chuyển.  
4. Viết CharacterMovement.cs: Cung cấp hàm SetPath(List\<Vector2Int\> path). Nhân vật sử dụng hàm Vector3.MoveTowards để đi tuần tự qua từng nút trong đường dẫn. Đổi hướng hình ảnh Sprites (FlipX) khi đổi hướng di chuyển để tạo sự tự nhiên.29

---

Phân tích phương thức mã nguồn của bạn khi có sự kiện "người chơi vừa đặt kệ chặn đường đi của NPC đang đi giữa chừng". Cơ chế hủy bỏ đường cũ và gọi lại luồng tìm đường được thực hiện ra sao để tránh vòng lặp vô tận (Infinite Loop)?

---

Sinh ra (Spawn) một NPC tại tọa độ (0, 0\) và yêu cầu di chuyển đến (10, 10). Trong lúc NPC đang đi, tôi sẽ đặt một kệ hàng chặn ngay tọa độ (5, 5). NPC phải khựng lại 1 nhịp (khoảng 0.1s), Console báo "Cập nhật lưới phát sinh \- Tính lại đường đi", sau đó NPC tìm con đường khác đi vòng qua kệ hàng một cách chính xác.

### **Bước 5: Cỗ Máy Trạng Thái Của Khách Hàng (Customer Finite State Machine)**

Trí tuệ điều hướng mới chỉ mang NPC đến nơi; trí tuệ hành vi mới quyết định việc nền kinh tế trò chơi có hoạt động hay không. AI khách hàng cần mô phỏng quá trình cân nhắc mua hàng y hệt dự án Phaser gốc.30

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Lập trình Cỗ Máy Trạng Thái Hữu Hạn (FSM) điều khiển vòng đời tư duy của Khách hàng, từ lúc bước vào cửa hàng cho đến khi ra quyết định mua thẻ bài và xếp hàng thanh toán.31

---

NPC là đối tượng có trí tuệ. Nó phải tìm kiếm các kệ hàng đang có thẻ bài, đánh giá giá bán, quyết định mua hay không mua dựa trên sự chênh lệch so với giá thị trường, và cuối cùng tiến đến quầy thu ngân.

1. ---

   Viết kiến trúc FSM thông qua CustomerController.cs, sử dụng Enum để định nghĩa các tiểu trạng thái: EnterShop, Wander, ExamineShelf, QueueAtCheckout, ExitShop.31  
2. Giao tiếp Dữ liệu: NPC phải có một Radar truy vấn (Query) tới hệ thống Quản lý Cửa hàng để lấy danh sách các kệ hàng Shelf đang có trạng thái HasItem \== true.  
3. Thuật toán Ra Quyết Định Kinh Tế (Economic Decision Engine): Khi ở trạng thái ExamineShelf, NPC đọc giá bán do người chơi thiết lập. Tính toán công thức xác suất mua hàng: Nếu (Giá Bán \<= Giá Thị Trường), tỷ lệ mua là 95%. Mỗi 5% giá tăng thêm so với giá trị thị trường, tỷ lệ quyết định mua giảm đi 15%. Sử dụng số ngẫu nhiên để xác định giao dịch thành công hay thất bại.  
4. Hiệu ứng Hình ảnh (Visual Cues): Tạo Instantiate một bong bóng trò chuyện (Speech Bubble) nhỏ lơ lửng trên đầu NPC hiển thị icon trái tim (đồng ý mua) hoặc icon tức giận (từ chối do giá cao).

---

Thiết kế mô hình chuyển đổi trạng thái bằng văn bản. Giải thích cơ chế "Xếp hàng" (Queueing) hoạt động thế nào khi có quá nhiều khách hàng cùng muốn thanh toán tại một quầy duy nhất? Bạn tính toán Offset tọa độ (đứng lùi lại phía sau nhau) ra sao?

---

Kiểm tra: Đặt một lá bài quý giá vào tủ kính với giá cắt cổ (đắt gấp 3 lần giá thị trường). Spawn một khách hàng. Khách hàng sẽ đi đến tủ bài đó, dừng lại 2 giây, bong bóng tức giận hiện lên, trạng thái chuyển thẳng sang ExitShop, đi ra ngoài cửa. Mọi quá trình phải diễn ra mượt mà và ghi log xác suất chính xác lên Console.

### **Bước 6: Tương Tác Kinh Tế Lõi Và Hiệu Ứng Trải Nghiệm Mở Gói (Core Loop)**

Vòng lặp tương tác chính của TCG Simulator bao gồm các hành vi vĩ mô: Quản lý hàng hóa, Trưng bày, Nhận tiền thu ngân và Mở gói tìm thẻ hiếm.32

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Kết nối tất cả các mô-đun thành Vòng Lặp Lối Chơi Cốt Lõi (Core Gameplay Loop): Cửa sổ Trưng Bày Hàng (Stocking UI), Hệ thống Thu Ngân (Checkout System), và Trình diễn Hoạt ảnh Mở Gói (Pack Opening Sequence).

---

Các mảnh ghép riêng rẽ (Khách hàng, Lưới, Dữ liệu thẻ) đã hoàn tất. Giờ là lúc người chơi thực sự tương tác với cửa hàng thông qua việc click chuột/chạm vào từng chiếc kệ để sắp xếp hàng hóa, tính tiền khách và dùng tiền lời mua các gói bài mới để mở.16

1. ---

   Giao diện Tương tác Kệ Hàng (Shelf UI Panel): Khi người chơi sử dụng tia chiếu (Raycast) chạm vào một Prefab Kệ hàng đã đặt, tạm dừng thời gian game (Pause) hoặc mở một Panel nổi lên màn hình. Panel này liệt kê các loại thẻ bài có trong Kho (InventoryManager) để người chơi kéo thả (Drag and Drop) vào danh sách hiển thị của Kệ, kèm trường nhập dữ liệu số Float để "Đặt Giá Cắt Cổ/Giá Khuyến Mãi".  
2. Cơ chế Thu Ngân (Checkout Mechanics): Khách hàng ở Bước 5 khi đã quyết định mua hàng và đứng vào ô lưới quầy thu ngân, chờ người chơi click vào quầy để hoàn tất giao dịch. Viết hàm ProcessTransaction() cập nhật trừ sản phẩm khỏi kệ, cộng tiền tệ (Currency) cho người chơi.  
3. Cơ chế Hiển Thị Mở Gói (The Pack Opening Spectacle): Xây dựng một Coroutine PlayPackOpeningRoutine(). Khi người chơi chọn mở gói trong kho, giao diện tối lại, một hiệu ứng Particle System (tia lửa rực rỡ) bùng nổ, và lần lượt 5 lá bài được lật ngửa theo tuần tự, hiển thị biểu diễn hình ảnh, thu hút sự chú ý trước khi ghi dữ liệu vào InventoryManager.11

---

Liệt kê chuỗi các sự kiện giao diện người dùng (UI Events) cho quá trình Mở Gói Thẻ. Bạn sẽ sử dụng cơ chế Unity Animation hay LeanTween / DoTween để quản lý chuỗi hiển thị đồ họa lật bài?

---

Quá trình chơi thử nghiệm toàn diện (Integration Test): Đặt kệ trống. Click mở giao diện Kệ, lấy thẻ từ Kho ra đặt giá. Đợi khách vào, mua hàng, tính tiền ở quầy. Tiền phải được cập nhật ở Góc trái màn hình ngay lập tức (Observer Pattern). Dùng số tiền đó click nút Mua Pack, mở Pack, và thấy hiệu ứng 5 thẻ bài mới xuất hiện. Bất kỳ lỗi gián đoạn hoặc đứt gãy luồng nào sẽ bị từ chối.

### **Bước 7: Mạng Lưới Kiến Trúc Giao Diện Phản Ứng Đa Nền Tảng (Cross-Platform UI Framework)**

Sự tương tác của người chơi hoàn toàn thông qua giao diện người dùng, và hệ thống này không được phép biến dạng dưới bất kỳ điều kiện hiển thị vật lý nào.12

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Tổ chức lại hệ thống Giao Diện Người Dùng theo các tiêu chuẩn thực hành tốt nhất năm 2024 về tính Phản ứng (Responsive Design), tối ưu cho cả thao tác kéo thả bằng chuột trên Màn hình siêu rộng (Ultrawide Desktop) và thao tác chạm đa điểm trên Màn hình nhỏ (Mobile).12

---

Trò chơi phải đạt chuẩn thẩm mỹ cao cấp. Text bị vỡ, icon bị lọt ra ngoài màn hình do thiết kế Notch (tai thỏ) của điện thoại, hoặc các nút bấm quá nhỏ để chạm ngón tay sẽ dẫn đến thất bại.

1. ---

   Thiết lập Cây Canvas (Canvas Tree): Cấu hình Canvas trung tâm thành Screen Space \- Overlay. Component Canvas Scaler sử dụng thuộc tính Scale With Screen Size với độ phân giải chuẩn 1920x1080 và giá trị match width/height động để giữ nguyên vẹn bố cục.14  
2. Quản lý Vùng An Toàn (Safe Area Management): Viết script SafeAreaFitter.cs. Script này truy vấn Screen.safeArea lúc khởi tạo và ép kích thước của các container giao diện lùi sâu vào trong màn hình, tránh khu vực camera khuyết đỉnh trên điện thoại.  
3. Kích thước Mục tiêu Chạm (Touch Targets): Tất cả các phím điều khiển phải có kích thước thiết kế tối thiểu (Minimum Hitbox) là 44x44 pixel ảo, bọc trong các thẻ Padding hợp lý.13  
4. Hiển thị Giá cả Lơ lửng (Floating World Space UI): Các tấm biển ghi giá tiền phía trên đầu kệ hàng phải sử dụng Canvas \- World Space riêng biệt. Script đính kèm phải ép Canvas này luôn xoay mặt về hướng Camera lưới chéo để tránh bị méo mó hình học (Billboard effect).  
5. Cấm tuyệt đối tính năng Best Fit trên component Text, vì nó gây hỗn loạn trong quá trình dựng phông chữ đa thiết bị. Sử dụng kích cỡ tĩnh với mỏ neo thích hợp.14

---

Mô tả cách bạn áp đặt các Điểm Neo (Anchors \- Min/Max) cho Thanh Tài chính ở góc trên bên trái màn hình, và Bảng Xây dựng ở mép dưới của màn hình để chúng không trôi dạt.

---

Giả lập chế độ phân giải trong Unity Editor: Chuyển đổi qua lại nhịp nhàng giữa độ phân giải 16:9 4K của PC và tỷ lệ hẹp 1080x2400 (iPhone Portrait Mode). Không có phần tử giao diện nào đè lên nhau, chữ không mờ đục và chức năng click/chạm đáp ứng trong vòng một khung hình xử lý.

### **Bước 8: Tuần Tự Hóa Dữ Liệu Và Khôi Phục Trạng Thái Trò Chơi (Save/Load Serialization)**

Mảnh ghép sinh tồn cuối cùng là một dự án quản lý dài hơi không thể bắt đầu lại từ số không mỗi khi tắt máy. Dự án phải ghi nhớ mọi tọa độ kệ hàng, mọi thẻ bài bên trong và dòng thời gian hiện tại.

**Siêu Lệnh Đệ Trình Cho Cursor:**

---

Thiết kế hệ thống Lưu và Tải (Save/Load System) cấu trúc cao, mã hóa toàn bộ dữ liệu trạng thái động của người chơi và mạng lưới vật thể vật lý vào không gian bộ nhớ dài hạn, tương thích hệ thống tập tin của cả HĐH máy tính và di động.

---

Không sử dụng PlayerPrefs cho mảng dữ liệu có kiến trúc lớn. Hệ thống cần tuần tự hóa (Serialize) toàn bộ đối tượng Kệ Hàng và nội dung phức hợp (Cards) đang nằm bên trong thành một chuỗi JSON siêu văn bản để lưu trữ.4

1. ---

   Viết GameData.cs hoạt động như một lớp chứa dữ liệu thuần túy (POCO \- Plain Old C\# Object), dễ dàng chuyển đổi qua lại thành JSON thông qua JsonUtility hoặc NewtonSoft JSON.  
2. Các điểm dữ liệu bắt buộc cần thu thập (Data Harvesting Payload):  
   * float PlayerMoney.  
   * Dictionary\<string, int\> PlayerInventoryPackAndCards.  
   * List\<PlacedShelfData\>: Mỗi phần tử là một object chứa ShelfID, Vector2Int Position (tọa độ lưới), Quaternion Rotation, và mảng danh sách các CardID đang được trưng bày kèm mức giá float.  
3. An Toàn Bộ Nhớ (Memory Safety): Dữ liệu được ghi thẳng xuống ổ đĩa tại vùng an toàn cục bộ Application.persistentDataPath thành tệp gamesave.dat. Viết cơ chế mã hóa Base64 đơn giản nhằm ngăn người chơi vào thư mục sửa đổi số tiền của bản thân.  
4. Quá Trình Rehydration (Phục sinh trạng thái): Khi trò chơi bắt đầu, GameManager đọc chuỗi JSON, truyền tín hiệu cho GridPlacementManager lập tức dựng lên (Instantiate) vô số các Prefabs kệ hàng, tự động ép dính tọa độ và nhồi nhét lại số thẻ bài đúng y như trước khi thoát.

---

Lập danh sách quy trình Phục sinh trạng thái. Bạn giải quyết vấn đề đồng bộ hóa vòng đời thế nào nếu đối tượng Cửa hàng (Shop Manager) được sinh ra trước khi đối tượng Quản lý Kệ hàng (Shelf Manager) sẵn sàng đón nhận lệnh tải tọa độ?

---

Chạy Play, trải nghiệm game 5 phút (Xây 4 tủ, đặt thẻ, có khách đến, nhận tiền 250$). Tắt Play Editor đột ngột. Nhấn Play lại ngay sau đó. Ngay tại khung hình đầu tiên sau màn hình Loading, toàn bộ 4 cái tủ, tiền và vị trí bài trưng bày phải khôi phục không thừa thiếu một giá trị nhỏ nhất. Không có thông báo lỗi Serialization xuất hiện.

## **Khái Luận Tổng Quan Kiến Trúc**

Báo cáo phân tích kỹ thuật kiến trúc toàn diện trên định ra một giải pháp di cư nền tảng (Platform Migration) minh bạch và triệt để. Quá trình dịch chuyển mã nguồn trò chơi TCG Simulator từ hệ sinh thái Web phản ứng (Phaser/Vue) sang môi trường lập trình C\# hướng thành phần chuyên biệt (Unity) không còn chứa đựng sự rủi ro phỏng đoán. Thông qua việc phân bổ 100% công sức cho việc khống chế định dạng trí tuệ nhân tạo bằng giao thức.cursorrules, cấu trúc 6 tầng kiểm duyệt Prompt 7, dự án khai thác tối đa sức mạnh của Unity trong thiết lập toán học Isometric nâng cao.3 Các thuật toán cốt lõi, từ việc truy xuất ngẫu nhiên độ hiếm thẻ bài dựa trên trọng số, tìm đường A\* dựa trên heuristic không gian chéo, cho đến nghệ thuật phản hồi giao diện kích thước động, đã được mô hình hóa và gói gọn thành các mô-đun siêu lệnh sắc bén. Với việc trung thành áp dụng các nguyên tắc này, sản phẩm trò chơi kỹ thuật số cuối cùng sẽ hoàn thiện với tính toàn vẹn hệ thống tuyệt đối, loại trừ hoàn toàn các vết nứt kỹ thuật xuyên nền tảng (Cross-platform zero-bug output).

#### **Works cited**

1. Isometric Game: 3 Ways to Do It \- 2D, 3D | Unity Tutorial \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=XeqKQBIa43g](https://www.youtube.com/watch?v=XeqKQBIa43g)  
2. Create an isometric tilemap \- Unity \- Manual, accessed May 3, 2026, [https://docs.unity3d.com/6000.3/Documentation/Manual/tilemaps/work-with-tilemaps/isometric-tilemaps/create-isometric-tilemap.html](https://docs.unity3d.com/6000.3/Documentation/Manual/tilemaps/work-with-tilemaps/isometric-tilemaps/create-isometric-tilemap.html)  
3. Creating an Isometric Tilemap \- Unity \- Manual, accessed May 3, 2026, [https://docs.unity3d.com/es/2019.4/Manual/Tilemap-Isometric-CreateIso.html](https://docs.unity3d.com/es/2019.4/Manual/Tilemap-Isometric-CreateIso.html)  
4. Grid \- Unity \- Manual, accessed May 3, 2026, [https://docs.unity3d.com/2022.2/Documentation/Manual/class-Grid.html](https://docs.unity3d.com/2022.2/Documentation/Manual/class-Grid.html)  
5. GitHub \- Donchitos/Claude-Code-Game-Studios: Turn Claude Code ..., accessed May 3, 2026, [https://github.com/Donchitos/Claude-Code-Game-Studios](https://github.com/Donchitos/Claude-Code-Game-Studios)  
6. GitHub \- affaan-m/everything-claude-code: The agent harness ..., accessed May 3, 2026, [https://github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)  
7. Cursor Prompts Guide: Mastering .cursorrules and AI Coding | QuantumByte Success Story, accessed May 3, 2026, [https://quantumbyte.ai/articles/cursor-prompts](https://quantumbyte.ai/articles/cursor-prompts)  
8. Boost Your C\# Unity Game Development with Cursor AI Rulesets | Neura Market Blog, accessed May 3, 2026, [https://www.neura.market/workflows-blog/boost-your-c-unity-game-development-with-cursor-ai-rulesets-29](https://www.neura.market/workflows-blog/boost-your-c-unity-game-development-with-cursor-ai-rulesets-29)  
9. Mouse Click Movement in Isometric Tilemap \- Unity Tutorial \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=b0AQg5ZTpac](https://www.youtube.com/watch?v=b0AQg5ZTpac)  
10. I made a script to automate opening packs :: TCG Card Shop Simulator General Discussions, accessed May 3, 2026, [https://steamcommunity.com/app/3070070/discussions/0/4852155152090590819/?l=english](https://steamcommunity.com/app/3070070/discussions/0/4852155152090590819/?l=english)  
11. Creating a Card Game System in Unity | Unity Coder Corner \- Medium, accessed May 3, 2026, [https://medium.com/unity-coder-corner/unity-creating-a-card-game-ac7f46365a50](https://medium.com/unity-coder-corner/unity-creating-a-card-game-ac7f46365a50)  
12. Best Practices for Building Responsive Design in 2024 \- DEV Community, accessed May 3, 2026, [https://dev.to/smkbukhari/best-practices-for-building-responsive-design-in-2024-2k22](https://dev.to/smkbukhari/best-practices-for-building-responsive-design-in-2024-2k22)  
13. Responsive Design: Best Practices, Principles & Examples (2026) \- UXPin, accessed May 3, 2026, [https://www.uxpin.com/studio/blog/best-practices-examples-of-excellent-responsive-design/](https://www.uxpin.com/studio/blog/best-practices-examples-of-excellent-responsive-design/)  
14. Unity UI System \- Best Practices \- DEV Community, accessed May 3, 2026, [https://dev.to/marbleit/unity-ui-system-best-practices-2o24](https://dev.to/marbleit/unity-ui-system-best-practices-2o24)  
15. Responsive Web Design: Best Practices for 2024 | by Abdulsamad | Medium, accessed May 3, 2026, [https://medium.com/@abdulsamad18090/responsive-web-design-best-practices-for-2024-492a42635a4c](https://medium.com/@abdulsamad18090/responsive-web-design-best-practices-for-2024-492a42635a4c)  
16. nasfadev/store-simulator-game \- GitHub, accessed May 3, 2026, [https://github.com/nasfadev/store-simulator-game](https://github.com/nasfadev/store-simulator-game)  
17. Tilemap Renderer Modes \- Unity \- Manual, accessed May 3, 2026, [https://docs.unity3d.com/2018.3/Documentation/Manual/Tilemap-Isometric-RenderModes.html](https://docs.unity3d.com/2018.3/Documentation/Manual/Tilemap-Isometric-RenderModes.html)  
18. How to import Isometric assets and set up pivot? : r/Unity3D \- Reddit, accessed May 3, 2026, [https://www.reddit.com/r/Unity3D/comments/fqkxsu/how\_to\_import\_isometric\_assets\_and\_set\_up\_pivot/](https://www.reddit.com/r/Unity3D/comments/fqkxsu/how_to_import_isometric_assets_and_set_up_pivot/)  
19. How to Write Cursor AI Prompts That Don't Waste Your Time \- Builder.io, accessed May 3, 2026, [https://www.builder.io/m/explainers/best-cursor-ai-prompts-for-coding](https://www.builder.io/m/explainers/best-cursor-ai-prompts-for-coding)  
20. Creating an Isometric Tilemap \- Unity \- Manual, accessed May 3, 2026, [https://docs.unity3d.com/2019.3/Documentation/Manual/Tilemap-Isometric-CreateIso.html](https://docs.unity3d.com/2019.3/Documentation/Manual/Tilemap-Isometric-CreateIso.html)  
21. Isometric Tilemap sorting order fails when using multiple sprites (even with identical PPU/Pivot) \- Stack Overflow, accessed May 3, 2026, [https://stackoverflow.com/questions/79875730/isometric-tilemap-sorting-order-fails-when-using-multiple-sprites-even-with-ide](https://stackoverflow.com/questions/79875730/isometric-tilemap-sorting-order-fails-when-using-multiple-sprites-even-with-ide)  
22. Master Grid Placement in Unity 2022 P1 \- Calculating Cell Position \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=l0emsAHIBjU](https://www.youtube.com/watch?v=l0emsAHIBjU)  
23. Creating a building grid-based placement system \[Unity/C\# tutorial\] \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=jEYzUAhYXHI](https://www.youtube.com/watch?v=jEYzUAhYXHI)  
24. Unity Placing Objects \- Grid Placement System P3 \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=i9W1kqUinIs](https://www.youtube.com/watch?v=i9W1kqUinIs)  
25. How to Create a Grid Placement System in Unity \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=CoRCCBlWJuM](https://www.youtube.com/watch?v=CoRCCBlWJuM)  
26. icaromag/unity3d-realtime-weighted-pathfinding \- GitHub, accessed May 3, 2026, [https://github.com/icaromag/unity3d-realtime-weighted-pathfinding](https://github.com/icaromag/unity3d-realtime-weighted-pathfinding)  
27. Simple A\* Pathfinding In Unity \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=ji-f-74zfIQ](https://www.youtube.com/watch?v=ji-f-74zfIQ)  
28. GridGraph \- A\* Pathfinding Project \- Arongranberg.com, accessed May 3, 2026, [https://arongranberg.com/astar/documentation/beta/gridgraph.html](https://arongranberg.com/astar/documentation/beta/gridgraph.html)  
29. character navigation in unity3d for isometric games \- Stack Overflow, accessed May 3, 2026, [https://stackoverflow.com/questions/35973002/character-navigation-in-unity3d-for-isometric-games](https://stackoverflow.com/questions/35973002/character-navigation-in-unity3d-for-isometric-games)  
30. Isometric 2D environments with Tilemap \- Unity, accessed May 3, 2026, [https://unity.com/blog/engine-platform/isometric-2d-environments-with-tilemap](https://unity.com/blog/engine-platform/isometric-2d-environments-with-tilemap)  
31. Working on customer shopping flow for my Konbini Simulator (Unity) : r/Unity3D \- Reddit, accessed May 3, 2026, [https://www.reddit.com/r/Unity3D/comments/1rk8x7s/working\_on\_customer\_shopping\_flow\_for\_my\_konbini/](https://www.reddit.com/r/Unity3D/comments/1rk8x7s/working_on_customer_shopping_flow_for_my_konbini/)  
32. Unity Card Game: Pack Opening C\# \#46 \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=VE66DFQCdF4](https://www.youtube.com/watch?v=VE66DFQCdF4)  
33. Unity Card Game: Open More Packs C\# \#57 \- YouTube, accessed May 3, 2026, [https://www.youtube.com/watch?v=ZUD6BwkrVYk](https://www.youtube.com/watch?v=ZUD6BwkrVYk)

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADAAAAAXCAYAAABNq8wJAAACzklEQVR4Xu2XS6hPURSHl1Der0SiXI+UKANFBsqAYuCR5BEzAwaiDFxREimiPCNSMhCDO1XCQJTELRIZmZkwUYqiPH5fe+979tlnn+N271GufPV1u2ef/9l7rbP2+u+/2X/+LYbLA3JGOtAP1sqV6cUmBnkXyy75RHbLF3KjHFzcWmKoPC/XpAOe7fKKd7+5YAMd3rN+HI/Lceaee0Iu8jYyQd7yXpNjorHZ8o3cZ0WQMevlRTkkuR5gMavkJ/ldrojGWCTy5u7L3XKKFcli7rveaf5ahbHmPhwywANTtsqvcok3wGfvJddybJGH5Ed506rBjpbn5PjkOsm64O1MxnoY0AGwWBZNiUz25lgoP8u93sBy+diqE8ewCOp6jrnFE8T80h1mc+Upq5YnMAc+NJewEuzyH3JXOpAQAiALcSaOyavR/zkI7rQcZcV8h+MbzO0j3nIOah9pJgvigWHytuUzksIEP60cAGVAt8q+2giCD/eQQbpbeOOBI+buy0F54SO5Oh4gqnd+gBuaoD4JgG6CQEYfWPLQDNR/3GJ52zxrsxWLy9V/gHnCXKVkhbK4YfnaC5ApMvZKTvJCbwII9T8zukbLJHF3zK0B6+ofagOgnujN1+OLGbaZq1v+xvQmgLj+Ayz0qLmuFjpfXf1DbQAT5Ws/EE8Q0yHfWr69hofuSK7HkF3qO4U9x94jgVjanAlxAJVAd8ovcmk6YG7xT819M48sD/XA2+MMlBK+sfeYO4ak0ABoqc+8dfUPob1TxrTTEmT1oHxvRY/fIM/Il9Z8BgKy32XlLyY61gcvm/WbuX02IroHaKmXvE3wdrBbTk3GeuAVLfOuk7OseeEBSuG5tXsKTSFJSCXUbfSBH0BfoXTY4GmHaguO3pQo/u681Wemm5sgfD+0ySZ50tt69mP4wcGPj7TV9od58rK540flEPcnoNW2GUDbz/u7+AU69o5hZbvGuQAAAABJRU5ErkJggg==>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEwAAAAYCAYAAABQiBvKAAAENElEQVR4Xu2YbahUVRSGl5iglJoahih4rSQkpUJSKjOJgvqRRIgaBheMrMA/KX6gIRaKHyCGZkEI4g8tLYgIQVBIEfxIyZBUUPyRiBCCQqTQH+t97t6Ls2fPmeOduR8hzAsPc2fvM3POfvfaa625Zm21dT9piFglJuQT95E+EM/kg1UaEJkuvhcnxRnxm5grBhaX1miQ2C5mx/dviH2RX8WUON4fGiyWia8jiyysyYUh4PPA9XxuuPhKjI/cU23DmjBspPg2sksMS+aeEBfFcitMTfW2+FI8EN8/JD6K3BZT43h/iGcbJd4Td8UN8VQyjzHwrDgl5lm43tf0kvghwjpKhbOHrXCciMm1QPwjno+4+OyhbAxhElyNr/2tFWKNBdPW1k51aZz43IpNdpGLf4oQCHXCHEwigh6NlIlFEy1LIq5XxXExIhlD/6dhRMZWCwWItOJrS8Vzf5yNuUj+8I3VG2qvW9iFxflEJjeMnQPXerEzee+qMuxJsVlsi7xg9flxjOi0sDkTI2+JvWJmcl2ZHhMbLRwz1vWvmF9zRfjeGdmY67kI+bfGaM7yAXFLTE4nSkR4cuPUMNynOKQGusoMYwFLxW4LeePByKdihxWpYJq4JF6xUDAoOvCZWCc+jNc1EsXnnfg3UXZNHLRw3DyHcb+x8ZpcHFc4Z9lmM8iXHRND04kSEQkYRgUEROgfEW/G96nKDHtNXLf6zeHBr1iIAjaBo8BGsjBEfwfnxSNxrEpsYLpJX1iRf92MsvzlIrrhD8vW5sdsj9VXvlSEJXngdzE6gpo1DNN5CB4mlX8PJhFxRG2rhnn+SnMqRmEYxrFp0Ch/obZh1kuGPS3+spBTqvSuhcLAa6pmDeM+VYYBf1MELltYKD0gVRiWhssrlSZ8F7lxv4X0wzNAo4SPGhrGbrFrR6xxk9ZhIb+U9We+UEpwrjLD1lp4aHJIKqLhtIWGmYUujGAQ1dS/q+oUuEj47+eDVnQDrAXyNiOVG3bBSoyl4tyx0OHm6hC/WFgIR6VM7BbHJRd9Dty0wjB+bnCs5/lFUdwbY6mOiJK/wYKx/vDgR7RMtCWwxeqbaESDTU/GcYdGCR9x8oBKPSmb64qa1eJPK5rSORaqCGW16jckIrryB2ATMAqorLx6K8ADnBDfWXE8+InycpxHGPe3hc/mfGL1kU5VTO/HZzfVXBG02Oob7zJ5J/CzNT55XROzIjSIj1u1Ua7J4qw1/2+dhy30YpAeNQoKbc6LyZhHD4sgEssiqDflDXVZf9ljEVnkt7wgtCqOAk1qnucQveJRK/rAvhC5jSYXmg2CbovcxLH0dqMn4rhx7H60YB65y3MK/0HYaaFj7wsR6SstpI97/ZrokdqGtSASNYk2T8ititxGD9RpIadCh3WvtWhVVHVfQ2+to1K0Hv1yoz4SObI7ha6t7ug/r9v33rVfGZMAAAAASUVORK5CYII=>