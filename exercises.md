# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Biển  Mã học viên: 2A202602416

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

**Trả lời:** Khi tạo service trên Render, `AGENT_API_KEY` khai báo `sync: false`
nên phải nhập tay. Giả sử tôi quên nhập (hoặc gõ sai tên biến thành
`AGENT_APIKEY`):

- **Có mặc định `"changeme"`:** app vẫn khởi động, `/health` vẫn 200, Render báo
  *Live* xanh. Tôi tưởng deploy thành công. Nhưng lúc đó API công khai đang được
  bảo vệ bằng khóa `changeme` — thứ ai đọc repo trên GitHub cũng biết. Bot quét
  Internet tìm ra URL, gọi `/ask` bằng `X-API-Key: changeme`, và mỗi request là
  tiền LLM của tôi. Tôi chỉ phát hiện khi nhìn hóa đơn.
- **Không có mặc định:** pydantic ném `ValidationError: agent_api_key Field
  required` ngay khi import `Settings`, container thoát, health check fail, deploy
  bị đánh dấu *Failed* và log chỉ thẳng tên biến thiếu. Lỗi hiện ra **đúng lúc
  tôi đang nhìn màn hình deploy**, sửa mất 1 phút, chưa request nào lọt qua.

Tức là "chết sớm" đổi một sự cố bảo mật âm thầm thành một lỗi cấu hình ồn ào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

