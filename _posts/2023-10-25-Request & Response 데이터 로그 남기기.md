---
title: Request & Response 데이터 로그 남기기
date: 2023-10-25 13:00 +0900
aliases: null
tags:
  - Spring
  - Spring AOP 로깅
  - MDC
  - ThreadLocal
  - ContentCachingRequestWrapper
  - ContentCachingResponseWrapper
  - Filter Interceptor
  - Logback
  - TaskDecorator
  - Async Event 로깅
  - 비동기 로깅
image: /assets/img/thumbnails/Spring.png
categories:
  - Skill
  - Spring
---

어떤 프로덕트를 개발할 때 `로깅`의 개념은 굉장히 중요하다<br>
로그는 Application, Network, ...등에서 발생하는 `모든 이벤트에 대한 기록`이다<br>
이러한 로그를 바탕으로 `모니터링, 오류 추적, ..`등을 진행할 수 있다

## WAS 요청/응답 흐름

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img1.png" alt="img"/>
</div>

위의 흐름은 WAS로 요청이 들어오고 응답이 될때까지의 간략한 흐름이다<br>
필자는 다음과 같이 로깅 메커니즘을 구현할 예정이다

- Spring AOP 메커니즘을 활용해서 전역적인 컴포넌트의 In/Out 데이터 로깅
- MDC 기반 요청별 흐름 추적

### MDC (Mapped Diagnostic Context)

일반적으로 웹 애플리케이션은 `멀티 쓰레드` 기반으로 Client의 요청을 처리한다<br>
이 과정에서 Client들의 요청을 `구분지을 수 있는 수단`없이 단순하게 로그만 남긴다면 해당 로그들이 어떤 요청을 처리하는 도중에 발생한 로그인지 파악하기 힘들 것이다

따라서 이러한 문제를 해결하기 위해서 `특정 요청에 대한 로그`들을 하나의 문맥으로 간주할 수 있으면 파악에 용이하다

> MDC는 `ThreadLocal`을 활용해서 현재 실행중인 문맥에 대해서 여러 정보들을 관리할 수 있는 공간이다

<br>

## 1. Spring AOP를 활용한 전역 컴포넌트 데이터 로깅 전략

간략하게 다음 4가지 컴포넌트를 통해서 AOP 메커니즘을 활용한 로깅 전략을 구축할 예정이다

### 1) 흐름 DepthLevel & ExecuteTime을 관리하는 `LoggingStatus`

```kotlin
class LoggingStatus(
    private val startTimeMillis: Long = System.currentTimeMillis(),
    private var depthLevel: Int = 0,
) {
    fun increaseDepth() {
        depthLevel++
    }

    fun decreaseDepth() {
        depthLevel--
    }

    fun depthPrefix(prefixString: String): String {
        if (depthLevel == 1) {
            return "|$prefixString"
        }
        val bar: String = "|" + " ".repeat(prefixString.length)
        return bar.repeat(depthLevel - 1) + "|$prefixString"
    }

    fun calculateTakenTime(): Long = System.currentTimeMillis() - startTimeMillis
}
```

### 2) Thread별로 LoggingStatus를 관리하는 `LoggingStatusManager`

```kotlin
@Component
class LoggingStatusManager {
    private val statusContainer = ThreadLocal<LoggingStatus>()

    fun getExistLoggingStatus(): LoggingStatus {
        return statusContainer.get()
            ?: throw IllegalStateException("ThreadLocal LoggingStatus not exists...")
    }

    fun syncStatus() {
        val status: LoggingStatus? = statusContainer.get()
        if (status == null) {
            statusContainer.set(LoggingStatus())
        }
    }

    fun clearResource() = statusContainer.remove()
}
```

### 3) 실질적으로 로그를 남기는 `LoggingTracer`

```kotlin
@Component
class LoggingTracer(
    private val loggingStatusManager: LoggingStatusManager,
) {
    private val log: Logger = logger()

    fun methodCall(
        methodSignature: String,
        args: Array<Any?>,
    ) {
        loggingStatusManager.syncStatus()

        val loggingStatus: LoggingStatus = loggingStatusManager.getExistLoggingStatus()
        loggingStatus.increaseDepth()

        if (log.isInfoEnabled) {
            log.info(
                "{} args={}",
                loggingStatus.depthPrefix(REQUEST_PREFIX) + methodSignature,
                args,
            )
        }
    }

    fun methodReturn(methodSignature: String) {
        val loggingStatus: LoggingStatus = loggingStatusManager.getExistLoggingStatus()

        if (log.isInfoEnabled) {
            log.info(
                "{} time={}ms",
                loggingStatus.depthPrefix(RESPONSE_PREFIX) + methodSignature,
                loggingStatus.calculateTakenTime(),
            )
        }
        loggingStatus.decreaseDepth()
    }

    fun throwException(
        methodSignature: String,
        exception: Throwable,
    ) {
        val loggingStatus: LoggingStatus = loggingStatusManager.getExistLoggingStatus()

        if (log.isInfoEnabled) {
            log.info(
                "{} time={}ms ex={}",
                loggingStatus.depthPrefix(EXCEPTION_PREFIX) + methodSignature,
                loggingStatus.calculateTakenTime(),
                exception.toString(),
            )
        }
        loggingStatus.decreaseDepth()
    }

    companion object {
        private const val REQUEST_PREFIX = "--->"
        private const val RESPONSE_PREFIX = "<---"
        private const val EXCEPTION_PREFIX = "<X--"
    }
}
```

