# Write up: [RE] Constellation

**Difficulty:** Hard

**Solved:** 39

## Thông tin của đề bài

`main.exe` là một công cụ đóng gói có giao diện đồ họa giúp gộp toàn bộ nội dung của một thư mục thành một tệp tin duy nhất. Tệp đầu ra mặc định sẽ lặp lại tên gốc làm phần mở rộng; chẳng hạn, tệp `test` sẽ trở thành `test.test`, và nếu tiếp tục đóng gói lần nữa, nó sẽ thành `test.test.test`. Tệp `flag.flag` được cung cấp là một phiên bản đã bị cố ý làm hỏng của một tệp tin đóng gói hợp lệ. Đối tượng được khôi phục là một tệp lưu trữ dạng thư mục; tệp `flag.py` sau khi được khôi phục sẽ thực hiện tính toán để tìm ra flag.

## Cấu trúc máy ảo VM bên trong file main.exe

Đoạn code thực thi máy ảo này được tác giả nén ở dạng file .zip và được để dưới cuối file. Do đó, khi giải bài này thì tác giả gợi ý rằng hãy dịch ngược từ dưới lên để kiểm tra và lấy thông tin của đoạn code mã hoá có khoảng cách đúng bằng kích thước file. 
Điểm làm cho bài này trở nên khó và cũng là quan trọng nhất trong bài này là chương trình này không chạy mã hoá theo tuần tự như bình thường mà mỗi đoạn mã hoá đều được đặt ở một điểm nút ngẫu nhiên nằm rải rác trong toàn bộ chương trình. Chính vì vậy mà khi phân tích thì ta phải để ý đến các thông số trên thanh ghi và các giá trị tức thời vào những lúc chương trình giao tiếp với hệ điều hành vì đoạn code này chạy theo lộ trình gần như là ngẫu nhiên để mã hoá.
Tác giả cũng gợi ý thêm là việc sử dụng các mốc I/O với máy chủ làm tham chiếu vì chúng sẽ rất hữu ích khi chúng ta theo dõi và phân tích các phép tính giữa chúng.
Mã nguồn gốc của chương trình này chỉ đảm nhận việc I/O và không lưu bất cứ bản sao riêng nào của các phép hoán vị và đa thức RS. Chính vì vậy mà khi phân tích thì ta có thể k cần quá để tâm đến mã nguồc gốc mà hãy coi nó như là các mốc để theo dõi thuật toán của chương trình.

**Đa thức RS:** là đa thức mã hoá Reed-Solomoon dùng để truyền dẫn dữ liệu và khôi phục dữ liệu bị mất hoặc do nhiễu.

## Cấu trúc file bị hỏng

Dùng các tool để đọc byte của file nén flag.flag và thấy được rằng mỗi số ở 28-byte đầu đều ở định dạng little-endian.

<img width="624" height="302" alt="image" src="https://github.com/user-attachments/assets/a4a3617a-9631-4ec7-bf46-fdf45ad29a2d" />

Các byte kế tiếp là `4d 8a 02 c1`, chúng là các khối và các khối có một bẩn đồ xoá gồm 33-byte mỗi khối. Byte đầu tiên là số lượng shard bị xoá và 32-byte còn lại là số hiệu vật lý của chính byte đó. Chính xác 32 shard trong mỗi khối đã được đặt về giá trị không. Dữ liệu shard thực tế bắt đầu tại vị trí offset này: `28 + 8 + 4 * (1 + 32) = 168`.
Một block là `256 * 192 = 49152` byte, do đó toàn bộ file là `168 + 4 * 49152 = 196776` byte.

## Cấu trúc Reed-Solomon của máy ảo

Trước khi file được nén, VM sẽ tạo một bản thiết kế với bố cục như sau: 

```

63 8d a6 f1
u32 chunk_size
u32 data_shards
u32 parity_shards
u32 block_count
byte field_exp[255]
byte coefficients[32 * 224]
u32 route_scores[4 * 256]

```

