# go反射

## 什么是反射？使用反射可以实现什么功能？

反射提供了一种可以操作任意类型数据结构的能力。通过反射你可以实现对任意类型的深度复制，可以对任意类型执行自定义的操作。另外由于golang可以给结构体定义tag熟悉，结合反射，于是就可以实现结构体与数据库表、结构体与json对象之间的相互转换。

## 使用反射需要注意什么事情？

使用反射时需要先确定要操作的值是否是期望的类型，是否是可以进行“赋值”操作的，否则reflect包将会毫不留情的产生一个panic。

# struct initializer示例

[go-struct-initializer](https://github.com/bulletRush/go-struct-initializer) 是我在学习golang反射过程中实现的一个可以根据struct tag初始化结构体的go library，这里对其中用到的反射技术做一下说明

```go
package structinitializer

import (
	"reflect"
	"fmt"
	"strconv"
	"strings"
)

const (
	ERROR_NOT_A_POINTER = "NotAPointer"
	ERROR_NOT_IMPLEMENT = "NotImplement"
	ERROR_POINTE_TO_NON_STRUCT = "PointToNonStruct"
	ERROR_TYPE_CONVERSION_FAILED = "TypeConversionFailed"
	ERROR_MULTI_ERRORS_FOUND = "MultiErrorsFound"
)

type Error struct {
	Reason  string
	Message string
	Stack   string
}

func NewError(reason, message string, stack string) error {
	return &Error{reason, message, stack}
}

func mergedError(errors []error, stackName string) error {
	if len(errors) == 0 {
		return nil
	}
	if len(errors) == 1 {
		return errors[0]
	}
	points := make([]string, len(errors))
	for i, err := range errors {
		points[i] = fmt.Sprintf("* %s", err)
	}
	return &Error{
		ERROR_MULTI_ERRORS_FOUND,
		strings.Join(points, "\n"),
		stackName,
	}
}

func (e *Error) Error() string {
	if e.Stack == "" {
		return fmt.Sprintf("%s: %s", e.Reason, e.Message)
	}
	return fmt.Sprintf("%s:(%s) %s", e.Reason, e.Stack, e.Message)
}

type InitialiserConfig struct {
	TagName string
}

type Initialiser struct {
	config *InitialiserConfig
}

func NewStructInitialiser() Initialiser {
	config := InitialiserConfig{
		TagName: "default",
	}
	initialiser := Initialiser{
		config: &config,
	}
	return initialiser
}

func (self *Initialiser) initialiseInt(stackName, tag string, val reflect.Value) error {
	if tag != "" {
		i, err := strconv.ParseInt(tag, 10, 64)
		if err != nil {
			return NewError(
				ERROR_TYPE_CONVERSION_FAILED,
				fmt.Sprintf(`"%s" can't convert to int64`, tag),
				stackName,
			)
		}
		if val.Int() == 0 {
			val.SetInt(int64(i))
		}
	}
	return nil
}

func (self *Initialiser) initialiseUint(stackName, tag string, val reflect.Value) error {
	if tag != "" {
		i, err := strconv.ParseUint(tag, 10, 64)
		if err != nil {
			return NewError(
				ERROR_TYPE_CONVERSION_FAILED,
				fmt.Sprintf(`"%s" can't convert to uint64`, tag),
				stackName,
			)
		}
		if val.Uint() == 0 {
			val.SetUint(uint64(i))  // 设置value
		}
	}
	return nil
}

func (self *Initialiser) initialiseString(stackName, tag string, val reflect.Value) error {
	if tag != "" && val.String() == ""{
		val.SetString(tag)
	}
	return nil
}

// 检查某个结构体是否存在某个域设置了default标签
func (self *Initialiser) checkStructHasDefaultValue(typ reflect.Type) bool {
	for i:=0; i<typ.NumField(); i++ {
		field := typ.Field(i)
		if field.Tag.Get(self.config.TagName) != "" {  // 获取struct field的指定tag
			return true
		}
		if field.Type.Kind() == reflect.Struct && self.checkStructHasDefaultValue(field.Type) {  // 
			return true
		}
	}
	return false
}

