## 1. File cấu hình `docker-compose.yml` mới

```yaml
version: '3'
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "127.0.0.1:8080:8080"
```

## 2. Giải thích cơ chế hoạt động và cách ngăn chặn truy cập trái phép

### Vấn đề gặp phải với Docker và UFW mặc định:
- Khi sử dụng cú pháp mặc định `ports: - "8080:8080"`, Docker hiểu ngầm là bind vào `0.0.0.0:8080` (lắng nghe trên tất cả các network interface của host, bao gồm cả card mạng public như `eth0`).
- Docker tự động can thiệp vào `iptables`, chèn các rule NAT trong chain `PREROUTING` và forward packet qua chain `DOCKER` trong bảng `FORWARD`.
- UFW thông thường cấu hình các rule lọc gói tin trên chain `INPUT`. Do các packet gửi tới container được Docker định tuyến qua chain `FORWARD` trước khi vào chain `INPUT` của host, các rule chặn cổng `8080` của UFW bị bypass hoàn toàn.

### Cách giải quyết bằng Port Binding `127.0.0.1:8080:8080`:
1. **Chỉ lắng nghe trên Loopback Interface**:
   - Cú pháp `127.0.0.1:8080:8080` chỉ thị Docker chỉ bind port `8080` vào địa chỉ Loopback `127.0.0.1`.
   - Tiến trình `docker-proxy` (hoặc rule iptables DNAT của Docker) sẽ chỉ bắt các request có destination IP là `127.0.0.1`.

2. **Chặn truy cập từ Internet / Mạng ngoài**:
   - Các gói tin từ bên ngoài internet gửi tới máy chủ sẽ có destination IP là địa chỉ Public IP của host (ví dụ: `203.0.113.10:8080`).
   - Do cổng `8080` không lắng nghe trên Public IP (`0.0.0.0`), gói tin sẽ bị kernel từ chối (Connection refused) hoặc bị drop theo rule mặc định của firewall, hoàn toàn không được chuyển tiếp vào container.

3. **Cho phép Nginx nội bộ truy cập (Reverse Proxy)**:
   - Nginx chạy trực tiếp trên cùng máy chủ (host) có thể giao tiếp thông qua loopback interface:
     ```nginx
     location / {
         proxy_pass http://127.0.0.1:8080;
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
     }
     ```
   - Khi đó, client bên ngoài chỉ có thể truy cập qua cổng công khai của Nginx (ví dụ: 80 hoặc 443 với SSL/TLS và xác thực), còn cổng 8080 của ứng dụng backend được bảo vệ tuyệt đối.
