1. # High level architecture

    ![Eureka High level Architecture](eureka_architecture.png)

    The architecture above depicts how Eureka is deployed at Netflix and this is how you would typically run it. There is one eureka cluster per region which knows only about instances in its region. There is at the least one eureka server per zone to handle zone failures.<br>
    
    Services register with Eureka and then send heartbeats to renew their leases every 30 seconds. If the client cannot renew the lease for a few times, it is taken out of the server registry in about 90 seconds. The registration information and the renewals are replicated to all the eureka nodes in the cluster. The clients from any zone can look up the registry information (happens every 30 seconds) to locate their services (which could be in any zone) and make remote calls.<br>

    每个区域有一个eureka集群， 该集群只知道区域内的示例。 为了应对zone failure， 每个可用区至少有一个eureka服务器。<br>
    服务向eureka注册自身， 然后每30秒发送heartbeat来更新leases(怎么翻译？租约？不太行)。 如果连续几次没有更新lease， 就在约90秒内从eureka服务器的注册表中拿出来。 注册信息和更新（续订）会复制到集群中的所有eureka节点。 任何可用区的clients都可以查询注册信息， 这每30秒发生一次， 从而定位服务(可能在任何可用区)并发起远程调用。<br>
2. 