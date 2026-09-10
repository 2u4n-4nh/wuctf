# WRITE UP: [RE] Flag Pointer Register

**Point:** 59

## Tổng quan

<img width="964" height="622" alt="image" src="https://github.com/user-attachments/assets/5e1d450c-6ff0-485f-aac6-d8f264f6ca3e" />

Có thể tìm challenge [ở đây](https://nnsc.tf/challenges?challenge=rev_Flag+Pointer+Register)

---

## Thu thập thông tin

Từ phần mô tả của đề bài, ta biết được rằng file này được cung cấp thẳng flag nhưng con trỏ lại trỏ đến sai vị trí và đưa ra thông báo sai.
Theo gợi ý từ đề bài là sử dụng debugger để có thể quan sát giá trị trả về trong thanh ghi RAX, sau đó kiểm tra cách các thanh ghi RCX, RDX, R8 và R9 được thiết lập trước khi thực hiện lời gọi hàm xuất dữ liệu.

## Phân tích động

Tiến hành phân tích động của file trên IDA pro, ta đặt breakpoint ở lệnh `call rbx; WriteFile` tương đoạn in ra chuỗi `Press ENTER to receive the flag.`.
Ta để chương trình tiếp tục chạy và theo dõi thanh ghi RAX.
Khi lệnh chạy liên tục và trước khi hàm `sub_140001000` được gọi, ta thực hiện theo đúng chuỗi trên là bấm enter nếu k bấm thì sẽ bị lỗi, k gọi được file.
Sau khi hàm `sub_140001000` được gọi thì thanh ghi RAX sẽ nhận được 1 chuỗi là `NNS{r4x_h4d_7h3_fl4g_bu7_rdx_p01n73d_70_7h3_wr0ng_buff3r}`.

## Phân tích tĩnh

Với bài này thì ta có thể phân tích tĩnh ra thẳng flag do hàm `start` chỉ gọi đến đúng 1 hàm khi đọc file là hàm `sub_140001000`.
Truy cập vào hàm `sub_140001000`:

```C
{
  __int64 i; // rax

  for ( i = 0; i != 57; ++i )
    byte_140003000[i] ^= byte_140002000[i & 7];
  return byte_140003000;
}
```

Ta thấy duy nhất có 1 vòng for để thực hiện phép XOR, ta truy cập lấy dữ liệu và thực hiện phép XOR với các dữ liệu bên tròn 2 mảng byte

```Python
byte_140003000 = [
    0x7F, 0x3C, 0xFA, 0x37, 0x91, 0x22, 0xFD, 0x04, 
    0x59, 0x46, 0xCD, 0x13, 0xD4, 0x7E, 0xB6, 0x04, 
    0x57, 0x1E, 0x9D, 0x2B, 0xBC, 0x74, 0xF0, 0x6C, 
    0x6E, 0x00, 0xCD, 0x34, 0xBC, 0x66, 0xB5, 0x6A, 
    0x5F, 0x45, 0x9A, 0x28, 0xBC, 0x21, 0xB5, 0x04, 
    0x06, 0x1A, 0x9A, 0x13, 0x94, 0x64, 0xB5, 0x35, 
    0x56,0x2D, 0xCB, 0x39, 0x85, 0x70, 0xB6, 0x29, 0x4C]
byte_140002000 = [0x31, 0x72, 0xA9, 0x4C, 0xE3, 0x16, 0x85, 0x5B]
flag = ""

for i in range(57):
    decrypted_byte = byte_140003000[i] ^ byte_140002000[i & 7]
    flag += chr(decrypted_byte)

print("Flag giải mã được là:")
print(flag)
```

Sau khi chạy đoạn code trên thì ta nhận được flag là `NNS{r4x_h4d_7h3_fl4g_bu7_rdx_p01n73d_70_7h3_wr0ng_buff3r}`

---

Chuỗi `NNS{r4x_h4d_7h3_fl4g_bu7_rdx_p01n73d_70_7h3_wr0ng_buff3r}` chính là flag của bài này
