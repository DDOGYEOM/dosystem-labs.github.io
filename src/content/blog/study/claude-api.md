---
title: 'Claude API 정리'
description: 'Claude API 기본 사용법 정리'
pubDate: '2026-11-25'
category: 'study' # mentoring 또는 study (저장 폴더도 같은 이름)
tags: ['claude', 'claude-api']
author: 'dongju-lee' # src/content/authors.json 의 키(아이디)
---


## 배경
 AI 모델 클로드(Claude)를 외부 앱, 웹사이트, 프로그램 등에서 코드로 호출해 사용할 수 있게 해주는 인터페이스로
 Claude에서는 두 가지 방법을 제공하며 아래와 같다

|  | Messages API | Claude Managed Agents |
| ----- | ----- | ----- |
| **무엇인가** | 모델에 직접 프롬프트를 보낼 수 있는 액세스 | 관리형 인프라에서 실행되는, 사전 구축되고 구성 가능한 에이전트 하네스 |
| **적합한 용도** | 커스텀 에이전트 루프 및 세밀한 제어 | 장기 실행 작업 및 비동기 작업 |

본문에서는 Messages API를 중점으로 다루며, 예시 코드는 **Anthropic Java SDK** 기준으로 작성한다

## API KEY 세팅
키 발급 과정은 생략 한다

### 조직과 워크스페이스
Claude Console은 **조직(Organization) → 워크스페이스(Workspace) → API 키** 구조로 되어 있음

| 단위 | 설명 |
| ----- | ----- |
| 조직 | 회사 계정. 결제, 멤버 관리, 전체 사용량 한도를 관리 |
| 워크스페이스 | 조직 안에서 프로젝트·환경별로 나눈 공간. 워크스페이스마다 비용·요청 한도를 따로 걸 수 있음 |
| API 키 | 보통 워크스페이스 단위로 발급함 (워크스페이스에 속하지 않은 키는 요청에 `anthropic-workspace-id` 헤더가 필요). 사용량과 비용도 키·워크스페이스 단위로 확인 가능 |

개발용과 운영용 워크스페이스를 나눠 두면 키 유출이나 비용 폭주가 생겨도 영향 범위를 줄일 수 있음

### 키 보관
`클라이언트 SDK`에서는 자동으로 환경변수 `ANTHROPIC_API_KEY` 키를 읽음.
키는 **발급할 때 한 번만 보여주므로** 바로 안전한 곳에 보관해야 함

**로컬: `.env` 파일**

```shell
# .env
ANTHROPIC_API_KEY=sk-ant-api03-...
```

Java는 `.env`를 자동으로 읽지 않으므로 실행 전에 환경변수로 올려줘야 함

```shell
# 터미널에서 실행할 때
set -a && source .env && set +a
./gradlew run
```

> 참고: IntelliJ에서는 Run Configuration의 `Environment variables`에 등록하면 됨.
> `.env`는 반드시 `.gitignore`에 추가해서 git에 올라가지 않게 해야 함

**운영: AWS SSM Parameter Store**

운영 서버에는 `.env` 파일을 두지 않고, SSM에 `SecureString`(암호화) 타입으로 저장한 뒤 앱이 시작될 때 읽어오도록 하고자 함.

- 서버(EC2, ECS 등)의 IAM 역할에 해당 파라미터 `ssm:GetParameter` 권한만 주면 됨
- 키를 교체할 때도 SSM 값만 바꾸고 앱을 재시작하면 되므로 코드 수정이 필요 없음

### 사용량·비용 한도
Console의 설정에서 한도를 걸어 두면 버그나 무한 루프로 호출이 폭주해도 비용이 한도 이상 나가지 않음

| 한도 | 설명 |
| ----- | ----- |
| 비용 한도 (spend limit) | 월 사용 금액 상한. 넘으면 요청이 거절됨 |
| 요청 한도 (rate limit) | 분당 요청 수·토큰 수 상한. 넘으면 429 에러 |

- 조직 전체 한도와 별개로 **워크스페이스별 한도**를 조직 한도보다 낮게 설정할 수 있음
- 한도에 가까워지면 알림을 받도록 설정해 두는 것을 권장

## 기본 요청 
기본 적인 요청 방법은 일반적인 API 요청과 유사함

| params | Java 빌더 메서드 | Type |
| ----- | ----- | ----- |
| model | `.model()` | `String` |
| max_tokens | `.maxTokens()` | `long` |
| messages | `.addUserMessage()` 등 | `String` 또는 블록 목록 |

