# Minecraft SS Integrity Checker v2

Công cụ hỗ trợ **Screen Share (SS)** cho staff server Minecraft, giúp phát hiện dấu hiệu sử dụng cheat client (hack client, X-ray, injector...) trên máy người chơi. Viết bằng Python, giao diện `customtkinter`.

> ⚠️ Tool chỉ **hỗ trợ**, không thay thế việc SS quan sát trực tiếp màn hình người chơi. Kết quả cần được staff đối chiếu thủ công trước khi kết luận vi phạm.

---

## Tính năng chính

Tool chấm điểm rủi ro (`RISK score`) dựa trên **11 lớp phát hiện** độc lập, mỗi lớp nhắm vào một cách cheat client có thể ẩn giấu dấu vết:

| # | Lớp quét | Mục đích |
|---|----------|----------|
| 1 | **Quét file & thư mục `.minecraft`** | Tên file đáng ngờ, hash SHA-256 trùng danh sách đã biết |
| 2 | **Quét sâu bên trong `.jar`** | Signature package, module pattern (killaura, esp, xray...), `MANIFEST.MF`, `fabric.mod.json`/`mods.toml`, phát hiện obfuscation (class tên random) |
| 3 | **Chuỗi ký tự đặc trưng (string signature)** | Domain, Discord invite, watermark còn sót trong constant pool `.class` dù đã đổi tên |
| 4 | **Resourcepack X-Ray qua texture** | So sánh độ trong suốt (alpha) của texture đá để phát hiện resourcepack X-ray không cần mod |
| 5 | **Log khởi động & crash report** | Bắt mod đã từng load kể cả khi `.jar` đã bị xoá sau khi chơi |
| 6 | **Windows Prefetch** | Chương trình từng chạy dù `.exe` gốc đã bị xoá |
| 7 | **Recent Files (`.lnk`)** | Shortcut file/folder vừa mở gần đây liên quan cheat |
| 8 | **Zone.Identifier (ADS)** | Truy vết nguồn gốc file tải về từ Internet (domain cheat client) |
| 9 | **Windows Defender Exclusions** | Phát hiện việc tự loại trừ `.minecraft`/file cheat khỏi bị quét |
| 10 | **Recycle Bin** | File cheat đã bị xoá gần đây trước khi SS (parse `$I` metadata NTFS) |
| 11 | **UserAssist (Registry)** | Chương trình GUI từng chạy (ROT13-decoded), lưu lâu hơn Prefetch |
| + | **Quét tiến trình runtime** | Tiến trình đáng ngờ, dòng lệnh khởi chạy `java`, module/DLL lạ nạp vào, class đang load qua `jcmd`, và **đọc trực tiếp RAM tiến trình** để bắt ghost client không ghi file |

### Điểm nổi bật kỹ thuật

- **Chống false positive**: dùng full package-path signature (`/net/wurstclient/`) thay vì từ khoá đơn (`fly`, `esp`) để tránh match nhầm code hợp lệ (`butterfly`, `despawn`...). Có danh sách riêng cho từ khoá độ tin cậy thấp (vd `krypton`, `thunder`) chỉ để tham khảo, không tự tính là bằng chứng.
- **Bắt file đã đổi tên**: quét signature/string bên trong `.jar` thay vì chỉ dựa tên file.
- **Bắt "ghost client"**: đọc thẳng bộ nhớ tiến trình `java(w).exe` để tìm watermark cheat kể cả khi không ghi file/log/DLL ra đĩa.
- **Bắt hành vi "xoá trước khi SS"**: kết hợp Prefetch, Recycle Bin, UserAssist, Zone.Identifier để tìm dấu vết còn sót sau khi cheat đã bị xoá.
- **Trích dẫn điều luật server**: với mod bị cấm tường minh (`SERVER_BANNED_MODS`), tool in ra đúng điều luật vi phạm thay vì gộp chung "nghi cheat".
- **Whitelist mod hợp pháp**: publisher đáng tin cậy (Microsoft, Oracle), launcher bên thứ 3 phổ biến (TLauncher, PrismLauncher, MultiMC), thư mục hệ thống Windows.

### Thang điểm rủi ro

| Điểm | Mức |
|------|-----|
| ≥ 60 | 🔴 HIGH RISK — cần kiểm tra kỹ / SS sâu hơn |
| 25–59 | 🟠 MEDIUM RISK — có dấu hiệu đáng ngờ |
| < 25 | 🟢 CLEAN — không phát hiện dấu hiệu rõ ràng |

