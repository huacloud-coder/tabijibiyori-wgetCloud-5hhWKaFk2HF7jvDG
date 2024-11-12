
## 一、概述


    木舟平台分为微服务平台和物联网平台， 上面几篇都是介绍如何通过网络组件接入设备，那么此篇文章就细致介绍下在木舟平台下如何构建微服务。


      木舟 (Kayak) 是什么?


       木舟(Kayak)是基于.NET6\.0软件环境下的surging微服务引擎进行开发的, 平台包含了微服务和物联网平台。支持异步和响应式编程开发，功能包含了物模型,设备,产品,网络组件的统一管理和微服务平台下的注册中心，服务路由，模块，中间服务等管理。还有多协议适配(TCP,MQTT,UDP,CoAP,HTTP,Grpc,websocket,rtmp,httpflv,webservice,等),通过灵活多样的配置适配能够接入不同厂家不同协议等设备。并且通过设备告警,消息通知,数据可视化等功能。能够让你能快速建立起微服务物联网平台系统。


## 二、构建服务


创建服务接口，继承IServiceKey，添加特性\[ServiceBundle("api/{Service}/{Method}")] 配置routepath，代码如下：




```
   [ServiceBundle("api/{Service}/{Method}")]
   public interface ITestApiService:IServiceKey
   { 
       public Task<string> SayHello(string name);
   }
```


创建服务实例，继承ProxyServiceBase, ITestApiService, ISingleInstance，如果只是业务处理只需继承ProxyServiceBase，继承ISingleInstance表示注入的生命周期 为单例模式，添加特性ModuleName标识一个服务多个实例，可以在调用的时候传入ServiceKey




```
    [ModuleName("Test")]
    public class TestService : ProxyServiceBase, ITestApiService, ISingleInstance
    {
        public Task<string> SayHello(string name)
        {
            return Task.FromResult($"{name} say:hello world");
        }
    }
```


 


## 二、身份鉴权


webapi调用必然会牵涉到身份鉴权，用户登录问题，而surging 已经集成了一套jwt验证机制


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112073448118-1268457881.png)


然后在Stage配置节上配置ApiGetWay




```
   "ApiGetWay": {
     "AccessTokenExpireTimeSpan": "240",
     "AuthorizationRoutePath": "api/sysuser/authentication",//身份鉴权服务的routepath
     "AuthorizationServiceKey": null,
     "TokenEndpointPath": "api/oauth2/token",//映射调用的routepath
     "CacheMode": "MemoryCache" //MemoryCache or  gateway.Redis save token
   }
```


 


然后在接口方法上加上  \[Authorization(AuthType \= AuthorizationType.JWT)] 特性，服务调用就要进行身份鉴权




```
    public interface IModuleService : IServiceKey
    {
        [Authorization(AuthType = AuthorizationType.JWT)]
        Taskbool>> Add(ModuleModel model);

        [Authorization(AuthType = AuthorizationType.JWT)]      
        Taskbool>> Modify(ModuleModel model);

        [Authorization(AuthType = AuthorizationType.JWT)]
        Task>> GetPageAsync(ModuleQuery query);

    }
```


## 三、缓存拦截


surging 可以支持拦截缓存，可以通过ServiceCacheIntercept特性进行配置，获取缓存可以通过CachingMethod.Get， 删除缓存可以通过CachingMethod.Remove，可以支持MemoryCache,Redis, 可以支持一，二级缓存，


启用EnableStageCache表示网关调用也可以走缓存拦截(注：不支持模型参数）




```
 [ServiceBundle("api/{Service}/{Method}")]
 public interface IProductService : IServiceKey
 {
     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts", CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
     Taskbool>> Add(ProductModel model);

     [Authorization(AuthType = AuthorizationType.JWT)]
     Task> GetProduct(int id);

     [Authorization(AuthType = AuthorizationType.JWT)]
     Task>> GetPageAsync(ProductQuery query);

     [Authorization(AuthType = AuthorizationType.JWT)]
     Task>> GetProductByCondition(ProductQuery query);

     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts", CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
     [ServiceLogIntercept]
     Taskbool>> DeleteById(List<int> ids);

     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts", CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
     Taskbool>> Modify(ProductModel model);


     [Authorization(AuthType = AuthorizationType.JWT)]
     Taskbool>> Validate(ProductModel model);

     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts",  CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
     Taskbool>> Stop(List<int> ids);

     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Remove, "GetProducts", CacheSectionType = "ddlCache", Mode = CacheTargetType.MemoryCache, EnableStageCache = true)]
     Taskbool>> Open(List<int> ids);

     [Authorization(AuthType = AuthorizationType.JWT)]
     [ServiceCacheIntercept(CachingMethod.Get, Key = "GetProducts", CacheSectionType = "ddlCache", EnableL2Cache = false, Mode = CacheTargetType.MemoryCache, Time = 480, EnableStageCache = true)]
     Task>> GetProducts();


 }
```


参数如果是非模型集合类型的参数，缓存key 会取第一个参数值，如果是模型参数就需要添加CacheKey特性，代码如下：




```
    public class PropertyThresholdQuery
    {
        [CacheKey(1)]
        public string PropertyCode {  get; set; }
        [CacheKey(2)]
        public string ProductCode { get; set; }
        [CacheKey(3)]
        public string DeviceCode { get; set; }
    }
```


## 四、服务管理


1\.平台是支持服务路由管理，此项功能除了可以查看元数据，服务节点，服务规则外，还可以在权重轮询负载算法情况下，改变权重可以让更多的访问调用到此服务节点上，还有可以优雅的移除服务节点


选择权重轮询负载分流算法，代码如下：




```
    [ServiceBundle("api/{Service}/{Method}")]
    public interface ITestApiService:IServiceKey
    {
        // [Authorization(AuthType = AuthorizationType.JWT)]
        [Command(ShuntStrategy =AddressSelectorMode.RoundRobin)]
        public Task<string> SayHello(string name);
    }
```


以下是编辑权重


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112081432912-212886244.png)


 


2\. 热部署中间服务


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112081547378-1105737965.png)


 3\. 黑白名单，添加IP地址或者IP段就能限制相关IP访问


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112082049029-1603276307.png)


 就比如访问api/testapi,结果如下：


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112082215551-66182892.png)


 4\.支持swagger API文档


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112082353401-1671071729.png)


## 五、分布式链路追踪


支持skywalking 分布式链路追踪


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112084233448-1870397079.png)


## 六 、构建发布


1\. 微服务发布：


发布微服务的时候，需要引用的是微服务，不要引用stage, 如下图


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112085127235-1888040987.png)


 2\. 网关发布， 引用服务接口和聚合服务(中间服务)模块，还有stage 模块


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112085346142-631529680.png)


 


![](https://img2024.cnblogs.com/blog/192878/202411/192878-20241112085430153-799648270.png)


 


## 七、总结


 以上是木舟平台如何构建服务，平台定于11月20日发布1\.0社区版本。也请大家到时候关注捧场。


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