> 참고: API 호출하는 클로드는 기본적으로 stateless 방식으로 대화 로그를 저장하지 않음

```groovy
// build.gradle
implementation("com.anthropic:anthropic-java:3.14.0")
```

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.messages.Message;
import com.anthropic.models.messages.MessageCreateParams;

// 환경변수 ANTHROPIC_API_KEY를 자동으로 읽음
AnthropicClient client = AnthropicOkHttpClient.fromEnv();

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .addUserMessage("지금 사용하고있는 모델의 정보와 버전 정보 출력")
    .build();

Message message = client.messages().create(params);
```

> 참고: `AnthropicClient`는 내부에 커넥션 풀을 가지고 있으므로 요청마다 만들지 말고 **하나를 만들어 재사용**해야 함

> 실제 응답 (JSON으로 출력)
```json 
{
  model: 'claude-opus-5-5',
  id: 'msg_011ChQxEbnwACiLDHDTGuStt',
  type: 'message',
  role: 'assistant',
  content: [
    {
      type: 'thinking',
      thinking: '',
      signature: '생략'
    },
    {
      type: 'text',
      text: '# 모델 정보\n' +
        '\n' +
        '| 항목 | 내용 |\n' +
        '|------|------|\n' +
        '| **모델 이름** | Claude |\n' +
        '| **개발사** | Anthropic |\n' +
        '| **모델 유형** | 대규모 언어 모델 (LLM) 기반 AI 어시스턴트 |\n' +
        '\n' +
        '## 버전 정보\n' +
        '\n' +
        '정확한 버전(예: Claude 3.5 Sonnet, Claude Opus 4 등)은 **확실하게 말씀드리기 어렵습니다.** 저는 제 버전을 직접 확인할 수 있는 시스템 정보에 접근할 수 없고, 현재 대화에도 버전이 명시되어 있지 않습니다. 추측으로 답하면 틀린 정보를 드릴 수 있습니다.\n' +
        '\n' +
        '## 정확한 버전 확인 방법\n' +
        '\n' +
        '- **웹/앱(claude.ai)**: 채팅 화면의 모델 선택 메뉴에서 확인\n' +
        '- **API 사용 시**: 요청에 지정한 `model` 파라미터 값 또는 응답 JSON의 `model` 필드 확인\n' +
        '  ```json\n' +
        '  {\n' +
        '    "model": "claude-...",\n' +
        '    ...\n' +
        '  }\n' +
        '  ```\n' +
        '- **타 서비스(플랫폼)를 통해 사용 중인 경우**: 해당 서비스의 설정이나 문서 확인\n' +
        '\n' +
        '더 궁금한 점이 있으면 말씀해 주세요!'
    }
  ],
  container: null,
  stop_reason: 'end_turn',
  stop_sequence: null,
  stop_details: null,
  usage: {
    input_tokens: 34,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
    cache_creation: { ephemeral_5m_input_tokens: 0, ephemeral_1h_input_tokens: 0 },
    output_tokens: 564,
    output_tokens_details: { thinking_tokens: 98 },
    service_tier: 'standard',
    inference_geo: 'global',
    speed: 'standard'
  },
  diagnostics: null,
  parsed_output: null
}
```

## 응답 읽기
주요 필드만 정리

| 필드 | Java | 설명 |
| ----- | ----- | ----- |
| content | `message.content()` | 응답 본문. 문자열이 아니라 **블록 목록** |
| stop_reason | `message.stopReason()` | 응답이 끝난 이유 (`end_turn`: 정상 종료, `max_tokens`: 길이 제한으로 잘림 등) |
| usage | `message.usage()` | 입력/출력 토큰 수. **과금 기준**이 되는 값 |

`content` 안의 블록은 `type`으로 종류를 구분함

| type | 설명 |
| ----- | ----- |
| text | 일반 텍스트 응답 |
| thinking | 모델의 사고 과정 블록 |
| tool_use | 도구 호출 요청 |

> 참고: `claude-opus-5-5`는 thinking이 항상 켜져 있어 `content` 앞쪽에 `thinking` 블록이 올 수 있음 (기본값에서는 내용이 빈 문자열로 옴).
> 그래서 `message.content().get(0)`처럼 첫 블록을 바로 읽으면 텍스트가 아닐 수 있음

텍스트만 꺼내려면 `text` 블록만 걸러서 읽음. 블록의 `.text()`는 `Optional`을 돌려주므로 텍스트 블록이 아니면 비어 있음

```java
String text = message.content().stream()
    .flatMap(block -> block.text().stream())
    .map(textBlock -> textBlock.text())
    .collect(Collectors.joining());

