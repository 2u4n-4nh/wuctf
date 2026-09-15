# Write up: [RE] Constellation

**Difficulty:** Hard

**Solved:** 39

## Thông tin của đề bài

`main.exe` là một công cụ đóng gói có giao diện đồ họa giúp gộp toàn bộ nội dung của một thư mục thành một tệp tin duy nhất. Tệp đầu ra mặc định sẽ lặp lại tên gốc làm phần mở rộng; chẳng hạn, tệp `test` sẽ trở thành `test.test`, và nếu tiếp tục đóng gói lần nữa, nó sẽ thành `test.test.test`. Tệp `flag.flag` được cung cấp là một phiên bản đã bị cố ý làm hỏng của một tệp tin đóng gói hợp lệ. Đối tượng được khôi phục là một tệp lưu trữ dạng thư mục; tệp `flag.py` sau khi được khôi phục sẽ thực hiện tính toán để tìm ra flag.

## Phân tích thông tin

Do file `main.exe` là một công cụ thực thi có thể đóng gói toàn bộ file đầu vào rồi trả về file được nén dưới dạng "tên gốc" sẽ được lặp lại làm phần mở rộng, và file `flag.flag` là một file py đã được nén thông qua file `main.exe`. Nhưng mấu chốt của file này là trước khi trả về thì thông tin của file được nén vẫn sẽ ở đoạn cuối của file `main.exe` với độ dài tương ứng của file đó. Do đó, khi phân tích bài này thì phải phân tích từ dưới lên lấy thông tin file, chạy ngược lại file `main.exe` để giải nén file `flag.flag` ra file `flag.py`.

Nhưng hàm thuật toán lại không được sắp xếp tuần tự mà được nằm rải rác trong các nút khác nhau ở các lệnh gọi máy chủ, không tuần tự từ trên xuống. Luồng thực thi k chạy thẳng mà chạy theo cách lộn xộn. Mã nguồn gốc của chương trình `main.exe` chỉ đảm nhận việc I/O.

## Cấu trúc máy ảo VM bên trong file main.exe


