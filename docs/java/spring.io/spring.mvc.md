## springMVC

### SpringMVC 的工作原理

- a. 用户向服务器发送请求，请求被 springMVC 前端控制器 DispatchServlet 捕获；
- b. DispatcherServle 对请求 URL 进行解析，得到请求资源标识符（URL），然后根据该 URL 调用 HandlerMapping
将请求映射到处理器 HandlerExcutionChain；
- c. DispatchServlet 根据获得 Handler 选择一个合适的 HandlerAdapter 适配器处理；
- d. Handler 对数据处理完成以后将返回一个 ModelAndView（）对象给 DisPatchServlet;
- e. Handler 返回的 ModelAndView()只是一个逻辑视图并不是一个正式的视图，DispatcherSevlet 通过
ViewResolver 试图解析器将逻辑视图转化为真正的视图 View;
- h. DispatcherServle 通过 model 解析出 ModelAndView()中的参数进行解析最终展现出完整的 view 并返回给
客户端;

### 处理器映射器
- BeanNameUrlHandlerMapping
    - 功能：寻找Controller,根据url请求去匹配bean的name属性url，从而获取Controller
- SimpleUrlHandlerMaping
    - 功能：寻找Controller,根据浏览器url匹配简单url的key，key又Controller的id找到Controller

- ControllerClassNameHandlerMapping
    - 功能：寻找Controller,根据类名（MyController）类名.do来访问,类名首字母小写

3个处理器映射器可以共存。

### 处理器适配器

- SimpleControllerHandlerAdapter
    - 功能：执行controller,调用controller里面方法，返回modelAndView。

- HttpRequestHandlerAdapter

2个处理器适配器可以共存。


### SpringMVC 常用注解
- @requestMapping 用于请求 url 映射。
- @RequestBody 注解实现接收 http 请求的 json 数据，将 json 数据转换为 java 对象。
- @ResponseBody 注解实现将 controller 方法返回对象转化为 json 响应给客户。

### 解决 get 和 post 乱码问题

解决 post 请求乱码:我们可以在 web.xml 里边配置一个 CharacterEncodingFilter 过滤器。 设置为 utf-8. 
解决 get 请求的乱码:有两种方法。对于 get 请求中文参数出现乱码解决方法有两个:
- 1.修改 tomcat 配置文件添加编码与工程编码一致。
- 2.另 外 一 种 方 法 对 参 数 进 行 重 新 编 码 String userName = New 
String(Request.getParameter(“userName”).getBytes(“ISO8859-1”), “utf-8”);