### 4) 전역적인 AOP 메커니즘을 적용한 `LoggingAop`

```kotlin
@Aspect
@Component
class LoggingAop(
    private val loggingTracer: LoggingTracer,
) {
    @Pointcut("execution(* com.sjiwon.logging..*(..))")
    private fun includeComponent() {
    }

    @Pointcut(
        """
            !execution(* com.sjiwon.logging.global.config..*(..))
            && !execution(* com.sjiwon.logging.global.decorator..*(..))
            && !execution(* com.sjiwon.logging.global.filter..*(..))
            && !execution(* com.sjiwon.logging.global.log..*(..))
        """,
    )
    private fun excludeComponent() {
    }

    @Around("includeComponent() && excludeComponent()")
    fun doLogging(joinPoint: ProceedingJoinPoint): Any? {
        val methodSignature: String = joinPoint.signature.toShortString()
        val args: Array<Any?> = joinPoint.args
        loggingTracer.methodCall(methodSignature, args)
        try {
            val result: Any? = joinPoint.proceed()
            loggingTracer.methodReturn(methodSignature)
            return result
        } catch (e: Throwable) {
            loggingTracer.throwException(methodSignature, e)
            throw e
        }
    }
}
```

<br>

## 2. MDC 기반 요청별 흐름 추적 Filter

```kotlin
enum class MdcKey {
    REQUEST_ID,
    REQUEST_IP,
    REQUEST_METHOD,
    REQUEST_URI,
    REQUEST_PARAMS,
    REQUEST_TIME,
    START_TIME_MILLIS,
}

class MdcLoggingFilter : Filter {
    override fun doFilter(
        request: ServletRequest,
        response: ServletResponse,
        chain: FilterChain,
    ) {
        setMdc(request as HttpServletRequest)
        chain.doFilter(request, response)
        MDC.clear()
    }

    private fun setMdc(request: HttpServletRequest) {
        MDC.put(REQUEST_ID.name, UUID.randomUUID().toString())
        MDC.put(REQUEST_IP.name, getClientIP(request))
        MDC.put(REQUEST_METHOD.name, getHttpMethod(request))
        MDC.put(REQUEST_URI.name, getRequestUriWithQueryString(request))
        MDC.put(REQUEST_PARAMS.name, getSeveralParamsViaParsing(request))
        MDC.put(REQUEST_TIME.name, LocalDateTime.now().toString())
        MDC.put(START_TIME_MILLIS.name, System.currentTimeMillis().toString())
    }
}
```

<br>

## 3. Request & Response 데이터 캐싱

### 데이터 캐싱의 필요성 (with Stream)

먼저 아래 4가지 케이스에 대해서 데이터를 중복으로 읽을 수 있는지 테스트해보자

```kotlin
data class Data(
    val id: Long,
    val name: String,
)
```

#### 1) QueryString

```kotlin
class QueryStringInterceptor : HandlerInterceptor {
    private val log: Logger = logger()

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
    ): Boolean {
        log.info("[Interceptor] ID=${request.getParameter("id")}, Name=${request.getParameter("name")}")
        return true
    }
}

@GetMapping("/api/v1")
fun queryString(@ModelAttribute data: Data): Data {
    log.info("[Controller] ID=${data.id}, Name=${data.name}")
    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img3.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img4.png" alt="img"/>
</div>

- QueryString으로 넘어오는 데이터들은 여러번 읽어도 정상적으로 읽을 수 있다

#### 2) FormData

```kotlin
class FormDataInterceptor : HandlerInterceptor {
    private val log: Logger = logger()

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
    ): Boolean {
        log.info("[Interceptor] ID=${request.getParameter("id")}, Name=${request.getParameter("name")}")
        return true
    }
}

