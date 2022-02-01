#### Measurement API

- ReadMeasurement(name []byte) (orgID, bucketID []byte, err error)

- CreateMeasurement(org, bucket []byte) ([]byte, error)



```go
func ReadMeasurement(name []byte) (orgID, bucketID []byte, err error) {
    // 返回organizeID（name的前8个字节）和bucketID（name的后8个字节）
	return name[:OrgIDLength], name[len(name)-BucketIDLength:], nil
}
```

```go
func CreateMeasurement(org, bucket []byte) ([]byte, error) {
    // 取org的前8个字节和bucket的后8个字节拼凑返回
	name := make([]byte, 0, MeasurementLength)
	name = append(name, org[:OrgIDLength]...)
	return append(name, bucket[:BucketIDLength]...), nil
}
```

