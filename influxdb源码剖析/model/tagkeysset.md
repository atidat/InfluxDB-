TagKeysSet：为融合tags提供set操作

```go
type TagKeysSet struct {
    i		int
    keys	[2][][]byte
    tmp		[][]byte
}
```

TagKeysSet API

func(s *TagKeysSet)Clear()：清除TagKeysSet的数据

func(s *TagKeysSet)KeysBytes() \[][]byte：返回keys其中一块数据（字典序）

func(s *TagKeysSet)Keys() \[]string：KeysBytes的[]string版本的实现

func(s *TagKeysSet)IsSupersetKeys(other Tags) bool：判断s是否是other的超集

func(s *TagKeysSet)IsSupersetBytes(other \[][]byte) bool：byte版本的超集判断

func(s *TagKeysSet)UnionKeys(other Tags)：并集merge，把other的key合入s中

func(s *TagKeysSet)UnionBytes(other \[][]byte)：byte版本的并集merge实现

