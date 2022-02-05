Statistic：监控服务所使用的统计

```go
type Statistic struct {
    Name	string					`json:"name"`
    Tags	map[string]string		`json:"tags"`	
    Values	map[string]interface{}	`json:"values"`
}
```

StatisticTags：可以和其他map融合，而不改变任意map

