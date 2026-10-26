# cpp-shorts 🐡

CLI tự động làm **YouTube Shorts dạy C++** với nhân vật cá nóc. Bạn đưa chủ đề (hoặc lấy bài kế tiếp trong giáo trình), nó tự viết script, **compile code thật**, tạo giọng đọc, render video, rồi **dừng lại chờ bạn duyệt** trước khi đăng.

## Có gì trong repo

```
README.md                 <- file này (dành cho bạn)
AGENTS.md  GEMINI.md      <- hướng dẫn cho CLI (Antigravity/OpenCode) đọc đầu tiên
docs/SPEC.md              <- đặc tả đầy đủ: pipeline, layout, review gate, các phase
docs/PROGRESS.md          <- CLI cập nhật tiến độ ở đây
mascot/pufferfish.svg     <- con cá nóc (các lớp có id)
mascot/pufferfish-rig-reference.html  <- bản chạy được, mở bằng trình duyệt để xem
prompts/script_system.md  <- system prompt cho Gemini viết script
curriculum.yaml           <- giáo trình 80 bài
pronunciations.yaml       <- sửa cách TTS đọc thuật ngữ
.env.example              <- mẫu cấu hình (copy thành .env)
```

## Kế hoạch

| Phase | CLI làm | Bạn kiểm tra |
|---|---|---|
| 1 | Khung project + port con cá nóc sang Remotion | Trong `npm run studio` cá giống bản demo: 5 pose, chớp mắt, phồng khi "wow" |
| 2 | Bài 1: script, compile code thật, giọng đọc | Script hợp lý, output thật, nghe giọng đọc |
| 3 | Render đủ: code gõ, hình minh họa, cảnh dưới nước, phụ đề bong bóng, thumbnail | Xem `draft.mp4` **trên điện thoại**, chữ không bị UI YouTube che |
| 4 | `approve` / `reject` / `next` / `batch` | Chạy `batch 3`, duyệt 1, từ chối 1 kèm ghi chú |
| 5 | (chỉ khi bạn yêu cầu) lịch đăng, upload, clip AI | |

Mỗi phase xong CLI sẽ **dừng chờ bạn nói "go"**.

## 1. Cài đặt trên Windows (làm 1 lần)

Mở **PowerShell** và chạy từng dòng (nếu `winget` không tìm thấy gói, thử `winget search <tên>`):

```powershell
winget install Git.Git
winget install OpenJS.NodeJS.LTS
winget install Python.Python.3.12
winget install Gyan.FFmpeg
winget install MSYS2.MSYS2
```

Cài trình biên dịch g++ (để compile code C++ thật trong video):

1. Mở **MSYS2 UCRT64** từ Start Menu, chạy: `pacman -S --needed mingw-w64-ucrt-x86_64-gcc`
2. Thêm `C:\msys64\ucrt64\bin` vào biến môi trường **PATH** (Start → "Edit the system environment variables" → Environment Variables → Path → New).

**Đóng hẳn PowerShell rồi mở lại**, kiểm tra:

```powershell
git --version; node -v; python --version; ffmpeg -version; g++ --version
```

Cả 5 lệnh phải ra phiên bản. Nếu activate venv sau này báo lỗi script, chạy một lần:
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

**API key:** vào Google AI Studio lấy API key (khác với gói Gemini Pro trong app chat). Kiểm tra free tier và model nào key của bạn dùng được.

## 2. Đưa lên GitHub

Tạo repo **private** tên `cpp-shorts` trên github.com (không tick thêm README). Giải nén zip, rồi trong thư mục đó:

```powershell
git init
git add .
git commit -m "init cpp-shorts"
git branch -M main
git remote add origin https://github.com/<tên-bạn>/cpp-shorts.git
git push -u origin main
```

Hoặc kéo thả cả thư mục lên trang repo (Add file → Upload files). Trước khi push, chắc chắn **không có file `.env`** (đã nằm trong `.gitignore`).

## 3. Cho CLI làm

```powershell
git clone https://github.com/<tên-bạn>/cpp-shorts.git
cd cpp-shorts
copy .env.example .env
notepad .env        # điền GEMINI_API_KEY, tự điền, đừng dán key vào chat
```

Mở Antigravity hoặc OpenCode trong thư mục này và nói:

```
Read AGENTS.md and docs/SPEC.md, then start Phase 1. Stop after each phase and wait for my go.
```

(Nếu CLI không tự đọc `AGENTS.md`, câu trên đã bảo nó đọc rồi.)

Nếu CLI cố làm hết một lần: "Stop. Follow the phases in docs/SPEC.md."

## 4. Sau khi xong, quy trình hằng ngày

```powershell
.venv\Scripts\Activate.ps1
python main.py next                      # hoặc: python main.py make "chủ đề bất kỳ"
python main.py review <slug>             # xem video + checklist
python main.py approve <slug>            # ổn -> vào thư mục ready/
python main.py reject <slug> --note "ví dụ khó hiểu, đổi ví dụ khác"
python main.py batch 7                   # cuối tuần làm 7 video, duyệt từng cái
```

Video đã `approve` nằm ở `ready/`. Bạn upload tay hoặc hẹn giờ trong YouTube Studio.

## 5. Dặn dò

- **Tự duyệt phần giải thích**: code compile thật nên không sai cú pháp, nhưng lời giải thích vẫn có thể sai ý.
- **Nghe giọng đọc**: sửa cách đọc thuật ngữ trong `pronunciations.yaml`.
- **Xem trên điện thoại** trước khi duyệt.
- **Short tối đa 3 phút**; kit nhắm 45-90 giây. Muốn video dài thì gom nhiều short lại.
- **Kiếm tiền**: YouTube đánh giá thấp nội dung hàng loạt, giống nhau. Giữ ví dụ, giọng, nhân vật riêng của bạn. Đừng kỳ vọng thu nhập sớm; mục tiêu đầu là học và có portfolio.
- **Upload tự động để sau**: API project chưa audit thì video bị khóa private.
- **Nhạc nền** (nếu thêm): dùng nhạc free-license (YouTube Audio Library).
- **Commit thường xuyên**, không đẩy `.env` hay file media lên GitHub.
- **Remotion** miễn phí cho cá nhân và công ty nhỏ; đọc lại license trên trang của họ cho chắc.
