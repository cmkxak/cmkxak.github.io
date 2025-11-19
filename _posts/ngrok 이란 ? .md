---
title:  "ngrok"
excerpt: "ngrok 활용하기."

toc: true
toc_sticky: true
 
date: 2025-11-19
last_modified_at: 2025-11-19
---

# ngrok으로 로컬 Spring Boot 앱을 외부에 공개하기

## 들어가며

Spring Boot로 개발하다 보면 로컬 환경에서 외부 서비스와 연동 테스트가 필요한 경우가 많습니다.

- 결제 시스템(PG사)의 callback URL 테스트
- OAuth 소셜 로그인의 redirect URI 설정
- 웹훅(Webhook) 수신 테스트
- 모바일 앱에서 로컬 API 호출
- 외부 팀원에게 개발 중인 API 공유

이럴 때 **ngrok**을 사용하면 로컬 서버를 간단하게 외부에 공개할 수 있습니다.

## ngrok이란?

ngrok은 로컬 서버를 안전한 터널을 통해 인터넷에 노출시켜주는 도구입니다.

```
인터넷 → ngrok 서버 → 터널 → 로컬 PC (localhost:8080)
```

### 작동 원리

```
┌─────────────┐      HTTPS      ┌──────────────┐      터널      ┌─────────────┐
│  외부 사용자  │ ────────────→  │ ngrok 서버    │ ───────────→  │ 로컬 PC      │
│             │                 │ (클라우드)     │               │ localhost    │
└─────────────┘                 └──────────────┘               └─────────────┘
                                                                 :8080
```

## 설치 및 설정

### 1. ngrok 설치

**macOS (Homebrew)**
```bash
brew install ngrok/ngrok/ngrok
```

**Windows (Chocolatey)**
```bash
choco install ngrok
```

**Linux / 직접 설치**
```bash
# 다운로드
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz

# 압축 해제
tar xvzf ngrok-v3-stable-linux-amd64.tgz

# 실행 가능하도록 이동
sudo mv ngrok /usr/local/bin/
```

### 2. 계정 생성 및 인증