@PostMapping("/api/v2")
fun formData(@ModelAttribute data: Data): Data {
    log.info("[Controller] ID=${data.id}, Name=${data.name}")
    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img5.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img6.png" alt="img"/>
</div>

- `application/x-www-form-urlencoded`로 넘어오는 데이터들은 여러번 읽어도 정상적으로 읽을 수 있다

#### 3) HTTP Request Body

```kotlin
class HttpRequestInterceptor(
    private val objectMapper: ObjectMapper,
) : HandlerInterceptor {
    private val log: Logger = logger()

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
    ): Boolean {
        val data: TestController.Data = objectMapper.readValue(request.inputStream)
        log.info("[Interceptor] ID=${data.id}, Name=${data.name}")
        return true
    }
}

@PostMapping("/api/v3")
fun request(@RequestBody data: Data): Data {
    log.info("[Controller] ID=${data.id}, Name=${data.name}")
    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img7.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img8.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img9.png" alt="img"/>
</div>

1. Interceptor에서 Request Data 읽기 성공
2. ArgumentResolver에서 Request Data를 Binding하는 과정에서 HttpMessageNotReadableException 발생

- I/O error while reading input message (with stream closed)

> Stream 데이터는 기본적으로 `단방향`으로 흘러가고 `한번 read한것은 다시 read할 수 없다`
{: .prompt-info }

이러한 이유로 인해 Interceptor에서 Request Body Data를 읽어버리면 이 Stream Data는 소비된 Data이고 따라서 이후에는 절대로 다시 읽을 수 없다

#### 4) HTTP Response Body

```kotlin
class HttpResponseInterceptor(
    private val objectMapper: ObjectMapper,
) : HandlerInterceptor {
    private val log: Logger = logger()

    override fun afterCompletion(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
        ex: Exception?,
    ) {
        val data: TestController.Data = objectMapper.readValue(request.inputStream)
        log.info("[Interceptor] ID=${data.id}, Name=${data.name}")
    }
}

@PostMapping("/api/v4")
fun response(@RequestBody data: Data): Data {
    log.info("[Controller] ID=${data.id}, Name=${data.name}")
    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img10.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img11.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img12.png" alt="img"/>
</div>

- 마찬가지로 `응답 데이터를 JSON으로 Deserialize할 때 읽을 데이터(Token)가 없기 때문에` 에러가 발생한다

### Spring에서 제공해주는 캐싱 컴포넌트

위에서 살펴봤듯이 InputStream & OutputStream은 어쨌든 Stream Data이고 이는 단방향이기 때문에 한번 read하거나 write했다면 그 이후에는 read나 write가 불가능하다<br>
이를 해결하기 위해서 직접 구현할 수도 있지만 Spring에서는 이러한 메커니즘을 구현한 구현체를 제공해준다

- 요청 데이터 캐싱 = ContentCachingRequestWrapper
- 응답 데이터 캐싱 = ContentCachingResponseWrapper

이를 활용해서 요청 & 응답 데이터를 캐싱하는 Filter를 구현해보자

```kotlin
class RequestResponseCachingFilter : OncePerRequestFilter() {
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val requestWrapper = ContentCachingRequestWrapper(request)
        val responseWrapper = ContentCachingResponseWrapper(response)
        filterChain.doFilter(requestWrapper, responseWrapper)
        responseWrapper.copyBodyToResponse()
    }
}
```

> 하지만 `ContentCachingRequestWrapper`의 경우 내부 캐싱 메커니즘을 이해하지 못하면 요청 데이터가 캐싱되지 않는 현상을 파악할 수 없다 
{: .prompt-danger }

#### ContentCachingRequestWrapper의 캐싱 메커니즘

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img13.png" alt="img"/>
</div>

ContentCachingRequestWrapper에서 caching되는 stream data를 읽기 위해서는 전제조건이 존재한다

> 요청 컨텐츠가 `반드시 소비`가 되어야 캐싱된다

- 이 말은 `요청 데이터를 1번 이상은 건드려야 캐싱`이 되고 캐싱 데이터를 읽을 수 있다는 것이다

<br>
현재 구현하고자 하는 로깅 구조는 다음과 같다

1. Filter에서 MdcKey 기반 MDC Map 설정 - MdcLoggingFilter
2. Filter에서 요청 데이터 캐싱 - RequestResponseCachingFilter
3. Filter's doFilter에서 Request 데이터 로깅 - RequestLoggingFilter
4. ...
5. ArgumentResolver에서 @RequestBody를 활용한 데이터 바인딩
6. ...
7. Filter에서 응답 데이터 캐싱 - RequestResponseCachingFilter
8. Filter's doFilter에서 Response 데이터 로깅 - RequestLoggingFilter

<br>
이러한 과정에서 Request 데이터를 로깅하는 것은 아래와 같은 2가지 점으로 인해 불가능하다

1. 캐싱없이 읽음 = ArgumentResolver에서 `Stream closed`로 인해 데이터 바인딩 실패
2. 캐싱 적용 = RequestResponseCachingFilter의 `ContentCachingRequestWrapper`은 이전에 최소 1번은 읽어야 캐싱되는데 여기서 처음 읽으니까 캐싱 불가능

<br>
간단한 테스트를 통해서 최소 1번은 읽혀야 실제로 캐싱되는지 확인해보자

```kotlin
class HttpCachingInterceptorV1(
    private val objectMapper: ObjectMapper,
) : HandlerInterceptor {
    private val log: Logger = logger()

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
    ): Boolean {
        val wrapper = request as ContentCachingRequestWrapper
        val wrapperData = objectMapper.readTree(wrapper.contentAsByteArray)
        log.info("[Interceptor] Data=$wrapperData")
        return true
    }
}

@PostMapping("/api/caching/v1")
fun cachingV1(
    @RequestBody data: Data,
    request: HttpServletRequest,
): Data {
    log.info("[Controller] Data=$data")

    val wrapper = request as ContentCachingRequestWrapper
    val wrapperData = objectMapper.readTree(wrapper.contentAsByteArray)
    log.info("[Controller - Caching] Data=$wrapperData")

    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img14.png" alt="img"/>
</div>

##### ✏️ 요청 데이터 CachingWrapper 커스터마이징

ContentCachingRequestWrapper로는 현재 구조에서 요청 데이터가 캐싱되지 않으므로 직접 커스터마이징하자

```kotlin
class ReadableRequestWrapper(
    request: HttpServletRequest,
) : HttpServletRequestWrapper(request) {
    private val params: MutableMap<String, Array<String>> = request.parameterMap
    private val encoding: Charset
    private val parts: Collection<Part>?
    val contentAsByteArray: ByteArray

    init {
        this.encoding = getEncoding(request.characterEncoding)
        this.parts = getMultipartParts(request)

        try {
            this.contentAsByteArray = request.inputStream.readAllBytes()
        } catch (e: Exception) {
            throw RuntimeException(e)
        }
    }

    private fun getEncoding(charEncoding: String): Charset {
        if (charEncoding.isBlank()) {
            return StandardCharsets.UTF_8
        }
        return Charset.forName(charEncoding)
    }

    private fun getMultipartParts(request: HttpServletRequest): Collection<Part>? {
        if (isMultipartRequest(request)) {
            return request.parts
        }
        return null
    }

    private fun isMultipartRequest(request: HttpServletRequest): Boolean {
        return !request.contentType.isNullOrBlank() && request.contentType.startsWith(MULTIPART_FORM_DATA_VALUE)
    }

    override fun getInputStream(): ServletInputStream {
        val byteArrayInputStream = ByteArrayInputStream(this.contentAsByteArray)

        return object : ServletInputStream() {
            override fun isFinished(): Boolean =
                throw UnsupportedOperationException("[ReadableRequestWrapper] isFinished() not supported")

            override fun isReady(): Boolean =
                throw UnsupportedOperationException("[ReadableRequestWrapper] isReady() not supported")

            override fun setReadListener(listener: ReadListener) = Unit

            override fun read(): Int = byteArrayInputStream.read()
        }
    }
}

class RequestResponseCachingFilter : OncePerRequestFilter() {
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val requestWrapper = ReadableRequestWrapper(request)
        val responseWrapper = ContentCachingResponseWrapper(response)
        filterChain.doFilter(requestWrapper, responseWrapper)
        responseWrapper.copyBodyToResponse()
    }
}
```

```kotlin
class HttpCachingInterceptorV2(
    private val objectMapper: ObjectMapper,
) : HandlerInterceptor {
    private val log: Logger = logger()

    override fun preHandle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        handler: Any,
    ): Boolean {
        val wrapper = request as ReadableRequestWrapper
        val wrapperData = objectMapper.readTree(wrapper.contentAsByteArray)
        log.info("[Interceptor] Data=$wrapperData")
        return true
    }
}

