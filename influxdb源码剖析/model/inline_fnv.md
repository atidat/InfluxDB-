该文件基于go原生库hash/fnv/fnv，实现了一种hash函数

// 基于原生库 hash/fnv/fnv.go的类型，实现了 Sum64()和Write()
type InlineFNV64a uint64