System.out.println(text);
System.out.println(message.stopReason());
System.out.println(message.usage());
```

`stop_reason`이 `max_tokens`라면 응답이 중간에 잘린 것이므로 `max_tokens` 값을 늘려서 다시 요청해야 함

## 에러 처리
API 호출이 실패하는 경우는 두 가지로 나뉨

1. **HTTP 에러**: 요청 자체가 실패함 → SDK가 예외(Exception)를 던짐
2. **비정상 종료**: 요청은 성공(HTTP 200)했지만 응답이 온전하지 않음 → `stop_reason`으로 확인

### HTTP 에러
주요 에러 코드 정리 (예외 클래스는 `com.anthropic.errors` 패키지)

| 상태 코드 | SDK 예외 클래스 | 원인 | 재시도 |
| ----- | ----- | ----- | ----- |
| 400 | `BadRequestException` | 요청 형식이나 파라미터가 잘못됨 | X |
| 401 | `UnauthorizedException` | API 키가 없거나 잘못됨 | X |
| 403 | `PermissionDeniedException` | 해당 키로 허용되지 않는 요청 | X |
| 404 | `NotFoundException` | 모델 ID 오타 등 리소스를 찾을 수 없음 | X |
| 429 | `RateLimitException` | 요청 한도 초과 | O |
| 500 이상 | `InternalServerException` | 서버 문제. 하위 클래스로 504 `GatewayTimeoutException`, 529(일시적 과부하) `OverloadedException`이 있음 | O |
| - | `AnthropicIoException` | 네트워크 연결 실패 | O |

에러 메시지 문자열을 비교하지 말고 SDK의 예외 클래스로 구분함.
HTTP 에러 예외는 모두 `AnthropicServiceException`을 상속하므로 **구체적인 클래스부터** `catch` 해야 함

```java
try {
    Message message = client.messages().create(params);
} catch (UnauthorizedException e) {
    System.err.println("API 키를 확인하세요");
} catch (BadRequestException e) {
    System.err.println("요청 파라미터 오류: " + e.getMessage());
} catch (RateLimitException e) {
    System.err.println("요청 한도 초과, 잠시 후 다시 시도하세요");
} catch (AnthropicServiceException e) {
    System.err.println("API 에러 " + e.statusCode() + ": " + e.getMessage());
} catch (AnthropicIoException e) {
    System.err.println("네트워크 연결 실패");
}
```

### 타임아웃과 재시도
SDK는 408, 409, 429, 5xx, 네트워크 에러, 타임아웃을 **기본 2번 자동 재시도**함 (대기 시간은 0.5초~8초 사이로 점점 늘리고, 응답에 `retry-after` 헤더가 있으면 그 값을 따름).
타임아웃 기본값은 **10분**이고, Java SDK에서는 `Duration`으로 지정함

```java
// 클라이언트 전체 설정
AnthropicClient client = AnthropicOkHttpClient.builder()
    .fromEnv()
    .maxRetries(5)                       // 재시도 횟수 (0이면 재시도 안 함)
    .timeout(Duration.ofSeconds(60))     // 60초
    .build();

