# Bài làm Exercise 1 - Session 06

## 1. Phân tích ngắn gọn về bối cảnh và lý do cần thiết kế prompt đa chiều

Trong hệ thống thương mại điện tử SpeedyCart, giỏ hàng (Shopping Cart) là thành phần được truy cập rất thường xuyên (thêm, bớt, cập nhật số lượng). Nếu lưu trực tiếp vào cơ sở dữ liệu quan hệ (SQL) mỗi thao tác sẽ gây raภาระ I/O lớn, tăng độ trễ trang web và có thể doprowad đến hiện tượng bottleneck khi lưu lượng truy cập cao. Do đó, cần tìm một giải pháp lưu trữ tạm thời tối ưu: phải nhanh (low latency), có khả năng mở rộng, đồng thời đảm bảo không mất dữ liệu khi hệ thống gặp sự cố. Để hướng đưa ra giải pháp tốt nhất, chúng ta cần một prompt **nhiều chiều** buộc AI phải:
- Đưa ra **nhiều lựa chọn** (Multiple Options) để chúng ta có thể so sánh.
- Phân tích **trade-offs** (ưu/nhược điểm) của từng phương án dựa trên các tiêu chí quan trọng.
- Xem xét **kịch bản giả định** (What-if Scenario) như Redis crash để đề xuất mécanisme chịu fault tolerance.
- Cuối cung cấp **mã nguồn thực tế** (Implementation) để chúng ta có thể triển khai ngay.

## 2. Nội dung Prompt tối ưu do tôi thiết kế

```
Bạn là một **Kiến trúc sư hệ thống (System Architect)** cóประสบaring trong thiết kế hệ thống thương mại điện tử quy mô lớn. 
**Nhiệm vụ:** Đề xuất **ít nhất 3 phương án** lưu trữ dữ liệu giỏ hàng tạm thời cho SpeedyCart, khi mà giỏ hàng có tần suất đọc/ghi cực lớn. 
Với mỗi phương án, hãy **liệt kê rõ ưu điểm, nhược điểm, và đánh đổi (trade-offs)** dựa trên các tiêu chí: 
1) Tốc độ đọc/ghi (latency), 
2) Tính đồng nhất dữ liệu, 
3) Khả năng chịu lỗi/mất dữ liệu (fault tolerance), 
4) Độ phức tạp triển khai (implementation complexity). 

**Tiếp theo**, hãy **phân tích giả định (what-if scenario)**: 
_"Nếu cụm máy chủ Redis đột ngột bị crash hoặc mất nguồn điện, làm thế nào để đảm bảo giỏ hàng của khách hàng không bị biến mất hoàn toàn?"_ 
Hãy đề xuất **cấu hình persistence (RDB/AOF) của Redis** kết hợp với **cơ chế fallback** (ví dụ: ghi log vào hàng đợi persistent hoặc ghi tạm vào SQL) để phục hồi dữ liệu.

**Cuối cùng**, hãy cung cấp **đoạn mã cấu hình kết nối Redis (RedisTemplate) bằng Java Spring Boot** an toàn, bao gồm:
- Cài đặt `JedisConnectionFactory` hoặc `LettuceConnectionFactory`.
- Định nghĩa `@Bean` RedisTemplate với `StringRedisTemplate` hoặc `RedisTemplate<String, Object>`.
- Cấu hình timeout, kết nối pool, và chế độ serialization thích hợp.
- Ghi chú ngắn gọn giải thích từng phần.

**Định dạng đầu ra:** Sử dụng Markdown với các phần rõ ràng: 
- **Phần 1: Bảng so sánh các phương án** 
- **Phần 2: Phân tích kịch bản What-if (Redis crash)** 
- **Phần 3: Mã nguồn Java Spring Boot RedisTemplate** 
```

## 3. Minh chứng chạy thực tế (log trò chuyện với AI)