@PostMapping("/api/caching/v2")
fun cachingV2(
    @RequestBody data: Data,
    request: HttpServletRequest,
): Data {
    log.info("[Controller] Data=$data")

    val wrapper = request as ReadableRequestWrapper
    val wrapperData = objectMapper.readTree(wrapper.contentAsByteArray)
    log.info("[Controller - Caching] Data=$wrapperData")

    return data
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img15.png" alt="img"/>
</div>

- 드디어 원하는대로 요청 데이터가 캐싱됨을 확인할 수 있다

#### ContentCachingRequestWrapper의 캐싱 메커니즘

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img16.png" alt="img"/>
</div>
<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img17.png" alt="img"/>
</div>

> `ContentCachingResponseWrapper`는 RequestWrapper와는 달리 Caching을 진행하는 `copyBodyToResponse()`가 오픈되어 있으므로 Filter Return 과정에서 호출함에 따라 캐싱이 정상적으로 진행된다

<br>

## 4. Request & Response 데이터 로깅 Filter

```kotlin
class RequestLoggingFilter(
    private val loggingStatusManager: LoggingStatusManager,
    vararg ignoredUrls: String,
) : Filter {
    private val log: Logger = logger()

    private val ignoredUrls: MutableSet<String> = HashSet()

    init {
        this.ignoredUrls.addAll(listOf(*ignoredUrls))
    }

    override fun doFilter(
        request: ServletRequest,
        response: ServletResponse,
        chain: FilterChain,
    ) {
        val httpRequest = request as HttpServletRequest
        val httpResponse = response as HttpServletResponse

        if (CorsUtils.isPreFlightRequest(httpRequest) || isIgnoredUrl(httpRequest)) {
            chain.doFilter(request, response)
            return
        }

        val stopWatch = StopWatch()
        try {
            stopWatch.start()

            loggingStatusManager.syncStatus()
            loggingRequestInfo(httpRequest)

            chain.doFilter(request, response)
        } finally {
            stopWatch.stop()

            loggingResponseInfo(httpResponse, stopWatch)
            loggingStatusManager.clearResource()
        }
    }

    private fun isIgnoredUrl(request: HttpServletRequest): Boolean {
        return PatternMatchUtils.simpleMatch(ignoredUrls.toTypedArray(), request.requestURI)
    }

    private fun loggingRequestInfo(httpRequest: HttpServletRequest) {
        log.info(
            "[Request START] = [Task ID = {}, IP = {}, HTTP Method = {}, Uri = {}, Params = {}, 요청 시작 시간 = {}]",
            MDC.get(REQUEST_ID.name),
            MDC.get(REQUEST_IP.name),
            MDC.get(REQUEST_METHOD.name),
            MDC.get(REQUEST_URI.name),
            MDC.get(REQUEST_PARAMS.name),
            MDC.get(REQUEST_TIME.name),
        )
        log.info("Request Body = {}", readRequestData(httpRequest))
    }

    private fun readRequestData(request: HttpServletRequest): String {
        if (request is ReadableRequestWrapper) {
            val bodyContents: ByteArray = request.contentAsByteArray

            if (bodyContents.isEmpty()) {
                return EMPTY_RESULT
            }
            return String(bodyContents, StandardCharsets.UTF_8)
        }
        return EMPTY_RESULT
    }

    private fun loggingResponseInfo(
        httpResponse: HttpServletResponse,
        stopWatch: StopWatch,
    ) {
        log.info("Response Body = {}", readResponseData(httpResponse))
        log.info(
            "[Request END] = [Task ID = {}, IP = {}, HTTP Method = {}, Uri = {}, HTTP Status = {}, 요청 처리 시간 = {}ms]",
            MDC.get(REQUEST_ID.name),
            MDC.get(REQUEST_IP.name),
            MDC.get(REQUEST_METHOD.name),
            MDC.get(REQUEST_URI.name),
            httpResponse.status,
            stopWatch.totalTimeMillis,
        )
    }

    private fun readResponseData(response: HttpServletResponse): String {
        if (response is ContentCachingResponseWrapper) {
            val bodyContents: ByteArray = response.contentAsByteArray

            if (bodyContents.isEmpty()) {
                return EMPTY_RESULT
            }
            return createResponse(bodyContents)
        }
        return EMPTY_RESULT
    }

    private fun createResponse(bodyContents: ByteArray): String {
        val result = String(bodyContents, StandardCharsets.UTF_8)
        if (result.contains("</html>")) {
            return EMPTY_RESULT
        }
        return result
    }

    companion object {
        private const val EMPTY_RESULT = "{ Empty }"
    }
}
```

<br>

## 5. 빈 등록 & 필터 로깅 구조

```kotlin
@Configuration
class WebLogConfig {
    @Bean
    fun firstFilter(): FilterRegistrationBean<MdcLoggingFilter> {
        return FilterRegistrationBean<MdcLoggingFilter>().apply {
            order = 1
            filter = MdcLoggingFilter()
            setName("mdcLoggingFilter")
            addUrlPatterns("/api/*")
        }
    }

    @Bean
    fun secondFilter(): FilterRegistrationBean<RequestResponseCachingFilter> {
        return FilterRegistrationBean<RequestResponseCachingFilter>().apply {
            order = 2
            filter = RequestResponseCachingFilter()
            setName("requestResponseCachingFilter")
            addUrlPatterns("/api/*")
        }
    }

    @Bean
    fun thirdFilter(loggingStatusManager: LoggingStatusManager): FilterRegistrationBean<RequestLoggingFilter> {
        return FilterRegistrationBean<RequestLoggingFilter>().apply {
            order = 3
            filter = RequestLoggingFilter(loggingStatusManager, *ignoredUrl)
            setName("requestLoggingFilter")
            addUrlPatterns("/api/*")
        }
    }

    companion object {
        private val ignoredUrl: Array<String> = arrayOf(
            "/favicon.ico",
            "/error*",
        )
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img18.png" alt="img"/>
</div>

<br>

## 6. Logback 설정

```xml
<!-- logback-appender.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<included>
  <timestamp key="BY_DATE" datePattern="yyyy-MM-dd"/>
  <property name="LOG_FILE_PATH" value="./logs"/>
  <property name="ARCHIVE_FILE_PATH" value="./logs/archive"/>
  <property name="LOG_PATTERN"
            value="[%d{yyyy-MM-dd HH:mm:ss.SSS, ${logback.timezone:-Asia/Seoul}}] [%-5level] [%thread] [%logger{1}] [%X{REQUEST_ID:-NO REQUEST ID}] %msg %n"/>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <layout class="ch.qos.logback.classic.PatternLayout">
      <Pattern>${LOG_PATTERN}</Pattern>
    </layout>
  </appender>

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE_PATH}/${BY_DATE}.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>${ARCHIVE_FILE_PATH}/%d{yyyy-MM-dd, ${logback.timezone:-Asia/Seoul}}_%i.log
      </fileNamePattern>
      <maxHistory>30</maxHistory>
      <maxFileSize>100MB</maxFileSize>
      <totalSizeCap>100MB</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>${LOG_PATTERN}</pattern>
    </encoder>
  </appender>
