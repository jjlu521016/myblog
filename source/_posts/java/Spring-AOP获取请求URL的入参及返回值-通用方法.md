---
title: ' Spring AOP获取请求URL的入参及返回值(通用方法)'
date: 2017.10.26 16:07:57
tags: 
    - AOP
    - Spring
---
以下代码为通用的代码，其中json解析使用的是fastJson，可以记录用户访问的ip、url、入参和出参
```
/**
 * @author jasonLu
 * @date 2017/10/26 9:57
 * @Description:获取请求的入参和出参
 */
@Component
@Aspect
public class RequestAspect {

    private static final Logger logger = LoggerFactory.getLogger(RequestAspect.class);

    @Pointcut("@within(org.springframework.stereotype.Controller) || @within(org.springframework.web.bind.annotation.RestController)")
    public void pointcut()
    {
        // 空方法
    }


    @Around("pointcut()")
    public Object handle(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        //IP地址
        String ipAddr = getRemoteHost(request);
        String url = request.getRequestURL().toString();
        String reqParam = preHandle(joinPoint,request);
        logger.info("请求源IP:【{}】,请求URL:【{}】,请求参数:【{}】",ipAddr,url,reqParam);

        Object result= joinPoint.proceed();
        String respParam = postHandle(result);
        logger.info("请求源IP:【{}】,请求URL:【{}】,返回参数:【{}】",ipAddr,url,respParam);
        return result;
    }

    /**
     * 入参数据
     * @param joinPoint
     * @param request
     * @return
     */
    private String preHandle(ProceedingJoinPoint joinPoint,HttpServletRequest request) {

        String reqParam = "";
        Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        Method targetMethod = methodSignature.getMethod();
        Annotation[] annotations = targetMethod.getAnnotations();
        for (Annotation annotation : annotations) {
            //此处可以改成自定义的注解
            if (annotation.annotationType().equals(RequestMapping.class)) {
                reqParam = JSON.toJSONString(request.getParameterMap());
                break;
            }
        }
        return reqParam;
    }

    /**
     * 返回数据
     * @param retVal
     * @return
     */
    private String postHandle(Object retVal) {
        if(null == retVal){
            return "";
        }
        return JSON.toJSONString(retVal);
    }


    /**
     * 获取目标主机的ip
     * @param request
     * @return
     */
    private String getRemoteHost(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return "0:0:0:0:0:0:0:1".equals(ip) ? "127.0.0.1" : ip;
    }

}
```