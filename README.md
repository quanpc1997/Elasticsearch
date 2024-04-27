# Elasticsearch

**Ưu điểm**:

    - Mã nguồn mở, hoàn toàn miễn phí, cộng đồng phát triển lớn.
    - Tốc độ truy vấn nhanh, kể cả các truy vấn phức tạp.
    - Hỗ trợ full-tex search: tách từ, tách câu, tạo chỉ mục cho dữ liệu.
    - Tìm kiếm mờ (fuzzy), tự động hoàn thành (autocomplete) giúp tìm kết quả kể cả khi sai chính tả.
    - Chạy trên server riêng, tương tác qua RESTful API, có thể tích hợp với hầu hết mọi hệ thống.
    - Dữ liệu lưu dạng JSON, rất linh hoạt trong trường hợp dữ liệu thường xuyên thay đổi cấu trúc.
    - Khả năng mở rộng, tính sẵn sàng cao do dựng theo mô hình cluster.

**Nhược điểm**:

    - Có đầy đủ các nhược điểm của NoSQL.
    - Không phù hợp với dữ liệu phải chỉnh sửa nhiều.
    - Không hỗ trợ transaction, không có ràng buộc quan hệ dẫn đến dữ liệu có thể sai.

## 1. Cấu trúc của Elasticsearch
### a. Elasticsearch Cluster
ES Cluster gồm có 3 phần:
    - **Master Node**: Quản lý mọi hoạt động, trạng thái của cụm.
    - **Client Node**(Coordinator/Ingest Node): Điều phối tương tác kết nối với client.
    - **Data Node**: Lưu trữ dữ liệu

![ES Cluster](/images/1-elasticsearch-cluster.webp)
_Hình 1: Cấu trúc một cụm ES_

ES có thể được xem là 1 loại NoSQL. Một số khái niệm ES tương tự với DBMS mà ta cần biết:

![ES Cluster](/images/2-es-compare-dbms.webp)
_Hình 2: Một số khái niệm của ES_
> :boom: **Chú ý:** ở đây Index tương đưong với Database. Index này không phải index đánh chỉ mục.

### b. Sharding
Hiểu đơn giản là chia dữ liệu thành các phần nhỏ hơn có tên là Shard. Một Shard sẽ chứa một tập hợp con dữ liệu và bản thân nó có đầy đủ chức năng và độc lập.
![Sharding](/images/3-sharding.webp)
_Hình 3: Sharding_

Như trong ví dụ mô tả ở hình 3 ta chia 1TB dữ liệu thành 4 Shard, mỗi Shard chứa 256GB dữ liệu và phân phối nó trên các node(Ví dụ hình 4)
![Sharding Distribution](/images/4--shard-distribution.webp)
_Hình 4: Phân chia Sharding_

**=> Việc sử dụng Sharding sẽ đảm bảo đưọc khả năng mở rộng -High Scalability**

Elasticsearch, chia Shard thành 2 loại đó là:
    - **Primary Shard**: Là dữ liệu gốc của hệ thống. Tại 1 thời điểm, mỗi dữ liệu của ES chỉ nằm trên 1 Primary Shard.
    - **Replica Shard**: Là dữ liệu sao chép của Primary Shard. Thông thường sẽ có rất nhiều 

