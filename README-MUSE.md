# Muse — Reynard fork mở thẳng Muse, ẩn toolbar

Gốc: https://github.com/minh-ton/reynard-browser (Reynard 0.20.1, trình duyệt Gecko cho iOS 13+)
Bản này: **Muse** — tên app `Muse`, Bundle ID `ai.muse.browser`, URL scheme `muse://`

## Đã đổi gì (12 file sửa + 1 workflow mới, xem `muse.patch`)

| Chức năng | Cách làm |
|---|---|
| Mở app vào thẳng Muse, giữ nguyên đoạn chat đang mở | `BrowserViewController.viewDidLoad`: sau `createInitialTab()` chỉ `browse(to: "https://muse.ai")` khi tab hoàn toàn trống (`(tab.url ?? "").isEmpty`). Tab khôi phục từ lần trước (đoạn chat muse.ai, hoặc link mở từ ngoài) được giữ nguyên, không bị ép về trang chủ. Tab mới cũng vậy: mặc định `NewTabSettings` đổi thành `.customURL` + `customNewTabURL = https://muse.ai` trong `BrowserPreferences.swift`. Màn hình mở đầu đổi từ `homepage` sang `lastTab` để không ép hiện trang chủ Reynard. |
| Ẩn thanh công cụ, chạm 3 ngón tay để hiện/ẩn | Mặc định thanh công cụ ẩn (`BrowserChrome.isAddressBarVisible = false`). Chạm 3 ngón tay vào màn hình thì hiện lên (chạm lại thì ẩn): `BrowserChrome.apply` dùng trạng thái toolbar chuẩn của Reynard khi hiện, `.hidden` khi ẩn; `BrowserViewController` neo nội dung web vào mép toolbar khi hiện, vào safe area khi ẩn; `isToolbarEnabled` báo cho Gecko biết để chừa đúng chỗ. |
| Nút quay lại/tiến chỉ hoạt động khi thanh công cụ đang hiện | `BrowserViewController.updateNavigationButtons()`: khi thanh công cụ đang hiện, `canGoBack`/`canGoForward` theo lịch sử thật của tab, `backPreviewImage` lấy `previewImages.backImage`; khi ẩn thì cả hai tắt như trước. Cử chỉ vuốt quay lại vẫn tắt hẳn. |
| Chia sẻ "Open in Muse" | `OpenInViewController.swift`: tiện ích chia sẻ mở app bằng `muse://open?url=...` (trước đây còn sót `reynard://open`, không mở được app). `SceneDelegate.resolvedBrowserURL` đã xử lý sẵn scheme `muse://open?url=`. |
| Gọn file .tipa | `tools/release/create-ipa.sh`: nén bằng `zip -r -9 -D` (nén tối đa, không ghi thư mục rỗng) thay cho `zip -r` — giảm ~800 KB. Chỉ đổi vỏ zip, gói app đã ký bên trong không động vào nên chữ ký vẫn nguyên. |
| Đổi tên / Bundle ID | `APP_DISPLAY_NAME = Muse` (`Reynard.xcconfig`); `ai.muse.browser`, `.Helper`, `.OpenIn` trong `project.pbxproj`, `Info.plist`, entitlements, `create-ipa.sh`, `OpenInViewController.swift`; scheme `reynard://` → `muse://` (`SceneDelegate.swift`, `Info.plist`). |
| Không tự cập nhật về Reynard | `BrowserUpdates.fetchUpdates()` tắt, file sideload đổi thành `Muse.ipa`. |
| Build cho fork | `.github/workflows/build.yml`: bỏ điều kiện `repository_owner == 'minh-ton'`, tên artifact đổi thành `Muse.ipa` / `Muse-TrollStore.tipa` / `Muse-Jailbroken.ipa`. Thêm `.github/workflows/build-muse.yml`: chỉ build bản TrollStore. |

## Cách build ra file .tipa (không build được trên máy Linux này)

Reynard dùng engine Gecko, phải build bằng Xcode trên macOS, tải Gecko, build Rust — máy Linux của Muse không có Xcode nên **không biên dịch trực tiếp được ở đây**. Cách đúng là build trên GitHub Actions (macOS 26, Xcode 26.6, theo workflow gốc):

1. Tạo repo GitHub mới (ví dụ `muse-browser`), push toàn bộ thư mục `muse-browser/` lên, kèm submodule:
   ```bash
   git submodule update --init --recursive
   git add -A && git commit -m "Muse fork" && git push
   ```
   Workflow sẽ tự `git submodule update` cho `engine/firefox` và `support/idevice` nên không cần push Gecko lên.
2. Vào repo → **Actions** → **Build Muse (TrollStore)** → **Run workflow**.
3. Chờ build xong, tải artifact **Muse-TrollStore.tipa** về.
   - Lần đầu build Gecko rất lâu (có thể vài giờ, cache sccache / Gecko deps giúp các lần sau nhanh hơn). Nếu job fail, xem log ở bước `Build Gecko` / `Build app` trước, đừng chạy lại ngay.
   - Không cần tài khoản Apple Developer cho bản TrollStore: workflow gốc build `--no-signing` rồi fake-sign bằng `ldid`.

## Cài lên iPhone iOS 17.0

Theo README gốc, bản TrollStore dùng cho **iOS 14 – 16.6.1 và 17.0** — đúng máy iOS 17.0 bạn đang tìm. Cài file `Muse-TrollStore.tipa` bằng TrollStore trên máy đó. Bản thường `Muse.ipa` chỉ dùng cho AltStore/SideStore trên iOS 17.4+.

## Lưu ý / chưa kiểm chứng

- Code Swift chỉ kiểm tra bằng mắt + cân bằng ngoặc, **chưa biên dịch bằng Xcode** — lỗi biên dịch (nếu có) sẽ hiện ở lần chạy Actions đầu tiên.
- Ẩn toolbar là ẩn hẳn cả thanh trên + thanh dưới và nút địa chỉ, mặc định ẩn. Muốn hiện lại thì chạm 3 ngón tay vào màn hình (chạm lại để ẩn); khi thanh công cụ đang hiện thì nút quay lại/tiến hoạt động bình thường.
- Đăng nhập Muse trong app vẫn qua web `muse.ai` như bình thường; Muse chỉ là vỏ trình duyệt, không lưu tài khoản riêng.
