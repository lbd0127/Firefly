---
title: AOP 三大功能快速集成：登录校验、参数校验与限流
published: 2026-10-27
tags: [Spring Boot, AOP, 切面编程, 参数校验, 登录校验, 限流, Redis]
category: 学习笔记
draft: false
---

# AOP 三大功能快速集成教程（登录校验 + 参数校验 + 限流）

> 通过 AOP + 自定义注解，一次集成三个功能，Controller 上加一行注解即可生效，业务代码零侵入。  
> 环境：Spring Boot 2.x + Java 8 + Redis。所有代码可直接复制粘贴。



---

## 〇、整体结构

```
请求 → Controller方法(@GlobalInterceptor + @VerifyParam)
     → OperationAspect 切面（@Before 前置拦截）
         ├─ checkLogin()           登录校验   → 失败抛 901 登录超时
         ├─ validateParams()       参数校验   → 失败抛 600 请求参数错误
         └─ checkRequestFrequency() 限流校验   → 超限抛 903 请求太频繁
     → 全部通过 → 执行真正的业务方法
```

需要创建的文件清单（共 7 个）：

| 文件                                           | 作用            |
| -------------------------------------------- | ------------- |
| `annotation/GlobalInterceptor.java`          | 方法级开关注解（三合一）  |
| `annotation/VerifyParam.java`                | 参数规则注解        |
| `entity/enums/VerifyRegexEnum.java`          | 内置常用正则        |
| `entity/enums/RequestFrequencyTypeEnum.java` | 限流窗口周期        |
| `entity/enums/ResponseCodeEnum.java`         | 统一错误码（可只取需要的） |
| `exception/BusinessException.java`           | 业务异常          |
| `aspect/OperationAspect.java`                | 核心切面类         |

---

## 一、引入依赖（pom.xml）

```xml
<dependencies>
    
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.4</version>
    </dependency>

    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.4</version>
    </dependency>
</dependencies>
```

`application.properties` 配置 Redis（已有则跳过）：

```properties
spring.redis.host=127.0.0.1
spring.redis.port=6379
```

> Spring Boot 自动开启 AOP，切面类加 `@Component` 即可，无需 `@EnableAspectJAutoProxy`。

---

## 二、错误码与业务异常

### 2.1 `ResponseCodeEnum.java`

```java
package com.easyjob.entity.enums;

public enum ResponseCodeEnum {
    CODE_200(200, "请求成功"),
    CODE_600(600, "请求参数错误"),
    CODE_901(901, "登录超时"),
    CODE_902(902, "权限不足"),
    CODE_903(903, "请求太频繁"),
    CODE_900(900, "服务器错误");

    private Integer code;
    private String msg;

    ResponseCodeEnum(Integer code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public Integer getCode() {
        return code;
    }

    public String getMsg() {
        return msg;
    }
}
```

### 2.2 `BusinessException.java`

```java
package com.easyjob.exception;

import com.easyjob.entity.enums.ResponseCodeEnum;

public class BusinessException extends RuntimeException {

    private ResponseCodeEnum codeEnum;

    public BusinessException(ResponseCodeEnum codeEnum) {
        super(codeEnum.getMsg());
        this.codeEnum = codeEnum;
    }

    public ResponseCodeEnum getCodeEnum() {
        return codeEnum;
    }
}
```

### 2.3 全局异常处理器（没有才需要建）

```java
package com.easyjob.controller;

import com.easyjob.entity.enums.ResponseCodeEnum;
import com.easyjob.exception.BusinessException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.ModelAndView;
import javax.servlet.http.HttpServletRequest;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandlerController {

    @ExceptionHandler(BusinessException.class)
    public Object handleBusinessException(HttpServletRequest request, BusinessException e) {
        // AJAX/接口请求返回 JSON
        Map<String, Object> result = new HashMap<>();
        result.put("code", e.getCodeEnum().getCode());
        result.put("msg", e.getCodeEnum().getMsg());
        result.put("data", null);
        return result;
    }

    @ExceptionHandler(Exception.class)
    public Object handleException(Exception e) {
        e.printStackTrace();
        Map<String, Object> result = new HashMap<>();
        result.put("code", ResponseCodeEnum.CODE_900.getCode());
        result.put("msg", ResponseCodeEnum.CODE_900.getMsg());
        return result;
    }
}
```

---

## 三、四个枚举/注解（直接复制）

### 3.1 `GlobalInterceptor.java` —— 方法级总开关