**Trả lời:** Dòng log thật lấy từ `docker compose logs agent` sau khi gọi `/ask`
3 lần với `X-User-Id: sv01`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T09:00:25.124961+00:00", "user_id": "sv01", "tokens_in": 289, "tokens_out": 52, "cost_usd": 7.455e-05}
```

(`tokens_in` = 289 vì lịch sử hội thoại của sv01 đã được gửi kèm vào prompt.)

1. **Tổng hợp chi phí theo user.** Vì mỗi dòng là JSON có `user_id` và
   `cost_usd`, tôi lọc và cộng được ngay, ví dụ
   `docker compose logs --no-log-prefix agent | grep ask_completed | jq -s 'group_by(.user_id) | map({user: .[0].user_id, usd: (map(.cost_usd) | add)})'`
   → trả lời câu "user nào tiêu nhiều tiền nhất hôm nay". Trên Render/Datadog
   thì là một query theo field. `print("đã trả lời xong")` không có user,
   không có số tiền — không có gì để cộng.
2. **Lọc và cảnh báo theo `level` / `event` trong một khoảng thời gian.** Có
   `timestamp` ISO-8601 (UTC) và `level`, nên đặt được cảnh báo kiểu "số dòng
   `level = error` trong 5 phút > 10 thì báo" hoặc vẽ biểu đồ số `ask_completed`
   theo phút, hay theo dõi `tokens_in` tăng dần (prompt phình to). Chuỗi tự do
   thì phải đoán bằng regex, và chỉ cần đổi câu chữ là cảnh báo hỏng.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `FROM python:3.11`) | 1.73 GB (~1770 MB) — nén: 446 MB |
| Multi-stage (`python:3.11-slim`) | 271 MB — nén: 63.9 MB |

(Bản 1 stage build từ Dockerfile gốc ở commit `306b897`. Số liệu từ
`docker images` và `docker image inspect --format '{{.Size}}'`.)

Giải thích: phần dung lượng chênh lệch đó là những gì?

**Trả lời:** Chênh ~1.46 GB, gần như toàn bộ đến từ **base image**, không phải
từ code hay thư viện của tôi. Xem `docker history`:

- Layer `pip install` ở bản 1 stage là **95.1 MB**, còn layer
  `COPY /install /usr/local` ở bản multi-stage là **65.5 MB** — chỉ lệch ~30 MB
  (bản multi-stage cài bằng `--no-cache-dir` nên không mang theo cache của pip
  trong `/root/.cache/pip`).
- Code (`app/`, `utils/`) chỉ ~140 KB ở cả hai bản.
- Phần còn lại (~1.4 GB) là `python:3.11` bản đầy đủ: một Debian gần như hoàn
  chỉnh gồm `gcc`/`g++`, `make`, header `-dev` của hàng loạt thư viện C
  (libssl, libffi, libpq, libxml...), `git`, `curl`, `wget`, ImageMagick, v.v.
  Những thứ đó chỉ cần khi **biên dịch**, không cần khi **chạy** một app
  FastAPI. `python:3.11-slim` bỏ hết chúng.

Multi-stage cho phép nếu cần compiler thì cài ở stage `builder`, rồi chỉ
`COPY --from=builder` kết quả sang runtime — compiler bị vứt lại. Ngoài dung
lượng, image nhỏ còn nghĩa là deploy nhanh hơn (Render phải kéo image) và ít
phần mềm hơn = ít CVE hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

**Trả lời:** Tôi đổi `SERVICE_VERSION = "1.0.0"` → `"1.0.1"` rồi build lại cả
hai bản với `--progress=plain` (sau đó trả file về như cũ):

**Dockerfile multi-stage của tôi:**

| Layer | Kết quả |
|-------|---------|
| `[builder 2/4] WORKDIR /build` | CACHED |
| `[builder 3/4] COPY requirements.txt .` | CACHED |
| `[builder 4/4] RUN pip install ... -r requirements.txt` | CACHED |
| `[runtime 2/6] COPY --from=builder /install /usr/local` | CACHED |
| `[runtime 3/6] RUN useradd ... appuser` | CACHED |
| `[runtime 4/6] WORKDIR /app` | CACHED |
| `[runtime 5/6] COPY app ./app` | **chạy lại** (1.8 s) |
| `[runtime 6/6] COPY utils ./utils` | **chạy lại** (0.1 s) |

Chỉ layer `COPY app` bị đổi checksum (vì `main.py` đổi), và Docker hủy cache
**từ layer đó trở xuống** — nên `COPY utils` chạy lại dù `utils/` không đổi.
Mọi thứ phía trên, gồm `pip install` tốn thời gian nhất, được dùng lại. Cả lần
build mất vài giây.

**Dockerfile 1 stage (`COPY . .` đứng trước `RUN pip install`):**

| Layer | Kết quả |
|-------|---------|
| `[2/4] WORKDIR /app` | CACHED |
| `[3/4] COPY . .` | chạy lại |
| `[4/4] RUN pip install -r requirements.txt` | **chạy lại — 114.9 s** |

Vì `COPY . .` chứa `main.py`, sửa một ký tự là layer đó đổi, kéo theo
`pip install` phía sau mất cache → tải và cài lại toàn bộ fastapi, uvicorn,
pydantic, redis... gần 2 phút, dù `requirements.txt` không hề đổi. Nhân với số
lần sửa code mỗi ngày và số lần CI chạy thì đó là rất nhiều thời gian chết.
Nguyên tắc: thứ **ít thay đổi** (dependency) đặt trước, thứ **hay thay đổi**
(code) đặt sau.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

**Trả lời:** Chuỗi sự kiện khi container chạy bằng root:

1. **Lỗ hổng trong app.** Ví dụ một thư viện dùng `pickle`/`yaml.load` không an
   toàn, hoặc code gọi `subprocess` với input của user → kẻ tấn công gửi request
   độc hại và **thực thi được lệnh** bên trong process uvicorn (RCE).
2. **Lệnh chạy với quyền của process** — tức là **root (uid 0)** trong container.
   Kẻ tấn công ghi được mọi file trong container, cài công cụ, đọc biến môi
   trường (`AGENT_API_KEY`, `REDIS_URL`), sửa code của app để cài backdoor.
3. **Thoát khỏi container.** Container không phải máy ảo — nó dùng chung kernel
   với host, và uid 0 trong container **chính là uid 0 trên host** (khi không
   bật user namespace). Root trong container có sẵn nhiều capability hơn, nên
   dễ khai thác hơn một lỗ hổng kernel/runtime (kiểu CVE runc 2019-5736),
   hoặc lợi dụng cấu hình sai như mount `/var/run/docker.sock` hay thư mục host.
4. **Root trên host.** File tạo ra trên volume mount mang owner root, và một
   lần thoát container thành công = toàn quyền trên máy host và mọi container
   khác chạy trên đó.

**`USER appuser` cắt chuỗi ở bước 2.** Trong image của tôi process chạy bằng
`appuser` (uid 10001) — tôi kiểm tra bằng `docker compose exec agent whoami` →
`appuser`. Kẻ tấn công có RCE thì cũng chỉ là một user thường: không cài được
package, không ghi được vào `/usr/local` (thư viện) hay các thư mục hệ thống,
không có capability của root. Để đi tiếp bước 3 họ phải tìm thêm một lỗ hổng
leo thang đặc quyền nữa, và nếu có thoát ra thì trên host họ cũng chỉ là uid
10001 — một user không tồn tại, không có quyền gì. Một lỗ hổng không còn đủ để
chiếm máy; cần hai.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

**Trả lời:** **20 request** trong 2 giây (gấp đôi hạn mức).

Cách làm: đợi tới cuối phút rồi dồn request vào hai bên mốc reset.

- `10:00:59` → gửi 10 request. Bộ đếm của phút `10:00` lên 10/10 — hợp lệ.
- `10:01:00` → đồng hồ sang phút mới, bộ đếm reset về 0.
- `10:01:00`–`10:01:01` → gửi thêm 10 request. Bộ đếm phút `10:01` lên 10/10 —
  vẫn hợp lệ.

Kết quả: 20 request trong khoảng 2 giây mà không bị chặn lần nào. Lặp lại mỗi
phút thì đỉnh tải luôn gấp đôi thứ hệ thống được thiết kế để chịu.

Với sliding window, lúc `10:01:00` tôi đếm các request có timestamp trong
`(now − 60s, now]` bằng `zremrangebyscore` + `zcard` → vẫn thấy đủ 10 request
của giây `:59`, nên request thứ 11 bị 429 và phải đợi tới `10:01:59` (khi 10
request cũ trôi ra khỏi cửa sổ). Bất kỳ khoảng 60 giây nào cũng không quá 10
request. Tôi thấy đúng hành vi này khi test: gọi 15 lần liền → 10 lần `200`
rồi `429`, và test `test_cua_so_truot_qua_thi_duoc_goi_lai` xác nhận sau 60
giây thì gọi lại được.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Trả lời:**

| | Rate limit | Cost guard |
|---|---|---|
| Đo cái gì | **Số request** | **Số tiền** (token × giá) |
| Cửa sổ | 60 giây gần nhất (trượt) | Tháng dương lịch (`cost:<user>:YYYY-MM`) |
| Bảo vệ khỏi | Burst / spam / bot làm quá tải service | Hóa đơn LLM vượt ngân sách |
| Mã lỗi | 429 — thử lại sau `Retry-After: 60` | 402 — hết ngân sách, chờ sang tháng |

Rate limit coi mọi request như nhau; cost guard biết request nào đắt.

**Rate limit cho qua, cost guard chặn:** một user gửi đều đặn 5 request/phút —
dưới hạn mức 10 — nhưng mỗi request dán vào 40.000 token tài liệu, cộng thêm
lịch sử hội thoại (tối đa 20 message) cũng bị gửi lại mỗi lượt. Không bao giờ
chạm 429, nhưng chạy suốt một ngày làm việc là đốt hết `MONTHLY_BUDGET_USD =
10`. Cost guard thấy `spent + cost > budget` → 402. Tương tự: một script chạy
nền 9 request/phút suốt 24/7 — lách rate limit hoàn hảo, nhưng tổng tiền cả
tháng thì không lách được.

**Cost guard cho qua, rate limit chặn:** user mới đầu tháng, ngân sách còn gần
nguyên $10, nhưng một vòng lặp bị lỗi ở client (hoặc bot) bắn 15 câu "test"
ngắn trong 1 giây. Mỗi câu chỉ tốn ~$0.00002 nên cost guard không phản ứng,
nhưng rate limit chặn từ request thứ 11 → 429 — đúng như tôi quan sát trên bản
deploy Render: `200 ×9` rồi `429 ×6`. Nó bảo vệ CPU, Redis và quota của nhà
cung cấp LLM khỏi burst, dù chưa tốn bao nhiêu tiền.

Cần cả hai; và cả hai đều phải chặn **trước** khi gọi LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

**Trả lời:** Giả sử liveness probe chạy mỗi 10 s, restart sau 3 lần fail liên
tiếp:

1. **t = 0 s** — Redis mất kết nối (restart, failover, mạng chập chờn). Cả 3
   container vẫn khỏe về mặt process, chỉ là không nói chuyện được với Redis.
2. **t ≈ 0–10 s** — Probe gọi endpoint gộp; nó `ping()` Redis → fail → trả 503.
   **Cả 3 container cùng fail một lúc**, vì chúng chung một dependency.
3. **t ≈ 20–30 s** — Đủ 3 lần fail. Orchestrator kết luận cả 3 container "chết"
   và **kill + restart đồng loạt cả 3**. Load balancer không còn instance nào
   → mọi request, kể cả request đang xử lý dở, trả **502/503**. Hệ thống sập
   hoàn toàn.
4. **t = 30 s** — Redis quay lại. Nhưng cả 3 container đang khởi động lại
   (kéo image, import app, chờ health check), nên vẫn chưa ai phục vụ.
5. **t ≈ 45–60 s+** — Container lần lượt lên lại. Nếu Redis còn chập chờn thì
   chúng vừa lên đã fail probe → vào vòng **restart loop**, backoff ngày càng
   dài. Sự cố 30 giây của Redis thành sự cố vài phút của toàn hệ thống.

**Tách ra (như tôi đã làm):** `/health` không chạm Redis → vẫn 200 → **không
container nào bị restart**. `/ready` trả 503 → load balancer **tạm ngừng gửi
traffic mới** (không kill ai). Khi Redis sống lại, `/ready` → 200 và traffic
quay về ngay, không cần khởi động lại gì. Tôi đã thử bằng `docker compose stop
redis`: `/health` → **200**, `/ready` → **503** `{"status":"not ready","redis":false}`;
`docker compose start redis` → `/ready` → **200** ngay, không container agent
nào bị restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

**Trả lời:** Tôi chạy 3 container agent sau nginx (round-robin), gọi 5 lượt với
`X-User-Id: sv01`:

| Lượt | Container xử lý (theo log) | `history_length` (Redis) |
|---|---|---|
| 1 | agent-2 | 0 |
| 2 | agent-1 | 2 |
| 3 | agent-3 | 4 |
| 4 | agent-2 | 6 |
| 5 | agent-1 | 8 |

Tăng đều 0 → 2 → 4 → 6 → 8 (mỗi lượt thêm 1 message user + 1 message
assistant) **dù đổi container liên tục**, vì cả 3 cùng đọc/ghi
`history:sv01` trong Redis (`LLEN` = 10, TTL ≈ 7 ngày).

**Nếu dùng dict Python:** mỗi container có một dict riêng trong RAM của nó.
`history_length` chỉ phản ánh số lượt **container đó** đã xử lý cho sv01. Với
round-robin qua 3 container sẽ thấy:

`0, 0, 0, 2, 2, 2, 4, 4, 4, …`

— tăng chậm gấp 3 lần, và câu trả lời "nhớ" lúc có lúc không tùy request rơi
vào container nào: hỏi "tôi vừa nói gì?" ở lượt 2 thì agent không biết. Tệ
hơn: khi một container restart (deploy bản mới, bị kill, scale down) thì phần
lịch sử trong RAM của nó mất sạch, con số quay về 0. Và tổng bộ nhớ cả cụm tăng
dần vì không có `ltrim`/TTL. Đó là lý do state phải nằm ngoài process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Trả lời:** Lần deploy lên Render thành công ngay (Live, `/ready` 200), vì tôi
đã chạy thử cả stack bằng Docker ở máy trước. Lỗi tôi gặp nằm ở khâu kiểm tra
đó, và nó chính là lỗi sẽ xảy ra **mỗi lần deploy lại** trên cloud:
**graceful shutdown không hoạt động trong container.**

- **Triệu chứng:** `docker stop` một container agent mất **31 giây** mới dừng
  (tôi đo bằng `date` trước/sau, với `-t 30`). Log không có dòng
  `{"event": "service_stopped"...}` và không có `INFO: Shutting down` của
  uvicorn — process bị **SIGKILL** sau khi hết 30 s chờ, chứ không tự thoát.
  Trong khi đó `pytest tests/test_cp4.py` vẫn **pass 19/19**, vì test gọi thẳng
  `lifecycle.request_shutdown()` bằng Python, không đi qua tín hiệu thật.
- **Tìm nguyên nhân:** nghi tín hiệu không tới được uvicorn, nên xem PID 1 của
  container bằng `cat /proc/1/cmdline`. `CMD ["sh", "-c", "uvicorn ..."]` làm
  **`sh` là PID 1**, uvicorn là process con. Docker (và Render) gửi SIGTERM cho
  PID 1; `sh` không chuyển tiếp tín hiệu cho con, và vì là PID 1 nên kernel
  cũng không áp dụng hành vi mặc định "chết khi nhận SIGTERM" → nó lờ đi.
  Handler `lifecycle.install()` của tôi không bao giờ được gọi.
- **Sửa:** thêm `exec` → `CMD ["sh", "-c", "exec uvicorn app.main:app --host
  0.0.0.0 --port ${PORT:-8000}"]`. `sh` vẫn thay được `${PORT}` nhưng sau đó
  **bị uvicorn thay thế**, uvicorn thành PID 1 (`/proc/1/cmdline` →
  `/usr/local/bin/uvicorn app.main:app ...`). Test lại: `docker stop` còn
  **1 giây**, log có đủ `Shutting down` → `service_stopped` → `Application
  shutdown complete`.
- Tôi cũng bỏ `startCommand` trong `railway.toml` vì nó ghi đè `CMD` bằng lệnh
  không có `exec`, sẽ đưa lỗi này quay lại nếu deploy bằng Railway.

Nếu không sửa, trên Render mỗi lần deploy bản mới (Render gửi SIGTERM rồi đợi
30 s mới SIGKILL), instance cũ sẽ bị kill cứng thay vì tắt êm, request đang xử
lý dở bị cắt → user thấy lỗi mỗi lần tôi deploy.