Thuật toán của máy ảo này là:

```

r520 = 1;
r521 = 0;
rs_field_loop:
jukai_store_byte(r500, r520);
r500 += 1;
r520 <<= 1;
r522 = r520;
r522 &= 256;
if (r522 == 0) goto rs_field_reduced;
r520 ^= 285;
rs_field_reduced:
r520 &= 255;
r521 += 1;
if (r521 < 255) goto rs_field_loop;

```

```

x = ((block + 1) * 0x045d9f3b)
  ^ ((shard + 0x1150) * 0x9e3779b1)
x ^= x >> 16
x *= 0x7feb352d
x ^= x >> 15
x *= 0x846ca68b
x ^= x >> 16

```

## Giải mã hàm thuật toán

Sử dụng AI để viết đoạn mã giải thuật toán trên:

```

selected = available_shards[:224]
matrix = [generator[row_index] for row_index, _ in selected]
inverse = gf_matrix_inverse(matrix)
data_shards = multiply(inverse, selected)

```

## Khôi phục bản ghi

Dữ liệu của bản ghi lặp lại 471 lần:

```

11 50 9d 02                    # file magic

a7 04 2c f1                    # record magic
u32 path_length                # 24
u32 blob_length                # 141
byte path[24]                  # binary token

d3 1f 02 8e                    # blob magic
u32 masked_index
byte encrypted_payload[128]
u8 payload_length              # 128
u32 tag

```

Mỗi bản ghi gồm 128-byte payload và 49-byte framing, do đó tổng kích thước là `4 + 471 * (128 + 49) = 83371` byte có 24-byte là mã định danh nhằm ẩn tên bản ghi chứ không phải thư mục.

Giá trị seed của VM là `0x5e17e11d`, giá trị của index seed là được tính toán bởi hàm sau:

```

path_seed = avalanche32(0x5e17e11d ^ 0x83b6a9d1 ^ 0xc001d00d)
          = 0xe3261780

index = masked_index ^ path_seed ^ 0x1150

```

Sau khi dùng thuật toán ở trên để xếp lại stt thật của 471 bản ghi và thu được tệp lưu trữ đã mã hoá có kích thước 60288 byte.
Máy ảo cũng tính toán ở phần cuối, nhưng giá trị này không được sử dụng làm giá trị xác thực cuối cùng. Để giải bài toán, chỉ cần các giá trị magic values, độ dài và chuỗi chỉ số liên tục là đủ.

## Giải mã chuỗi XOR stream của tác giả

Viết đoạn mã giải thuật toán: 

```

key = avalanche32(0x5e17e11d ^ 0x50524144)
state = avalanche32(key ^ 0x9e3779b9)

state = avalanche32(state + 0x6d2b79f5 + offset)
keystream += little_endian_u32(state)
plain = encrypted XOR keystream

```

Kết quả thu được là: `72 19 b4 0d 5e a1 03 cc`.

## Khôi phục file và lấy flag

Sau khi giải mã xong XOR ở bước trước và thu thập được đường dẫn tương đối, ta khôi phục được tệp tin phân tán chứa 139 mục:

```

u8 kind                 # 1: directory, 2: file
u32 path_length
byte path[path_length]  # UTF-8, separated with '/'

if kind == 2:
    u64 file_size
    byte data[file_size]

```

Sau khi khôi phục thành công 139 thư mục này thì phải bỏ các byte thừa ở cuối dữ liệu, ta sẽ thu được một cây thư mục hoàn chỉnh gồm 42 thư mục và 97 file trong đó có file flag.py
Chạy file flag.py thì ta thu được flag là `pwnsec{c0d3ebc92d57e1db301dbf8a6a9595b3e003078fc977c204ab6ff6372353a6da60a423b3d9b57fd1c7618b976df1500f2d3b137d9b5e57e92909a140ae6ee99f}`.
