# Triển khai hệ thống thống SIEM sử dụng mã nguồn mở Wazuh
# I. Triển khai Hệ thống SIEM: Azure VM + Wazuh (Docker) + Telegram Bot Alerts
Hệ thống SIEM phục vụ cho bài tập lớn được triển khia trên nền tảng máy ảo Azure VM cho phép truy cập linh hoạt và thuận tiện cho các thành viên 
trong việc vài đặt, theo dõi trạng thái các Agents cũng như triển khai các kịch bản tấn công

## 1. Cấu hình VM và thông tin liên quan
Cấu hình VM được lựa chọn như sau
- Size: Standard D2s v3
- vCPUs: 2
- RAM: 8 GiB
- OS: Ubuntu Linux Server 24.04 LTS
- IP tĩnh
- Phương thức kết nối: SSH key

## 2. Triển khai Wazuh thông qua Docker

### Cập nhật hệ thống
```bash
sudo apt update && sudo apt upgrade -y
```

### Cài đặt Docker & Docker Compose
```bash
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
```

### Tải và triển khai Wazuh
```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker
docker-compose up -d
```

### Truy cập giao diện Wazuh
- Mở trình duyệt và nhập:
  ```
  https://<VM_Public_IP>/app/wazuh
  ```
 Lúc này Wazuh sẽ yêu cầu tài khoản và mật khẩu để tiếp tục
 ```
  Tài khoản: kibanaserver
  Mật khẩu:  kibanaserver
  Hoặc
  Tài khoản: admin
  Mật khẩu:  SecretPassword
 ```
 
---

## 3. Thiết lập Telegram Bot và Alerts

### Tạo Telegram Bot
1. Vào Telegram, tìm **BotFather**.
2. Gõ `/newbot` → đặt tên và username.
3. Nhận **API Token**.

### Lấy Chat ID
1. Vào Telegram, tìm **@bot_username**.
2. Bắt đầu một cuộc trò chuyện với bot, nếu muốn triển khai trong một nhóm, thêm bot vào và bắt đầu cuộc trò chuyện.
3. Truy cập link sau để lấy Chat ID: `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates`


### Tích hợp Telegram bot với Wazuh
1. Truy cập Wazuh Dashboard
2. Vào **Menu** (góc trải trên) chọn **Notification** sau đó chọn **Create Channel**
3. Nhập tên và mô tả cho Channel (VD: Telegram bot)
4. Tại **Channel Type** chọn `Custom webhook`
5. Giữ nguyên các cài đặt, tại ô **Webhook URL** nhập
`https://api.telegram.org/bot<BOT_TOKEN>/sendMessage`
6. Lưu **Channel** vừa tạo

### Thiết lập Alert cho việc cảnh báo
1. Vào **Menu** chọn **Alert** sau đó chuyển đến tab `Monitor`
2. Tạo **Monitor** mới
3. Chọn các chế độ cho Monitor, ở đây chọn `Per Query Monitor`
4. Chọn kiểu query, để linh hoạt trong thiết lập truy vấn, ở đây chọn `Extraction Query Editor`
5. Thiết lập query dựa trên nhu cầu sử dụng
6. Tạo **Trigger** cho Alert, sau đó tạo **Action**
7. Tại tab Action, ở mục Channel, chọn Channel có Telegram bot vừa tạo
8. Thiết lập message về định dạng `JSON`
9. Lưu Monitor

## Kết quả
- **Azure VM**: Máy chủ Ubuntu hoạt động trên Azure.
- **Wazuh (Docker)**: SIEM triển khai thành công.
- **Telegram Bot**: Nhận cảnh báo bảo mật trực tiếp trên điện thoại.


# II. Hướng dẫn triển khai Wazuh Agent

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

