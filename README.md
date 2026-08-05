# download-redirect

Trang tĩnh **mở app / tải app theo OS** cho QR code.

Thứ tự ưu tiên khi quét QR trên mobile:

1. **App đã cài** → mở thẳng app, kèm query param lấy từ URL.
2. **App chưa cài** → về App Store / CH Play.
3. **Desktop hoặc không nhận diện được** → hiện trang chọn thủ công.

Độc lập hoàn toàn với web production Angular. **QR code trỏ tới URL của trang này**, không phải trỏ tới web production.

## Cơ chế mở app

Có 2 lớp, chạy nối tiếp nhau:

**Lớp 1 — Universal Link (iOS) / App Links (Android).** OS chặn URL *trước khi* trình duyệt load trang, mở thẳng app với đúng URL gốc (kèm param). Đây là đường đi tốt nhất và `index.html` thậm chí không chạy. Điều kiện: 2 file verification phải phục vụ được tại

- `https://<domain>/.well-known/assetlinks.json`
- `https://<domain>/.well-known/apple-app-site-association` (Content-Type `application/json`)

`netlify.toml` đã set sẵn header + rewrite cho 2 đường dẫn này.

**Lớp 2 — fallback trong `index.html`**, chạy khi lớp 1 trượt (chưa verify domain, mở từ webview app khác, iOS bỏ qua Universal Link khi điều hướng nội bộ cùng domain):

- **Android**: điều hướng sang `intent://<host><path><query>#Intent;scheme=https;package=…;S.browser_fallback_url=<store>;end`. Nhắm thẳng package nên mở được app kể cả khi App Links chưa verify; chưa cài thì Chrome tự đưa về CH Play.
- **iOS**: cần custom URL scheme, khai ở hằng `CONFIG.IOS_APP_URL_BASE` trong `index.html`. **Hiện đang để trống** → iOS bỏ qua bước này và về thẳng App Store. Điền vào (vd `'bidvhome://open'`) khi biết scheme thật của app.

Nếu app không mở sau `STORE_FALLBACK_DELAY` (1.5s) thì rơi về store; timer bị huỷ ngay khi trang mất focus (dấu hiệu app đã mở).

## Truyền param sang app

Mọi query param trên URL QR được chuyển tiếp nguyên vẹn vào app, trừ các param điều khiển của trang (`CONFIG.CONTROL_PARAMS`):

| Param | Tác dụng |
| --- | --- |
| `noapp=1` | Bỏ qua mở app, đi thẳng store |
| `env=sit` | Dùng package `vn.com.bidv.bidvhome.sit` cho `intent://` (mặc định: prod) |

Ví dụ `https://<domain>/?campaign=qr01&ref=hn` → app nhận `?campaign=qr01&ref=hn`.
Path cũng được giữ: `https://<domain>/promo/123?x=1` → app nhận `/promo/123?x=1`.

## Deploy (chọn 1)

### Cloudflare Pages
1. Push repo lên GitHub.
2. Cloudflare Dashboard → Workers & Pages → Create → Pages → kết nối repo.
3. Build command: để trống. Output/Root directory: `download-redirect`.
4. Deploy → nhận URL cố định, ví dụ `https://bidvhome-dl.pages.dev`.

### Netlify
```bash
npm i -g netlify-cli
netlify deploy --dir=download-redirect --prod
```

### Vercel
```bash
npm i -g vercel
vercel deploy download-redirect --prod
```

### GitHub Pages
Đặt `index.html` vào branch/thư mục Pages đang trỏ tới (vd `/docs`), bật Pages trong Settings.

## Sau khi deploy

1. Lấy URL cố định (vd `https://bidvhome-dl.pages.dev`).
2. Tạo QR code encode đúng URL đó (bất kỳ trình tạo QR nào).
3. Thay ảnh `src/assets/images/QR-test.png` bằng QR mới.

QR **không cần in/tạo lại** về sau: nếu URL store đổi hoặc muốn trỏ về web production, chỉ sửa `index.html` rồi deploy lại — nội dung QR giữ nguyên.

## Đồng bộ URL store

Các URL store trong `index.html` phải khớp với
`src/app/pages/utilities/constants/download-app.const.ts` (`URL_APP.IOS`, `URL_APP.ANDROID`).
Khi đổi một bên, nhớ đổi bên còn lại.

## Việc còn phải làm (phía app / hạ tầng)

- [ ] **SHA256 fingerprint cho package prod** trong `.well-known/assetlinks.json` đang tạm dùng lại fingerprint của bản `.sit`. Lấy fingerprint thật ở Play Console → *App integrity → App signing key certificate* và thay vào entry `vn.com.bidv.bidvhome`.
- [ ] **Custom URL scheme iOS**: điền `CONFIG.IOS_APP_URL_BASE` trong `index.html` (hiện để trống).
- [ ] **App Android** phải khai `intent-filter` cho `https://<domain>` với `android:autoVerify="true"`.
- [ ] **App iOS** phải bật capability *Associated Domains* với `applinks:<domain>`.

## Kiểm tra sau khi deploy

```bash
curl -I https://<domain>/.well-known/assetlinks.json                # 200 + application/json
curl -I https://<domain>/.well-known/apple-app-site-association     # 200 + application/json
```

- Android App Links: `adb shell pm get-app-links vn.com.bidv.bidvhome` → phải thấy `verified`.
- Universal Link iOS: kiểm tra tại https://app-site-association.cdn-apple.com/a/v1/&lt;domain&gt;
- Google verify: https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://&lt;domain&gt;&amp;relation=delegate_permission/common.handle_all_urls