// 요청 하나만 따로 설정 (두 번째 인자)
Message message = client.messages().create(
    params,
    RequestOptions.builder().timeout(Duration.ofSeconds(10)).build()
);
```

- 타임아웃도 재시도 대상이라 **최대 대기 시간**은 `timeout × (maxRetries + 1)`까지 늘어날 수 있음
- 응답이 긴 요청(`max_tokens`가 큰 경우)은 타임아웃을 줄이기보다 스트리밍을 쓰는 게 맞음
- 429가 자주 나면 재시도 횟수를 늘리기보다 동시 요청 수를 줄이거나 조직의 요청 한도(rate limit)를 확인해야 함

### stop_reason으로 비정상 종료 확인
HTTP 200으로 응답이 와도 예외가 나지 않을 뿐, 응답이 잘렸거나 거절된 것일 수 있음.
그래서 `content`를 읽기 전에 `stop_reason`을 먼저 확인해야 함

| stop_reason | 의미 | 처리 |
| ----- | ----- | ----- |
| end_turn | 정상 종료 | 그대로 사용 |
| max_tokens | `max_tokens` 한도에 걸려 응답이 중간에 잘림 | `max_tokens`를 늘려 다시 요청 |
| stop_sequence | 지정한 `stop_sequences` 문자열을 만나 멈춤 | 의도한 종료이므로 그대로 사용 |
| tool_use | 모델이 도구 호출을 요청함 | 도구 실행 후 결과를 보내 대화를 이어감 |
| pause_turn | 서버 도구 실행 중 일시 정지됨 | 응답을 그대로 다시 보내 이어서 진행 |
| refusal | 안전 정책에 따라 응답을 거절함 | `stop_details`로 사유 확인 |
| model_context_window_exceeded | 모델의 컨텍스트 윈도우 한도에 도달해서 멈춤 | 입력(대화 기록 등)을 줄여서 다시 요청 |

```java
String stopReason = message.stopReason().map(StopReason::asString).orElse("");

switch (stopReason) {
    case "end_turn", "stop_sequence" -> {
        // 정상 응답
    }
    case "max_tokens" -> System.err.println("응답이 잘렸음. max_tokens를 늘려서 다시 요청 필요");
    case "refusal" -> {
        // stopDetails는 refusal일 때만 값이 있음
        message.stopDetails().ifPresent(d ->
            System.err.println("요청 거절됨: " + d.category() + " " + d.explanation()));
    }
    default -> System.err.println("처리하지 않은 stop_reason: " + stopReason);
}
```

## OUTPUT 구조화 하기
프롬프트에 "JSON으로 답해줘"라고만 쓰면 설명 문장이 붙거나 필드가 빠지는 등 형식이 깨질 수 있음.
**Structured Outputs** 기능을 쓰면 응답이 지정한 JSON 스키마를 반드시 따르도록 강제할 수 있음

방법은 두 가지

| 방법 | 설명 |
| ----- | ----- |
| `OutputConfig` + `JsonOutputFormat` | JSON 스키마를 직접 작성. 응답은 문자열이라 Jackson 등으로 직접 파싱 필요 |
| `.outputConfig(클래스)` | record/클래스로 정의. SDK가 스키마 생성, 파싱까지 해줌 (**권장**) |

> 참고: 예전 문서에 나오는 `output_format` 파라미터는 deprecated 되었으므로 `output_config.format`을 사용

### 방법 1: JSON 스키마 직접 작성

```java
JsonOutputFormat.Schema schema = JsonOutputFormat.Schema.builder()
    .putAdditionalProperty("type", JsonValue.from("object"))
    .putAdditionalProperty("properties", JsonValue.from(Map.of(
        "model_name", Map.of("type", "string"),
        "developer", Map.of("type", "string"),
        "model_type", Map.of("type", "string"),
        "version", Map.of("type", "string")
    )))
    .putAdditionalProperty("required", JsonValue.from(
        List.of("model_name", "developer", "model_type", "version")))
    .putAdditionalProperty("additionalProperties", JsonValue.from(false))
    .build();

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .addUserMessage("지금 사용하고있는 모델의 정보와 버전 정보 출력")
    .outputConfig(OutputConfig.builder()
        .format(JsonOutputFormat.builder().schema(schema).build())
        .build())
    .build();

Message message = client.messages().create(params);

// 응답 text가 JSON 문자열이므로 직접 파싱
String json = message.content().stream()
    .flatMap(block -> block.text().stream())
    .map(textBlock -> textBlock.text())
    .findFirst()
    .orElseThrow();
JsonNode data = new ObjectMapper().readTree(json);
System.out.println(data.get("model_name").asText()); // Claude
```

```json
{
  model: 'claude-opus-5-5',
  id: 'msg_011ChQyvueU7LZeJwF8avuNG',
  type: 'message',
  role: 'assistant',
  content: [
    {
      type: 'text',
      text: '{"model_name":"Claude","developer":"Anthropic","model_type":"대규모 언어 모델 (Large Language Model, AI 어시스턴트)","version":"정확한 버전 정보는 확인할 수 없습니다"}'
    }
  ],
  ~ 생략
}
```


### 방법 2: 클래스로 정의 (권장)
record(또는 클래스)를 넘기면 SDK가 JSON 스키마를 자동으로 만들고, 응답도 그 타입으로 파싱해서 돌려줌

```java
import com.anthropic.models.messages.StructuredMessageCreateParams;