```java
package com.easyjob.annotation;

import com.easyjob.entity.enums.RequestFrequencyTypeEnum;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)   // 必须是 RUNTIME，否则切面反射读不到
public @interface GlobalInterceptor {

    /**
     * 是否校验登录，默认校验
     */
    boolean checkLogin() default true;

    /**
     * 是否校验参数，默认校验
     */
    boolean checkParams() default true;

    /**
     * 限流阈值：窗口期内每个 IP 最多请求次数，0 表示不限流
     */
    int requestFrequencyThreshold() default 0;

    /**
     * 限流统计周期
     */
    RequestFrequencyTypeEnum frequencyType() default RequestFrequencyTypeEnum.NO_LIMIT;
}
```

### 3.2 `VerifyParam.java` —— 参数规则

```java
package com.easyjob.annotation;

import com.easyjob.entity.enums.VerifyRegexEnum;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.PARAMETER, ElementType.FIELD})   // 可标在方法参数或 DTO 字段上
@Retention(RetentionPolicy.RUNTIME)
public @interface VerifyParam {

    /** 正则校验 */
    VerifyRegexEnum regex() default VerifyRegexEnum.NO;

    /** 最小长度，-1 不校验 */
    int min() default -1;

    /** 最大长度，-1 不校验 */
    int max() default -1;

    /** 是否必填 */
    boolean required() default false;
}
```

### 3.3 `VerifyRegexEnum.java` —— 内置正则库

```java
package com.easyjob.entity.enums;

public enum VerifyRegexEnum {
    NO("", "不校验"),
    POSITIVE_INTEGER("^[0-9]*[1-9][0-9]*$", "正整数"),
    NUMBER_LETTER_UNDER_LINE("^\\w+$", "数字、英文字母或下划线"),
    EMAIL("^[\\w-]+(\\.[\\w-]+)*@[\\w-]+(\\.[\\w-]+)+$", "邮箱"),
    PHONE("(1[0-9])\\d{9}$", "手机号码"),
    COMMON("^[a-zA-Z0-9_\\u4e00-\\u9fa5]+$", "数字，字母，中文，下划线"),
    PASSWORD("^(?=.*\\d)(?=.*[a-zA-Z])[\\da-zA-Z~!@#$%^&*_]{8,18}$", "8-18位数字字母特殊字符"),
    ACCOUNT("^[0-9a-zA-Z_]{1,}$", "数字、英文字母或下划线"),
    MONEY("^[0-9]+(.[0-9]{1,2})?$", "金额");

    private String regex;
    private String desc;

    VerifyRegexEnum(String regex, String desc) {
        this.regex = regex;
        this.desc = desc;
    }

    public String getRegex() {
        return regex;
    }

    public String getDesc() {
        return desc;
    }
}
```

### 3.4 `RequestFrequencyTypeEnum.java` —— 限流窗口

```java
package com.easyjob.entity.enums;

public enum RequestFrequencyTypeEnum {
    DAY(60 * 60 * 24, "一天"), HOUR(60 * 60, "一小时"), MINUTE(60, "一分钟"),
    SECONDS(1, "一秒"), NO_LIMIT(0, "不限制");

    private Integer seconds;
    private String desc;

    RequestFrequencyTypeEnum(Integer seconds, String desc) {
        this.seconds = seconds;
        this.desc = desc;
    }

    public Integer getSeconds() {
        return seconds;
    }

    public String getDesc() {
        return desc;
    }
}
```

---

## 四、RedisUtils（限流用，只保留核心方法）

```java
package com.easyjob.component;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Component("redisUtils")
public class RedisUtils<V> {

    @Resource
    private RedisTemplate<String, V> redisTemplate;

    public V get(String key) {
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 自增，并在第一次自增（result == 1）时设置过期时间 —— 限流的时间窗口
     */
    public long increment(String key, long delta, long time) {
        if (delta < 0) {
            throw new RuntimeException("递增因子必须大于0");
        }
        long result = redisTemplate.opsForValue().increment(key, delta);
        if (result == 1) {
            redisTemplate.expire(key, time, TimeUnit.SECONDS);
        }
        return result;
    }
}
```

---


## 五、核心切面类 OperationAspect（三合一）

