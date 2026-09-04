# Excalibur — Session Handoff (Season 2)

> File này ghi lại toàn bộ bối cảnh, mục tiêu, trạng thái và việc cần làm để một phiên làm việc mới (context đã reset) có thể tiếp tục ngay mà không phải khám phá lại từ đầu.
>
> Ngày cập nhật: 2026-08-28. Workspace: `C:\excalibur`.

---

## 1. Dự án & mục tiêu (đích đến)

- Tên dự án: **excalibur** (app iOS tên hiển thị `kfun`, bundle id `com.34306.excalibur1`).
- Dùng **DarkSword exploit** (`kexploit_opa334`) để chiếm **kernel read/write** trên iOS, từ đó thực hiện các tính năng customize:
  - SpringBoard injection (inject tweak)
  - Bypass giới hạn 3 app (3-app limit)
  - Decrypt app → IPA
  - Bật JIT
  - Ghi đè file hệ thống
  - Custom MobileGestalt (Dynamic Island, hidden features…)
  - Memory finder / editor
- **Mục tiêu trước mắt (đang làm dở):** đọc/ghi bộ nhớ game **FreeFire (Unity)** mà **không có task port** (không có entitlement `task_for_pid`). Cụ thể: resolve được **base address của `UnityFramework`** trong tiến trình game để làm nền cho cheat/memory edit.

---

## 2. Môi trường / thiết bị mục tiêu

- iPhone 13 (`iPhone13,2`), chip **T8101 (A15)**, iOS **26.0.1 (23A355)**.
- Kernel: `Darwin Kernel Version 25.0.0`, xnu-12377.2.9, RELEASE_ARM64.
- PAC bật (arm64e).

---

## 3. Bố cục code hiện tại

- `darksword-kexploit-fun/` — app iOS (SwiftUI + Objective-C++ bridge).
  - `esp/FFBridge.mm` — cầu nối chính: phát hiện game, attach, resolve module base.
  - `kexploit/` — kernel exploit (`kexploit_opa334`, `krw`, `kutils`, `offsets`, `xpf`, `vnode`…).
  - `TaskRop/` — **RemoteCall**: chạy kernel-call trong ngữ cảnh tiến trình khác mà không cần task port.
  - `research/`, `utils/`, `kpf/` — tiện ích hỗ trợ.
- Build: `darksword-kexploit-fun.xcodeproj`. Dự án dùng **file-system-synchronized groups** → mọi file `.m`/`.mm` nằm trong thư mục `darksword-kexploit-fun` tự động được include vào target (không cần khai báo từng file trong `project.pbxproj`).

---

## 4. Lịch sử bug & cách sửa (rất quan trọng, đừng làm lại)

### Bug A — Panic reboot khi attach (ĐÃ SỬA)

- **Triệu chứng:** chạy exploit ở tab Home, lấy được pid khoảng ~1 giây thì **panic + reboot**.
- **Panic log (`bug_type 210`):** `Unexpected fault in kernel static region at pc 0xfffffff02631603c`, `esr=0x96000006` (translation fault level 0), task `pid ...: kfun`.
- **Nguyên nhân gốc:** code cũ đọc bộ nhớ game bằng **pmap walk thủ công** (`kread_remote` / `pmap_walk` / `kphys_read64`). App log: `[PMAP] walk l1 invalid: l0=0x897120003 l1=0` → decode bảng trang sai → đọc địa chỉ kernel unmapped → panic.
- **Cách sửa:** bỏ hoàn toàn pmap walk cho việc đọc bộ nhớ game, chuyển sang **RemoteCall** (`init_remote_call` + `remote_read`).
  - `kread_remote` / `pmap_walk` / `kphys_read64` vẫn còn trong `kexploit/kutils.m` nhưng **không còn được gọi** (code chết).
  - `pmap_read_resolve()` giờ chỉ quét task struct của chính mình để resolve offset, không walk bộ nhớ game.

### Bug B — GitHub Actions build fail (ĐÃ SỬA)

