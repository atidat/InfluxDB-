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
- Tag：k/v类型的tag pair

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

- ParsePoints：解析字节流为Point的集合

- ParsePointsString：解析字符串类型的Point信息为Point集合

- ParsePointsWithPrecision：ParsePoints的precision版本，该函数返回Point切片，gc时这块内存不会被回收

- parsePoint：解析buf为Point，buf内容主要包括measurement+tag、fields和timestamp，'\n'分割

    ```go
    func ParsePoints(buf []byte) ([]Point, error) {
       return ParsePointsWithPrecision(buf, time.Now().UTC(), "n")
    }
    
    func ParsePointsWithPrecision(buf []byte, defaultTime time.Time, precision string) ([]Point, error) {
        // bytes.Count 表意是统计'\n的数量，本质为统计按'\n'分隔后的元素数量
        points := make([]Point, 0, bytes.Count(buf, []byte{'\n'})+1)
        var (
            pos    int
            block  []byte
            failed []string
        )
        for pos < len(buf) {
            // scanline返回扫描到的第一个换行符位置pos和buf[start:pos]内容
            // 如buf为 " asdfasdf\nasdfasd\n"，则block为" asdfasdf\n"
            pos, block = scanLine(buf, pos)
            pos++
    
            if len(block) == 0 { continue }
            // 返回block后第一个非空白字符的位置
            start := skipWhitespace(block, 0)
            if start >= len(block) { continue }
            if block[start] == '#' { continue } // #开头的为注释行
            if block[len(block)-1] == '\n' { block = block[:len(block)-1] } // 删除换行符
    
            // block[start:]示例："asdfasdf"
            pt, err := parsePoint(block[start:], defaultTime, precision)
            if err != nil {
                failed = append(failed, fmt.Sprintf("unable to parse '%s': %v", string(block[start:]), err))
            } else {
                points = append(points, pt)
            }
        }
        if len(failed) > 0 { return points, fmt.Errorf("%s", strings.Join(failed, "\n")) }
        return points, nil
    }
    
    // parsePoint：point buf分为3个block，分别是measurement+tag、field和timestamp
    func parsePoint(buf []byte, defaultTime time.Time, precision string) (Point, error) {
        // part1: scan the first block which is measurement[,tag1=value1,tag2=value2...]
        // pos：measurement+tag部分的结尾位置，key：measurement+tags
        pos, key, err := scanKey(buf, 0)
        if err != nil { return nil, err }
        // measurement name is required
        if len(key) == 0 { return nil, fmt.Errorf("missing measurement") }
        if len(key) > MaxKeyLength { return nil, fmt.Errorf("max key length exceeded: %v > %v", len(key), MaxKeyLength) }
    
        // part2: scan the second block is which is field1=value1[,field2=value2,...]
        // pos：fields k/v部分的结尾位置，fields：{k1/v1, k2/v2, ..., kn/vn}
        pos, fields, err := scanFields(buf, pos)
        if err != nil { return nil, err }	
        if len(fields) == 0 { return nil, fmt.Errorf("missing fields") }
    
        var maxKeyErr error
        // walkFields迭代每一对fields的k/v，如果有一对k/v不满足fn，则退出迭代
        err = walkFields(fields, func(k, v []byte) bool {
            // seriesKeySize 返回measurement+tag + 4 + 单个key的长度
            // TODO 不明白measurement+tag的长度在这里的意义
            if sz := seriesKeySize(key, k); sz > MaxKeyLength {
                maxKeyErr = fmt.Errorf("max key length exceeded: %v > %v", sz, MaxKeyLength)
                return false
            }
            return true
        })
        if err != nil { return nil, err }
        if maxKeyErr != nil { return nil, maxKeyErr }
    
        // part3: scan the last block which is an optional integer timestamp
        pos, ts, err := scanTime(buf, pos)
        if err != nil { return nil, err }
    
        pt := &point{
            key:    key,
            fields: fields,
            ts:     ts,
        }
    
        if len(ts) == 0 {
            pt.time = defaultTime
            // SetPrecision：设置point的time，时间精度为precision指定的格式，如n代表纳秒
            pt.SetPrecision(precision)
        } else {
            // parseIntBytes是个0内存分配数字解析，先把数字byte切片，先转化成string，再转换成int64
            ts, err := parseIntBytes(ts, 10, 64)
            if err != nil { return nil, err }
            // SafeCalcTime：时间戳 * 精度作为UTC()的基础，返回Time格式的时间
            pt.time, err = SafeCalcTime(ts, precision)
            if err != nil { return nil, err }
    
            // 检查时间戳信息后的位置是否都是空格，存在非空格字符则非法
            for pos < len(buf) {
                if buf[pos] != ' ' { return nil, ErrInvalidPoint }
                pos++
            }
        }
        return pt, nil
    }
   ```

- ParseKey：从[]byte的Point信息中获取measurement和tags信息

