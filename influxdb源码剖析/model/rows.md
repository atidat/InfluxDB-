Row：定义为从语句执行后的单行

```go
type Row struct {
    Name 	string				`json:"name,omitempty"`
    Tags	map[string]string	`json:"tags,omitempty"`
    Columns	[]string			`json:"columns,omitempty"`
    Values	[][]interface{}		`json:"values,omitempty"`
    Partial	bool				`json:"partial,omitempty"`
}
```

Row API

- SameSeries：判断两行row是否相同



Rows：定义为*Row的集合，实现了Swap、Less和Len三种方法，故可排序

