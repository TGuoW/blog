有些web站点会为每个用户生成特定版本的URL来追踪用户的身份，一般会对真正的URL进行扩展，在路径的开始或结束的地方添加一些状态信息。

用户浏览站点时，web服务器会动态生成一些超级链接以维护URL的状态信息。这种改动过的包含用户状态信息的URL被称为胖URL。

通过胖URL技术Web服务器可以将若干个独立的HTTP事务绑定成一个会话。当用户首次访问一个web网点时，会生成唯一ID，服务器会将该ID加入URL，然后重新导向胖URL。

以后URL再收到对胖URL请求，就会将该用户相关所有的增量状态重写进输出超级链接，使其成为胖URL，以维护用户ID.

胖URL的缺点： 

1. 胖URL会给新用户带来困扰 
2. 无法实现共享URL 
3. 胖URL破坏了可用于公共访问缓存 
4. 重写HTML页面会使URL变胖，带来额外的服务器负荷 
5. 当用户进行站点跳转时，就会丢失胖URL会话 
6. 在会话间是非持久的