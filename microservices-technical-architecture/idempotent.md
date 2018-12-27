# 幂等与去重处理

幂等一词源自数学概念，在程序中如果相同条件下多次请求对资源的影响表现一致则称请求为幂等请求，对应的接口为幂等接口。

在分布式环境下之所有强调幂等是由于对通信链路的不信任，我们的请求可能由于网络问题而出错进而需要做重试，一般的网络通讯类库都提供重试机制并且默认启用重试功能，而如果我们对应的接口没有做请求去重就可以导致重复处理引发数据错误，所以在分布式系统中接口的幂等性非常重要。一个非常典型的场景如下：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-idempotent1.png?sanitize=true)

服务A请求服务B完成某业务处理，请求已发到服务B并且服务B完成了业务处理，但在响应处理完成时由于网络原因未能发送到服务A或是响应超时触发服务A放弃等待等原因导致服务A误认为请求失败，服务A进行了重试，此时就引发了业务处理异常。

请求方式多见于同步的HTTP、异步的MQ，我们在接口设计之初就要考虑幂等设计，对HTTP而言目前几乎都是REST方式，上文我们介绍过REST概念，它由HTTP请求方法代表对资源的操作方式，实际上这些HTTP请求方式已经隐含了幂等约定：GET、PUT、DELETE原则要保证幂等，而POST不需要幂等。为了支持这个约定我们在GET、PUT、DELETE请求中只要带上要操作的资源Id，代码逻辑都以这一Id为条件展开即可保证幂等。

当然如果真这么简单就好了，在实际开发中还是会有很多例外的，比如某一GET请求实现的是针对资源的Excel导出，虽然结果是幂等的，但这是一个很耗CPU与内存的操作会有一定的延时，请求方可能会误以为超时或网络问题而重试（试想前端的一个导出按钮）进而导致资源的浪费，再比如像用户注册这一POST请求我们要求重试场景下必须保证业务上的幂等性，不能多出相同的用户记录，还有像支付场景下我们可能通过MQ调用，也会面临网络问题但肯定不能因为重试而产生重复消费的情况。这些我们也可以理解成为保证幂等而进行的去重，只是这些幂等的实现成本不同，形式多样。

常见的我们可以通过数据库主键或唯一性约束来实现幂等，这种方式相对简单，但通用性欠缺，因为有些场景是不依赖于数据库的，并且这一方案在性能上需要权衡，毕竟数据库的资源是非常宝贵的，如果预估有会有大量重复请求时这就非常得不优雅了。
针对不同的场景我们可以考虑用分布式锁实现，分布式锁一般适用于需要长时间处理的任务，在任务处理期间防止重复请求，如数据导出、复杂计算等，由于这些操作本身就要求串行处理，所以加锁对性能地影响有限（锁粒度为请求条件），我们也可以使用状态机，很多情况下请求对象都是带状态的，并且状态的跃迁是单向的，如订单的状态多为 `已下单，待付款-> 已付款，待发货 -> 已发货，待签收` ，那么对于发货请求只能是针对已下单，待付款的订单，对其状态进行判断即可实现去重。

上面的方案都是有场景限定的，更通用的做法如下图所示：

![](https://raw.githubusercontent.com/gudaoxuri/Microservices-Architecture/master/resources/images/ms-idempotent2.png?sanitize=true)

基本流程是请求执行业务前先判断请求是否存在，如果存在则返回错误，否则写入请求并执行业务处理，处理完成后更新请求的状态并返回业务处理结果，如果业务处理错误可以选择删除对应的请求以便进行重试。这一流程要求请求必须有可区分是否是重试的标识，并且对业务处理的前置判断和后置更新最好在框架层面实现以对业务操作透明化，做到最小化的业务逻辑侵入。

我们可以借助Redis实现（如果要记录的资源量很大时可以考虑使用BloomFilter，注意精度问题），要求请求方做一定的策略，用Redis记录请求Token，请求Token是请求方为同一请求（包含重试）生成唯一凭证用于判断是否是重复请求，当然这不符合迪米特法则 [Law of Demeter](https://en.wikipedia.org/wiki/Law_of_Demeter)，对于REST请求而言，一个简化的做法是只关注需要语义幂等的操作（如PUT、DELETE、GET）,直接使用URI做为请求Token再加上过期时间，比如 `PUT /user/001` 幂等有效时间30秒，则在30秒内同一个URI请求都视为重复直接过滤，这种做法可简化请求方操作但仅限于REST请求且符合REST规范。

主流的MQ实现在 `autocommit=true` 时天然实现了幂等，但考虑业务处理可能出错的情况我们一般会将autocommit设置成false，在业务处理成功后再提交，这时就需要使用上述幂等方案了：在接收到消息时写入请求Token以实现去重判断（Token可为Topic+Offset）提交后删除Token，整体上可以做到对业务透明。

幂等的业务操作部分成功时如何处理？比如业务处理有2个步骤而只成功了1个，那是否允许请求重试？一个最直接的方法是将2个步骤用事务包裹，要么全部成功，要么全部失败，失败后删除请求Token允许重试。我们还有会遇到一些复杂的情况，微服务化后很有可能这2步操作是对不同服务的调用，此时我们只有两个选择：1）引入分布式事务（见上文），分布式事务相对较重，如非必须建议慎用，2）要求被调用的接口也实现幂等，这种做法相对优雅，也比较推荐，从中我们发现幂等是有传递性的，它要求我们请求对应的接口及其接口调用的子孙接口都必须幂等，这也是幂等在微服务下实施的难点，所以使用一套对业务透明的通用幂等方案非常有必要。

幂等在服务设计中非常重要，在一些关键业务中如不实现幂等轻则产生一些脏数据，重则会导致资产损失。