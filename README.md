# logQuery
设计思路

1、日志客户端
	扩展slf4j接口（个人对其不是非常了解，临时看了下感觉不太容易下手，以前看到过对logback的Appender扩展的，我从这里下手的）。每次append将消息收集到内存中的queue中，到达一定大小 或 时间发送给日志服务器（采用socket通信，自定义了下消息类型，int类型的总长度 + byte类型的类型 + 消息具体内容的对象流）。
	针对发送采用了策略模式，便于扩展
2、日志收集端
	启动serverSocket，接受客户端连接，连接后为其开启一个线程服务。线程循环着接受客户端传来的数据，首先读取自定义消息 int类型的总长度，然后使用readFully等待获取到所有数据，转换为具体对象。将数据批量存到mysql数据库中，此时获取到了其主键ID，对每条日志内容进行分词（根据空格拆分，去重及过滤空白字符），放入key为词，value为RoaringBitmap（ID）的ConcurrentHashMap中。【该ConcurrentHashMap为单例】

3、日志查询端
	前端ajax请求，解析请求数据，此处通过请求的时间 从DB里按时间倒叙筛选出只含ID的数据集，然后通过请求的关键词信息 对 Bitmap进行相应操作（与、或、非），筛选出合适的id，再次根据排序好的id去数据库中捞取数据（默认limit为30）
		a、ajax跨域问题 通过拦截器拦截，设计消息头为"Access-Control-Allow-Origin", "*"解决


---------------------------
索引思路：
D1种
	按 词 进行分表，表中存的是含有该词的 日志ID（每个ID至多存一次） 及 日志时间【这两个字段都做索引】。
	便于进行联合搜索，另外也进行了数据的持久化
	
D2种
	使用Bitmap，存在内存中便于进行操作
	问题：
		a、占用内存较多
			解决思路：
				RoaringBitmap可以一定程度上优化该问题
		b、题目要求是在 日志查询端 中查询，而Bitmap是在 日志收集端 中建立的
			解决思路：
				思路1、Bitmap持久化处理，定时定量同步给 日志查询端
				思路2、将Store单独独立出一个系统，提供一些相关的基本服务，比如：增删改查。其他系统通过RPC或HTTP请求。可以一定减少其他系统对该库的连接 及 将索引及数据相关操作由一个系统管理，便于维护
				思路3、将日志查询端 和 日志收集端 系统合并为一个。
				
		c、不方便与DB的数据进行操作
			时间上依赖DB排序，但关键词的索引依赖内存中Bitmap，造成从DB中根据时间搜索出数据来都会内存爆掉
				解决思路：
					思路1、拆表，某表只存id和时间 及 level、host等一些数据量很小的数据 ，另外有个表存放id 和 全部数据信息。
						可以通过时间筛选出id
					思路2、查询时根据时间范围只筛选出ID集合，然后进行bitMap运算，然后再根据ID及limit去控制最终返回的数据