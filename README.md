# Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

## 1. Bối cảnh
- Nhánh `main` có thêm 2 commit mới sửa `config.json` (đổi port, thêm env).
- Nhánh `feature-api` cũng sửa cùng các dòng trong `config.json` qua 2 commit.
- Khi rebase `feature-api` lên `main`, xung đột phát sinh **ở từng chặng** (từng commit được replay).

## 2. Mục tiêu
Đưa các commit của `feature-api` lên **nối tiếp ngay sau** commit mới nhất của `main`,
tạo lịch sử **thẳng tắp, không có merge commit**.

## 3. Thiết lập ban đầu

Trên main:
```bash
git checkout main
printf '{\n  "port": 8080,\n  "debug": false\n}\n' > config.json
git add config.json
git commit -m "init config"
```

Tạo nhánh feature-api và sửa 2 lần:
```bash
git checkout -b feature-api
printf '{\n  "port": 9000,\n  "debug": false\n}\n' > config.json
git add config.json
git commit -m "feat: change port"
printf '{\n  "port": 9000,\n  "debug": true\n}\n' > config.json
git add config.json
git commit -m "feat: enable debug"
```

Quay lại main, sửa 2 lần (gây lệch với feature-api):
```bash
git checkout main
printf '{\n  "port": 8081,\n  "debug": false\n}\n' > config.json
git add config.json
git commit -m "update port on main"
printf '{\n  "port": 8081,\n  "debug": false,\n  "env": "production"\n}\n' > config.json
git add config.json
git commit -m "add env config"
```

## 4. Rebase và giải quyết xung đột từng chặng

```bash
git checkout feature-api
git rebase main
```

### Chặng 1 — replay commit "feat: change port"
Git báo:
```
CONFLICT (content): Merge conflict in config.json
error: could not apply xxxxxxx... feat: change port
```
`git status` lúc này báo đang ở trạng thái `rebase in progress; interactive`.

Mở `config.json`, thấy các ký hiệu `<<<<<<<`, `=======`, `>>>>>>>`. Xử lý thủ công:
giữ port của feature (9000) nhưng không làm mất env của main. Nội dung đúng:
```json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}
```
Rồi tiếp tục:
```bash
git add config.json
git rebase --continue
```

### Chặng 2 — replay commit "feat: enable debug"
Nếu phát sinh conflict tiếp, mở `config.json` sửa lại cho `debug` thành `true` và
giữ nguyên các dòng khác. Nội dung cuối:
```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```
Rồi:
```bash
git add config.json
git rebase --continue
```
(Nếu chặng này không conflict, Git tự áp dụng và rebase hoàn tất luôn.)

## 5. Kiểm tra kết quả

```bash
git status
git log --graph --oneline
```

Kết quả mong đợi — lịch sử **thẳng tắp, KHÔNG có merge commit**, các commit feature-api
nằm nối tiếp ngay sau commit mới nhất của main:
```
* <hash> feat: enable debug
* <hash> feat: change port
* <hash> add env config
* <hash> update port on main
* <hash> init config
```

## 6. So sánh Merge vs Rebase khi giải quyết xung đột

| | Merge | Rebase |
|---|-------|--------|
| Xung đột | Giải quyết **một lần** tại thời điểm gộp | Giải quyết **từng chặng**, mỗi commit replay có thể conflict riêng |
| Lịch sử | Có thêm **merge commit** (rẽ nhánh) | **Thẳng tắp**, không merge commit |
| Lệnh tiếp tục | `git commit` sau khi resolve | `git add` + `git rebase --continue` cho mỗi chặng |
| Hủy giữa chừng | `git merge --abort` | `git rebase --abort` |
| Dùng khi | Muốn giữ nguyên dấu vết lịch sử nhánh | Muốn lịch sử gọn, tuyến tính trước khi merge/push |

> Lưu ý: chỉ rebase nhánh **chưa chia sẻ công khai**. Không rebase nhánh người khác đang dùng vì nó viết lại lịch sử.

## 7. Ảnh chụp `git log --graph --oneline`