---

## Yêu cầu hệ thống

- **Windows** (bắt buộc cho các lớp quét sâu: Prefetch, Recycle Bin, UserAssist, Defender exclusions, memory scan, registry). Chạy trên OS khác vẫn hoạt động nhưng các lớp trên sẽ tự động báo "không khả dụng".
- **Python 3.9+**
- Nên chạy với quyền **Administrator** để:
  - Đọc đầy đủ `C:\Windows\Prefetch`
  - Đọc đầy đủ vùng nhớ tiến trình `java(w).exe`
  - Liệt kê module/DLL nạp vào tiến trình qua PowerShell
- Nên cài **JDK đầy đủ** (không chỉ JRE) để có lệnh `jcmd`, phục vụ quét class đang load trong JVM runtime.

### Thư viện Python

Bắt buộc:
```
pip install customtkinter
```

Tùy chọn (nếu thiếu, tool tự động bỏ qua lớp liên quan và báo rõ trong log):
```
pip install pillow    # quét X-ray qua texture resourcepack
pip install psutil    # quét tiến trình, cmdline, kết nối mạng, module DLL
```

`ctypes` / `winreg` (memory scan, UserAssist) đã có sẵn trong Python trên Windows, không cần cài thêm.

---

## Cách sử dụng

1. Cài các thư viện cần thiết ở trên.
2. Chạy tool trên máy người chơi trong lúc **Screen Share** (nên mở sẵn Minecraft để quét được tiến trình runtime và memory scan):
   ```
   python check.py
   ```
   (hoặc chạy bản `.exe` đã đóng gói qua PyInstaller, xem phần dưới)
3. Nhấn **START SCAN** — tool sẽ tự động quét lần lượt 11 lớp và hiển thị log trực tiếp, kèm điểm RISK cập nhật realtime.
4. Nhấn **EXPORT REPORT** để lưu log ra file `.txt` trong thư mục `ss_reports/` (đặt tên theo timestamp).

---

## Đóng gói thành `.exe` (PyInstaller)

```
pip install pyinstaller
pyinstaller --onefile --noconsole --name "SS_IntegrityChecker" check.py
```

File `.exe` sẽ nằm trong thư mục `dist/`. Do tool cần đọc registry, Prefetch, và bộ nhớ tiến trình, nên khuyến khích người dùng chạy `.exe` này **với quyền Administrator** ("Run as administrator") để có kết quả đầy đủ nhất.

---

## Giới hạn & lưu ý

- Tool chỉ hỗ trợ SS, **không thay thế** việc quan sát trực tiếp màn hình.
- Danh sách hash (`KNOWN_HASHES`) và signature cần được cập nhật định kỳ vì cheat client mới ra liên tục.
- Nếu thiếu quyền Admin hoặc thiếu `jcmd`, lớp quét runtime (module/class scan) sẽ báo "bỏ qua" thay vì báo sai — lúc đó tool chỉ còn dựa vào quét file, không bắt được ghost client thật sự.
- Kết quả từ các lớp độ tin cậy thấp (🟡 tham khảo) **không** được coi là bằng chứng vi phạm, chỉ để staff xem xét thêm.
- Windows giới hạn số lượng entry Prefetch (~1024 file gần nhất) nên chương trình chạy quá lâu trước đó có thể không còn dấu vết.

---

## Cấu trúc code (tham khảo nhanh)

```
check.py
├── CONFIG              # danh sách signature, keyword, whitelist
├── UI (customtkinter)  # cửa sổ chính, log box, nút Scan/Export
├── CORE HELPERS        # log(), add(), hash, extract_strings...
├── scan_jar_internals  # lớp 2 + 3
├── scan_resourcepack_xray / scan_resourcepacks   # lớp 4
├── scan_logs                                     # lớp 5
├── scan_prefetch                                 # lớp 6
├── scan_recent_files                             # lớp 7
├── scan_zone_identifier                          # lớp 8
├── scan_defender_exclusions                      # lớp 9
├── scan_recycle_bin                              # lớp 10
├── scan_userassist                                # lớp 11
├── scan_processes / scan_process_cmdline /
│   scan_loaded_modules / scan_loaded_classes /
│   scan_process_memory_strings                    # quét runtime + memory scan
├── scan_files          # điều phối quét file & jar
└── run_scan            # điều phối toàn bộ pipeline + tổng kết điểm
```