- **Triệu chứng:** workflow "Build unsigned IPA" fail ở step `Build without Apple code signing` (xcodebuild).
- **Nguyên nhân:** `TaskRop/RemoteCall.h` thiếu `extern "C"`. `FFBridge.mm` là Objective-C++ (`.mm`) → hiểu các hàm là C++ (name mangling); `RemoteCall.m` là Objective-C (`.m`) → định nghĩa C symbols → **link error Undefined symbols** (`init_remote_call`, `remote_read`, `destroy_remote_call`…).
- **Cách sửa:** bọc toàn bộ khai báo trong `RemoteCall.h` bằng `#ifdef __cplusplus extern "C" { ... }`.

---

## 5. Trạng thái hiện tại (đã có gì)

- Build unsigned IPA **pass** (commit `6f972ef`).
- Runtime log mới nhất (10:47 PM) cho thấy:
  - `[init_remote_call]` **hoạt động**: `Injected threads: 2`, `Inject EXC_GUARD ... OK` trên 2 thread, `firstExceptionPort=0x290b`, `secondExceptionPort=0x5513`, `process: FreeFire, pid: 1272`.
  - Kernel r/w READY, XPF finish, spray sockets OK.
  - **Không còn** `[PMAP] walk l1 invalid`, không còn panic.
- Log được paste **dừng ở `[EXPLOIT] starting kexploit_opa334...`** — chưa thấy kết quả cuối `[FF] attached pid=... UnityFramework base=0x...`.

---

## 6. Cần làm tiếp (next steps)

1. Chạy lại app và **lấy đầy đủ log sau** dòng `[EXPLOIT] starting kexploit_opa334...`, xác nhận:
   - `ff_attach_pid` trả về `UnityFramework base` (khác `0`).
   - Không panic sau khi attach.
2. Nếu `UnityFramework base=0` hoặc `init_remote_call failed`: debug RemoteCall —
   - `vm_map_remote_page()` trong `TaskRop/VM.m` (remap trang game qua vm_map);
   - PAC sign trong `sign_state()` (`TaskRop/RemoteCall.m`);
   - resolve `off_task_all_image_info_addr`.
3. Sau khi resolve được base: tiếp tục các TODO trong `README.md` (SpringBoard inject, bypass 3-app, decrypt app, JIT, MobileGestalt, memory editor).

---

## 7. Thông tin kỹ thuật đã resolve (dùng lại được)

- `allproc` head: `0xfffffff00aa5bf38`, offset từ kernel base: `0x3a57f38`.
- `off_task_all_image_info_addr` = `0x3d0` (resolve runtime bằng cách quét task struct tìm địa chỉ dyld `all_image_info`).
- `off_vm_map_pmap` = `0x40`.
- Kernel base (phiên log đó): `0xfffffff007004000` (slide thay đổi mỗi lần boot).

---

## 8. Build / CI

- Repo: `https://github.com/userlethekhoi/Free-Fire-DarksWord.git` (branch `main`).
- Workflow:
  - `.github/workflows/build-unsigned-ipa.yml` — **chạy tự động khi push lên `main`** (build unsigned, upload artifact + tạo Release `FreeFireDS.ipa`).
  - `.github/workflows/build-ipa.yml` — build **signed** IPA (manual `workflow_dispatch`, cần các secrets: `DEVELOPMENT_TEAM`, `IOS_CERTIFICATE`, `IOS_CERTIFICATE_PASSWORD`, `IOS_PROVISIONING_PROFILE`, `IOS_PROVISIONING_PROFILE_NAME`).
  - `.github/workflows/build.yml` — build unsigned, manual.
- IPA unsigned cần **re-sign** trước khi cài lên máy.
- **Lưu ý CI:** `gh` CLI chưa đăng nhập; không có token nên không tải được log Actions (API trả 403). Muốn lấy log lỗi build: `gh auth login` / set `GH_TOKEN`, hoặc paste log thủ công từ trang Actions.

---

## 9. Cách test nhanh

1. Re-sign IPA (hoặc build signed qua workflow).
2. Cài lên iPhone 13 / iOS 26.0.1.
3. Mở app → tab Home → chạy exploit.
4. Mở game FreeFire.
5. Đọc log console (app in log qua `printf`/`NSLog`).
