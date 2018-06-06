# PWA主要技术Service Worker相关需要了解的知识 #

## Service Worker概念和用法 ##
我们平常浏览器窗口中跑的页面运行的是主JavaScript线程，DOM和window全局变量都是可以访问的。而Service Worker是走的另外的线程，可以理解为在浏览器背后默默运行的一个线程，脱离浏览器窗体，因此，window以及DOM都是不能访问的。

由于Service Worker走的是另外的线程，因此，就算这个线程翻江倒海也不会阻塞主JavaScript线程，也就是不会引起浏览器页面加载的卡顿之类。同时，由于Service worker设计为完全异步，同步API（如XHR和localStorage）不能在Service Worker中使用。

除了上面的些限制外，Service Worker对我们的协议也有要求，就是必须是https协议的，但本地开发也弄个https协议是很麻烦的，好在还算人性化，Service Worker在http://localhost或者http://127.0.0.1这种本地环境下的时候也是可以跑起来的。

最后，Service workers大量使用Promise，因为通常它们会等待响应后继续，并根据响应返回一个成功或者失败的操作，这些场景非常适合Promise。

## Service Worker的兼容性 ##
桌面端Chrome和Firefox可用，IE不可用。最新safari，edge已支持。https://caniuse.com/#search=service%20work

## 和PWA技术的关系 ##
PWA的核心技术包括：
- web App Manifest – 在主屏幕添加app图标，定义手机标题栏颜色之类
- Service Worker – 缓存，离线开发，以及地理位置信息处理等
- App Shell – 先显示APP的主结构，再填充主数据，更快显示更好体验
- Push Notification – 消息推送，之前有写过“简单了解HTML5中的Web Notification桌面通知”

有此可见，Service Worker仅仅是PWA技术中的一部分，但是又独立于PWA。也就是虽然PWA技术面向移动端，但是并不影响我们在桌面端渐进使用Service Worker，考虑到自己厂子用户70%~80%都是Chrome内核，收益或许会比预期的要高。

## 你需要知道 CacheStorage ##
Cache 对象受到 CacheStorage 的管理，在 W3C 规范中，CacheStorage 对应到内核的ServiceWorkerCacheStorage 对象。前端从这些情况可以得到哪些信息呢？资源的存储不是按照资源的域名处理的，而是按照 Service Worker 的origin 来处理，所以 Cache 的资源是无法跨域共享的，意思就是说，不同域的 Service Worker 无法共享使用对方的 Cache，即使是 Foreign Cache 请求的跨域资源，同样也是存放在这个 origin 之下。因为ServiceWorkerCache 通过 cacheName 标记缓存版本，所以就会存在多个版本的 ServiceWorkerCache 资源。为什么需要 cacheName 来标记版本呢？假设当前域名下所有的覆盖式发布的静态资源和接口数据全部存储在同一个 cacheName 里面，业务部署更新后，无法识别旧的冗余资源，单靠前端无法完全清除。这是因为 Service Worker 不知道完整的静态资源路径表，只能在客户端发起请求时去做判断，那些当前不会用到的资源不代表以后一定不会使用到。假如静态资源是非覆盖式发布，那么冗余的资源就更多了。这里要特别注意的是，Cache 不会过期，只能显式删除如果版本更新后，更换 cacheName，这意味着旧 cacheName 的资源是不会使用到了，业务逻辑可以放心的把旧cacheName 对应的所有资源全部清除，而无需知道完整的静态资源路径表。那cacheName 是不是只是在这种情况下才能发挥作用呢？其实不是的，使用过 webpack 工具的同学知道 vender配置，vender 主要是把最不经常变动的第三方的库文件打包在一起，避免与频繁更新的资源打包一起，提高客户端缓存使用率，还有就是 common 的配置，把公用的组件打包在一起，减少代码冗余，因此，cacheName 也可以根据这种情况进行设置，最大化利用缓存空间，提高缓存利用率。
由于 Service Worker 相关缓存的底层存储都使用了系统的文件系统（File System），而文件系统一般是不支持多进程访问的，当统一域名下有两个不同的 Service Worker 是无法同时对同一资源进行操作的。

## 缓存大小限制 ##
一般来说，Service Worker 层面对 ServiceWorkerCache 的限制都会大于浏览器对每个域名的限制，所以，通常可理解为，ServiceWorkerCache 仅受浏览器 QuotaManager 对域名可使用存储的限制。对于前端开发同学来说，必须有清理冗余缓存的业务逻辑，并且提高缓存资源的使用率。
比如，系统磁盘可用空间为 570M， 浏览器全局已使用空间为 30M，那么 每个域名可使用 Temporary 类型存储限额 = （570+30）/ 3 / 5 = 40M。虽然 ServiceWorkerCache 在 Service Worker 层面的限制为 512M，非常大，但它也不能超出每个域名的限制（40M），即同一域名下的 ServiceWorkerCache 也只能使用 40M。

## Service Worker更多的应用场景 ##
Service Worker除了可以缓存和离线开发，其可以应用的场景还有很多，举几个例子（参考自MDN）：
 - 后台数据同步
 - 响应来自其它源的资源请求，
 - 集中接收计算成本高的数据更新，比如地理位置和陀螺仪信息，这样多个页面就可以利用同一组数据
 - 在客户端进行CoffeeScript，LESS，CJS/AMD等模块编译和依赖管理（用于开发目的）
 - 后台服务钩子
 - 自定义模板用于特定URL模式
 - 性能增强，比如预取用户可能需要的资源，比如相册中的后面数张图片



## ServiceWorkers/WebWorkers/WebSockets的区别 ##
Service worker非常适合创建一个优秀的离线web app。当你可以的时候，它们让你与服务器交互(从服务器获取新数据，或者将更新的信息推回到服务器)，这样你的应用程序就可以工作，而不管你的用户的连接情况如何。

Web Workers对于任何具有强大功能的Web app来说都是非常完美的。它们让你把复杂的功能推到后台线程中，这样你的主JS就可以继续工作，比如设置事件监听器和其他UI交互。然后，当复杂的功能完成时，他们会报告他们的结果，让JS更新Web app。

WebSockets客户端和服务器之间有一个持久的连接，并且双方可以在任何时候开始发送数据。WebSockets对任何需要经常与服务器进行通信、并且可以从服务器能够直接与客户沟通的web app来说是非常完美的。



## service work 进阶 ##
https://blog.csdn.net/i10630226/article/details/78885664