```java
package com.easyjob.aspect;

import com.easyjob.annotation.GlobalInterceptor;
import com.easyjob.annotation.VerifyParam;
import com.easyjob.component.RedisUtils;
import com.easyjob.entity.constants.Constants;
import com.easyjob.entity.dto.SessionUserDto;   // 换成你项目的登录用户 DTO
import com.easyjob.entity.enums.RequestFrequencyTypeEnum;
import com.easyjob.entity.enums.ResponseCodeEnum;
import com.easyjob.exception.BusinessException;
import com.easyjob.utils.StringTools;            // 没有可替换为 StringUtils.isBlank
import com.easyjob.utils.VerifyUtils;
import org.apache.commons.lang3.ArrayUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.List;

@Aspect
@Component("operationAspect")
public class OperationAspect {

    @Resource
    private RedisUtils redisUtils;

    private final Logger logger = LoggerFactory.getLogger(OperationAspect.class);

    /** 视为"基本类型"直接校验的参数，其他类型走对象字段反射校验 */
    private static final String[] BASE_TYPE_ARRAY =
            new String[]{"java.lang.String", "java.lang.Integer", "java.lang.Long"};

    // ==================== 入口：前置拦截 ====================

    @Before("@annotation(com.easyjob.annotation.GlobalInterceptor)")
    public void interceptorDo(JoinPoint point) {
        Object target = point.getTarget();
        Object[] arguments = point.getArgs();
        Method method = ((MethodSignature) point.getSignature()).getMethod();
        GlobalInterceptor interceptor = method.getAnnotation(GlobalInterceptor.class);
        if (interceptor == null) {
            return;
        }

        // 1. 登录校验
        if (interceptor.checkLogin()) {
            checkLogin();
        }

        // 2. 参数校验
        if (interceptor.checkParams()) {
            validateParams(method, arguments);
        }

        // 3. 限流校验
        if (interceptor.frequencyType() != RequestFrequencyTypeEnum.NO_LIMIT
                && interceptor.requestFrequencyThreshold() != 0) {
            String fullMethodName = target.getClass().getName() + "." + method.getName();
            checkRequestFrequency(fullMethodName, interceptor.frequencyType(),
                    interceptor.requestFrequencyThreshold());
        }
    }

    // ==================== 功能一：登录校验 ====================

    void checkLogin() {
        HttpServletRequest request = ((ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes()).getRequest();
        SessionUserDto sessionUser =
                (SessionUserDto) request.getSession().getAttribute(Constants.SESSION_KEY);
        if (sessionUser == null) {
            throw new BusinessException(ResponseCodeEnum.CODE_901);  // 901 登录超时
        }
    }

    // ==================== 功能二：参数校验 ====================

    private void validateParams(Method method, Object[] arguments) {
        Parameter[] parameters = method.getParameters();
        for (int i = 0; i < parameters.length; i++) {
            Parameter parameter = parameters[i];
            Object value = arguments[i];
            VerifyParam verifyParam = parameter.getAnnotation(VerifyParam.class);
            if (verifyParam == null) {
                continue;   // 未标注的参数不校验
            }
            String paramTypeName = parameter.getParameterizedType().getTypeName();
            if (ArrayUtils.contains(BASE_TYPE_ARRAY, paramTypeName)) {
                // 基本类型直接校验
                checkValue(value, verifyParam);
            } else {
                // 对象参数：反射遍历 DTO 字段校验
                checkObjValue(parameter, value);
            }
        }
    }

    @SuppressWarnings({"unchecked", "rawtypes"})
    private void checkObjValue(Parameter parameter, Object value) {
        try {
            String typeName = parameter.getParameterizedType().getTypeName();
            Class classz = Class.forName(typeName);
            Field[] fields = classz.getDeclaredFields();
            for (Field field : fields) {
                VerifyParam fieldVerifyParam = field.getAnnotation(VerifyParam.class);
                if (fieldVerifyParam == null) {
                    continue;
                }
                field.setAccessible(true);
                Object resultValue = field.get(value);
                checkValue(resultValue, fieldVerifyParam);
            }
        } catch (BusinessException e) {
            throw e;
        } catch (Exception e) {
            logger.error("校验参数失败", e);
            throw new BusinessException(ResponseCodeEnum.CODE_600);
        }
    }

    private void checkValue(Object value, VerifyParam verifyParam) {
        Boolean isEmpty = value == null || StringTools.isEmpty(value.toString());
        Integer length = value == null ? 0 : value.toString().length();

        // 1. 必填校验
        if (isEmpty && verifyParam.required()) {
            throw new BusinessException(ResponseCodeEnum.CODE_600);
        }
        // 2. 长度校验
        if (!isEmpty && (verifyParam.max() != -1 && verifyParam.max() < length
                || verifyParam.min() != -1 && verifyParam.min() > length)) {
            throw new BusinessException(ResponseCodeEnum.CODE_600);
        }
        // 3. 正则校验
        if (!isEmpty && !StringTools.isEmpty(verifyParam.regex().getRegex())
                && !VerifyUtils.verify(verifyParam.regex(), String.valueOf(value))) {
            throw new BusinessException(ResponseCodeEnum.CODE_600);
        }
    }

    // ==================== 功能三：限流校验 ====================

    void checkRequestFrequency(String fullMethodName, RequestFrequencyTypeEnum frequencyType,
                               Integer threshold) {
        HttpServletRequest request = ((ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes()).getRequest();
        String ip = getIpAddr(request).replace(":", "");
        String redisKey = fullMethodName + ip;   // key = 类全名.方法名 + IP
        Integer cuCount = (Integer) redisUtils.get(redisKey);
        if (cuCount != null && cuCount > threshold - 1) {
            throw new BusinessException(ResponseCodeEnum.CODE_903);  // 903 请求太频繁
        }
        redisUtils.increment(redisKey, 1, frequencyType.getSeconds());
    }

    protected String getIpAddr(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");
        if (ip != null && ip.length() != 0 && !"unknown".equalsIgnoreCase(ip)) {
            // 多次反向代理后会有多个ip值，第一个ip才是真实ip
            if (ip.indexOf(",") != -1) {
                ip = ip.split(",")[0];
            }
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_CLIENT_IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("X-Real-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

两个辅助类（项目里没有就复制）：

```java
// StringTools 只需要这个方法
public static boolean isEmpty(String s) {
    return null == s || "".equals(s.trim());
}

