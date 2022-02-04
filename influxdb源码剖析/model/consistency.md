ConsistencyLevel表示写入成功返回之前，所需要的复制条件



type ConsistencyLevel int



const (

​	ConsistencyLevelAny	iota	  // 允许提示切换，此时还没发生写入

​	ConsistencyLevelOne				// 至少需要一个数据节点确认写入

​	ConsistencyLevelQuorum		// 需要法定数量的数据节点确认写入

​	ConsistencyLevelAll				   // 

)