func (self *Initialiser) initialisePtr(stackName, tag string, val reflect.Value) error {
	if tag == "-" {
		return nil
	}
	typ := val.Type().Elem()
	if tag == "" {
		if typ.Kind() != reflect.Struct {
			return nil
		} else if !self.checkStructHasDefaultValue(typ) {
			return nil
		}
	}
        // 结构体存在至少一个域设置了default标签才有初始化的必要
	realVal := reflect.New(typ)  // 根据结构体类型创建一个新的结构体
        // Indirect可以获取到value对应的真实value（指针指向的value）
	if err := self.initialise(stackName, tag, reflect.Indirect(realVal)); err != nil {
		return err
	}
	val.Set(realVal)
	return nil
}

func (self *Initialiser) initialise(stackName string, tag string, val reflect.Value) error {
	typ := val.Type()
	kind := typ.Kind()
	if kind == reflect.Struct {
		return self.initialiseStruct(stackName, val)
	}
	switch kind {
	case reflect.Int:
		fallthrough
	case reflect.Int8:
		fallthrough
	case reflect.Int16:
		fallthrough
	case reflect.Int32:
		fallthrough
	case reflect.Int64:
		return self.initialiseInt(stackName, tag, val)
	case reflect.Uint:
		fallthrough
	case reflect.Uint8:
		fallthrough
	case reflect.Uint16:
		fallthrough
	case reflect.Uint32:
		fallthrough
	case reflect.Uint64:
		fallthrough
	case reflect.Uintptr:
		return self.initialiseUint(stackName, tag, val)
	case reflect.String:
		return self.initialiseString(stackName, tag, val)
	case reflect.Ptr:
		return self.initialisePtr(stackName, tag, val)
	default:
		return NewError(ERROR_NOT_IMPLEMENT, "not implement", stackName)
	}
	return NewError(ERROR_NOT_IMPLEMENT, "not implement", stackName)
}

func (self *Initialiser) initialiseStruct(stackName string, structVal reflect.Value) error {
	structs := make([]reflect.Value, 1, 5)
	structs[0] = structVal
	errors := make([]error, 0)
	for len(structs) > 0 {
		structVal := structs[0]
		structs = structs[1:]
		structTyp := structVal.Type() // 结构体类型信息
		for i:=0; i<structTyp.NumField(); i++ { // 循环结构体的所有域 
			structField := structTyp.Field(i) // 获取结构体域信息
			tag := structField.Tag // 获取结构体域的标签
			defaultTag := tag.Get(self.config.TagName);
			if structField.Anonymous {  // 这个域是一个结构体组合
				structs = append(structs, structVal.Field(i))
				continue
			}
			fieldName := structField.Name // 获取结构体域的名字
			if stackName != "" {
				fieldName = fmt.Sprintf("%s.%s", stackName, fieldName)
			}
			err := self.initialise(
				fieldName,
				defaultTag,
				structVal.Field(i), // 获取结构体域对应的值信息
			)
			if err != nil {
				errors = append(errors, err)
			}
		}
	}
	return mergedError(errors, stackName)
}

func (self *Initialiser) Initialise(inf interface{}) error {
	typ := reflect.TypeOf(inf)
	kind := typ.Kind() // kind与type的区别在于：kind无法区分基本类型的别名
	if kind != reflect.Ptr { // 只有指针才能执行初始化操作
		return NewError(
			ERROR_NOT_A_POINTER,
			fmt.Sprintf("%s not a pointer", typ),
			"",
		)
	}
	val := reflect.ValueOf(inf).Elem() // 获取指针对应的值信息
	kind = val.Kind()
	if kind != reflect.Struct {
		return NewError(
			ERROR_POINTE_TO_NON_STRUCT,
			fmt.Sprintf("%s point to non struct", typ),
			"",
		)
	}
	return self.initialiseStruct(val.Type().Name(), val)
}

func InitializeStruct(struct_ interface{}) error {
	initialiser := NewStructInitialiser()
	return initialiser.Initialise(struct_)
}
```

```go
package structinitializer

import (
	"testing"
	"fmt"
	"reflect"
)

type Validator interface {
	Valid() bool
}

type TNormStruct struct  {
	AInt int
	AStr string
}

type TStruct struct {
	AInt int `default:"10"`
	AStr string `default:"hello"`
}

func (self *TStruct) Valid() bool {
	return self.AStr == "hello" && self.AInt == 10
}

