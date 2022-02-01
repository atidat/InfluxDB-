#### Measurement_schema 基础类型

1. SchemaType：区分bucket支持的<u>schema模式</u>，分为implicit和explicit schema两种模式
2. SemanticColumnType：区分measurement的<u>列类型</u>，分为timestamp、tag和field三种列类型
3. SchemaColumnDataType：区分<u>列数据类型</u>，分为Float、Integer、Unsigned、string和Boolean五种数据类型

上述3种类型均实现了以下4个函数：

- String() string

- Ptr ()\*SchemaType/\*SemanticColumnType/\*SchemaColumnDataType

- UnmarshalJson(d []byte)  error

- MarshalJson() ([]byte, error)

SchemaType还实现了Equals(other *SchemaType) bool

SchemaColumnDataType还实现了ToFieldType() models.FiedleType

4. MeasurementSchema：数据结构包括ID、OrigID、BucketID、Name、Columns和CRUDLog 6个基本数据

  对外函数Validate() error

  ```go
  import "go.uber.org/multierr"
  
  func (m *MeasurementSchema) Validate() (err error) {
      // multierr.Append 把两个err合并到一个err中
      // 对m.Name的长度、特殊字符、utf-8编码校验
  	err = multierr.Append(err, m.validateName("name", m.Name))
      // 对m.Columns各列进行time、tag和field类型校验
  	err = multierr.Append(err, m.validateColumns())
  	return
  }
  ```

5. columns：数据结构包括indices和columns，其中indices内容为数字，columns为列的信息；columns实现了sort.Sort的三个接口，故类型可排序

6. MeasurementSchemaColumn：数据结构包括Name、Type和DataType，其中Type为列类型，分为time、tag和field三类，DataType为数据类型，分为Float、Integer、Unsigned、string和Boolean五种数据类型。



#### Measurement_schema API

- SchemaTypeFromString(s string) *SchemaType

- SemanticColumnTypeFromString(s string) *SemanticColumnType
- SchemaColumnDataTypeFromString(s string) *SchemaColumnDataType
- ValidateMeasurementSchemaName(name string) error

```go
// 返回Schema类型（本质为name的string类型转SchemaType类型）
func SchemaTypeFromString(s string) *SchemaType {
	switch s {
	case "implicit": return SchemaTypeImplicit.Ptr()
	case "explicit": return SchemaTypeExplicit.Ptr()
	default: return nil
	}
}
```

```go
// 返回ColumnType的三种基础类型（timestamp、tag和field）中的一种
func SemanticColumnTypeFromString(s string) *SemanticColumnType {
	switch s {
	case "timestamp": return SemanticColumnTypeTimestamp.Ptr()
	case "tag": return SemanticColumnTypeTag.Ptr()
	case "field": return SemanticColumnTypeField.Ptr()
	default: return nil
	}
}
```

```go
// 返回ColumnDataType的五种基础类型中的一种
func SchemaColumnDataTypeFromString(s string) *SchemaColumnDataType {
	switch s {
	case "float": return SchemaColumnDataTypeFloat.Ptr()
	case "integer": return SchemaColumnDataTypeInteger.Ptr()
	case "unsigned": return SchemaColumnDataTypeUnsigned.Ptr()
	case "string": return SchemaColumnDataTypeString.Ptr()
	case "boolean": return SchemaColumnDataTypeBoolean.Ptr()
	default: return nil
	}
}
```

```go
func ValidateMeasurementSchemaName(name string) error {
    // name长度校验
	if len(name) == 0 {	return ErrMeasurementSchemaNameTooShort }
	if len(name) > 128 { return ErrMeasurementSchemaNameTooLong }

    // name特殊字符校验1
	if err := models.CheckToken([]byte(name)); err != nil {
		return &influxerror.Error{
			Code: influxerror.EInvalid,
			Err:  err}
	}
    // name特殊字符校验2
	if strings.HasPrefix(name, "_") { return ErrMeasurementSchemaNameUnderscore	}
	if strings.Contains(name, `"`) || strings.Contains(name, `'`) {
		return ErrMeasurementSchemaNameQuotes
	}
	return nil
}

func CheckToken(a []byte) error {
    // utf-8编码校验
	if !utf8.Valid(a) {	return errInvalidUTF8 }
	for _, r := range string(a) {
        // 打印、换行符校验
		if !unicode.IsPrint(r) { return errNonPrintable	}
		if r == unicode.ReplacementChar { return errReplacementChar	}
	}
	return nil
}
```

