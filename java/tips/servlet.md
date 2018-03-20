##### servlet本质
1，servlet是个规范  
2，servlet容器才是实现处理请求的实体（如tomcat）  
3，而对于需要实现业务的一方来说只需要知道实现一个个servlet即可

一方面容器实现根据servlet规范来实现一套可以处理从请求到调用servlet然后响应的系统，另一方面web框架也是根据servlet规范来实现一套可以根据servlet请求给出相应的应用。是一个面向接口的好例子。    
