# BTL-ANM-HK252

# Hướng dẫn Triển khai và Quản trị Hệ thống Wazuh SIEM

Tài liệu này hướng dẫn chi tiết các bước cài đặt, kết nối và xử lý sự cố (Troubleshooting) khi triển khai Wazuh Agent lên các máy trạm (Ubuntu/Linux) và kết nối với Wazuh Manager (triển khai qua Docker trên Azure).

---

## 1. Lưu ý Tối quan trọng (Điều kiện tiên quyết)

> **LUẬT BẤT THÀNH VĂN:** Phiên bản của Wazuh Agent cài trên máy trạm **BẮT BUỘC** phải nhỏ hơn hoặc bằng phiên bản của Wazuh Manager.
>
> _Ví dụ: Nếu Manager trên Azure đang chạy bản `4.4.4`, bạn tuyệt đối không được cài Agent bản mới nhất (ví dụ `4.14.x`), nếu không kết nối sẽ liên tục bị từ chối (Closing connection)._

---

## 2. Cài đặt Wazuh Agent (Máy trạm Ubuntu)

Để đảm bảo tương thích phiên bản với Manager, chúng ta sẽ chỉ định đích danh phiên bản cần cài đặt (ở đây ví dụ là `4.4.4`):

```bash
# 1. Cập nhật danh sách gói
sudo apt-get update

# 2. Cài đặt chính xác phiên bản Wazuh Agent (Thay số version nếu cần)
sudo apt-get install wazuh-agent=4.4.4-1 -y
```

---

## 3. Cấu hình Kết nối và Xác thực (Authentication)

Sau khi cài đặt, Agent lúc này giống như một "xác không hồn". Bạn cần trỏ IP và xin chìa khóa từ Manager.

### Bước 3.1: Đăng ký chìa khóa (Key) với Manager

Sử dụng công cụ `agent-auth` để xin cấp quyền. Thay `<IP_MANAGER>` bằng IP thực tế của Azure, và `<TÊN_AGENT>` bằng tên bạn muốn hiển thị trên Dashboard.

```bash
sudo /var/ossec/bin/agent-auth -m <IP_MANAGER> -A <TÊN_AGENT>
```

_Dấu hiệu thành công: Terminal trả về dòng `INFO: Valid key received`._

### Bước 3.2: Khai báo địa chỉ Manager trong file cấu hình

Mở file cấu hình cốt lõi của Agent:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Tìm đến khối `<client> -> <server>` và sửa thẻ `<address>` thành IP của Manager:

```xml
  <client>
    <server>
      <!-- Sửa IP tại đây -->
      <address>172.x.x.x</address>
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
    <!-- Các cấu hình khác giữ nguyên -->
```

_(Bấm `Ctrl + O` -> `Enter` để lưu, `Ctrl + X` để thoát)._

---

## 4. Khởi động và Kiểm tra trạng thái

Kích hoạt Agent để nó bắt đầu gửi log về trung tâm:

```bash
# Khởi động Agent
sudo systemctl enable wazuh-agent
sudo systemctl restart wazuh-agent

# Kiểm tra log để xác nhận kết nối thành công
sudo grep -i "connect" /var/ossec/logs/ossec.log | tail -n 5
```

_Nếu log hiển thị **`INFO: (1410): Connected to the server`**, xin chúc mừng, hệ thống đã thông mạng! Hãy F5 lại trang Dashboard để xem Agent chuyển sang trạng thái Active._

---

## 5. Các Lỗi Thường Gặp (Troubleshooting)

### Lỗi 1: `Closing connection to server` liên tục

- **Triệu chứng:** Trong log liên tục hiện dòng `Trying to connect...` rồi lập tức `Closing connection...`.
- **Nguyên nhân:** 99% do **Xung đột phiên bản (Version Mismatch)**. Agent mang phiên bản cao hơn Manager.
- **Cách xử lý:** Gỡ cài đặt Agent hiện tại (`sudo apt-get remove --purge wazuh-agent -y`), xóa thư mục `/var/ossec` và cài lại đúng phiên bản cũ khớp với Manager (xem lại Mục 2).

### Lỗi 2: `Duplicate agent name: <tên_máy> (from manager)`

- **Triệu chứng:** Lệnh `agent-auth` thất bại và báo lỗi trùng tên.
- **Nguyên nhân:** Tên Agent bạn đang cố đăng ký đã tồn tại trong cơ sở dữ liệu của Manager từ những lần thử lỗi trước đó.
- **Cách xử lý:** Đổi một cái tên mới hoàn toàn khi chạy lệnh đăng ký:
  ```bash
  sudo /var/ossec/bin/agent-auth -m <IP> -A <TÊN_HOÀN_TOÀN_MỚI>
  ```

---

## 6. Quản lý và Dọn dẹp Agent

### Tạm dừng Agent (Không gửi log nữa)

```bash
sudo systemctl stop wazuh-agent
```

### Xóa vĩnh viễn các Agent rác/bị lỗi (Thực hiện trên Manager)

Nếu trên Dashboard có nhiều Agent ở trạng thái `Never connected` hoặc `Disconnected`, bạn có thể xóa chúng tận gốc bằng lệnh sau trên máy chủ Manager (áp dụng cho Docker):

```bash
# Thay 001 bằng ID của Agent cần xóa
sudo docker exec single-node_wazuh.manager_1 /var/ossec/bin/manage_agents -r 001
```

_(Sau khi chạy lệnh, tải lại trang Dashboard để thấy thay đổi)._

---

_Tài liệu được đúc kết từ quá trình thực chiến cấu hình SIEM._

```</TÊN_AGENT></IP_MANAGER>

```
