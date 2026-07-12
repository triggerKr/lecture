
기존 Tomcat 환경(starter-web) 유지하며 컨트롤러로 프록시하기 (추천)
만약 이미 스프링 부트 2.7 프로젝트에 많은 MVC 기능(spring-boot-starter-web)과 다른 비즈니스 로직들이 들어가 있어서 위와 같은 Netty 기반 Gateway를 쓰기 부담스럽다면, RestTemplate을 활용해 컨트롤러에서 직접 파이썬을 호출하는 방식이 가장 안전합니다. (스프링 부트 2.7에는 RestClient가 없으므로 RestTemplate을 씁니다.)

1. RestTemplate 빈(Bean) 등록 설정
파이썬 마이크로서비스들과의 통신을 담당할 RestTemplate을 커스텀 설정합니다. 타임아웃 설정을 넉넉히 주는 것이 마이크로서비스 환경에서 좋습니다.

Java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate pythonRestTemplate() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(5000); // 5초
        factory.setReadTimeout(30000);    // 30초 (파이썬 작업 시간에 따라 조절)
        return new RestTemplate(factory);
    }
}
2. Imap-Gateway용 프록시 컨트롤러 구현
Vue3가 보낸 HTTP 메서드(GET, POST 등)와 헤더, 바디, 쿼리 스트링을 그대로 캡처해서 파이썬 서비스로 토스해 주는 범용 컨트롤러입니다.

Java
import org.springframework.http.*;
import org.springframework.util.StreamUtils;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.HttpStatusCodeException;
import org.springframework.web.client.RestTemplate;

import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.Collections;

@RestController
@RequestMapping("/api/imap")
public class ImapProxyController {

    private final RestTemplate restTemplate;
    // 실제 운영 중인 파이썬 imap-gateway 주소 (설정파일로 빼는 것을 추천)
    private final String pythonServiceUrl = "http://your-python-imap-gateway-address:port";

    public ImapProxyController(RestTemplate pythonRestTemplate) {
        this.restTemplate = pythonRestTemplate;
    }

    @RequestMapping(value = "/**", method = {RequestMethod.GET, RequestMethod.POST, RequestMethod.PUT, RequestMethod.DELETE})
    public ResponseEntity<?> proxyRequest(HttpMethod method, HttpServletRequest request) throws URISyntaxException, IOException {
        
        // 1. Vue3가 요청한 주소에서 /api/imap 제거 후 파이썬용 URI 생성
        String extraPath = request.getRequestURI().replace("/api/imap", "");
        String queryString = request.getQueryString();
        String targetUrl = pythonServiceUrl + extraPath + (queryString != null ? "?" + queryString : "");
        URI uri = new URI(targetUrl);

        // 2. 요청 헤더 복사
        HttpHeaders headers = new HttpHeaders();
        Collections.list(request.getHeaderNames())
                .forEach(headerName -> headers.add(headerName, request.getHeader(headerName)));

        // 3. 요청 바디(Body) 복사
        byte[] body = StreamUtils.copyToByteArray(request.getInputStream());

        HttpEntity<byte[]> httpEntity = new HttpEntity<>(body, headers);

        try {
            // 4. 파이썬 마이크로서비스로 요청 대리 수행
            return restTemplate.exchange(uri, method, httpEntity, byte[].class);
        } catch (HttpStatusCodeException e) {
            // 파이썬 쪽에서 에러(4xx, 5xx)가 난 경우 그대로 Vue3에게 전달
            return ResponseEntity.status(e.getRawStatusCode())
                    .headers(e.getResponseHeaders())
                    .body(e.getResponseBodyAsByteArray());
        }
    }
}
이 방식(방법 2)의 장점 (10개 이상의 파이썬 서비스 제어)
이 구조를 잡아두시면, 나중에 다른 파이썬 마이크로서비스(예: analytics-service, user-service 등)를 스프링 부트에 추가로 연결할 때도 클래스를 새로 만들 필요가 거의 없습니다.

@RequestMapping("/api/{serviceName}/") 형태로 구조를 조금만 바꾸면, URL의 첫 번째 경로를 파이썬 서비스 이름으로 인식하여 매핑된 파이썬 서버 주소로 다이내믹하게 라우팅해 주는 나만의 미니 API 게이트웨이를 자바로 쉽게 완성할 수 있습니다.

스프링 부트 2.7.18 환경에 가장 안정적인 서블릿 방식이므로 기존 시스템 레이아웃을 깨뜨리지 않고 연동하기에 가장 수월하실 것입니다.