</included>

<!-- logback-local.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="logback/logback-appender.xml"/>
  
  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
  </root>
</configuration>
```

- `[%X{REQUEST_ID:-NO REQUEST ID}]`를 통해서 MDC에 저장한 클라이언트별 고유한 REQUEST_ID를 적용한다

```yaml
logging:
  config: classpath:logback/logback-local.xml
```

<br>

## 로깅 실전 테스트

```kotlin
// Api
@RestController
class BookApi(
    private val bookService: BookService,
) {
    data class Request(
        val name: String,
    )

    @PostMapping("/api/v1/books")
    fun save(
        @RequestBody request: Request,
    ): Book = bookService.save(request.name)
}

// Service
@Service
class BookService(
    private val bookRepository: BookRepository,
) {
    fun save(name: String): Book {
        return bookRepository.save(Book(name = name))
    }
}

// Repository
interface BookRepository {
    fun save(book: Book): Book
}

@Repository
class MemoryBookRepository : BookRepository {
    private val datasource: MutableMap<Long, Book> = ConcurrentHashMap()

    override fun save(book: Book): Book {
        val idField: Field = book::class.java.getDeclaredField("id").apply {
            isAccessible = true
        }
        val idValue: Long = book.id
        if (idValue == 0L) {
            val identifier: Long = datasource.size.toLong() + 1
            idField.set(book, identifier)
        }
        datasource[book.id] = book
        return datasource[book.id]!!
    }
}
```

```text
[2024-03-30 16:47:37.505] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [1d43260b-8bbd-4e30-919e-74fa614b829e] [Request START] = [Task ID = 1d43260b-8bbd-4e30-919e-74fa614b829e, IP = 0:0:0:0:0:0:0:1, HTTP Method = POST, Uri = /api/v1/books, Params = [], 요청 시작 시간 = 2024-03-30T16:47:37.500111] 
[2024-03-30 16:47:37.507] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [1d43260b-8bbd-4e30-919e-74fa614b829e] Request Body = {"name": "Spring"} 
[2024-03-30 16:47:37.669] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |--->BookApi.save(..) args=[Request(name=Spring)] 
[2024-03-30 16:47:37.669] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |    |--->BookService.save(..) args=[Spring] 
[2024-03-30 16:47:37.669] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |    |    |--->MemoryBookRepository.save(..) args=[Book(id=0, name=Spring)] 
[2024-03-30 16:47:37.671] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |    |    |<---MemoryBookRepository.save(..) time=166ms 
[2024-03-30 16:47:37.671] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |    |<---BookService.save(..) time=166ms 
[2024-03-30 16:47:37.671] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [1d43260b-8bbd-4e30-919e-74fa614b829e] |<---BookApi.save(..) time=166ms 
[2024-03-30 16:47:37.704] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [1d43260b-8bbd-4e30-919e-74fa614b829e] Response Body = {"id":1,"name":"Spring"} 
[2024-03-30 16:47:37.704] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [1d43260b-8bbd-4e30-919e-74fa614b829e] [Request END] = [Task ID = 1d43260b-8bbd-4e30-919e-74fa614b829e, IP = 0:0:0:0:0:0:0:1, HTTP Method = POST, Uri = /api/v1/books, HTTP Status = 200, 요청 처리 시간 = 198ms] 
```

- 성공적으로 전역적인 Request & Response 데이터 + 컴포넌트 호출 구조를 로깅할 수 있다

<br>

## 비동기 호출 & TaskDecorator

위와 같은 로깅 구조에서 `비동기 처리` 역시 동일한 문맥을 보장받을 수 있을까?

```kotlin
@Configuration
@EnableAsync
class AsyncConfig

