# Bài tập: Tiến trình chạy nền với nohup, ps, kill

## Mục tiêu

Chạy một script giám sát ở chế độ nền, đảm bảo nó vẫn tiếp tục chạy sau khi đóng phiên SSH, sau đó tìm và dừng tiến trình một cách an toàn.

**Kịch bản:** Một shell script giám sát ghi thời gian hệ thống vào file log định kỳ. Script phải chạy liên tục kể cả khi quản trị viên đã thoát SSH, và có thể được dừng an toàn khi cần.

## Script `loop-monitor.sh`

```bash
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
```

- `while true; do ... done`: vòng lặp vô hạn, script chạy mãi cho đến khi bị dừng bằng tín hiệu.
- `sleep 5`: tạm dừng 5 giây giữa mỗi lần ghi, tránh chiếm CPU.
- `>> /tmp/monitor.log`: **ghi nối** (append) một dòng thời gian vào cuối file log `/tmp/monitor.log`, không ghi đè nội dung cũ.

## Các lệnh đã thực hiện

1. Cấp quyền thực thi cho script:

```bash
chmod +x loop-monitor.sh
```

2. Chạy script ở chế độ nền và không bị dừng khi thoát SSH:

```bash
nohup ./loop-monitor.sh > /dev/null 2>&1 &
```

- `nohup`: cho tiến trình bỏ qua tín hiệu **SIGHUP** (hangup) được gửi khi phiên SSH/terminal đóng, nên tiến trình không bị dừng theo.
- `&`: chạy tiến trình ở chế độ nền, trả lại dấu nhắc lệnh ngay.
- `> /dev/null 2>&1`: bỏ đi toàn bộ output chuẩn (stdout) và output lỗi (stderr), không tạo file `nohup.out` (script đã tự ghi vào log riêng).

3. Lấy PID của tiến trình (`-f` so khớp với toàn bộ dòng lệnh):

```bash
pgrep -f loop-monitor.sh
```

4. Xem chi tiết tiến trình (user, PID, CPU, RAM, thời điểm bắt đầu, lệnh):

```bash
ps aux | grep loop-monitor.sh
```

5. Xem 10 dòng cuối của file log để xác nhận script vẫn đang ghi:

```bash
tail -n 10 /tmp/monitor.log
```

6. Gửi tín hiệu SIGTERM để dừng tiến trình một cách an toàn:

```bash
kill -15 <PID>
```

## Giải thích tín hiệu

| Tín hiệu | Số | Ý nghĩa |
|----------|----|---------|
| `SIGTERM` | 15 | **Yêu cầu** tiến trình kết thúc. Tiến trình có thể bắt tín hiệu này để dọn dẹp (đóng file, xóa file tạm…) rồi thoát một cách "êm". Đây là tín hiệu mặc định của `kill`. |
| `SIGKILL` | 9 | Kernel **giết ngay lập tức**. Tiến trình không thể bắt, chặn hay bỏ qua, nên không có cơ hội dọn dẹp. |

Vì vậy luôn thử `kill -15` trước. Chỉ dùng `kill -9` khi tiến trình không chịu thoát sau SIGTERM.

## Kết quả

Nội dung bằng chứng (output của `pgrep`, `ps`, `tail` log, lệnh `kill -15` và `ps` sau khi kill):

```text
# pgrep -f loop-monitor.sh (after reconnecting SSH)
# ps aux | grep loop-monitor.sh
# tail -n 10 /tmp/monitor.log
System time: Wed Oct  7 04:38:50 AM +07 2026
System time: Wed Oct  7 04:38:54 AM +07 2026
System time: Wed Oct  7 04:38:55 AM +07 2026
System time: Wed Oct  7 04:38:59 AM +07 2026
System time: Wed Oct  7 04:39:00 AM +07 2026
System time: Wed Oct  7 04:39:04 AM +07 2026
System time: Wed Oct  7 04:39:05 AM +07 2026
System time: Wed Oct  7 04:39:09 AM +07 2026
System time: Wed Oct  7 04:39:10 AM +07 2026
System time: Wed Oct  7 04:39:15 AM +07 2026
# kill -15 
# ps aux | grep loop-monitor.sh (after kill)
(khong con tien trinh - da tat)
```

**Nhận xét:**

- File log `/tmp/monitor.log` vẫn có các dòng thời gian mới được ghi sau khi kết nối lại SSH. Điều này chứng tỏ tiến trình chạy bằng `nohup` **không bị dừng** khi phiên SSH đóng.
- Lưu ý về bằng chứng:
  - Phần output của `pgrep` và `ps aux` trống (không ghi lại được PID).
  - Lệnh `kill -15` không kèm PID.
  - Log có hai dòng ghi gần nhau trong mỗi chu kỳ 5 giây, nhiều khả năng đang có hai bản script chạy song song.
  - Để xác nhận đầy đủ, cần chạy lại `pgrep -af loop-monitor.sh`, `kill -15 <PID>` với PID thực tế, và kiểm tra `ps aux | grep [l]oop-monitor.sh` không còn kết quả.