- ParseKeyBytes：同上

- ParseKeyBytesWithTags：同上，解析成tags的具体实现

    ```go
    func ParseKey(buf []byte) (string, Tags) {
    	name, tags := ParseKeyBytes(buf)
    	return string(name), tags
    }
    
    func ParseKeyBytes(buf []byte) ([]byte, Tags) {
    	return ParseKeyBytesWithTags(buf, nil)
    }
    
    func ParseKeyBytesWithTags(buf []byte, tags Tags) ([]byte, Tags) {
        // 扫描buf内容，返回下一part的状态和name尾的位置
    	state, i, _ := scanMeasurement(buf, 0)
    
    	var name []byte
    	if state == tagKeyState {
    		tags = parseTags(buf, tags)
    		name = buf[:i-1] // i是逗号后一位的位置
    	} else {
            // 如果下一part不是tagKeyState的状态，意味着buf中没有tag的信息
    		name = buf[:i]
    	}
    	return unescapeMeasurement(name), tags
    }
    ```

- ParseTags

- ParseTagWithTags

- parseTags：解析tags的具体实现

  ```go
  func ParseTags(buf []byte) Tags {
  	return parseTags(buf, nil)
  }
  
  func ParseTagsWithTags(buf []byte, tags Tags) Tags {
  	return parseTags(buf, tags)
  }
  
  func parseTags(buf []byte, dst Tags) Tags {
  	if len(buf) == 0 { return nil }
  
  	n := bytes.Count(buf, []byte(","))
  	if cap(dst) < n {
  		dst = make(Tags, n)
  	} else {
  		dst = dst[:n]
  	}
  	if dst == nil {
  		dst = Tags{}
  	}
  
  	var i int
      // 从buf中扫描每一对k/v，关键扫描点=和,
  	walkTags(buf, func(key, value []byte) bool {
  		dst[i].Key, dst[i].Value = key, value
  		i++
  		return true
  	})
  	return dst[:i]
  }
  ```

- ParseName：返回measurement名

- ValidPrecision：仅支持ns、us、ms和s四种精度

- scanMeasurement：返回已扫描状态后的下一个状态、name后的一个位置和错误

- scanKey：使用一块新内存存储measurement+有序tags数据

  ```go
  /* 
  1、扫描measurement
  2、轮询tag key，确保未使用已保留tag key
  3、轮询tag key，判断tag key是否重复或有序
  4、使用插入排序，重排indices以获得有序tag key的索引
  5、轮询indices，判断tag key是否重复或有序
  */
  func scanKey(buf []byte, i int) (int, []byte, error) {
  	start := skipWhitespace(buf, i)
  	i = start
  	sorted := true // 先假定tag有序
      // indices存储每一个tag开始的位置，如'cpu,host=a,region=b,zone=c'，则indices=[4,11,20]
  	indices := make([]int, 100)
  	commas := 0
  
  	state, i, err := scanMeasurement(buf, i)
  	if err != nil { return i, buf[start:i], err }
  	if state == tagKeyState { // Optionally scan tags if needed.
          // i：下一个位置；commas：逗号的数量；indices：每个tag key的首字符位置
          // 其实根据scanTags的代码看，根据len(indices)可以知道commas，即commas无用
  		i, commas, indices, err = scanTags(buf, i, indices)
  		if err != nil { return i, buf[start:i], err }
  	}
  
  	// 迭代每一个tag key，确保没有使用已保留的tag key，比如_measurement和_field
  	for j := 0; j < commas; j++ {
  		_, key := scanTo(buf[indices[j]:indices[j+1]-1], 0, '=')
  		for _, reserved := range reservedTagKeys {
  			if bytes.Equal(key, reserved) { return i, buf[start:i], fmt.Errorf("cannot use reserved tag key %q", key)
  			}
  		}
  	}
  
  	// 通过判断相邻的两个tag key，判断tag key是否有序或重复tag key
  	for j := 0; j < commas-1; j++ {
  		// get the left and right tags
  		_, left := scanTo(buf[indices[j]:indices[j+1]-1], 0, '=')
  		_, right := scanTo(buf[indices[j+1]:indices[j+2]-1], 0, '=')
  		if cmp := bytes.Compare(left, right); cmp > 0 {
  			sorted = false; break
  		} else if cmp == 0 {
  			return i, buf[start:i], fmt.Errorf("duplicate tags")
  		}
  	}
  
  	if !sorted && commas > 0 {
  		measurement := buf[start : indices[0]-1]
  
          // 使用插入排序，对tag key进行排序。不过buf中没改变内容，改变的是indices内索引的位置
  		indices := indices[:commas]
  		insertionSort(0, commas, buf, indices)
  
  		// 使用一块新内存空间，用来保存measurement+有序tags数据（可以理解，更改buf确实危险）
  		b := make([]byte, len(buf[start:i]))
  		pos := copy(b, measurement)
  		for _, i := range indices {
  			b[pos] = ','
  			pos++
  			_, v := scanToSpaceOr(buf, i, ',')
  			pos += copy(b[pos:], v)
  		}
  
  		// 基于已经有序的indices，二次确认tag key是否有序（没想明白为什么再确认一次）
  		for j := 0; j < commas-1; j++ {
  			_, left := scanTo(buf[indices[j]:], 0, '=')
  			_, right := scanTo(buf[indices[j+1]:], 0, '=')
  			if bytes.Equal(left, right) { return i, b, fmt.Errorf("duplicate tags") }
  		}
  		return i, b, nil
  	}
  	return i, buf[start:i], nil
  }
  
  ```

