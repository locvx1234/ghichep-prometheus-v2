Prometheus cung cấp ngôn ngữ truy vấn để có thể tổng hợp dữ liệu thời gian thực. Kết quả của việc truy vấn có thể dùng để vẽ biểu đồ, hiện thị dữ liệu dạng bảng, hay là truyền cho các hệ thống khác qua HTTP API.

### Các loại Expression:

- **Instant vector** : giá trị của một điểm tại một thời điểm
- **Range vector** : tập hợp các điểm trong một khoảng thời gian
- **Scalar** : giá trị số dạng float
- **String** : giá trị dạng chuối, chưa sử dụng


Tùy thuộc vào use-case để sử dụng loại expression nào. Ví dụ để vẽ đồ thị không sử dụng Range vector mà phải dùng Instance vector.


### Instant vector selectors

Cho phép select chuỗi thời gian và giá trị tại mỗi thời điểm. Định dạng truy vấn đơn giản nhất là chỉ cần tên metric. 

Ví dụ :

```
http_requests_total
```

Có thể filter theo label bằng dấu `{}`

```
http_requests_total{job="prometheus",group="canary"}
```

Có thể sử dụng Regex [RE2 syntax](https://github.com/google/re2/wiki/Syntax) trong chuỗi truy vấn:

- `=` : chọn label có giá trị chính xác như thế
- `!=` : chọn label có giá trị khác giá trị cung cấp
- `=~` : chọn label có giá trị regex-match với giá trị cung cấp
- `!~` : chọn label có giá trị không regex-match với giá trị cung cấp

Ví dụ:

```
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

Biểu thức trên chọn những gía trị của metric `http_requests_total` với `environment` là `staging`, `testing` hoặc `development`, và `method` khác `GET`

### Range Vector Selectors

Hoạt động giống như Instant vector selectors , thay vì trả lại giá trị đơn mà nó trả về một tập hợp các giá trị trong khoảng thời gian được chọn. Sử dụng `[]` ở cuối biểu thức để xác định khoảng thời gian. 

Ví dụ:

```
http_requests_total{job="prometheus"}[5m]
```

Các đơn vị thời gian được sử dụng là : `s`, `m`, `h`, `d`, `w`, `y`

### Thay đổi offset 

Sử dụng offset để thay đổi mốc thời gian để chọn Instance vector hoặc Range vector

```
http_requests_total offset 5m
```

Biểu thức trên trả về một Instance vector của `http_requests_total` tại thời điểm 5 phút trước thời điểm hiện tại