record ModelInfo(
    @JsonProperty("model_name") String modelName,
    String developer,
    @JsonProperty("model_type") String modelType,
    String version
) {}

StructuredMessageCreateParams<ModelInfo> params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .addUserMessage("지금 사용하고있는 모델의 정보와 버전 정보 출력")
    .outputConfig(ModelInfo.class) // 타입이 지정된 빌더로 바뀜
    .build();

client.messages().create(params).content().stream()
    .flatMap(block -> block.text().stream())
    .forEach(typed -> {
        ModelInfo info = typed.text(); // String이 아니라 ModelInfo
        System.out.println(info.modelName());
    });
```

> 참고: 필드 설명이 필요하면 `@JsonPropertyDescription("모델 이름")`처럼 Jackson 어노테이션을 붙이면 스키마에 같이 들어감

### 주의할 점
- 모든 `object`에 `additionalProperties: false`가 필요함 (스키마를 직접 작성할 때는 빠뜨리지 않도록 주의)
- 재귀 스키마, 숫자 범위(`minimum`, `maximum`), 문자열 길이(`minLength`, `maxLength`) 같은 제약은 지원하지 않음
- 새 스키마로 처음 요청할 때는 스키마 컴파일 때문에 조금 느림. 같은 스키마는 마지막으로 사용한 시점부터 24시간 동안 캐시됨 (쓸 때마다 24시간이 다시 시작됨)
- `stop_reason`이 `max_tokens`이거나 `refusal`이면 스키마를 따르지 않는 응답이 올 수 있으므로 **파싱 전에 `stop_reason`을 먼저 확인**해야 함 (에러 처리 참고)

## 프롬프트 캐싱
매 요청마다 보내는 긴 시스템 프롬프트, 문서, 이전 대화 같은 **반복되는 앞부분(prefix)을 서버에 캐싱**해서 다음 요청을 싸고 빠르게 만드는 기능.
캐시는 `tools` → `system` → `messages` 순서의 앞부분이 **완전히 같을 때만** 적중하므로, 안 바뀌는 내용은 앞에, 바뀌는 내용은 뒤에 둬야 함

```java
// 방법 1: 자동 캐싱 (권장) - 최상위에 넣으면 마지막 블록까지 알아서 캐싱
MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .cacheControl(CacheControlEphemeral.builder().build())
    .system("아주 긴 사내 문서 내용...")
    .addUserMessage("핵심 내용을 요약해줘")
    .build();

// 방법 2: 직접 지정 - 캐싱할 블록에 붙임 (요청당 최대 4개)
.systemOfTextBlockParams(List.of(
    TextBlockParam.builder()
        .text("아주 긴 사내 문서 내용...")
        .cacheControl(CacheControlEphemeral.builder()
            .ttl(CacheControlEphemeral.Ttl.TTL_1H) // 생략하면 기본 TTL 5분
            .build())
        .build()))
```

| 구분 | 가격 (기본 입력 대비) |
| ----- | ----- |
| 캐시 쓰기 | 1.25배 (TTL 1시간은 2배) |
| 캐시 읽기 | 약 0.1배 (모델마다 다름, 예: `claude-opus-5-5`·`claude-sonnet-5-5`는 0.05배, `claude-fable-5-2`·`claude-mythos-5-2`·`claude-fable-5-1`·`claude-mythos-5-1`은 0.025배) |

적중 여부는 응답의 `message.usage().cacheReadInputTokens()`로 확인. 같은 요청을 두 번 보냈을 때 0보다 크면 적중한 것

### 주의할 점
- 최소 길이(`claude-opus-5-5`는 512 토큰)보다 짧으면 에러 없이 **조용히 캐싱되지 않음**
- 시스템 프롬프트에 현재 시간, UUID 등을 넣으면 매번 앞부분이 달라져 계속 캐시 미스

## 파일
PDF, 이미지 같은 파일을 Claude에게 보내는 방법은 세 가지

| 방법 | 설명 |
| ----- | ----- |
| base64 | 파일 내용을 base64 문자열로 요청에 직접 넣음. 매 요청마다 파일 전체를 다시 보냄 |
| URL | 공개된 URL을 넘기면 서버가 직접 가져감 |
| Files API | 파일을 한 번 업로드하고 받은 `file_id`로 여러 요청에서 재사용 (**권장**) |

### 방법 1: base64로 직접 첨부
한 번만 쓰는 작은 파일이면 업로드 없이 바로 보내도 됨

```java
String data = Base64.getEncoder().encodeToString(Files.readAllBytes(Path.of("report.pdf")));

