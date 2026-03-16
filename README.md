# Elasticsearch Setup with Docker Compose

Repo này cung cấp cấu hình Docker Compose để chạy Elasticsearch + Kibana ở 3 chế độ:

- **Single node**: đơn giản, phù hợp dev/local.
- **Multi node (3 node cluster, full TLS)** với HTTPS cho client.
- **Multi node (3 node cluster, HTTP client, TLS-only transport)**: client chỉ dùng HTTP trong mạng nội bộ, TLS chỉ dùng cho giao tiếp giữa các node.

---

## 1. Yêu cầu

- Docker
- Docker Compose (hoặc `docker compose` đi kèm Docker mới)

---

## 2. Chuẩn bị file môi trường

Copy file mẫu:

```bash
cp .env.example .env
```

Mở file `.env` và chỉnh các giá trị cho phù hợp:

- **`STACK_VERSION`**: version Elasticsearch/Kibana (ví dụ `8.15.0`).
- **`CLUSTER_NAME`**: tên cluster (ví dụ `es-single-node` hoặc `es-multi-node`).
- **`LICENSE`**: `basic` hoặc `trial`.
- **`ES_PORT`**: port Elasticsearch trên host (mặc định 9200).
- **`KIBANA_PORT`**: port Kibana trên host (mặc định 5601).
- **`ELASTIC_PASSWORD`**: mật khẩu cho user `elastic`.
- **`KIBANA_PASSWORD`**: mật khẩu cho user `kibana_system`.

Lưu ý: Không commit file `.env` (đã được ignore trong `.gitignore`).

---

## 3. Chạy single node (`docker-compose-single-node.yml`)

### 3.1. Khởi động

```bash
docker compose -f docker-compose-single-node.yml up -d
```

Compose sẽ:

- Tạo CA & TLS certificate cho Elasticsearch/Kibana.
- Khởi động `elasticsearch` dạng **single-node** với security + SSL bật sẵn.
- Khởi động `kibana` trỏ tới Elasticsearch.

### 3.2. Kiểm tra

- Kiểm tra health:

```bash
docker compose -f docker-compose-single-node.yml ps
```

- Test Elasticsearch:

```bash
curl -k --cacert certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" https://localhost:${ES_PORT}
```

- Truy cập Kibana trên browser:

```text
http://localhost:${KIBANA_PORT}
```

Đăng nhập bằng user `elastic` + `${ELASTIC_PASSWORD}`.

### 3.3. Dừng & xóa container

```bash
docker compose -f docker-compose-single-node.yml down
```

Nếu muốn xóa kèm volume data:

```bash
docker compose -f docker-compose-single-node.yml down -v
```

---

## 4. Chạy multi node cluster full TLS (`docker-compose-multi-node.yml`)

### 4.1. Khởi động

```bash
docker compose -f docker-compose-multi-node.yml up -d
```

Compose sẽ:

- Tạo CA & TLS certificate cho 3 node:
  - `elasticsearch-01`, `elasticsearch-02`, `elasticsearch-03`.
- Khởi động 3 node trong cùng `cluster.name=${CLUSTER_NAME}`.
- Dùng `cluster.initial_master_nodes` & `discovery.seed_hosts` để các node join cluster.
- Khởi động `kibana-cluster` trỏ vào `https://elasticsearch-01:9200`.

### 4.2. Kiểm tra cluster

- Xem trạng thái container:

```bash
docker compose -f docker-compose-multi-node.yml ps
```

- Kiểm tra thông tin cluster:

```bash
curl -k --cacert certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" \
  https://localhost:${ES_PORT}/_cluster/health?pretty
```

- Kiểm tra các node trong cluster:

```bash
curl -k --cacert certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" \
  https://localhost:${ES_PORT}/_cat/nodes?v
```

- Truy cập Kibana:

```text
http://localhost:${KIBANA_PORT}
```

### 4.3. Dừng & xóa container

```bash
docker compose -f docker-compose-multi-node.yml down
```

Hoặc xóa kèm volume:

```bash
docker compose -f docker-compose-multi-node.yml down -v
```

---

## 5. Chạy multi node cluster HTTP client, TLS-only transport (`docker-compose.yml`)

### 5.1. Khởi động

```bash
docker compose -f docker-compose.yml up -d
```

Compose sẽ:

- Tạo CA & TLS certificate cho 3 node:
  - `elasticsearch-01`, `elasticsearch-02`, `elasticsearch-03`.
- Khởi động 3 node trong cùng `cluster.name=${CLUSTER_NAME}`.
- Bật security (`xpack.security.enabled=true`) với:
  - **Transport TLS bật** (giữa các node sử dụng cert từ `elasticsearch-setup`).
  - **HTTP không dùng TLS** (`xpack.security.http.ssl.enabled=false`) để client/Kibana nói chuyện qua `http://`.
- Khởi động `kibana-cluster` trỏ vào `http://elasticsearch-01:9200` với user `kibana_system`.

### 5.2. Kiểm tra cluster

- Xem trạng thái container:

```bash
docker compose -f docker-compose.yml ps
```

- Kiểm tra thông tin cluster:

```bash
curl -u "elastic:${ELASTIC_PASSWORD}" \
  http://localhost:${ES_PORT}/_cluster/health?pretty
```

- Kiểm tra các node trong cluster:

```bash
curl -u "elastic:${ELASTIC_PASSWORD}" \
  http://localhost:${ES_PORT}/_cat/nodes?v
```

- Truy cập Kibana:

```text
http://localhost:${KIBANA_PORT}
```

### 5.3. Dừng & xóa container

```bash
docker compose -f docker-compose.yml down
```

Hoặc xóa kèm volume:

```bash
docker compose -f docker-compose.yml down -v
```

---

## 6. Ghi chú thêm

- **TLS & security**:
  - `docker-compose-single-node.yml`: bật `xpack.security.enabled=true` + TLS cho cả HTTP & transport, client cần dùng `https://` kèm CA và user/password (`elastic` / `kibana_system` / user khác nếu bạn tạo thêm).
  - `docker-compose-multi-node.yml`: tương tự, full TLS cho HTTP & transport.
  - `docker-compose.yml`: bật `xpack.security.enabled=true` + TLS cho **transport only**, client/Kibana nói chuyện qua `http://` nhưng vẫn cần user/password.
- **Data persistence**: Dữ liệu được lưu trong các volume Docker:
  - Single node: `elasticsearch-data`, `kibana-data`.
  - Multi node: `elasticsearch-data-01/02/03`, `kibana-data`.
- **Tùy chỉnh memory**: có thể chỉnh `${MEM_LIMIT}` trong `.env` để phù hợp RAM máy bạn.