1. [ngrok.com](https://ngrok.com) 회원가입
2. 대시보드에서 authtoken 확인
3. 토큰 등록

```bash
ngrok config add-authtoken YOUR_AUTH_TOKEN
```

## Spring Boot와 함께 사용하기

### 기본 사용법

**1. Spring Boot 앱 실행**
```bash
./mvnw spring-boot:run
# 또는
./gradlew bootRun
```

**2. ngrok으로 터널 생성**
```bash
ngrok http 8080
```

**출력 결과:**
```
Session Status                online
Account                       user@email.com
Version                       3.5.0
Region                        Korea (kr)
Latency                       5ms
Web Interface                 http://127.0.0.1:4040
Forwarding                    https://abc123def.ngrok-free.app -> http://localhost:8080

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

이제 `https://abc123def.ngrok-free.app`로 외부에서 접근 가능합니다!

### Spring Boot 컨트롤러 예제

```java
@RestController
@RequestMapping("/api")
public class TestController {
    
    @GetMapping("/hello")
    public ResponseEntity<String> hello() {
        return ResponseEntity.ok("Hello from Local Spring Boot!");
    }
    
    @PostMapping("/webhook")
    public ResponseEntity<Map<String, String>> webhook(@RequestBody Map<String, Object> payload) {
        System.out.println("웹훅 수신: " + payload);
        
        Map<String, String> response = new HashMap<>();
        response.put("status", "success");
        response.put("message", "Webhook received");
        
        return ResponseEntity.ok(response);
    }
}
```

**테스트:**
```bash
# 외부에서 접근 가능
curl https://abc123def.ngrok-free.app/api/hello

# 웹훅 테스트
curl -X POST https://abc123def.ngrok-free.app/api/webhook \
  -H "Content-Type: application/json" \
  -d '{"event":"payment.success","orderId":"12345"}'
```

## 실전 예제: PG사 결제 연동

### 1. 결제 Controller

```java
@Controller
@RequestMapping("/payment")
@Slf4j
public class PaymentController {
    
    @Autowired
    private PaymentService paymentService;
    
    // 결제 요청 페이지
    @GetMapping("/request")
    public String paymentRequest(Model model) {
        String orderNo = "ORDER" + System.currentTimeMillis();
        model.addAttribute("orderNo", orderNo);
        model.addAttribute("amount", 10000);
        return "payment-request";
    }
    
    // PG사로부터 결제 결과 수신 (Server-to-Server)
    @PostMapping("/callback")
    @ResponseBody
    public String paymentCallback(@RequestBody PaymentCallbackDto callback) {
        log.info("=== 결제 Callback 수신 ===");
        log.info("주문번호: {}", callback.getOrderNo());
        log.info("결제금액: {}", callback.getAmount());
        log.info("결과코드: {}", callback.getResultCode());
        
        // 결제 검증 및 DB 저장
        boolean isValid = paymentService.verifyAndSave(callback);
        
        if (isValid) {
            return "OK"; // PG사에 응답
        } else {
            return "FAIL";
        }
    }
    
    // 사용자 브라우저 리다이렉트 (결제 완료 페이지)
    @GetMapping("/result")
    public String paymentResult(@RequestParam String orderNo, Model model) {
        Payment payment = paymentService.findByOrderNo(orderNo);
        model.addAttribute("payment", payment);
        return "payment-result";
    }
}
```

### 2. application.yml 설정

```yaml
spring:
  application:
    name: payment-demo

server:
  port: 8080

# ngrok URL (환경변수로 관리 추천)
app:
  base-url: ${BASE_URL:http://localhost:8080}
  
payment:
  pg:
    merchant-id: test_merchant
    secret-key: ${PG_SECRET_KEY}
```

### 3. 결제 요청 페이지 (Thymeleaf)

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>결제 테스트</title>
</head>
<body>
    <h1>결제 테스트</h1>
    
    <form id="paymentForm" method="post" action="https://pg.settlebank.co.kr/payment">
        <input type="hidden" name="MID" value="test_merchant">
        <input type="hidden" name="OrderNo" th:value="${orderNo}">
        <input type="hidden" name="Amt" th:value="${amount}">
        <input type="hidden" name="GoodsName" value="테스트 상품">
        
        <!-- ngrok URL 사용 -->
        <input type="hidden" name="ReturnURL" 
               th:value="${@environment.getProperty('app.base-url')} + '/payment/callback'">
        <input type="hidden" name="ResultURL" 
               th:value="${@environment.getProperty('app.base-url')} + '/payment/result'">
        
        <button type="submit">결제하기</button>
    </form>
</body>
</html>
```

### 4. ngrok 실행 및 환경변수 설정

```bash
# 1. ngrok 실행
ngrok http 8080

# 2. 출력된 URL 확인 (예: https://abc123.ngrok-free.app)

# 3. Spring Boot 재시작 (환경변수 포함)
BASE_URL=https://abc123.ngrok-free.app ./mvnw spring-boot:run
```

또는 IntelliJ IDEA에서:
```
Run → Edit Configurations → Environment Variables
BASE_URL=https://abc123.ngrok-free.app
```

## ngrok 고급 기능

### 1. 요청 검사 (Inspect)

ngrok 실행 시 제공되는 Web Interface 활용:
```
http://127.0.0.1:4040
```

여기서 모든 HTTP 요청/응답을 실시간으로 확인 가능!

### 2. 포트 지정
```bash
# 다른 포트 사용
ngrok http 8081

# 특정 도메인 사용 (유료 플랜)
ngrok http 8080 --domain=myapp.ngrok.app
```

### 3. 기본 인증 추가
```bash
ngrok http 8080 --basic-auth="username:password"
```

### 4. 설정 파일 사용

`~/.ngrok2/ngrok.yml`:
```yaml
version: "2"
authtoken: YOUR_AUTH_TOKEN
tunnels:
  spring-boot:
    proto: http
    addr: 8080
    inspect: true
  spring-boot-secure:
    proto: http
    addr: 8080
    auth: "admin:secret123"
```

실행:
```bash
ngrok start spring-boot
# 또는
ngrok start spring-boot-secure
```

### 5. 여러 터널 동시 실행

```bash
# API 서버
ngrok http 8080 --region=kr &

# 프론트엔드
ngrok http 3000 --region=kr &
```

## 보안 고려사항

### ⚠️ 주의할 점

1. **민감한 데이터 노출 방지**
```java
@RestController
public class SecurityController {
    
    @GetMapping("/admin")
    public String admin(HttpServletRequest request) {
        // ngrok 헤더 확인
        String ngrokHeader = request.getHeader("X-Forwarded-For");
        if (ngrokHeader != null) {
            // 개발 환경에서만 허용
            throw new AccessDeniedException("Admin access not allowed via ngrok");
        }
        return "admin page";
    }
}
```

2. **환경별 설정 분리**
```java
@Configuration
@Profile("local")
public class NgrokConfig {
    
    @Value("${app.base-url}")
    private String baseUrl;
    
    @PostConstruct
    public void init() {
        if (baseUrl.contains("ngrok")) {
            log.warn("⚠️  ngrok 사용 중 - 민감한 데이터 노출 주의!");
        }
    }
}
```

3. **운영 환경에서는 절대 사용 금지**
```yaml
# application-prod.yml
app:
  base-url: https://api.yourcompany.com
```

### ✅ 안전하게 사용하기

```java
@Component
public class RequestLoggingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // ngrok을 통한 요청인지 확인
        String host = httpRequest.getHeader("Host");
        if (host != null && host.contains("ngrok")) {
            log.info("ngrok 요청: {} {}", httpRequest.getMethod(), 
                     httpRequest.getRequestURI());
        }
        
        chain.doFilter(request, response);
    }
}
```

## 실무 팁

### 1. 고정 URL 사용 (유료 플랜)

무료 플랜은 재시작할 때마다 URL이 변경됩니다.
```bash
# 무료: 매번 다른 URL
https://a1b2c3.ngrok-free.app
https://x9y8z7.ngrok-free.app  # 재시작 시