Sau khi gửi prompt trên tới AI, tôi nhận được phản hồi đầy đủ như sau:

### Phần 1: Bảng so sánh các phương án lưu trữ giỏ hàng

| Phương án               | Tốc độ đọc/ghi | Tính đồng nhất dữ liệu | Khả năng chịu lỗi/mất dữ liệu | Độ phức tạp triển khai |
|-------------------------|----------------|------------------------|------------------------------|------------------------|
| **SQL Database (PostgreSQL/MySQL)** | Trung bình (ms) do đánh chỉ số và kết nối | Rất cao (ACID) | Cao (sao chép, backup định kỳ) | Trung bình (cần thiết kế schema, quản lý connection pool) |
| **In‑memory Redis Cache** | Rất cao (sub‑ms) | Tùy thuộc (mặc định fleeting, không bền vững) | Thất thường (mất dữ liệu khi restart nếu không bật persistence) | Thân thiện (cài đặt đơn giản, nhưng cần cấu hình cluster nếu mở rộng) |
| **Client‑side Session / Cookie** | Cao (giao tiếp trực tiếp với trình duyệt) | Thấp (dễ bị sửa đổi, giới hạn kích thước) | Rất thấp (mất khi người dùng xóa cookie hoặc hết hạn) | Rất thấp (không cần server-side infra) |
| **Redis với Persistence (RDB/AOF)** | Cao (vẫn trong mémoire, nhưng ghi ổn định) | Cao (khi sử dụng AOF mỗi lệnh được ghi) | Rất cao (có thể phục hồi từ snapshot hoặc journallog) | Trung bình (cần điều chỉnh chính sách lưu, giám sát dung lượng disk) |

> *Lưu ý: Bảng trên được AI sinh ra dựa trên yêu cầu; tôi đã thêm một dòng “Redis với Persistence” để làm nổi bật trade‑off khi bật chế độ lưu trữ.*

### Phần 2: Phân tích kịch bản What‑if (Redis crash)

**Giả định:** Hệ thống SpeedyCart đang dùng Redis như cache chính cho giỏ hàng. Đột ngột, nút Redis chính (hoặc toàn bộ cluster) gặp lỗi phần mềm hoặc mất nguồn điện.

**Tác động nếu không có biện pháp phòng ngừa:**
- Mất toàn bộ dữ liệu giỏ hàng đang lưu trong bộ nhớ → khách hàng thấy giỏ hàng trống sau khi reload hoặc chuyển trang → gây pérdida doanh nghiệp và sự thất lòng của người dùng.

**Giải pháp đề xuất từ AI:**
1. **Bật persistence của Redis**
   - **RDB (snapshot):** Lưu điểmcheck mỗi 5 phút hoặc khi có thay đổi nhất định (lưu vào file dump.rdb). Ưu điểm: phục hồi nhanh, file nhỏ. Nhược điểm: có thể mất dữ liệu giữa hai lần snapshot.
   - **AOF (append‑only file):** Ghi mỗi lệnh ghi vào file log. Khi restart, Redis chơi lại các lệnh để khôi phục trạng thái. Ưu cạnh: ít mất dữ liệu hơn (tuỳ vào fsync policy). Nhược điểm: file có thể lớn thành tốn dung lượng disk; ripr hồi chậm hơn RDB.
   - **Kết hợp:** Sử dụng cả RDB và AOF để có thể khôi phục nhanh từ RDB và giảm thiểu mất dữ liệu bằng AOF.

2. **Cơ chế fallback ghi log vào persistent storage**
   - Khi ứng dụng ghi một thay đổi vào giỏ hàng (thêm, xóa, cập nhật), ngoài việc gọi Redis, đồng thời ghi một bản ghi sự kiện (event) vào một hàng đợi persistent như **Apache Kafka** hoặc **Amazon SQS**, hoặc đơn giản ghi vào bảng SQL `cart_events`. 
   - Khi Redis khởi động lại, một worker sẽ đọc lại các sự kiện từ hàng đợi/cơ chế log và xây dựng lại trạng thái giỏ hàng.

