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
| `env=prod\|uat\|sit` | Ghi đè môi trường tự nhận từ path (xem mục dưới). Giá trị lạ bị bỏ qua |

Ví dụ `https://<domain>/E9Y8/?campaign=qr01&ref=hn` → app nhận `?campaign=qr01&ref=hn`.
Path cũng được giữ: `https://<domain>/E9Y8/promo/123?x=1` → app nhận `/promo/123?x=1`.

## Ba môi trường

Môi trường được nhận từ **path prefix trên URL** — chính là path mà app khai trong AASA, nên không cần thêm param:

| Env | Path prefix | Android package | iOS appID |
| --- | --- | --- | --- |
| `prod` | `/E9Y8` | `vn.com.bidv.bidvhome` | `3GRX94WRGL.vn.com.bidv.bidvHomes` |
| `uat` | `/SyXu` | `vn.com.bidv.bidvhome.uat` | `MD4U8MPEK7.vn.com.bidv.bidvHomes.uat` |
| `sit` | `/Ga5k` | `vn.com.bidv.bidvhome.sit` | `MD4U8MPEK7.vn.com.bidv.bidvHomes.sit` |

Thứ tự ưu tiên: `?env=` → path prefix → `CONFIG.DEFAULT_ENV` (`prod`).

Prefix bị **cắt khỏi path truyền cho app** vì nó là định danh môi trường, không phải route: `/E9Y8/promo/123` → app nhận `/promo/123`.

AASA khai `/Ga5k` cho **cả sit lẫn uat**. Code gán `/Ga5k` → sit và dùng `/SyXu` (riêng của uat) cho uat. Muốn mở app uat từ URL `/Ga5k/…` thì thêm `?env=uat`.

Đổi cấu hình ở `CONFIG.ENV` trong `index.html`. Mỗi env có thêm khoá tuỳ chọn:

| Khoá | Không khai thì |
| --- | --- |
| `scheme` | dùng `CONFIG.APP_SCHEME` chung |
| `schemeHost` | dùng `CONFIG.APP_SCHEME_HOST` (`open`) |
| `iosStore` / `androidStore` | dùng store mặc định (prod) |

Nên đặt `scheme` **khác nhau cho từng env** — custom scheme không xác thực chủ sở hữu, máy cài 2 bản mà chung scheme thì OS mở bản nào là tuỳ hên xui.

## Deploy

### Netlify (khuyến nghị)

Netlify xử lý được cả 3 thứ mà GitHub Pages không làm được: phục vụ `.well-known/`, set `Content-Type` cho AASA, và rewrite path prefix ở mọi độ sâu.

#### 1. Tạo site từ Git