# 유료: 고정 URL
https://myapp.ngrok.app  # 항상 동일
```

### 2. 팀원과 공유

```bash
# 터널 정보 확인
curl http://localhost:4040/api/tunnels

# 팀원에게 URL 공유
echo "API 테스트: https://abc123.ngrok-free.app/api/hello"
```

### 3. Docker와 함께 사용

```yaml
# docker-compose.yml
version: '3.8'
services:
  spring-boot:
    build: .
    ports:
      - "8080:8080"
    environment:
      - BASE_URL=${BASE_URL}
  
  ngrok:
    image: ngrok/ngrok:latest
    environment:
      - NGROK_AUTHTOKEN=${NGROK_AUTHTOKEN}
    command: http spring-boot:8080
    ports:
      - "4040:4040"
```

```bash
docker-compose up -d
```

### 4. GitHub Actions에서 테스트

```yaml
name: Integration Test with ngrok

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
      
      - name: Start Spring Boot
        run: ./mvnw spring-boot:run &
      
      - name: Install ngrok
        run: |
          wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
          tar xvzf ngrok-v3-stable-linux-amd64.tgz
          ./ngrok authtoken ${{ secrets.NGROK_AUTH_TOKEN }}
      
      - name: Start ngrok
        run: ./ngrok http 8080 &
      
      - name: Run tests
        run: ./mvnw test
```

## 대안 도구들

| 도구 | 특징 | 가격 |
|------|------|------|
| **ngrok** | 가장 안정적, 풍부한 기능 | 무료/유료 |
| **localtunnel** | 간단, 설치 빠름 | 무료 |
| **serveo** | SSH 기반, 설치 불필요 | 무료 |
| **Cloudflare Tunnel** | 무료, 고성능 | 무료 |

## 트러블슈팅

### 문제 1: CORS 에러

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://*.ngrok-free.app", "https://*.ngrok.app")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true);
    }
}
```

### 문제 2: 세션 유지 안됨

```yaml
# application.yml
server:
  servlet:
    session:
      cookie:
        same-site: none
        secure: true  # HTTPS 필수
```

### 문제 3: 무료 플랜 제한

```
ERR_NGROK_108: Tunnel has been rate limited
```

→ 유료 플랜 전환 또는 재시작 후 대기

## 마치며

ngrok은 로컬 개발 환경에서 외부 서비스와 연동 테스트를 할 때 매우 유용한 도구입니다.

**핵심 정리:**
- ✅ 로컬 서버를 외부에 즉시 공개 가능
- ✅ HTTPS 자동 제공
- ✅ 웹훅, 결제 콜백 등 실전 테스트 가능
- ⚠️ 보안 주의 필요 (개발 환경에서만!)
- ⚠️ 운영 환경에서는 절대 사용 금지

Spring Boot 프로젝트에서 PG 결제, OAuth 소셜 로그인, 웹훅 등을 테스트할 때 ngrok을 활용해보세요!

---

**참고 자료:**
- [ngrok 공식 문서](https://ngrok.com/docs)
- [Spring Boot 공식 문서](https://spring.io/projects/spring-boot)
- [ngrok Java SDK](https://github.com/ngrok/ngrok-java)
