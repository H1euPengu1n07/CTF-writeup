# Brain Rot

![alt text](image-6.png)

## Description

![alt text](image-7.png)

## Recon

Đọc source, thấy có file `main.js`:

![alt text](image.png)

```text
Somewhere in the sprawl a maintenance terminal leaks the host filesystem
through the pinned-vulnerable Vite dev server (CVE-2025-30208).
```

Quét endpoint:

![alt text](image-1.png)

File `package.json` xác nhận phiên bản Vite là `5.4.14`.

![alt text](image-2.png)

Vite có endpoint đặc biệt `/@fs/` dùng để phục vụ các file nằm ngoài thư mục root của project theo đường dẫn tuyệt đối trên filesystem.

![alt text](image-3.png)

## Bypass CVE-2025-30208

CVE-2025-30208 là lỗ hổng của Vite dev server: có thể vượt qua cơ chế allow list của `/@fs/` bằng cách thêm query string `?raw??` hoặc `?import&raw??` vào cuối đường dẫn. Vite dùng dấu `?` để cắt bỏ query khi kiểm tra allow list, nhưng phần `??` dư ra khiến chuỗi sau khi xử lý không còn khớp với đường dẫn thực, làm bước kiểm tra bị bỏ qua và file vẫn được đọc và trả về nguyên văn.

![alt text](image-5.png)

## Flag

```text
PTITCTF{youtu.be/UsjsYMo3O1Q?si=d0om_scr0ll1ng_n0_gO0d_57ae60d8f963}
```