// VerifyUtils
public static boolean verify(VerifyRegexEnum regexEnum, String value) {
    if (StringTools.isEmpty(regexEnum.getRegex())) {
        return true;
    }
    return Pattern.matches(regexEnum.getRegex(), value);
}
```

登录校验适配说明：上面切面用的是 **Session 取登录用户**。如果你的项目用 JWT，把 `checkLogin()` 换成：

```java
void checkLogin() {
    HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes()).getRequest();
    String token = request.getHeader("token");
    // jwtUtil.getTokenData 按你项目的 JWT 工具类调整
    SessionUserDto user = jwtUtil.getTokenData(Constants.JWT_KEY, token, SessionUserDto.class);
    if (user == null) {
        throw new BusinessException(ResponseCodeEnum.CODE_901);
    }
}
```

---

## 六、使用示例（Controller 中一行注解搞定）

```java
@RestController
public class DemoController {

    // 1. 最简用法：登录校验 + 参数校验 全开（默认值）
    @RequestMapping("/updatePwd")
    @GlobalInterceptor
    public ResponseVO updatePwd(@VerifyParam(required = true, regex = VerifyRegexEnum.PASSWORD) String password) {
        return ResponseVO.success();
    }

    // 2. 登录接口：关闭登录校验（否则鸡生蛋问题），参数校验保留
    @RequestMapping("/login")
    @GlobalInterceptor(checkLogin = false)
    public ResponseVO login(@VerifyParam(regex = VerifyRegexEnum.PHONE) String phone,
                            @VerifyParam(required = true) String password) {
        return ResponseVO.success();
    }

    // 3. 验证码/发短信接口：免登录 + 限流（每分钟每 IP 最多 1 次）
    @RequestMapping("/sendCode")
    @GlobalInterceptor(checkLogin = false,
            frequencyType = RequestFrequencyTypeEnum.MINUTE,
            requestFrequencyThreshold = 1)
    public ResponseVO sendCode(@VerifyParam(regex = VerifyRegexEnum.EMAIL) String email) {
        return ResponseVO.success();
    }

    // 4. 对象参数校验：@VerifyParam 标在 DTO 字段上，Controller 参数只写 @VerifyParam
    @RequestMapping("/register")
    @GlobalInterceptor(checkLogin = false)
    public ResponseVO register(@VerifyParam RegisterForm form) {
        return ResponseVO.success();
    }
}

public class RegisterForm {
    @VerifyParam(required = true, regex = VerifyRegexEnum.PHONE)
    private String phone;

    @VerifyParam(required = true, regex = VerifyRegexEnum.PASSWORD)
    private String password;

    @VerifyParam(required = true, min = 2, max = 20)
    private String nickName;
    // getter/setter 省略
}
```

校验失败/超限时前端收到的统一响应：

```json
{ "code": 600, "msg": "请求参数错误" }
{ "code": 901, "msg": "登录超时" }
{ "code": 903, "msg": "请求太频繁" }
```

---

## 七、注意事项

1. **注解保留策略**必须是 `RetentionPolicy.RUNTIME`，否则切面运行期反射读不到；
2. **对象校验只处理一层字段**：反射遍历 DTO 字段，嵌套对象不会深入；`BASE_TYPE_ARRAY` 只认 String/Integer/Long，其他类型参数会走对象分支，需按需扩展；
3. **限流算法是固定窗口**（Redis INCR + 首次 EXPIRE），窗口边界处可能存在临界突刺，一般业务足够；要求更严可用滑动窗口/令牌桶；
4. **限流 key = 方法全名 + IP**，集群环境天然共享计数；但反代环境必须确保网关写入 `X-Forwarded-For`，否则所有人共享网关 IP 被一起限流；
5. **校验顺序**：登录 → 参数 → 限流，限流放在最后意味着每次合法请求也计入 Redis，如果希望"未登录不计次"可把限流分支挪到最前；
6. 校验全部走 `@Before` 前置通知，非法请求在进入业务方法前就被拦截并抛异常。
