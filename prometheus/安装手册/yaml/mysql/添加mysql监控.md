通过prometheus安装手册可知
Exporters将数据采集的target通过http的形式暴露给Prometheus server，Prometheus server通过访问该exporter提供的endpoints端点，获取到需要采集的监控数据。
而Exporters分为两类：

直接采集：这类的exporters内置在了相应的应用中，能够直接提供target端点，比如etcd、kubernetes组件，都直接内置了用于向Prometheus暴露监控数据的端点。

间接采集：原有的监控目标不支持prometheus，需要通过prometheus提供的Client Library编写该监控目标的监控采集程序，比如redis、tomcat、mysql等应用，需要有外置的exporters先采集应用的监控项，再通过exporters的http接口把metrics暴露给prometheus server

所以我们需要安装mysql-exporter同时暴露一个svc
再创建一个ServiceMonitor监控暴露出的svc

完成之后如下图所示：![image](/uploads/a59c6e8eaf82f80dfd5e32f6bda26999/image.png)