1. [app.netlify.com](https://app.netlify.com) → **Add new site** → **Import an existing project** → **GitHub** → chọn repo `qr-home`.
2. Branch: `main`. **Build command và Publish directory để trống** — `netlify.toml` trong repo đã khai sẵn, và nó luôn thắng cấu hình trên giao diện.
3. **Deploy site**.

Xong bước này có URL tạm dạng `https://<tên-ngẫu-nhiên>.netlify.app`. Đổi tên ở *Site configuration → Site details → Change site name*.

Từ đây mỗi lần push lên `main` là Netlify tự deploy lại.

> Dùng Git chứ đừng dùng `netlify deploy` từ máy local: publish directory là `.`, nên CLI sẽ upload cả `node_modules/`. Bản deploy từ Git thì clone sạch theo `.gitignore`.

#### 2. Gắn custom domain

Bước bắt buộc — deeplink không chạy trên `*.netlify.app` vì app không khai domain đó.

*Site configuration → Domain management → Add a domain* → nhập domain (vd `qr.bidvhome.vn`).

Rồi trỏ DNS ở nhà cung cấp domain:

| Loại domain | Bản ghi |
| --- | --- |
| Subdomain (`qr.bidvhome.vn`) | `CNAME` → `<tên-site>.netlify.app` |
| Apex (`bidvhome.vn`) | `A` → `75.2.60.5`, hoặc `ALIAS`/`ANAME` → `<tên-site>.netlify.app` |

Đợi Netlify cấp chứng chỉ Let's Encrypt (*Domain management → HTTPS*, thường vài phút). **Phải có HTTPS hợp lệ trước khi test Universal Link** — iOS từ chối AASA trên chứng chỉ sai hoặc self-signed.

Bật luôn *Force HTTPS*.

#### 3. `netlify.toml` làm gì

| Khai báo | Tác dụng |
| --- | --- |
| `publish = "."` | Phục vụ thẳng từ gốc repo, không cần build |
| 2 khối `[[headers]]` | `Content-Type: application/json` cho AASA (file không đuôi) và `assetlinks.json` |
| Rewrite `/apple-app-site-association` | iOS < 13 tìm ở root trước, rewrite 200 về `.well-known/` |
| Rewrite `/E9Y8/*`, `/SyXu/*`, `/Ga5k/*` | Cùng một `index.html` phục vụ ở cả 3 prefix, **mọi độ sâu** |

Rewrite phải là `status = 200`. Đổi thành 301 là URL trên thanh địa chỉ mất prefix, mà mất prefix là mất luôn điều kiện iOS chịu mở app.

Thêm môi trường mới thì thêm một khối `[[redirects]]` tương ứng, khớp với `CONFIG.ENV` trong `index.html`.

#### 4. Kiểm tra

```bash
D=qr.bidvhome.vn

# Phải 200 + application/json, KHÔNG được redirect
curl -sI "https://$D/.well-known/apple-app-site-association" | head -3
curl -sI "https://$D/.well-known/assetlinks.json" | head -3

# Rewrite prefix — cả nông lẫn sâu, đều phải 200 và trả về index.html
curl -sI "https://$D/E9Y8/" | head -1
curl -sI "https://$D/E9Y8/promo/123?x=1" | head -1

# Apple CDN đã lấy được AASA chưa (sau khi domain đã có HTTPS)
curl -s "https://app-site-association.cdn-apple.com/a/v1/$D"
```

Dòng `curl -sI` của AASA phải thấy `HTTP/2 200` và `content-type: application/json`. Thấy `301`/`302` là hỏng — iOS không đi theo redirect cho file này.

### GitHub Pages (đang dùng)

Settings → Pages → Source: **Deploy from a branch**, branch `main`, folder `/ (root)`.

Hai điều kiện bắt buộc để deeplink chạy được:

**1. `.nojekyll` ở root.** Đã có sẵn trong repo. Không có file này, Jekyll bỏ qua mọi thư mục bắt đầu bằng dấu chấm → `.well-known/` không được publish → App Links / Universal Links không bao giờ verify.

**2. Custom domain.** OS chỉ đọc file verification ở **root của domain**:

```
https://<domain>/.well-known/assetlinks.json
```

Project site mặc định (`https://<user>.github.io/qr-home/`) đặt file ở `/qr-home/.well-known/…` — sai chỗ, và `<user>.github.io/.well-known/…` thuộc repo khác. **Deeplink không thể hoạt động trên project site.** Cách xử lý:

| Cách | Kết quả |
| --- | --- |
| Thêm custom domain (file `CNAME` + DNS) | Site chạy ở root → deeplink OK ✅ |
| Đổi tên repo thành `<user>.github.io` | Site chạy ở root → deeplink OK ✅ |
| Giữ nguyên project site | Chỉ redirect ra store, **không mở được app** ❌ |

Nếu vẫn chạy ở subpath, set `CONFIG.BASE_PATH = '/qr-home'` trong `index.html` để path truyền sang app không dính tên thư mục.

**Hạn chế không khắc phục được trên GitHub Pages:** không set được `Content-Type` cho `.well-known/apple-app-site-association` (file không có đuôi → phục vụ dưới dạng `application/octet-stream`). iOS có thể từ chối. Kiểm tra bằng lệnh ở mục *Kiểm tra sau khi deploy*; nếu Apple CDN không trả về nội dung thì phải chuyển sang host set được header (Netlify / Cloudflare Pages).

Thêm một hạn chế nữa: GitHub Pages không có rewrite, nên `/E9Y8/promo/123` không khớp file nào. Workflow phải copy `index.html` thành `404.html` để hứng — trang vẫn chạy, nhưng mã trạng thái trả về là 404. Netlify không dính vì đã có rewrite thật.

## Sau khi deploy

1. Chốt URL cuối cùng, đúng path prefix của môi trường (vd `https://qr.bidvhome.vn/E9Y8/`).
2. Tạo QR code encode đúng URL đó (bất kỳ trình tạo QR nào).
3. Thay ảnh `src/assets/images/QR-test.png` bằng QR mới.

**Chốt domain trước khi in QR.** Đổi domain sau là phải in lại toàn bộ; đổi URL store hay logic điều hướng thì không, chỉ cần sửa `index.html` rồi deploy.

QR **không cần in/tạo lại** về sau: nếu URL store đổi hoặc muốn trỏ về web production, chỉ sửa `index.html` rồi deploy lại — nội dung QR giữ nguyên.

## Đồng bộ URL store

Các URL store trong `index.html` phải khớp với
`src/app/pages/utilities/constants/download-app.const.ts` (`URL_APP.IOS`, `URL_APP.ANDROID`).
Khi đổi một bên, nhớ đổi bên còn lại.

## Ràng buộc từ file verification

`.well-known/apple-app-site-association` (bản chính thức từ team mobile) khai `paths` cụ thể chứ không phải `"*"`, tức **giới hạn Universal Link theo path** (bảng đầy đủ ở mục *Ba môi trường*).

Nghĩa là trang này **phải nằm dưới path prefix tương ứng** thì iOS mới mở app. Đặt ở `/` hay `/qr-home/` sẽ không bao giờ kích hoạt, kể cả khi domain và AASA đều đúng.

→ URL cho QR bản prod phải có dạng `https://<domain>/E9Y8/<gì đó>?param=…`, tức file `index.html` phải deploy vào thư mục `E9Y8/` (và `SyXu/`, `Ga5k/` cho uat/sit).

Cấu trúc path prefix ngẫu nhiên 4 ký tự này là dấu hiệu của Firebase Dynamic Links (đã ngừng hoạt động từ 25/08/2025). Nếu đúng vậy, hỏi team mobile xem bản app hiện hành còn khai path nào khác không — nhiều khả năng cần build mới để khai domain/path của trang này.

## Việc còn phải làm (phía app / hạ tầng)

Chi tiết cấu hình phía app (entitlements iOS, `intent-filter` Android, lệnh test): [docs/app-setup.md](docs/app-setup.md) — gửi thẳng file này cho team mobile.

- [ ] **Chốt domain**: trang này phải chạy trên đúng domain mà app khai trong *Associated Domains* (`applinks:`) và `intent-filter`. Domain khác → Universal Link không bao giờ chạy.
- [ ] **Chốt path**: URL phải nằm dưới path prefix ở bảng trên.
- [ ] **Custom URL scheme**: điền `CONFIG.APP_SCHEME` trong `index.html` (hiện để trống). Đây là cách duy nhất mở app khi chưa có domain/path đúng.
- [ ] **App Android** phải khai `intent-filter` cho `https://<domain>` với `android:autoVerify="true"`.

## Kiểm tra sau khi deploy

```bash
curl -I https://<domain>/.well-known/assetlinks.json                # 200 + application/json
curl -I https://<domain>/.well-known/apple-app-site-association     # 200 + application/json
```

- Android App Links: `adb shell pm get-app-links vn.com.bidv.bidvhome` → phải thấy `verified`.
- Universal Link iOS: kiểm tra tại https://app-site-association.cdn-apple.com/a/v1/&lt;domain&gt;
- Google verify: https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://&lt;domain&gt;&amp;relation=delegate_permission/common.handle_all_urls