type TWrongStruct struct {
	AInt int `default:"hhhhh"`
	AStr string `default:"hello"`
}

func (self *TWrongStruct) Valid() bool {
	return false
}

type TEmbeddStruct struct  {
	AStruct TStruct
	AInt int `default:"101"`
}

func (self *TEmbeddStruct) Valid() bool {
	return self.AStruct.Valid() && self.AInt == 101
}

type TEmbeddStruct2 struct  {
	AStruct TStruct
	AInt int
}

type TAnonymousStruct struct  {
	TStruct
	AUint uint16 `default:"20"`
}

func (self *TAnonymousStruct) Valid() bool  {
	return self.TStruct.Valid() && self.AUint == 20
}

type TPointerStruct struct {
	AStructPtr *TStruct
	AIntPtr *int32 `default:"20"`
	AStrPtr *string `default:"-"`
	AUintPtr *uint
	ANormStruct *TNormStruct
}

func (self *TPointerStruct) Valid() bool {
	if self.AStructPtr != nil && self.AStructPtr.Valid() && self.AIntPtr != nil && *self.AIntPtr == 20 && self.AStrPtr == nil && self.AUintPtr == nil && self.ANormStruct == nil {
		return true
	}
	return false
}

func AssertTrue(t *testing.T, a bool, msg string) {
	if !a {
		t.Errorf(msg)
	}
}

func TestCheckStructHasDefaultValue(t *testing.T) {
	self := NewStructInitialiser()
	AssertTrue(t, self.checkStructHasDefaultValue(reflect.TypeOf(TStruct{})), "TStruct check failed!")
	AssertTrue(t, !self.checkStructHasDefaultValue(reflect.TypeOf(TNormStruct{})), "TNormStrcut check succeed!")
	AssertTrue(t, self.checkStructHasDefaultValue(reflect.TypeOf(TEmbeddStruct2{})), "TEmbeddStruct2 check failed!")
}

func checkError(t *testing.T, err error, wantReason string) {
	if err == nil {
		t.Errorf("except non pointer err!")
		return
	} else {
		if err1 := err.(*Error); err1.Reason != wantReason {
			t.Errorf("want '%s' but '%s': %s", wantReason, err1.Reason, err1.Message)
			return
		}
	}
	fmt.Printf("%#v\n", err)
}

func TestNonPointer(t *testing.T) {
	aStruct := TStruct{}
	err := InitializeStruct(aStruct)
	checkError(t, err, ERROR_NOT_A_POINTER)
}

func TestPointToNonStruct(t *testing.T) {
	aInt := 100
	err := InitializeStruct(&aInt)
	checkError(t, err, ERROR_POINTE_TO_NON_STRUCT)
}

func TestWrongTag(t *testing.T) {
	aStruct := TWrongStruct{}
	err := InitializeStruct(&aStruct)
	checkError(t, err, ERROR_TYPE_CONVERSION_FAILED)
}

func TestDefaultSetted(t *testing.T) {
	aStruct := TStruct{
		AInt: 103,
	}
	err := InitializeStruct(&aStruct)
	if err != nil || aStruct.AInt != 103 {
		t.Errorf("BOOM!!! rewrite a user setted value? err: %s", err)
	}
	fmt.Printf("%#v\n", aStruct)
}

func checkStruct(t *testing.T, v Validator) {
	err := InitializeStruct(v)
	if err != nil {
		t.Errorf("unexcept err: %#v, aStruct: %#v\n", err, v)
		return
	}

	if !v.Valid() {
		t.Errorf("init incorret: %#v\n", v)
		return
	}
	fmt.Printf("%#v\n", v)
}

func TestEmbeddStruct(t *testing.T) {
	aStruct := TEmbeddStruct{}
	checkStruct(t, &aStruct)
}

func TestAnonymousStrct(t *testing.T) {
	aStruct := TAnonymousStruct{}
	checkStruct(t, &aStruct)
}

func TestNormalStruct(t *testing.T) {
	aStruct := TStruct{}
	checkStruct(t, &aStruct)
}

func TestPointerStruct(t *testing.T) {
	aStruct := TPointerStruct{}
	checkStruct(t, &aStruct)
}
```

# 参考
* [Package reflect](https://golang.org/pkg/reflect/)
* [The Laws of Reflection](https://blog.golang.org/laws-of-reflection)
