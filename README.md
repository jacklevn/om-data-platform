# om-data-platform

Repo dùng chung giữa máy cá nhân và máy công ty.

## Trạng thái

Repo mới khởi tạo, chưa chọn tech stack.

## Làm việc trên 2 máy

Nguyên tắc: **mở máy ra thì `pull` trước, xong việc thì `push`.**

```bash
# Đầu buổi làm việc
git pull

# Cuối buổi làm việc
git add .
git commit -m "mô tả thay đổi"
git push
```

Lần đầu trên một máy mới:

```bash
git clone https://github.com/jacklevn/om-data-platform.git
```

## Lưu ý

Thư mục `data/` và các file `.env` không được đẩy lên git (xem `.gitignore`).
