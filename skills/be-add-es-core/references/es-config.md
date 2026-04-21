# ElasticsearchConfig 참고 코드

ES 클라이언트 Bean 설정. ApiKey 인증, 전용 ObjectMapper.

```java
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.json.jackson.JacksonJsonpMapper;
import co.elastic.clients.transport.rest_client.RestClientTransport;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.json.JsonMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.module.paramnames.ParameterNamesModule;
import lombok.extern.slf4j.Slf4j;
import org.apache.http.Header;
import org.apache.http.HttpHost;
import org.apache.http.message.BasicHeader;
import org.elasticsearch.client.RestClient;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
@Slf4j
public class ElasticsearchConfig {

    @Value("${elasticsearch.url}")
    private String esUrl;

    @Value("${elasticsearch.api-key}")
    private String esApiKey;

    /** ES 전용 ObjectMapper (Instant → epoch_millis, 유연한 파싱) */
    @Bean
    @Qualifier("esObjectMapper")
    public ObjectMapper esObjectMapper() {
        ObjectMapper mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())
                .addModule(new ParameterNamesModule())
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                .configure(DeserializationFeature.FAIL_ON_TRAILING_TOKENS, false)
                .build();
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, true);
        mapper.configure(SerializationFeature.WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS, false);
        return mapper;
    }

    /** ES RestClient (ApiKey 인증) */
    @Bean
    public RestClient restClient() {
        return RestClient
                .builder(HttpHost.create("https://" + esUrl))
                .setHttpClientConfigCallback(httpAsyncClientBuilder ->
                        httpAsyncClientBuilder
                                .setSSLHostnameVerifier((host, sslSession) -> true)
                                .setKeepAliveStrategy((response, context) ->
                                        Duration.ofMinutes(5).toMillis()))
                .setDefaultHeaders(new Header[]{
                        new BasicHeader("Authorization", "ApiKey " + esApiKey)
                })
                .build();
    }

    /** ElasticsearchClient */
    @Bean
    public ElasticsearchClient esClient(
            RestClient restClient,
            @Qualifier("esObjectMapper") ObjectMapper esObjectMapper) {
        var transport = new RestClientTransport(
                restClient, new JacksonJsonpMapper(esObjectMapper));
        return new ElasticsearchClient(transport);
    }
}
```

## 필요 properties

```yaml
elasticsearch:
  url: localhost:9200
  api-key: your-api-key-here
```