DocumentBlockParam doc = DocumentBlockParam.builder()
    .source(Base64PdfSource.builder().data(data).build())
    .build();

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .addUserMessageOfContentBlockParams(List.of(
        ContentBlockParam.ofDocument(doc),
        ContentBlockParam.ofText(TextBlockParam.builder().text("이 문서를 요약해줘").build())
    ))
    .build();
```

> 참고: 이미지는 `ImageBlockParam` + `ContentBlockParam.ofImage(...)`를 사용. URL은 `.source(UrlPdfSource.builder().url("https://...").build())`

### 방법 2: Files API
파일을 업로드하면 `file_id`가 발급되고, 이후 요청에서는 id만 넘기면 됨.
같은 파일로 여러 번 질문할 때 매번 파일을 다시 보내지 않아도 됨

```java
import com.anthropic.models.files.FileMetadata;
import com.anthropic.models.files.FileUploadParams;

// 1. 업로드
FileMetadata uploaded = client.files().upload(
    FileUploadParams.builder()
        .file(Path.of("report.pdf"))
        .build());
System.out.println(uploaded.id()); // file_011C...

// 2. file_id로 첨부
DocumentBlockParam doc = DocumentBlockParam.builder()
    .fileSource(uploaded.id())
    .build();

MessageCreateParams params = MessageCreateParams.builder()
    .model("claude-opus-5-5")
    .maxTokens(1024L)
    .addUserMessageOfContentBlockParams(List.of(
        ContentBlockParam.ofDocument(doc),
        ContentBlockParam.ofText(TextBlockParam.builder().text("이 문서를 요약해줘").build())
    ))
    .build();
```

파일 종류에 따라 content 블록의 `type`을 맞춰야 함

| 파일 | MIME 타입 | 블록 type |
| ----- | ----- | ----- |
| PDF | `application/pdf` | `document` |
| 텍스트 | `text/plain` | `document` |
| 이미지 | `image/jpeg`, `image/png`, `image/gif`, `image/webp` | `image` |

> 참고: .docx, .xlsx는 바로 못 보내므로 PDF나 텍스트로 변환해서 보내야 함

### 크기 제한

| 항목 | 제한 |
| ----- | ----- |
| 요청 하나 전체 크기 | 128MB (1st-party Claude API 기준, 플랫폼마다 다름. base64로 넣으면 원본보다 약 1.33배 커짐) |
| PDF 페이지 수 | 요청당 600페이지 (컨텍스트 1M 미만이면 100페이지) |
| Files API 업로드 | 파일 하나 500MB |

- Files API로 올려도 **메시지에 첨부할 때는 요청 크기·페이지 제한이 그대로 적용**됨
- 제한을 넘는 PDF는 페이지 단위로 나눠서 여러 번 요청해야 함

### 스캔 PDF
Claude는 PDF의 각 페이지를 **텍스트 추출 + 페이지 이미지**로 함께 읽음.
그래서 글자가 이미지로만 들어있는 스캔 PDF도 별도 OCR 없이 일반 PDF와 같은 방법으로 보내면 됨

- 페이지마다 이미지 토큰이 추가되므로 같은 분량의 텍스트보다 **토큰을 더 많이 씀**
- 스캔 품질이 낮거나(흐림, 기울어짐) 손글씨가 많으면 인식 정확도가 떨어질 수 있음

파일 관리

```java
client.files().list();                     // 목록 조회
client.files().retrieveMetadata(fileId);   // 정보 조회
client.files().delete(fileId);             // 삭제
```

### 주의할 점
- 파일 하나 최대 500MB, 조직당 1TB까지 저장 가능
- 업로드, 조회, 삭제는 무료. 메시지에 첨부하면 그 내용만큼 **입력 토큰으로 과금**됨
- 파일은 직접 삭제하거나 만료 시각(`expires_at`)이 지날 때까지 남아 있음. 업로드할 때 `expires_in_seconds`로 만료 시간을 지정할 수 있음
- 예전 문서의 `client.beta().files()` + `files-api-2025-04-14` 헤더는 이제 필요 없음. `client.files()`를 사용
