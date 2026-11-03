## 前言

访问流水如果写在每个接口方法体内，业务代码会混进 IP、浏览器和入库细节。换存储或补一个字段时，要逐个接口改，漏掉的接口就没有记录。更合适的切分是：用一个只打在方法上的标记说明“记什么”，用环绕通知在方法成功后收集参数，再用异步方法落库。业务方法只保留标记，不关心流水怎么存。

## 编写自定义接口

标记本身没有逻辑，只声明业务名、行为种类、端，以及这次要不要入库。行为用枚举，避免调用点各写各的字符串。

```java
@Target(value = ElementType.METHOD)
@Retention(value = RetentionPolicy.RUNTIME)
public @interface AccessLog {

    /** 展示用名称，可用 {0} 引用第 0 个参数 */
    String module() default "";

    ActionKind kind();

    Channel channel() default Channel.WEB;

    boolean store() default true;
}
```

```java
public enum Channel {
    WEB, ADMIN, APP
}
```

枚举同时保存中文说明和入库短码。`fromCode` 用 `Objects.equals` 比较字符串内容；若写成 `==`，只有常量池里碰巧是同一对象时才匹配，从数据库读回的短码会全部落空。

```java
public enum ActionKind {

    OPEN_TAG("打开标签", "open_tag"),
    OPEN_CATEGORY("打开分类", "open_category"),
    READ_ARTICLE("阅读正文", "read_article"),
    LIKE_ARTICLE("给文章点赞", "like_article"),
    CLICK_FRIEND("点开友链", "click_friend"),
    RUN_SEARCH("执行搜索", "run_search"),
    PLAY_LESSON("播放课程", "play_lesson"),
    ENTER_VIEW("进入页面", "enter_view"),
    OPEN_ARCHIVE("打开归档", "open_archive"),
    VIEW_WRITER("查看作者", "view_writer"),
    ADD_REPLY("新增回复", "add_reply"),
    DROP_REPLY("移除回复", "drop_reply"),
    FLAG_REPLY("标记不当回复", "flag_reply"),
    OPEN_COLUMN("打开栏目", "open_column");

    private final String label;
    private final String code;

    ActionKind(String label, String code) {
        this.label = label;
        this.code = code;
    }

    public String label() {
        return label;
    }

    public String code() {
        return code;
    }

    public static ActionKind fromCode(String code) {
        for (ActionKind item : values()) {
            if (Objects.equals(item.code, code)) {
                return item;
            }
        }
        return null;
    }

    /**
     * 从参数名到实参的映射里抽出“模块主键”和“附加文本”。
     * 不同行为关心的字段不一样，集中在这里，切面就不用再写分支。
     */
    public static Extra extraOf(ActionKind kind, Map<String, Object> args, String moduleName) {
        String moduleId = "";
        String extra = "";
        switch (kind) {
            case VIEW_WRITER:
                extra = text(args.get("writerName"));
                break;
            case OPEN_CATEGORY:
            case OPEN_COLUMN:
                moduleId = text(args.get("categoryId"));
                break;
            case OPEN_TAG:
                moduleId = text(args.get("tagId"));
                break;
            case RUN_SEARCH:
                extra = text(args.get("keyword"));
                break;
            case OPEN_ARCHIVE:
                extra = text(args.get("monthKey"));
                break;
            case READ_ARTICLE:
            case LIKE_ARTICLE:
                moduleId = text(args.get("articleId"));
                break;
            case ENTER_VIEW:
                extra = moduleName;
                break;
            case ADD_REPLY:
            case DROP_REPLY:
            case FLAG_REPLY:
                extra = replyBody(args.get("reply"));
                break;
            case CLICK_FRIEND:
                moduleId = text(args.get("linkId"));
                break;
            case PLAY_LESSON:
                moduleId = text(args.get("lessonId"));
                break;
            default:
                break;
        }
        return new Extra(moduleId, extra);
    }

    private static String text(Object value) {
        return value == null ? "" : String.valueOf(value);
    }

    @SuppressWarnings("unchecked")
    private static String replyBody(Object reply) {
        if (reply == null) {
            return "";
        }
        if (reply instanceof Map) {
            Object body = ((Map<String, Object>) reply).get("body");
            return text(body);
        }
        try {
            Method reader = reply.getClass().getMethod("getBody");
            return text(reader.invoke(reply));
        } catch (ReflectiveOperationException ex) {
            return "";
        }
    }

    public static final class Extra {
        public final String moduleId;
        public final String note;

        public Extra(String moduleId, String note) {
            this.moduleId = moduleId;
            this.note = note;
        }
    }
}
```

