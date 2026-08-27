# Machine Love

![alt text](image-3.png)

## Description

![alt text](image-4.png)

## Recon

Đọc JavaScript phía client, có thể thấy file video được nạp thông qua một request dạng:

```text
/somepath?file=koi%20o%20shite/Machine%20Love.mp4
```

![alt text](image.png)

Nghĩa là tham số `file` chỉ định đường dẫn tới file cần đọc, gợi ý cho hướng khai thác Path Traversal / Local File Read. Mình thử đọc thẳng absolute path `/etc/passwd`, server trả về:

```text
absolute path rejected
```

-> Server có filter kiểm tra input và từ chối các đường dẫn tuyệt đối (bắt đầu bằng `/`). Tuy nhiên, filter này kiểm tra chuỗi *sau khi* server đã tự động giải mã một lớp URL encoding, nên nếu mã hóa hai lần thì lớp filter sẽ không nhận ra ký tự nguy hiểm.

## Khai thác

Ý tưởng là dùng **double URL encoding** để qua mặt filter. Khi request đến, server giải mã một lần biến `%252f` thành `%2f`; chuỗi lúc này (`%2f`) không chứa ký tự `/` nên vượt qua được bước kiểm tra. Đến khi giá trị thực sự được dùng để mở file, nó lại được giải mã thêm một lần nữa thành `/`:

```text
/ -> %2f -> %252f
```

Áp dụng tương tự cho toàn bộ các ký tự trong payload (dấu chấm, dấu gạch, tên file) để chúng đều bị encode hai lần, qua đó tránh mọi bước kiểm tra chuỗi của filter:

![alt text](image-1.png)

Dò dần từng cấp thư mục bằng `../` (được encode hai lần thành `%252e%252e%252f`), đến cấp thứ 3 thì tìm được file `flag.txt`:

```text
/somepath?file=%252e%252e%252f%252e%252e%252f%252e%252e%252f%2566%256c%2561%2567%252e%2574%2578%2574
```

Trong đó `%252e%252e%252f` = `../` và `%2566%256c%2561%2567%252e%2574%2578%2574` = `flag.txt`.

![alt text](image-2.png)

## Flag:

```text
PTITCTF{youtu.be/sqK-jh4TDXo?si=t3t0Oo_a701c8e2564d}
```
