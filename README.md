# download-redirect

Trang tĩnh **redirect tải app theo OS** cho QR code.

- iOS (iPhone/iPad) → App Store
- Android → CH Play
- Desktop / không nhận diện được → hiện trang chọn thủ công (2 nút store)

Độc lập hoàn toàn với web production Angular — chỉ là 1 file `index.html`, deploy được lên bất kỳ static host miễn phí nào. **QR code trỏ tới URL của trang này**, không phải trỏ tới web production.

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