参数名要和控制器方法的形参一致。阅读正文取 `articleId`，搜索取 `keyword`，回复三类行为则从名为 `reply` 的对象上读 `body`。新增行为时只扩展这个枚举，不要回到接口里面写库。

## 编写AOP代码

切面使用环绕通知：先 `proceed`，业务抛错时不记成功流水；业务返回后，收集失败只打错误日志，不影响已经得到的返回值。

```java
@Aspect
@Component
@Slf4j
public class VisitLogAspect {

    private final VisitWriter visitWriter;

    public VisitLogAspect(VisitWriter visitWriter) {
        this.visitWriter = visitWriter;
    }

    @Pointcut("@annotation(accessLog)")
    public void marked(AccessLog accessLog) {
    }

    @Around("marked(accessLog)")
    public Object around(ProceedingJoinPoint joinPoint, AccessLog accessLog) throws Throwable {
        Object returned = joinPoint.proceed();
        try {
            capture(joinPoint, accessLog);
        } catch (Exception ex) {
            log.warn("访问流水写入失败", ex);
        }
        return returned;
    }

    private void capture(ProceedingJoinPoint joinPoint, AccessLog accessLog) {
        HttpServletRequest request = currentRequest();
        String moduleName = JoinPointReads.fillTemplate(joinPoint.getArgs(), accessLog.module());
        String ip = request == null ? "-" : ClientIp.resolve(request);
        String ua = request == null ? "-" : request.getHeader("User-Agent");
        log.info("module={} ip={} {} {} ua={}",
                moduleName, ip, request == null ? "-" : request.getMethod(),
                request == null ? "-" : request.getRequestURI(), ua);
        if (!accessLog.store()) {
            return;
        }
        Map<String, Object> args = JoinPointReads.argumentMap(joinPoint);
        ActionKind.Extra extra = ActionKind.extraOf(accessLog.kind(), args, moduleName);
        String userId = request == null ? "" : String.valueOf(request.getAttribute("userId"));
        if ("null".equals(userId)) {
            userId = "";
        }
        visitWriter.append(userId, ip, ua, accessLog.kind().code(), extra.moduleId, extra.note);
    }

    private HttpServletRequest currentRequest() {
        RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
        if (!(attributes instanceof ServletRequestAttributes)) {
            return null;
        }
        return ((ServletRequestAttributes) attributes).getRequest();
    }
}
```

`JoinPointReads` 负责两件和业务无关的事：把注解里的 `{0}` 换成对应实参，以及按形参名收集实参。下标从 0 开始，和数组一致。源码里若把 `{1}` 当成第一个参数，阅读成本更高，也容易和 JSON 模板混淆。

```java
public final class JoinPointReads {

    private static final Pattern SLOT = Pattern.compile("\\{(\\d+)}");

    private JoinPointReads() {
    }

    public static String fillTemplate(Object[] args, String template) {
        if (template == null || template.isEmpty()) {
            return "";
        }
        Matcher matcher = SLOT.matcher(template);
        StringBuffer buffer = new StringBuffer();
        while (matcher.find()) {
            int index = Integer.parseInt(matcher.group(1));
            String replacement = (args != null && index >= 0 && index < args.length)
                    ? String.valueOf(args[index])
                    : "";
            matcher.appendReplacement(buffer, Matcher.quoteReplacement(replacement));
        }
        matcher.appendTail(buffer);
        return buffer.toString();
    }

    public static Map<String, Object> argumentMap(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String[] names = signature.getParameterNames();
        Object[] values = joinPoint.getArgs();
        Map<String, Object> map = new LinkedHashMap<>();
        if (names == null) {
            return map;
        }
        for (int i = 0; i < names.length; i++) {
            Object value = values[i];
            if (value instanceof ServletRequest
                    || value instanceof ServletResponse
                    || value instanceof MultipartFile
                    || value instanceof BindingResult) {
                continue;
            }
            map.put(names[i], value);
        }
        return map;
    }

    public static Method methodOf(JoinPoint joinPoint) throws NoSuchMethodException {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method declared = signature.getMethod();
        return joinPoint.getTarget().getClass()
                .getMethod(declared.getName(), declared.getParameterTypes());
    }
}
```

编译时若没有加上 `-parameters`，`getParameterNames()` 会得到 `arg0` 这种占位名，枚举里按 `articleId` 取值就会落空。Maven 编译器插件需要打开参数名保留。

