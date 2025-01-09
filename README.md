

# 一、背景


　　　最近为了做微服务高可用和优化上线流程，我参与了一个微服务的改造开发。
　　　主要包括redis切换哨兵模式、接入高可用xxljob集群、配置和升级脚本优化。

# 二、问题描述



  项目改造提测后，测试发现一个依赖远程http调用的功能不可用



# 三、问题分析



  查看被调用方日志发现通用Token解析错误如下图


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109110905889-626493354.png)
 





 
说明调用时传入了错误的Token，查看调用方代码：




```
# 使用hutool工具发起远程http请求
HashMap headers = getHeader(modelSyncGitDto);
HttpRequest request = HttpUtil.createRequest(Method.POST, url);
HttpResponse execute = request
        .addHeaders(headers)
        .timeout(2000)
        .body(JSONUtil.toJsonStr(modelSyncGitDto))
        .execute();
# 通用工具生成jwt token放入header
private HashMap getHeader(ModelSyncGitDto modelSyncGitDto){
    String userJwtToken = JwtUtils.getJwtToken(filterConfig.getAcSecret(),null,modelSyncGitDto.getComputerUserName(),modelSyncGitDto.getOsName());
    HashMap headers = new HashMap<>(200);
    headers.put("Token", userJwtToken);
    return headers;
}
```


 


以上调用代码无问题，继续排查被调用方解析token的代码


```
# 被调用方获取token
String token = CookieUtils.resolveCookie(request, tokenKey);
if (Objects.isNull(token) || token.isBlank()){
    token = request.getHeader(tokenKey);
}
if (Objects.isNull(token) || token.isBlank()){
    token = request.getParameter(tokenKey);
}
return token;
```


 


发现被调用方优先从cookie获取代码，未从header获取对应Token；
调用方未设置cookie，怀疑是请求后response缓存带的。增加调用方日志查看


```
log.info("http请求返回，cookie={}", execute.getCookies());
```




![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109110918880-187737579.png)
 





请求返回后在response设置了cookie，但每次请求都是*createRequest，理论不会携带cookie。添加disableCookie验证一下：*


```
# 创建request增加禁用cookie
HttpRequest request = HttpUtil.createRequest(Method.POST, url);
HttpResponse execute = request
        .disableCookie()
        .addHeaders(headers)
        .timeout(2000)
        .body(JSONUtil.toJsonStr(modelSyncGitDto))
        .execute();
```


 


发布后恢复正常，说明请求携带了cookie信息。
后期查看文档hutool的HttpUtil的确有这么一个缓存cookie的问题，参考


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109110935039-795472700.png)
 





[https://github.com/dromara/hutool/issues/583](https://github.com)
# 


# 四、梅开二度


　　提交代码并告诉测试妹纸“修复好了”。测试验证的确无问题，但遭到测试妹纸的灵魂拷问：
“线上为啥没问题，这次版本不允许夹杂额外代码上线”。
　　的确，还是要找到问题的原因。
## 4\.1 配置对比


通过对比线上配置，发现被调用方的common依赖引入了


```
server:
  servlet:
    session:
      # session超时时间，不能小于1分钟
      timeout: 481m
      # 浏览器 Cookie 中 SessionID 名称
      cookie:
        name: Token
        path: /
      tracking-modes: COOKIE
```


 


以上配置指定了cookie key为Token，且过期时间为8小时；去掉被调用方以上配置，调用方请求打印cookie如下；
***无server.servlet.session配置：***


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109110946038-640852371.png)


说明被调用方spring security默认会将Cookie设置在当前应用所在的域下放置SESSION。
## 4\.2 二次请求


***无server.servlet.session配置：***


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109111001097-269097180.png)
 





cookie中返回了SEESION和Token，初步怀疑是缓存返回导致；
## 4\.2 日志对比


相同代码，通过对比其他环境调用方打印的日志，方向无论有无***server.servlet.session***配置，返回的response都为空


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109111016060-120697125.png)
 





说明还是代码配置问题。
## 4\.3 终极原因


当调用方请求被调用方时，配置的ip不是localhost、127\.0\.0\.1、节点IP时，请求调用返回的response都不会携带cookie。


![](https://img2024.cnblogs.com/blog/1076066/202501/1076066-20250109111023784-942334188.png)
 





HttpUtil通过配置请求ip，cookie跨域导致cookie无法保存，也无法携带。
线上httpUtl发起的请求ip为另一台服务器nginx地址，返回的response无cookie。
# 五、总结


以上问题原因为2个导致：
1. 本版本hutool的HttpUtil请求缓存了上一次请求的cookie，并在下一次携带；
2. Spring Security 默认会将Cookie设置在当前应用所在的域下，即localhost。请求方也为localhost所在节点，由于http同源策略，导致返回的response中存在cookie。


 
 

 本博客参考[westworld加速](https://xbsj9.com)。转载请注明出处！