- scanTags：

  ```go
  // scanTags examines all the tags in a Point, keeping track of and
  // returning the updated indices slice, number of commas and location
  // in buf where to start examining the Point fields.
  func scanTags(buf []byte, i int, indices []int) (int, int, []int, error) {
  	var (
  		err    error
  		commas int
  		state  = tagKeyState
  	)
  
  	for {
  		switch state {
  		case tagKeyState:
  			// 提前手动扩容（为什么不使用append？因为append会在原来的内存基础上扩容，万一新的后半部分的空间，有其他有效数据，会怎么样呢）
  			if commas >= len(indices) {
  				newIndics := make([]int, cap(indices)*2)
  				copy(newIndics, indices)
  				indices = newIndics
  			}
  			indices[commas] = i
  			commas++
  
  			i, err = scanTagsKey(buf, i)
  			state = tagValueState // tag value always follows a tag key
  		case tagValueState:
  			state, i, err = scanTagsValue(buf, i)
  		case fieldsState:
  			// Grow our indices slice if we had exactly enough tags to fill it
  			if commas >= len(indices) {
  				// The parser is in `fieldsState`, so there are no more
  				// tags. We only need 1 more entry in the slice to store
  				// the final entry.
  				newIndics := make([]int, cap(indices)+1)
  				copy(newIndics, indices)
  				indices = newIndics
  			}
  			indices[commas] = i + 1
  			return i, commas, indices, nil
  		}
  		if err != nil { return i, commas, indices, err }
  	}
  }
  ```

- scanFields：返回fields最后的value后的索引、第一个field key~最后一个field value的数据和错误

  ```go
  func scanFields(buf []byte, i int) (int, []byte, error) {
  	start := skipWhitespace(buf, i)
  	i = start
  	quoted := false
  	equals := 0	// 记录‘=’的数量
  	commas := 0 // 记录‘,’的数量
  
      // 里面各种检查field key/value1是否非法
  	for {
  		if i >= len(buf) { break }
  		if buf[i] == '\\' && i+1 < len(buf) { // 转义字符
  			i += 2; continue
  		}
  
  		// field value被“”包围的场景
  		if buf[i] == '"' && equals > commas {
  			quoted = !quoted
  			i++
  			continue
  		}
  
  		// If we see an =, ensure that there is at least on char before and after it
  		if buf[i] == '=' && !quoted {
  			equals++
  
  			// check for "... =123" but allow "a\ =123"
  			if buf[i-1] == ' ' && buf[i-2] != '\\' {
  				return i, buf[start:i], fmt.Errorf("missing field key")
  			}
  
  			// check for "...a=123,=456" but allow "a=123,a\,=456"
  			if buf[i-1] == ',' && buf[i-2] != '\\' {
  				return i, buf[start:i], fmt.Errorf("missing field key")
  			}
  
  			// check for "... value="
  			if i+1 >= len(buf) {
  				return i, buf[start:i], fmt.Errorf("missing field value")
  			}
  
  			// check for "... value=,value2=..."
  			if buf[i+1] == ',' || buf[i+1] == ' ' {
  				return i, buf[start:i], fmt.Errorf("missing field value")
  			}
  
  			if isNumeric(buf[i+1]) || buf[i+1] == '-' || buf[i+1] == 'N' || buf[i+1] == 'n' {
  				var err error
  				i, err = scanNumber(buf, i+1)
  				if err != nil { return i, buf[start:i], err }
  				continue
  			}
  			// If next byte is not a double-quote, the value must be a boolean
  			if buf[i+1] != '"' {
  				var err error
  				i, _, err = scanBoolean(buf, i+1)
  				if err != nil { return i, buf[start:i], err }
  				continue
  			}
  		}
  
  		if buf[i] == ',' && !quoted { commas++ }
  		// reached end of block?
  		if buf[i] == ' ' && !quoted { break }
  		i++
  	}
  
  	if quoted { return i, buf[start:i], fmt.Errorf("unbalanced quotes") }
  	// check that all field sections had key and values (e.g. prevent "a=1,b"
  	if equals == 0 || commas != equals-1 {
  		return i, buf[start:i], fmt.Errorf("invalid field format")
  	}
  	return i, buf[start:i], nil
  }
  ```

  