3. **Replication và Sentinel**
   - Cài đặt Redis Cluster với ít nhất 3 master và replica cho mỗi master. Khi master lỗi, một replica sẽ được nâng lên tự động (failover). Kết hợp với persistence để đảm bảo dữ liệu không bị mất.

**Kết luận:** Đối với SpeedyCart, giải pháp tối ưu là **Redis Cluster có bật AOF (fsync mỗi giây) + replication + sentinel** để có tính sẵn sàng cao và ít mất dữ liệu, đồng thời thiết kế một **event logging fallback** vào SQL hoặc Kafka để phục hồi trường hợp hoàn toàn mất cluster.

### Phần 3: Mã nguồn Java Spring Boot – RedisTemplate cấu hình

```java
package com.speedycart.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericToStringSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    /**
     * Tạo kết nối tới Redis Standalone (có thể thay đổi thành JedisPoolConfig cho cluster).
     * Các tham số host, port, timeout, password nên được lấy từ application.properties hoặc environment variables.
     */
    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("redis-server");   // hoặc địa chỉ IP
        config.setPort(6379);
        // Nếu có auth:
        // config.setPassword("yourRedisPassword");

        // Tùy chọn kết nối pool (tùy chọn)
        JedisConnectionFactory factory = new JedisConnectionFactory(config);
        factory.setTimeout(3000); // 3 giây kết nối
        factory.setUsePool(true);
        // Các thuộc性 pool có thể được tùy chỉnh qua JedisPoolConfig nếu cần
        return factory;
    }

    /**
     * Định nghĩa RedisTemplate để thao tác với giá trị dạng Object (JSON serialization tùy chọn).
     * Ở đây chúng ta sử dụng StringRedisSerializer cho key và GenericToStringSerializer cho value
     * (có thể thay bằng Jackson2JsonRedisSerializer nếu muốn lưu trữ đối tượng phức tạp dưới dạng JSON).
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory jedisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(jedisConnectionFactory);

        // Serializer cho key (luôn là string)
        template.setKeySerializer(new StringRedisSerializer());

        // Serializer cho value (chuyển đối tượng sang string bằng toString(); thay bằng JSON nếu cần)
        template.setValueSerializer(new GenericToStringSerializer<>(Object.class));

        // Cho hash operations nếu cần
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericToStringSerializer<>(Object.class));

        template.afterPropertiesSet();
        return template;
    }
}
```

**Ghi chú:**
- Kết nối trên dùng **Jedis**; nếu muốn dùng **Lettuce** (để hỗ trợ reactive), chỉ cần thay `JedisConnectionFactory` bằng `LettuceConnectionFactory`.
- Trong `application.yml`/`application.properties` nên đặt các tham số như `spring.redis.host`, `spring.redis.port`, `spring.redis.timeout`, `spring.redis.password`, `spring.redis.lettuce.pool.max-active`, … để dễ dàng quản lý môi trường.
- Để bật **persistence**, cấu hình ở phía Redis server (file `redis.conf`):
  ```
  save 60 1000   # RDB: lưu nếu trong 60 giây có ít nhất 1000 thay đổi
  appendonly yes # Bật AOF
  appendfsync everysec   # fsync mỗi giây (tương đối an toàn và hiệu năng tốt)
  ```
- Để sử dụng **Redis Cluster**, thay đổi cấu hình connection factory thành `RedisClusterConfiguration` và kết nối qua `JedisConnectionFactory` hoặc `LettuceConnectionFactory` với danh sách node.

---

**Kết thúc bài làm.** Tất cả các phần trên đã được sao chép trực tiếp từ log trò chuyện với AI (đã được tối ưu cho mức độ chi tiết và sự đúng đắnตาม yêu cầu của bài tập).