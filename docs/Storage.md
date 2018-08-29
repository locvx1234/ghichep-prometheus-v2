Prometheus thích hợp với local storage và cả remote storage.

### Local storage

Các sample được nhóm thành các block two-hour (2 giờ sinh nhóm thành một block). Mỗi block bao gồm một thư mục chứa một hoặc nhiều file chunk, chứa time series sample cũng như là metadata file và index file trong khoảng thời gian đó. Block của sample hiện tại chưa đầy đủ, và sẽ được lưu trữ trong RAM. Nó được đảm bảo bằng cơ chế write-ahead-log (WAL) để có thể tạo lại khi Prometheus bị sự cố. Khi series bị xóa qua API, các bản ghi về việc xóa sẽ được lưu riêng trong file tombstone. 

Cấu trúc thư mục lưu trữ sẽ dạng như sau: 

```
├── 01CP1SZ8SHQXY20TMEDNAKSVE3
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01CP2EJEAK804E9T46SCV3ZSD4
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01CP335KATK83XWGDE6MXDKDC6
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01CP335M8MNVYEXV481AZDJ14B
│   ├── chunks
│   │   └── 000001
│   ├── index
│   ├── meta.json
│   └── tombstones
├── lock
└── wal
    ├── 000001
    ├── 000084
    └── 000085
```

Prometheus có một số cờ để cấu hình storage, quan trọng nhất là : 

- `--storage.tsdb.path` : Đường dẫn Prometheus ghi vào database, mặc định là `data/`
- `--storage.tsdb.retention` : Khoảng thời gian lưu dữ liệu, nó sẽ bị xóa khi vượt quá khoảng thời gian này, mặc định `15d`

Trung bình, Prometheus cần 1-2 bytes mỗi sample. Nên để tính dung lượng cần cho lưu trữ, sử dụng công thức sau: 

```
needed_disk_space = retention_time_seconds * ingested_samples_per_second * bytes_per_sample
```

### Remote storage integrations

Lưu trữ cục bộ bị giới hạn về dung lượng và thời gian lưu trữ. Để giải quyết vấn đề này, Prometheus cung cấp giải phải remote storage. 

Mô hình như sau:

![remote_integration](https://raw.githubusercontent.com/locvx1234/ghichep-prometheus-v2/master/images/remote_integrations.png)

Prometheus sẽ write/read qua Adapter - [remote_storage_adapter](https://github.com/prometheus/prometheus/blob/release-2.3/documentation/examples/remote_storage/remote_storage_adapter/README.md). Và thông qua Adapter này để write hoặc read từ Third-party storage.

Third-party này là một Time series database như Influx, Graphite, OpenTSDB

#### Cài đặt:

Yêu cầu đã có [Go](https://golang.org/doc/install#tarball)


Trên InfuxDB, tạo database tên là `prometheus` và cấp quyền cho user `prom` truy cập

```
> influx
>> auth
>> CREATE DATABASE prometheus;
>> CREATE USER "prom" with password 'prom';
>> GRANT ALL ON prometheus TO prom;
>> ALTER RETENTION POLICY "autogen" ON "prometheus" DURATION 1d
REPLICATION 1 SHARD DURATION 1d DEFAULT;
>> SHOW RETENTION POLICIES ON prometheus;
```

Trên Prometheus server: 

```
go get github.com/prometheus/prometheus/documentation/examples/remote_storage/remote_storage_adapter
```

Running:

```
INFLUXDB_PW=prom $GOPATH/bin/remote_storage_adapter \
-influxdb-url=http://192.168.30.67:8086 \
-influxdb.username=prom \
-influxdb.database=prometheus \
-influxdb.retention-policy=autogen
```


Sửa file cấu hình prometheus `/etc/prometheus/prometheus.yml`

```
# Remote write configuration (for Graphite, OpenTSDB, or InfluxDB).
remote_write:
  - url: "http://localhost:9201/write"

# Remote read configuration (for InfluxDB only at the moment).
remote_read:
  - url: "http://localhost:9201/read"
```

Trong đó `localhost:9201` là địa chỉ `remote_storage_adapter` khi chạy



Restart Prometheus 

```
systemctl restart prometheus
```

Tại sao lại là InfluxDB, nó có những ưu điểm gì, tham khảo : 

https://github.com/locvx1234/ghichep-prometheus-v2/blob/master/docs/Using%20Prometheus%20with%20InfluxDB%20for%20Metrics%20Storage%20-%20FileId%20-%20115469.pdf