落库方法加 `@Async`，让请求线程在业务返回后不必等待插入。读缓存和写缓存必须使用同一把键；只要前缀差一个冒号，每次都会打到外部解析，缓存形同虚设。

```java
@Service
public class VisitWriter {

    private final StringRedisTemplate redis;
    private final VisitRepository repository;
    private final RegionLookup regionLookup;

    public VisitWriter(StringRedisTemplate redis, VisitRepository repository, RegionLookup regionLookup) {
        this.redis = redis;
        this.repository = repository;
        this.regionLookup = regionLookup;
    }

    @Async
    public void append(String userId, String ip, String ua, String action,
                       String moduleId, String note) {
        VisitRow row = new VisitRow();
        row.setUserId(userId);
        row.setIp(ip);
        row.setAction(action);
        row.setModuleId(moduleId);
        row.setNote(note);
        row.setBrowser(UserAgentBrief.browser(ua));
        row.setOs(UserAgentBrief.os(ua));

        String cacheKey = "geo:" + ip;
        String region = redis.opsForValue().get(cacheKey);
        if (region == null || region.isEmpty()) {
            region = regionLookup.locate(ip);
            if (region != null && !region.isEmpty()) {
                redis.opsForValue().set(cacheKey, region, Duration.ofHours(12));
            }
        }
        row.setRegion(region == null ? "" : region);
        repository.insert(row);
    }
}
```

`@Async` 要生效，启动类上还得有 `@EnableAsync`。只有注解、没有这个开关时，方法仍在调用方线程里同步执行，页面耗时会把查库算进去。异步方法如果写在同一个类的内部自调用上，代理也不会介入，所以 `append` 放在独立的 `VisitWriter` 里，由切面注入后调用。

`UserAgentBrief` 只做很粗的判断，够用即可，不必在切面里堆浏览器规则：

```java
public final class UserAgentBrief {

    private UserAgentBrief() {
    }

    public static String browser(String ua) {
        if (ua == null) {
            return "unknown";
        }
        if (ua.contains("Edg/")) {
            return "edge";
        }
        if (ua.contains("Chrome/")) {
            return "chrome";
        }
        if (ua.contains("Firefox/")) {
            return "firefox";
        }
        return "other";
    }

    public static String os(String ua) {
        if (ua == null) {
            return "unknown";
        }
        if (ua.contains("Windows")) {
            return "windows";
        }
        if (ua.contains("Android")) {
            return "android";
        }
        if (ua.contains("Mac OS")) {
            return "mac";
        }
        return "other";
    }
}
```

## 指定接口进行收集

在需要记账的方法上标 `AccessLog`。环绕通知能拿到形参，因此回复正文不用在业务方法里再抄一份给日志。

```java
@AccessLog(module = "新增回复", kind = ActionKind.ADD_REPLY)
@PostMapping("/replies")
public ReplyView create(@Valid @RequestBody ReplyForm reply, BindingResult binding) {
    if (binding.hasErrors()) {
        throw new IllegalArgumentException("回复参数不合法");
    }
    if (reply.getBody() == null || reply.getBody().length() > 400) {
        throw new IllegalArgumentException("回复长度需在 400 字以内");
    }
    Reply saved = replyService.create(reply);
    return ReplyView.of(saved);
}
```

`BindingResult` 会被 `argumentMap` 跳过，`reply` 留下。枚举的 `ADD_REPLY` 分支调用 `getBody()`，流水的附加文本就是回复内容，模块主键留空。搜索接口则写成：

```java
@AccessLog(module = "搜索 {0}", kind = ActionKind.RUN_SEARCH, store = true)
@GetMapping("/search")
public List<ArticleBrief> search(@RequestParam("keyword") String keyword) {
    return articleQuery.find(keyword);
}
```

这里形参名是 `keyword`，和 `RUN_SEARCH` 分支一致；模板 `{0}` 会把同一个词填进日志行的 module 字段，便于只看应用日志、不查库时也能读懂。

只想打一行日志、不入库的探针，把 `store` 设为 `false`。切面在 `proceed` 之后仍会打印，但不会调用 `VisitWriter`。

请求走完并等待异步线程结束后，表里应有 IP、行为短码、模块主键和附加文本。若方法已经返回而表始终为空，先确认 `@EnableAsync` 是否打开，再确认编译参数名是否保留，最后核对 Redis 读写是不是同一把 `geo:` 前缀。
