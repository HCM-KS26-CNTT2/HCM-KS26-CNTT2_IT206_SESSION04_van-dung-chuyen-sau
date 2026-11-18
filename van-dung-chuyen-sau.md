## [Vận dụng chuyên sâu] Thiết kế giải pháp Prompting ứng dụng thực tế cho Kỹ thuật Prompting Nâng cao (Logic & Chuỗi)
### PHẦN 1: THIẾT KẾ GIẢI PHÁP (I/O & LUỒNG XỬ LÝ)

- Bài toán nghiệp vụ: Xây dựng hệ thống tự động tiếp nhận khiếu nại đổi trả sản phẩm lỗi của khách hàng, phân tích nguyên nhân kỹ thuật từng bước (CoT) và chia nhỏ chuỗi công việc thành các bước độc lập (Prompt Chaining) để chuyển giao cho các phòng ban liên quan (Kho vận, Kỹ thuật bảo hành, CSKH).
- Đầu vào (Input):
  + Thông tin đơn hàng và mô tả lỗi từ khách hàng ([CUSTOMER_COMPLAINT]).
  + Lịch sử mua hàng và trạng thái bảo hành ([ORDER_HISTORY]).
- Đầu ra mong muốn (Output):
  + Báo cáo phân tích nguyên nhân từng bước (Chain-of-Thought).
  + Quyết định duyệt đổi trả tự động (Approved / Rejected / Needs Review).
  + Email phản hồi khách hàng và phiếu chuyển giao nội bộ cho bộ phận kho vận.
----
### PHẦN 2: MASTER PROMPT THỰC THI (ÁP DỤNG CHAIN-OF-THOUGHT & CHAINING)

BƯỚC 1: PHÂN TÍCH NGUYÊN NHÂN TỪNG BƯỚC (CHAIN-OF-THOUGHT) 
```
Bạn là Trưởng ban Kiểm định Chất lượng & CSKH tại TechShop, chuyên sử dụng tư duy logic từng bước để xử lý các khiếu nại phức tạp của khách hàng.
Hãy phân tích khiếu nại của khách hàng dưới đây theo phương pháp suy luận từng bước (Chain-of-Thought) để đưa ra phán đoán chính xác nhất.
Action & Logic Steps:
- Bước 1: Đọc và xác thực thông tin khiếu nại so với chính sách đổi trả (7 ngày đầu lỗi nhà sản xuất được đổi mới).
- Bước 2: Phân tích nguyên nhân kỹ thuật tiềm ẩn dựa trên mô tả lỗi của khách hàng.
- Bước 3: Đánh giá xem lỗi thuộc trách nhiệm của nhà sản xuất, đơn vị vận chuyển hay do người dùng sử dụng sai cách.
- Bước 4: Đưa ra kết luận cuối cùng (Duyệt đổi mới / Từ chối / Chuyển kỹ thuật kiểm tra trực tiếp) kèm mức độ tự tin.

Dữ liệu đầu vào:
- [CUSTOMER_COMPLAINT]: "Tôi mua chiếc laptop ngày hôm qua, mở hộp ra thì màn hình có một đường sọc ngang màu xanh và không thể tắt được."
- [ORDER_HISTORY]: "Đơn hàng #TS98234, giao thành công ngày hôm qua, còn nguyên tem bảo hành."

Format Output Yêu cầu: Trình bày rõ ràng từng bước suy luận (Step 1 đến Step 4) trước khi đưa ra kết luận cấu trúc JSON.
```


BƯỚC 2: CHUỖI TIẾP THEO (PROMPT CHAINING - TỰ ĐỘNG SOẠN EMAIL & PHIẾU KHO)
(Lưu ý: Đầu ra của Bước 1 sẽ được tự động truyền làm đầu vào cho bước này).

