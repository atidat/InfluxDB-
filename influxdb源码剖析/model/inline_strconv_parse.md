解析字节流为各种类型数据，如int64、uint64、float64和bool



func parseIntBytes(b []byte, base  int, bitSize int) (int64, error)

func parseUintBytes(b []byte, base int, bitSize int) (uint64, error)

func parseFloatBytes(b []byte, bitSize int) (float64, error)

func parseBoolBytes(b []byte) (bool, error)

上述的操作都会通过unsafeBytesToString，先将[]byte转化成string

```
// 在原内存空间上进行类型转换
func unsafeBytesToString(in []byte) string {
   return *(*string)(unsafe.Pointer(&in))
}
```