@RestController
class HelloApi(
    private val helloService: HelloService,
) {
    @GetMapping("/api/v1/async")
    fun async(): String {
        helloService.async()
        return "ok"
    }

    @GetMapping("/api/v1/event")
    fun event(): String {
        helloService.event()
        return "ok"
    }
}

@Service
class HelloService(
    private val eventPublisher: ApplicationEventPublisher,
) {
    @Async
    fun async() {
    }

    fun event() {
        eventPublisher.publishEvent(HelloEvent(id = 1L))
    }
}

@Component
class EventHandler {
    @Async
    @EventListener
    fun execute(event: HelloEvent) {
    }
}
```

```text
// @Async
[2024-03-30 17:03:14.473] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] [Request START] = [Task ID = 1b30f0bb-ba49-413c-ada6-0fabea70a2ea, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/async, Params = [], 요청 시작 시간 = 2024-03-30T17:03:14.467496200] 
[2024-03-30 17:03:14.474] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] Request Body = { Empty } 
[2024-03-30 17:03:14.500] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.l.LoggingTracer] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] |--->HelloApi.async() args=[] 
[2024-03-30 17:03:14.502] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.l.LoggingTracer] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] |<---HelloApi.async() time=29ms 
[2024-03-30 17:03:14.502] [INFO ] [task-1] [c.s.l.g.l.LoggingTracer] [NO REQUEST ID] |--->HelloService.async() args=[] 
[2024-03-30 17:03:14.503] [INFO ] [task-1] [c.s.l.g.l.LoggingTracer] [NO REQUEST ID] |<---HelloService.async() time=1ms 
[2024-03-30 17:03:14.516] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] Response Body = ok 
[2024-03-30 17:03:14.516] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [1b30f0bb-ba49-413c-ada6-0fabea70a2ea] [Request END] = [Task ID = 1b30f0bb-ba49-413c-ada6-0fabea70a2ea, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/async, HTTP Status = 200, 요청 처리 시간 = 42ms] 