```
Bạn là chuyên viên tự động hóa quy trình vận hành và chăm sóc khách hàng tại TechShop.
Dựa trên kết quả phân tích kiểm định ở bước trước, hãy tự động hóa việc tạo ra 2 tài liệu sau:
1. Email phản hồi khách hàng: Lịch sự, thấu cảm, thông báo quyết định đổi mới sản phẩm và hướng dẫn thủ tục gửi trả hàng miễn phí.
2. Phiếu điều phối nội bộ kho vận (Warehouse Ticket): Cấu trúc dạng JSON gồm các trường: `ticket_id`, `action_required` ("Thu hồi và đổi mới"), `priority` ("High"), `note`.

Xử lý lỗi & Đứt gãy ngữ cảnh:
- Nếu dữ liệu đầu vào bị thiếu mã đơn hàng hoặc không rõ mô tả lỗi, hệ thống phải tự động trả về: {"error": "Thiếu dữ liệu kiểm định, yêu cầu nhân viên CSKH nhập bổ sung"}.
- Trường hợp phản hồi của AI bị cắt cụt giữa chừng do giới hạn token (đứt gãy ngữ cảnh), yêu cầu hệ thống kích hoạt lệnh nối tiếp: "Continue generating from where you left off, do not repeat previous text."
```
----
###  PHẦN 3: KẾT QUẢ CHẠY THỬ THỰC TẾ 
#### *Kết quả phản hồi từ AI (Bước 1 - Chain-of-Thought):*
**Bước 1: Đọc và xác thực thông tin khiếu nại so với chính sách đổi trả**
- **Thông tin khiếu nại:** Khách hàng mua laptop ngày hôm qua, khi mở hộp phát hiện màn hình có một đường sọc ngang màu xanh và không thể tắt.
- **Lịch sử đơn hàng:** Đơn hàng **#TS98234**, giao thành công **ngày hôm qua** (trong vòng 7 ngày đầu), sản phẩm **còn nguyên tem bảo hành**.
- **Đánh giá chính sách:** Theo chính sách đổi trả của TechShop, sản phẩm mua trong vòng 7 ngày đầu tiên nếu gặp lỗi từ nhà sản xuất và còn nguyên tem bảo hành sẽ được áp dụng chính sách **đổi mới 1-1**.
**Bước 2: Phân tích nguyên nhân kỹ thuật tiềm ẩn**
-   **Hiện tượng:** Xuất hiện đường sọc ngang màu xanh cố định trên màn hình ngay từ lần mở hộp đầu tiên (out-of-box failure).
-   **Nguyên nhân tiềm ẩn:**    -   Lỗi tấm nền LCD/OLED panel (Dead/defective pixel lines).
    -   Lỗi cáp tín hiệu màn hình (Flex cable) bị lỏng hoặc đứt ngầm trong quá trình lắp ráp tại nhà máy.
    -   Lỗi vi mạch điều khiển hiển thị (T-Con board).
### Bước 3: Đánh giá trách nhiệm
-   **Thời gian phát sinh:** Lỗi xuất hiện ngay khi mở hộp (Day 1 / Out-of-box failure).
-   **Tình trạng vật lý:** Sản phẩm còn nguyên tem bảo hành, vừa mới giao ngày hôm qua, loại trừ hoàn toàn các tác động va đập hoặc sử dụng sai cách từ phía người dùng.
-   **Xác định trách nhiệm:** Lỗi thuộc về **nhà sản xuất (lỗi linh kiện phần cứng hoặc lỗi khâu kiểm tra xuất xưởng - QC)**.
### Bước 4: Kết luận cuối cùng và Mức độ tự tin
-   **Kết luận:**  **Duyệt đổi mới sản phẩm** và tiến hành tạo phiếu thu hồi, điều phối kho vận.
-   **Mức độ tự tin:** 100%

#### Kết quả phản hồi từ AI (Bước 2 - Prompt Chaining & JSON Output):
### 1. Email phản hồi khách hàng
**Tiêu đề:** [TechShop] Thông báo duyệt đổi mới sản phẩm cho đơn hàng #TS98234

Kính gửi Quý khách hàng,

Lời đầu tiên, TechShop xin gửi lời xin lỗi chân thành nhất đến Quý khách vì sự cố không mong muốn khi sản phẩm nhận được gặp lỗi hiển thị màn hình ngay trong lần mở hộp đầu tiên. Chúng tôi rất thấu hiểu sự bất tiện và trải nghiệm chưa hoàn hảo này của Quý khách.

Sau khi tiếp nhận và kiểm tra nhanh thông tin khiếu nại đối với đơn hàng **#TS98234**, bộ phận Kiểm định Chất lượng TechShop đã chính thức **phê duyệt đổi mới 1-1** cho sản phẩm của Quý khách.

Để tiến hành thủ tục đổi mới hoàn toàn **miễn phí**, xin Quý khách vui lòng thực hiện theo các bước sau:
1.  **Đóng gói sản phẩm:** Đặt lại laptop vào hộp nguyên bản, kèm đầy đủ phụ kiện, hóa đơn và quà tặng kèm (nếu có).
2.  **Gửi trả hàng miễn phí:** Nhân viên giao nhận của đơn vị vận chuyển đối tác sẽ liên hệ để đến tận nơi thu hồi sản phẩm tại địa chỉ của Quý khách hoàn toàn miễn phí (Quý khách không cần thanh toán bất kỳ chi phí vận chuyển nào).
3.  **Nhận sản phẩm mới:** Ngay sau khi kho vận nhận được sản phẩm lỗi và xác nhận nguyên tem bảo hành, TechShop sẽ lập tức gửi đi sản phẩm mới nguyên seal đến Quý khách trong thời gian sớm nhất.

Nếu Quý khách có bất kỳ thắc mắc hoặc cần hỗ trợ khẩn cấp, xin vui lòng phản hồi trực tiếp qua email này hoặc liên hệ hotline chăm sóc khách hàng của TechShop.

Trân trọng,

**Chuyên viên Chăm sóc Khách hàng TechShop**
### 2. Phiếu điều phối nội bộ kho vận (Warehouse Ticket)
```json 
{
  "ticket_id": "TS98234",
  "action_required": "Thu hồi và đổi mới",
  "priority": "High",
  "note": "Khách hàng gặp lỗi sọc màn hình ngay khi mở hộp (lỗi NSX trong thời hạn đổi trả 7 ngày). Đã duyệt đổi mới 1-1."
}
```