#### b.1. HA trong ES Cluster
ES sau khi chia Index thành các Primary Shard và các Replica Shard. Nó sẽ phân bố chúng một cách tự động trên các node của hệ thống(nếu hệ thống có nhiều hơn 1 node). Khi một node chết thì nó sẽ có cơ chế tự động chuyển các shard đó sang các node còn sống.
[Đọc thêm](https://viblo.asia/p/elasticsearch-zero-to-hero-2-co-che-hoat-dong-cua-elasticsearch-38X4ENMXJN2)

## 2. Cơ chế hoạt động CRUD

### Câu hỏi khi phỏng vấn

> **Câu 1**: Khi một document đưọc thêm vào index. Thì document đó sẽ đưọc lưu vào shard nào?

**Trả lời**: Nó sẽ phân chia xem shard đó được lưu vào shard nào theo công thức sau:

**shard = hash(routing) % number_of_primary_shards**

Trong đó:
    - **routing**: Giá trị sử dụng để định tuyến, mặc định sẽ là trường _id
    - **number_of_primary_shards**: Số lượng Primary Shard (ở ví dụ này là Shard=3)


> **Câu 2**: Khi có một bản ghi bị CRUD thì cơ chế đồng bộ giữa Primary Shard và Replica Shard như thế nào?

**Đối với Create, Update, Delete**
**B1:** Client gửi yêu cầu đến Master Node.
**B2:** Master Node sẽ hash theo _id. Xác định dữ liệu thuộc Primary Shard nào. Ví dụ là P1 đi.
**B3:** Sau khi đã xử lý xong ở P1. Nó sẽ thông báo đến các Replica của của nó ở các node khác(Ví dụ là có 2 replica ở Node 1 và Node 2).
**B4:** Kết quả thành công sẽ trả dữ liệu về cho Client.

**Đối với Read**
**B1:** Client gửi yêu cầu đến cho Master Node.
**B2:** Master Node sẽ hash theo _id. Xác định dữ liệu thuộc Primary Shard nào. Ví dụ là P1 đi.
**B3:** Mastẻ Node dựa theo tình hình chịu tải từ các Node, điều hướng truy vấn tới 1 node chứa dữ liệu. Ví dụ là Node 2
**B4:** Node 2 thực hiện truy vấn dữ liệu và trả về kết quả.

> **Câu 3**: Cơ chế xử lý đồng thời nhiều dữ liệu một lúc như thế nào?
Thực ra không khác nhiều so với xử lý đơn.

**Đối với Read**
Sử dụng MGET APIs: Xử lý tương tự như một documents nhưng xử lý đa luồng song song để có hiệu năng tốt nhất.
**B1**: Client gửi MGET requests tới Master Node.
**B2**: Master Node tạo multi-get tới các shard tương ứng trên các Data Node.
**B3**: Tổng hợp kết quả và trả về cho Client.

**Đối với Create, Update, Delete**
Sử dụng BULK APIs
**B1:** Client gửi Bulk request đến Master Node.
**B2:** Master Node tạo 1 bulk request trên mỗi shard, sau đó chuyển requests tương ứng đến Primary Shard.
**B3:** Tại mỗi Primary Shard, sau khi xử lý thành công. Gửi request(n) đến Replica Shards, đồng thời xử lý tiếp request(n+1).
**B4:** Khi tất cả các Shard thành công. Master Node tổng hợp lại và gửi kết quả về Client.

## 3. Xử lý dữ liệu trong ES
![Xử lý dữ liệu](/images/5-solve-data.webp)
ES xử dụng **Inverted Index**. 
Nhìn vào hình trên ta thấy ES sẽ cắt những câu trong văn bản thành từng từ một và loại bỏ các từ ít ý nghĩa như a, an, every, for, ... Sau đó nó sẽ ghi chủ mục vào dưói dạng document:position. Ví dụ **bright** 1:2 tức là nằm ở document 1 vị trí thứ 2.
Giả sử người dùng tra từ "blue sky" thì nó sẽ tìm kiếm từng từ một blue nằm cả ở document 1 và 3. Sky thì nằm trong document 2 và 3 nên cụm tìm kiếm đưộc tìm thấy sẽ là document 3.

Việc phân tách(lập chỉ mục) như thế nào tách lấy 1 từ hay 2 từ, lấy số hay bỏ số phụ thuộc vào quá trình Analyzer.
![Analyzer](/images/6-analyzer.webp) mà ở đây là bộ **Tokenizer**.
[Đọc thêm](https://viblo.asia/p/elasticsearch-zero-to-hero-2-co-che-hoat-dong-cua-elasticsearch-38X4ENMXJN2)


