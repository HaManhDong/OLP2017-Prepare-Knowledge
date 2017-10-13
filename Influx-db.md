### **Influxdb**

Mỗi **influx database** sẽ có các **measurement** (độ đo - ví dụ như nhiệt độ, độ ẩm) cho các độ đo ta cần lưu trữ, data trong mỗi **measurement** được tổ chức theo **"time series"**. Time series có 0 -> nhiều các **points**, mỗi **point** bao gồm:

-  **time**: gía trị timestap tại thời điểm thêm data vào
- **measure**: độ đo của gía trị (e.g. "temperature")
- it nhất 1 key-value **field**:  the measured value itself (e.g. "temperature=30")
- zero to many key-value **tags**: containing any **metadata** about the value (e.g. "region=HaNoi")

Ta có thể coi 1 **measurement** tương đương với 1 **table** trong **SQL**, luôn được đánh chỉ mục (primary index) trên trường **time**, **tags** và **fields** là các **columns** trong table, trong đó **tags** được đánh chỉ mục, còn **fields** thì không. Còn các **points** có thể coi là các **record** (bản ghi) trong csdl quan hệ, nhưng các **points** trong cùng 1 **measurent** có thể có số lượng value, metadata khác nhau, còn tronng csdl quan hệ thì các record phải có cấu trúc giống nhau. Do đó, sự khác biệt giữa **influxdb** với các scdl quan hệ là ta có thể có hàng triệu các **measurements** mà ta không cần phải định nghĩa trước schemas như trong csdl quan hệ, và các gía trị **null** sẽ **không được lưu**.

Định dạng của point trong InfluxDB sử dụng **Line Protocol**, với format như sau:

```sh
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]
```

Ví dụ về insert các point vào trong influxdb

```sh
> insert cpu,host="donghm",region=HaNoi value=0.64
> insert cpu,host="donghm",region=HaNoi cpu=0.64
> insert cpu,host=donghm2,region=HaNoi2 cpu=0.69 1434067467000000000
> select * from cpu
name: cpu
time                cpu  host     region value
----                ---  ----     ------ -----
1434067467000000000 0.69 donghm2  HaNoi2 
1507543318765262548      "donghm" HaNoi  0.64
1507543335300223464 0.64 "donghm" HaNoi
```
**Note**: trong trường hợp ta không cung cấp timestamp khi insert 1 point thì **InfluxDB** sẽ lấy **the local current timestamp** làm time stampe cho point đó.

Các port mặc định trong influxdb

- **8086:** HTTP API port
- **8083:** Administrator interface port, if it is enabled (was reject in v1.3)
- **2003:** Graphite support, if it is enabled

Pull influxdb image

```sh
$ docker pull influxdb
```

The InfluxDB image exposes a shared volume under /var/lib/influxdb, so you can mount a host directory to that point to access persisted container data, by use option **-v** in command `docker run`.

Tạo influxdb container  với **database** tạo sẵn là **tem_humi**:

```sh
----- Sử dụng image influxdb -----
$ docker run -dit -p 8086:8086 \
  -e INFLUXDB_DB=tem_humi \
  --expose 8086 \
  -v /home/donghm/influxdb/tem_humi:/var/lib/influxdb
  --name tem_humi_influxdb influxdb

----- Sử dụng image tutum/influxdb -----
$ docker run -dit -p 8083:8083 -p 8086:8086 \
  -e PRE_CREATE_DB="tem_humi" \
  --expose 8083 --expose 8086 \
  -v /home/donghm/influxdb/tem_humi:/data
  --name tem_humi_influxdb influxdb
```

	-d, --detach, Run container in background and print container ID
	-i, --interactive, Keep STDIN open even if not attached
	-t, --tty, Allocate a pseudo-TTY
	-p, --publish list, Publish a container's port(s) to the host (mapping port từ máy thật đến port tương ứng trên container)
	-e, --env list, Set environment variables
	--expose list, Expose a port or a range of ports (mở port trên container)
	-v, --volume list, Bind mount a volume

Execute container:

```sh
$ docker exec -it tem_humi_influxdb /bin/bash

root@57994c0559bc:/# apt-get update
root@57994c0559bc:/# apt install net-tools
root@57994c0559bc:/# netstat -ntpl
```

Ngoài ra ta có thể tạo database thông qua câu lệnh:
```sh
$ curl -G http://localhost:8086/query --data-urlencode "q=CREATE DATABASE tem_humi"
```

Để output trả về với định dạng là date time:
```sh
$ influx
> precision rfc3339 // yyyy-mm-ddTh:m:s:ms 
				  // 2017-10-05T02:25:17.160327049Z

// select value with filter time
> select * from tem where time > now() - 1h
```

**Lưu ý:   InfluxDB uses the server’s local nanosecond timestamp in UTC if the timestamp is not included with the point.**

**Database in project 1 OLP**

```sh
database name: temp_humi
measures: tem (temperature) and humi (humiditi)
```