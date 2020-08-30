2020-08-19， `Apache Shrio`发布了`CVE-2020-13933`的漏洞, 其等级为高，影响范围为`<=1.5.3`。 在处理身份验证请求时存在`权限绕过`, 可发送特制的`HTTP`请求, 绕过身份验证过程并获得对应用的访问.  现对该漏洞进行分析。 

# Diff
可以看一下`1.5.3`和`1.5.4`版本的diff，看变动哪些文件
[DIFF链接](https://github.com/apache/shiro/compare/shiro-root-1.5.3...shiro-root-1.6.0)
在`web/src/main/java/org/apache/shiro/web/filter/mgt/DefaultFilter.java`中，看到diff内容为

```
package org.apache.shiro.web.filter.mgt;

  import org.apache.shiro.util.ClassUtils;
+ import org.apache.shiro.web.filter.InvalidRequestFilter;
  import org.apache.shiro.web.filter.authc.*;
  import org.apache.shiro.web.filter.authz.*;
  import org.apache.shiro.web.filter.session.NoSessionCreationFilter;
@@ -48,7 +49,8 @@
     rest(HttpMethodPermissionFilter.class),
     roles(RolesAuthorizationFilter.class),
     ssl(SslFilter.class),
 -   user(UserFilter.class);
 +   user(UserFilter.class),
 +   invalidRequest(InvalidRequestFilter.class);

    private final Class<? extends Filter> filterClass;
```
可以看到添加了一个`Filter`:`InvalidRequestFilter`
在[InvalidRequestFilter.java](https://github.com/apache/shiro/compare/shiro-root-1.5.3...shiro-root-1.6.0#diff-aaf47dacbed7db2ff19000baf3bdc4af)
```java
public class InvalidRequestFilter extends AccessControlFilter {

    private static final List<String> SEMICOLON = Collections.unmodifiableList(Arrays.asList(";", "%3b", "%3B"));

    private static final List<String> BACKSLASH = Collections.unmodifiableList(Arrays.asList("\\", "%5c", "%5C"));
```
对`\\`和`;`符号做了标记, 在[testFilterBlocks](https://github.com/apache/shiro/compare/shiro-root-1.5.3...shiro-root-1.6.0#diff-1ee3bc21740941899519a4a2280a78f8)函数中:
```java
    @Test
    void testFilterBlocks() {
        InvalidRequestFilter filter = new InvalidRequestFilter()
        assertPathBlocked(filter, "/\\something")
        assertPathBlocked(filter, "/%5csomething")
        assertPathBlocked(filter, "/%5Csomething")
        assertPathBlocked(filter, "/;something")
        assertPathBlocked(filter, "/%3bsomething")
        assertPathBlocked(filter, "/%3Bsomething")
        assertPathBlocked(filter, "/\u0019something")
    }

```
其中上可以断定, 是`;`或者`\\`字符出的问题. 

## 环境搭建
- `git clone https://github.com/l3yx/springboot-shiro.git`
- 在idea中修改`om.xml`文件，将`shiro`的版本改为`1.5.3`
    ```xml
    <dependency>
         <groupId>org.apache.shiro</groupId>
         <artifactId>shiro-web</artifactId>
         <version>1.5.3</version>
    </dependency>
    <dependency>
         <groupId>org.apache.shiro</groupId>
         <artifactId>shiro-spring</artifactId>
         <version>1.5.3</version>
    </dependency>
    ```
- 修改`LoginController.java` 为   因为 在 `CVE-2020-11989`中有两种利用方式， 所以测试了两种， 分别是 
    ```java

        @GetMapping("/admin/{name}")
        public String admin(@PathVariable String name) {
            return "admin page";
        }
    //    @GetMapping("/admin/page")
    //    public String admin() {
    //        return "admin page";
    //    }
    ```

如diff中的， 可以分别测试 `localhost:8080/shiro/admin/page` 下的不同路径，
- `localhost:8080/%3bshiro/admin/page`, 
- `localhost:8080/shiro/%3badmin/page`,
- `localhost:8080/shiro/admin/%3bpage`

其中第三种， 在如上`controller`中可以成功， 下边详细分析一下，

## Tomcat远程调试
- 在idea中导出war包, 并放在`tomcat`的`webapps`目录下
- 在`tomcat/bin/catalina.sh`下添加一行`debug`命令: `export JAVA_OPTS='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005'`
- 在idea中配置远程 调试IP
- 配置断点即可


## 详细分析
- 先看调用链
    ![调用链](https://pic4.zhimg.com/80/v2-176b1bda17e97d112437f7e1378e526b_720w.png)

- 调用链前半部分为 `tomcat`请求，然后调用 了`spring`的`filter`,  接着又调用了`shiro`的`filter`最后调用了`spring`的`dispacher`， 在 `org.apache.shiro.web.util.WebUtils#getPathWithinApplication` 函数中， 调用了
    ```java

    public static String getPathWithinApplication(HttpServletRequest request) {
        return normalize(removeSemicolon(getServletPath(request) + getPathInfo(request)));
    }
    ```
    ![getPathWithinApplication](https://pic1.zhimg.com/80/v2-443aef267ee0ed5803e029b25c71f464_720w.png)

- 由上可知， `uri` = `"/admin/;aa" ` 而 `org.apache.shiro.web.util.WebUtils#removeSemicolon` 为
    ```java
    private static String removeSemicolon(String uri) {
        int semicolonIndex = uri.indexOf(59);
        return semicolonIndex != -1 ? uri.substring(0, semicolonIndex) : uri;
    }
    ```
    即只获取 分号`（; )`前作为返回值 此时返回  `"/admin/"`

- 而`org.apache.shiro.web.util.WebUtils#normalize(java.lang.String, boolean)`  如下，可以看到仅过滤 不同类型的\ /， 所以这里返回 的仍然是 `"/admin/"`
    ```java

    private static String normalize(String path, boolean replaceBackSlash) {
        if (path == null) {
            return null;
        } else {
            String normalized = path;
            if (replaceBackSlash && path.indexOf(92) >= 0) {
                normalized = path.replace('\\', '/');
            }
     
            if (normalized.equals("/.")) {
                return "/";
            } else {
                if (!normalized.startsWith("/")) {
                    normalized = "/" + normalized;
                }
     
                while(true) {
                    int index = normalized.indexOf("//");
                    if (index < 0) {
                        while(true) {
                            index = normalized.indexOf("/./");
                            if (index < 0) {
                                while(true) {
                                    index = normalized.indexOf("/../");
                                    if (index < 0) {
                                        return normalized;
                                    }
     
                                    if (index == 0) {
                                        return null;
                                    }
     
                                    int index2 = normalized.lastIndexOf(47, index - 1);
                                    normalized = normalized.substring(0, index2) + normalized.substring(index + 3);
                                }
                            }
     
                            normalized = normalized.substring(0, index) + normalized.substring(index + 2);
                        }
                    }
     
                    normalized = normalized.substring(0, index) + normalized.substring(index + 1);
                }
            }
        }
    ```

- 在 `org.apache.shiro.web.filter.mgt.PathMatchingFilterChainResolver#getChain `中，代码片段为， 即将 `”/admin/"` 处理为 `"/admin"`
    ```java

    String requestURI = this.getPathWithinApplication(request);
    if (requestURI != null && !"/".equals(requestURI) && requestURI.endsWith("/")) {
        requestURI = requestURI.substring(0, requestURI.length() - 1);
    }
    ```

- 同样是这个函数， 继续对比 `pathPattern` 和` requestURI`,  而` /admin/* ` 和 `/admin` 导致的不匹配，所以绕过了认证
    ```java

        pathPattern = (String)var6.next();
        if (pathPattern != null && !"/".equals(pathPattern) && pathPattern.endsWith("/")) {
            pathPattern = pathPattern.substring(0, pathPattern.length() - 1);
        }
    } while(!this.pathMatches(pathPattern, requestURI));
    ```
    ![](https://pic4.zhimg.com/80/v2-e817844349682904519f59bc76143966_720w.png)

- 继续跟踪在 `org.springframework.web.util.UrlPathHelper#decodeAndCleanUriString` 函数中
  
    ```java
    private String decodeAndCleanUriString(HttpServletRequest request, String uri) {
        uri = this.removeSemicolonContent(uri);
        uri = this.decodeRequestString(request, uri);
        uri = this.getSanitizedPath(uri);
        return uri;
    }
    ```
    分别去除了分号， URL解码，去除  `//`  , 最后获取到了 `admin/;aa ` 绕过了认证
    ![](https://pic4.zhimg.com/80/v2-6a77615f95932a8772f2ac00cfe02e08_720w.png)

以上原因即为不同的中间件对于路径去过滤不同， 对于URL编码的解码时间不同，导致了这个漏洞  
![](https://pic4.zhimg.com/80/v2-330bf5fea19b489fcec9885bf18b0887_720w.png)
类似这个 ： [how-i-chained-4-bugs-features-into-rce-on-amazon](http://blog.orange.tw/2018/08/how-i-chained-4-bugs-features-into-rce-on-amazon.html)

## 参考:
- [Apache Shiro 身份验证绕过漏洞 (CVE-2020-11989)](https://xlab.tencent.com/cn/2020/06/30/xlab-20-002/)
- [Apache Shiro权限绕过漏洞分析(CVE-2020-11989)](https://xz.aliyun.com/t/7964)