// ApplicationEventPublisher
[2024-03-30 17:04:26.409] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [d6d5104c-df73-49f5-81c7-657ab813f7af] [Request START] = [Task ID = d6d5104c-df73-49f5-81c7-657ab813f7af, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/event, Params = [], 요청 시작 시간 = 2024-03-30T17:04:26.394100900] 
[2024-03-30 17:04:26.417] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [d6d5104c-df73-49f5-81c7-657ab813f7af] Request Body = { Empty } 
[2024-03-30 17:04:26.481] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [d6d5104c-df73-49f5-81c7-657ab813f7af] |--->HelloApi.event() args=[] 
[2024-03-30 17:04:26.488] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [d6d5104c-df73-49f5-81c7-657ab813f7af] |    |--->HelloService.event() args=[] 
[2024-03-30 17:04:26.488] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [d6d5104c-df73-49f5-81c7-657ab813f7af] |    |<---HelloService.event() time=79ms 
[2024-03-30 17:04:26.488] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.l.LoggingTracer] [d6d5104c-df73-49f5-81c7-657ab813f7af] |<---HelloApi.event() time=79ms 
[2024-03-30 17:04:26.488] [INFO ] [task-1] [c.s.l.g.l.LoggingTracer] [NO REQUEST ID] |--->EventHandler.execute(..) args=[HelloEvent(id=1)] 
[2024-03-30 17:04:26.488] [INFO ] [task-1] [c.s.l.g.l.LoggingTracer] [NO REQUEST ID] |<---EventHandler.execute(..) time=0ms 
[2024-03-30 17:04:26.521] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [d6d5104c-df73-49f5-81c7-657ab813f7af] Response Body = ok 
[2024-03-30 17:04:26.521] [INFO ] [http-nio-8080-exec-1] [c.s.l.g.f.RequestLoggingFilter] [d6d5104c-df73-49f5-81c7-657ab813f7af] [Request END] = [Task ID = d6d5104c-df73-49f5-81c7-657ab813f7af, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/event, HTTP Status = 200, 요청 처리 시간 = 114ms] 
```

- 로깅 결과로 알 수 있듯이 현재 구조에서 `비동기 처리`는 동일한 문맥을 제공해주지 않는다

### TaskDecorator

기본적으로 비동기 처리는 기존 쓰레드가 아닌 별도의 쓰레드에서 진행된다<br>
따라서 위의 로깅 메커니즘에서 적용한 `ThreadLocal` 역시 쓰레드가 전환되면 이용할 수 없는 저장소가 되버린다

Spring 4.3부터 제공되는 `TaskDecorator`를 활용하면 비동기 처리를 진행하는 전/후에 추가적인 작업을 부여할 수 있다

- 비동기 처리 전에 MDC 문맥에 대한 Copy를 진행하게 되면 비동기 처리를 하더라도 동일한 로깅 문맥을 이어나갈 수 있을듯 하다

```kotlin
class MdcTaskDecorator : TaskDecorator {
    override fun decorate(runnable: Runnable): Runnable {
        val context = MDC.getCopyOfContextMap()

        return Runnable {
            if (context != null) {
                MDC.setContextMap(context)
            }
            runnable.run()
        }
    }
}

@Configuration
@EnableAsync
class AsyncConfig : AsyncConfigurer {
    private val log: Logger = logger()

