#### point 基础类型

**接口**

Point：定义写入数据库的数据

```go
type Point interface {
    Name() []byte	// 返回该point的measurement名
    SetName(string)	// 更新该point的measurement名
    // Tag操作
    Tags() Tags		// 返回该point的tag set
    ForEachTag(fn func(k, v []byte)bool)	// 调用函数fn轮询每一个tag，知道fn返回false
    AddTag(key, value string)	// 创建/更新tag key的值为value
    SetTags(tags Tags)			// 替换tags
    HasTag(tag []byte) bool		// 判断tag是否存在
    // Fields操作
    Fields() (Fields, error)	// 返回该point的fileds
    // Time操作
    Time() time.Time			// 返回该Point的时间戳
    SetTime(t time.Time)		// 更新该Point的时间戳
    UnixNano() int64			// 返回该Point的纳秒时间戳
    HashID() uint64				// 返回Point的key的非加密校验和
    Key() []byte				// 返回该Point的key（tag连接的measurement） TODO
    String() string				// 返回该Point的字符串格式
    MarshalBinary() ([]byte, error)		// 返回该Point的二进制流
    PercisionString(precision string) string	// 返回给Point的指定字符串格式
    RoundedString(d time.Duration) string		// 返回该Point的字符串格式（如果该Point有时间戳，则四舍五入）
    Split(size int) []Point			// 返回相同时间戳的长度小于size的多个point集合
    Round(d time.Duration)			// 四舍五入该point的时间戳
    StringSize() int				// 返回该Point的字符串长度
    AppendString(buf []byte) []byte	// 添加String()的值到指定buf
    FieldIterator() FieldIterator	// 返回该Point的field迭代器（无需在内存中重建field的kv字典）
}
```

FieldIterator：低分配的field迭代器（TODO 什么是低分配？）

```go
type FieldIterator interface {
    Next() bool				// 判断是否还有没轮询的field
    FieldKey() []byte		// 返回当前field的key	
    Type() FieldType		// 返回当前field的类型
    StringValue() string	// 返回当前field的字符串值
    IntegerValue() (int64, error)	// 返回当前field的有符号整形值
    UnsignedValye() (uint64, error)	// 返回当前field的无符号整型值
    BooleanValue() (bool, error)	// 返回当前field的布尔值
    FloatValue() (float64, error)	// 返回当前field的浮点值
    Reset()							// 恢复初始状态
}
```



**类型**

- Points：基于时间戳排序的Point列表

- point：Point接口的默认实现

   ```go
    type point struct {
        time time.Time		// 
        key []byte			// measurement和tags的文本编码，key基于tags的排序
        fields []byte		// field data的文本编码
        ts []byte			// 时间戳的文本编码
        cachedFields map[string]interface{}		// 已解析fields的缓存版本（TODO 什么意思）
        cachedName string	// 已解析key名的缓存版本
        cachedTags Tags		// 已解析tags的缓存版本
        it fieldIterator	// TODO
    }
   ```
Tag：k/v类型的tag pair

   ```go
    type Tag struct {
        Key	[]byte
        Value	[]byte
    }
   ```

- Tags：已排序的Tag的集合

- Fields：point的field的k/v键值对

- fieldIterator

  ```go
  type fieldIterator struct {
      start, end 	int
      key, keybuf []byte
      valueBuf	[]byte
      fieldType	FieldType	// type FieldType int
  }
  ```
  
  

#### point API

ParsePoints：解析字节流为Point的集合


ParsePointsWithPrecision：ParsePoints的precision版本，该函数返回Point切片，gc时这块内存不会被回收

```go
func ParsePoints(buf []byte) ([]Point, error) {
   return ParsePointsWithPrecision(buf, time.Now().UTC(), "n")
}

func ParsePointsWithPrecision(buf []byte, defaultTime time.Time, precision string) ([]Point, error) {
	points := make([]Point, 0, bytes.Count(buf, []byte{'\n'})+1)
	var (
		pos    int
		block  []byte
		failed []string
	)
	for pos < len(buf) {
		pos, block = scanLine(buf, pos)
		pos++

		if len(block) == 0 {
			continue
		}

		start := skipWhitespace(block, 0)

		// If line is all whitespace, just skip it
		if start >= len(block) {
			continue
		}

		// lines which start with '#' are comments
		if block[start] == '#' {
			continue
		}

		// strip the newline if one is present
		if block[len(block)-1] == '\n' {
			block = block[:len(block)-1]
		}

		pt, err := parsePoint(block[start:], defaultTime, precision)
		if err != nil {
			failed = append(failed, fmt.Sprintf("unable to parse '%s': %v", string(block[start:]), err))
		} else {
			points = append(points, pt)
		}

	}
	if len(failed) > 0 {
		return points, fmt.Errorf("%s", strings.Join(failed, "\n"))
	}
	return points, nil

}
```

