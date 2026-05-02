# Triển khai Hệ thống SIEM: Azure VM + Wazuh (Docker) + Telegram Bot Alerts
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