    override fun getAsyncExecutor(): Executor {
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = 10
            maxPoolSize = 100
            queueCapacity = 30
            setRejectedExecutionHandler(ThreadPoolExecutor.CallerRunsPolicy())
            setAwaitTerminationSeconds(60)
            setTaskDecorator(MdcTaskDecorator())
            setThreadNamePrefix("Asynchronous Thread-")
            initialize()
        }
    }

    override fun getAsyncUncaughtExceptionHandler(): AsyncUncaughtExceptionHandler {
        return AsyncUncaughtExceptionHandler { ex: Throwable, method: Method, params: Array<Any?> ->
            log.error("Asynchronous method thrown exception... -> Method = {}, Params = {}", method, params, ex)
        }
    }
}
```

<div style="text-align: left">
  <img src="/assets/img/posts/2023-10-25-Request%20&%20Response%20데이터%20로그%20남기기/img19.png" alt="img"/>
</div>

```text
// Async
[2024-03-30 17:13:21.994] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [f42c476a-f9dc-4cba-b73f-0759b1680843] [Request START] = [Task ID = f42c476a-f9dc-4cba-b73f-0759b1680843, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/async, Params = [], 요청 시작 시간 = 2024-03-30T17:13:21.989088400] 
[2024-03-30 17:13:21.996] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [f42c476a-f9dc-4cba-b73f-0759b1680843] Request Body = { Empty } 
[2024-03-30 17:13:22.020] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.l.LoggingTracer] [f42c476a-f9dc-4cba-b73f-0759b1680843] |--->HelloApi.async() args=[] 
[2024-03-30 17:13:22.024] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.l.LoggingTracer] [f42c476a-f9dc-4cba-b73f-0759b1680843] |<---HelloApi.async() time=30ms 
[2024-03-30 17:13:22.025] [INFO ] [Asynchronous Thread-1] [c.s.l.g.l.LoggingTracer] [f42c476a-f9dc-4cba-b73f-0759b1680843] |--->HelloService.async() args=[] 
[2024-03-30 17:13:22.025] [INFO ] [Asynchronous Thread-1] [c.s.l.g.l.LoggingTracer] [f42c476a-f9dc-4cba-b73f-0759b1680843] |<---HelloService.async() time=0ms 
[2024-03-30 17:13:22.040] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [f42c476a-f9dc-4cba-b73f-0759b1680843] Response Body = ok 
[2024-03-30 17:13:22.040] [INFO ] [http-nio-8080-exec-3] [c.s.l.g.f.RequestLoggingFilter] [f42c476a-f9dc-4cba-b73f-0759b1680843] [Request END] = [Task ID = f42c476a-f9dc-4cba-b73f-0759b1680843, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/async, HTTP Status = 200, 요청 처리 시간 = 45ms] 

// ApplicationEventPublisher
[2024-03-30 17:13:58.393] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.f.RequestLoggingFilter] [4032ccc5-8968-4314-9515-4150de7b53dc] [Request START] = [Task ID = 4032ccc5-8968-4314-9515-4150de7b53dc, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/event, Params = [], 요청 시작 시간 = 2024-03-30T17:13:58.393071100] 
[2024-03-30 17:13:58.393] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.f.RequestLoggingFilter] [4032ccc5-8968-4314-9515-4150de7b53dc] Request Body = { Empty } 
[2024-03-30 17:13:58.394] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |--->HelloApi.event() args=[] 
[2024-03-30 17:13:58.394] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |    |--->HelloService.event() args=[] 
[2024-03-30 17:13:58.395] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |    |<---HelloService.event() time=2ms 
[2024-03-30 17:13:58.395] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |<---HelloApi.event() time=2ms 
[2024-03-30 17:13:58.395] [INFO ] [Asynchronous Thread-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |--->EventHandler.execute(..) args=[HelloEvent(id=1)] 
[2024-03-30 17:13:58.395] [INFO ] [Asynchronous Thread-2] [c.s.l.g.l.LoggingTracer] [4032ccc5-8968-4314-9515-4150de7b53dc] |<---EventHandler.execute(..) time=0ms 
[2024-03-30 17:13:58.396] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.f.RequestLoggingFilter] [4032ccc5-8968-4314-9515-4150de7b53dc] Response Body = ok 
[2024-03-30 17:13:58.397] [INFO ] [http-nio-8080-exec-2] [c.s.l.g.f.RequestLoggingFilter] [4032ccc5-8968-4314-9515-4150de7b53dc] [Request END] = [Task ID = 4032ccc5-8968-4314-9515-4150de7b53dc, IP = 0:0:0:0:0:0:0:1, HTTP Method = GET, Uri = /api/v1/event, HTTP Status = 200, 요청 처리 시간 = 3ms] 
```

- 비동기 처리에서도 성공적으로 MDC 문맥을 유지하고 로깅을 진행할 수 있게 되었다

<br>
> 관련된 코드는 [깃허브](https://github.com/sjiwon/devlog-codes/tree/main/spring/logging/request-response-logging-with-mdc-logback)에서 확인할 수 있습니다
