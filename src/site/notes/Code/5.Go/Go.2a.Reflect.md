---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.2a.Reflect/","title":"Reflect","noteIcon":""}
---


# Reflect

> [Go 语言反射的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect/)
> [深入理解 go reflect - 反射基本原理 - 掘金 (juejin.cn)](https://juejin.cn/post/7183132625580605498#heading-7)

反射使程序能够在运行时检查变量和值，求出它们的类型

在 Go 语言中，reflect 实现了运行时反射，reflect 包会帮助识别 interface{} 变量的 [[Code/5.Go/Go.0a.Go数据类型#interface 实现\|底层具体类型和具体值]]

## 三大法则

### reflect 将 interface{} 变量转换成反射对象

由于 `reflect.TypeOf`、`reflect.ValueOf ` 两个方法的入参都是 interface{} 类型，所以在方法执行的过程中会发生类型转换，一切其他类型变量首先被转换为 interface{}类型

当一个变量被转换成 teflect 对象时，Go 语言会在编译期间完成类型转换，将变量的类型和值转换至 interface{}，并等待运行期间使用 reflect 包获取接口中存储的信息

### reflect 可以从反射对象获取 interface{} 变量

reflect 中的 `reflect.Value.Interface` 能将反射对象还原成 interface 类型的变量，但只能获取 interface{} 类型的变量，如果想要将其还原成最原始的状态还需要经过类型断言
```go
v := reflect.ValueOf(1)
v.Interface().(int)
```

### reflect 需要通过指针修改原始变量

由于 Go 语言的函数调用都是值传递的，所以只能用迂回的方式改变原变量：先获取变量指针对应的 `reflect.Value`，再通过 `reflect.Value.Elem` 方法得到可以被设置的变量
可以通过 `reflect.Value.CanSet` 来判断一个反射对象的值是否是可更改的

```go
i := 1
v := reflect.ValueOf(&i)
v.Elem().SetInt(10)

fmt.Println(reflect.Valueof(i).CanSet()) // false
fmt.Println(reflect.Valueof(v).CanSet()) // false
fmt.Println(reflect.Valueof(v).Elem().CanSet()) // true
```

## 类型

`interface{}` 类型在 Go 语言内部通过 `reflect.emptyInterface` 结构体表示
其中 `rtype` 字段用于表示变量的类型， `word` 字段指向内部封装的数据
```go
type emptyInterface struct {
	typ  *rtype // 变量的类型
	word unsafe.Pointer // 内部封装的数据
}
```

`reflect.Type` 是反射包定义的一个接口，表示 interface{} 的类型信息
```go
type Type interface {
  Align() int
  FieldAlign() int
  Method(int) Method // 获取当前类型对应方法的引用
  MethodByName(string) (Method, bool)
  NumMethod() int
  ...
  Implements(u Type) bool // 判断当前类型是否实现了某个接口
  ...
}
```

`reflect.Value` 是一个结构体，表示 interface{}的具体值
```go
type Value struct {
        // 包含过滤的或者未导出的字段
}

func (v Value) Addr() Value
func (v Value) Bool() bool
func (v Value) Bytes() []byte
...
```

`reflect.Kind` 表示该类型的特定类别(e.g. struct)

## 函数

`reflect.TypeOf()` 函数将传入的变量隐式转换成 `reflect.emptyInterface` 类型并获取其中存储的类型信息 `reflect.rtype`
```go
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}

func toType(t *rtype) Type {
	if t == nil {
		return nil
	}
	return t
}
```
`reflect.rtype ` 是一个实现了 `reflect.Type` 接口的结构体，该结构体实现的 `reflect.rtype.String` 方法可以获取当前类型的名称

`reflect.ValueOf()` 函数中先调用了 `reflect.escapes` 保证当前值逃逸到堆上，然后通过 `reflect.unpackEface` 从接口中获取数据的运行时表示 `reflect.Value`
```go
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	escapes(i)

	return unpackEface(i)
}

func unpackEface(i interface{}) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if ifaceIndir(t) {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```

`NumField()` 方法返回结构体中字段的数量
`Field(i int)` 方法返回第 i 个字段的 `reflect.Value`

`reflect.Indirect()` 可以将指针 Value 转换为指针所指向对象的 Value，当传入值不为指针时不进行转换直接返回

`Int` 和 `String` 可以取出 `reflect.Value` 作为 `int64` 和 `string`

`CanAddr` 判断反射对象是否可以寻址，返回 `true` 时可以通过 `Addr` 方法获取反射对象的地址
`CanSet` 判断反射对象是否可以修改

### Kind

`Kind` 方法用于判断变量的基本类型

go 的全部基本类型如下
```go
type Kind uint

const (
   Invalid Kind = iota
   Bool
   Int
   Int8
   Int16
   Int32
   Int64
   Uint
   Uint8
   Uint16
   Uint32
   Uint64
   Uintptr
   Float32
   Float64
   Complex64
   Complex128
   Array
   Chan
   Func
   Interface
   Map
   Pointer
   Slice
   String
   Struct
   UnsafePointer
)
```

### Elem

`reflect.Value` 的 `Elem` 方法的作用是**获取指针指向的值，或者获取接口的动态值**，其他类型值触发 panic

`reflect.Type` 的 `Elem` 方法的作用是**获取数组、chan、map、指针、切片的关联元素的类型信息**，其他类型值触发 panic
```go
t1 := reflect.TypeOf([3]int{1, 2, 3}) // 数组 [3]int
fmt.Println(t1.String()) // [3]int
fmt.Println(t1.Elem().String()) // int
```

> 使用 `Key` 方法获取 map 类型 key 的类型信息

> 在定义结构体时，可以为每个字段指定一个[[Code/5.Go/Go.0a.Go数据类型#标签\|标签]]，可以使用 `Elem()` 读取这些标签

## 语法

> [一篇带你全面掌握go reflect 反射的用法 - timelesszhuang - 博客园 (cnblogs.com)](https://www.cnblogs.com/timelesszhuang/p/go-reflect.html)

```go
var i int = 10
var s string = "hello"
user := User{
	Id:     7,
	Name:   "杰克逊",
	Weight: 65.5,
	Height: 1.68,
}
```

### reflect.Type

#### 获取 Type

```go
typeI := reflect.TypeOf(1)       
typeS := reflect.TypeOf("hello") 
fmt.Println(typeI) // int
fmt.Println(typeS) // string

typeUser := reflect.TypeOf(&User{}) 
fmt.Println(typeUser)                  // *User
fmt.Println(typeUser.Kind())           // ptr
fmt.Println(typeUser.Elem().Kind())    // struct
```

#### 指针 Type 转为非指针

```go
typeUserPtr := reflect.TypeOf(&User{}) 
typeUser := reflect.TypeOf(User{})
assert.IsEqual(typeUser.Elem(), typeUserPtr)
```

### reflect.Value

#### 获取 Value

```go
iValue := reflect.ValueOf(1)
sValue := reflect.ValueOf("hello")
userPtrValue := reflect.ValueOf(&user)
fmt.Println(iValue)       //1
fmt.Println(sValue)       //hello
fmt.Println(usrPtrValue) //&{7 杰克逊  65 1.68}
```

#### Value 转为 Type

```go
iType := iValue.Type()
sType := sValue.Type()
userType := usrPtrValue.Type()
//在Type和相应Value上调用Kind()结果一样的
fmt.Println(iType.Kind() == reflect.Int, iValue.Kind() == reflect.Int, iType.Kind() == iValue.Kind())  
fmt.Println(sType.Kind() == reflect.String, sValue.Kind() == reflect.String, sType.Kind() == sValue.Kind()) 
fmt.Println(usrType.Kind() == reflect.Ptr, usrPtrValue.Kind() == reflect.Ptr, usrType.Kind() == usrPtrValue.Kind())
```

#### 指针/非指针 Value 互相转换

```go
usrValue1 := usrPtrValue1.Elem()           // 指针转为非指针
usrValue2 := reflect.Indirect(usrPtrValue1) // 指针转为非指针
usrPtrValue2 := usrValue1.Addr()            // 非指针转为指针

fmt.Println(usrValue1.Kind(), usrValue2.Kind())       // struct struct
fmt.Println(usrPtrValue1.Kind(), usrPtrValue2.Kind()) //ptr ptr
```

#### 获取原始数据

有两种常用方法获取反射对象的原始数据
- 通过 Interface()函数把 Value 转为 interface{}，再从 interface{}进行类型断言强制类型转换为原始数据类型
- 在 Value 上直接调用 Int()、String()等一步到位
```go
fmt.Printf("origin value iValue is %d %d\n", iValue.Interface().(int), iValue.Int())
fmt.Printf("origin value sValue is %s %s\n", sValue.Interface().(string), sValue.String())
usr := usrValue.Interface().(User)
fmt.Printf("id=%d name=%s weight=%.2f height=%.2f\n", usr.Id, usr.Name, usr.Weight, usr.Height)
usr2 := usrPtrValue.Interface().(*User)
fmt.Printf("id=%d name=%s weight=%.2f height=%.2f\n", usr2.Id, usr2.Name, usr2.Weight, usr2.Height)

```

#### 修改原始数据

```go
valueI := reflect.ValueOf(&i) //由于go语言所有函数传的都是值，所以要想修改原来的值就需要传指针
valueS := reflect.ValueOf(&s)
valueUser := reflect.ValueOf(&user)
valueI.Elem().SetInt(8) //由于valueI对应的原始对象是指针，通过Elem()返回指针指向的对象
valueS.Elem().SetString("golang")
valueUser.Elem().FieldByName("Weight").SetFloat(68.0) //FieldByName()通过Name返回类的成员变量
```
强调一下，**要想修改原始数据的值，给 ValueOf 传的必须是指针**，而指针 Value 不能调用 Set 和 FieldByName 方法，所以得先通过 Elem()转为非指针 Value

未导出成员的值不能通过反射进行修改

```go
addrValue := valueUser.Elem().FieldByName("addr")
if addrValue.CanSet() {
	addrValue.SetString("北京")
} else {
	fmt.Println("addr是未导出成员，不可Set") //以小写字母开头的成员相当于是私有成员
}
```

##### 修改 Slice

```go
users := make([]*User, 1, 5) //len=1，cap=5
users[0] = &User{
	Id:     7,
	Name:   "杰克逊",
	Weight: 65.5,
	Height: 1.68,
}


sliceValue := reflect.ValueOf(&users) //准备通过Value修改users，所以传users的地址
if sliceValue.Elem().Len() > 0 {      //取得slice的长度
	sliceValue.Elem().Index(0).Elem().FieldByName("Name").SetString("令狐一刀")
	fmt.Printf("1st user name change to %s\n", users[0].Name)
}
```
可以修改 slice 的 cap，但新的 cap 必须位于原始的 len 到 cap 之间，即只能把 cap 改小
```go
sliceValue.Elem().SetCap(3)
```
通过把 len 改大，可以实现向 slice 中追加元素的功能
```go
sliceValue.Elem().SetLen(2)
//调用reflect.Value的Set()函数修改其底层指向的原始数据
sliceValue.Elem().Index(1).Set(reflect.ValueOf(&User{
	Id:     8,
	Name:   "李达",
	Weight: 80,
	Height: 180,
}))
fmt.Printf("2nd user name %s\n", users[1].Name)
```

##### 修改 map

`Value.SetMapIndex()`：往 map 里添加一个 key-value 对
`Value.MapIndex()`： 根据 Key 取出对应的 map
```go
u1 := &User{
	Id:     7,
	Name:   "杰克逊",
	Weight: 65.5,
	Height: 1.68,
}
u2 := &User{
	Id:     8,
	Name:   "杰克逊",
	Weight: 65.5,
	Height: 1.68,
}
userMap := make(map[int]*User, 5)
userMap[u1.Id] = u1

mapValue := reflect.ValueOf(&userMap)                                                         //准备通过Value修改userMap，所以传userMap的地址
mapValue.Elem().SetMapIndex(reflect.ValueOf(u2.Id), reflect.ValueOf(u2))                      //SetMapIndex 往map里添加一个key-value对
mapValue.Elem().MapIndex(reflect.ValueOf(u1.Id)).Elem().FieldByName("Name").SetString("令狐一刀") //MapIndex 根据Key取出对应的map
for k, user := range userMap {
	fmt.Printf("key %d name %s\n", k, user.Name)
}
```

#### 判断空 Value

```go
var i interface{} //接口没有指向具体的值
v := reflect.ValueOf(i)
fmt.Printf("v持有值 %t, type of v is Invalid %t\n", v.IsValid(), v.Kind() == reflect.Invalid)

var usr *User = nil
v = reflect.ValueOf(usr) //Value指向一个nil
if v.IsValid() {
	fmt.Printf("v持有的值是nil %t\n", v.IsNil()) //调用IsNil()前先确保IsValid()，否则会panic
}

var u User //只声明，里面的值都是0值
v = reflect.ValueOf(u)
if v.IsValid() {
	fmt.Printf("v持有的值是对应类型的0值 %t\n", v.IsZero()) //调用IsZero()前先确保IsValid()，否则会panic
}
```

#### 调用函数

```go
valueFunc := reflect.ValueOf(Add)     // 函数也是一种数据类型
typeFunc := reflect.TypeOf(Add)
argNum := typeFunc.NumIn()            // 函数输入参数的个数
args := make([]reflect.Value, argNum) // 准备函数的输入参数
for i := 0; i < argNum; i++ {
	if typeFunc.In(i).Kind() == reflect.Int {
		args[i] = reflect.ValueOf(3) //给每一个参数都赋3
	}
}
sumValue := valueFunc.Call(args) //返回[]reflect.Value，因为go语言的函数返回可能是一个列表
if typeFunc.Out(0).Kind() == reflect.Int {
	sum := sumValue[0].Interface().(int) //从Value转为原始数据类型
	fmt.Printf("sum=%d\n", sum)
}
```

#### 调用成员方法

```go
valueUser := reflect.ValueOf(&user)              //必须传指针，因为BMI()在定义的时候它是指针的方法
bmiMethod := valueUser.MethodByName("BMI")       //MethodByName()通过Name返回类的成员变量
resultValue := bmiMethod.Call([]reflect.Value{}) //无参数时传一个空的切片
result := resultValue[0].Interface().(float32)
fmt.Printf("bmi=%.2f\n", result)

//Think()在定义的时候用的不是指针，valueUser可以用指针也可以不用指针
thinkMethod := valueUser.MethodByName("Think")
thinkMethod.Call([]reflect.Value{})

valueUser2 := reflect.ValueOf(user)
thinkMethod = valueUser2.MethodByName("Think")
thinkMethod.Call([]reflect.Value{})
```

### 创建对象

#### 获取 struct 成员变量的信息

```go
typeUser := reflect.TypeOf(User{}) // 需要用struct的Type，不能用指针的Type
fieldNum := typeUser.NumField()    // 成员变量的个数
for i := 0; i < fieldNum; i++ {
	field := typeUser.Field(i)       // 返回第 i 个字段的 Value
	fmt.Printf("%d %s offset %d anonymous %t type %s exported %t json tag %s\n", i,
		field.Name,            //变量名称
		field.Offset,          //相对于结构体首地址的内存偏移量，string类型会占据16个字节
		field.Anonymous,       //是否为匿名成员
		field.Type,            //数据类型，reflect.Type类型
		field.IsExported(),    //包外是否可见(即是否以大写字母开头)
		field.Tag.Get("json")) //获取成员变量后面``里面定义的tag
}

//可以通过FieldByName获取Field
if nameField, ok := typeUser.FieldByName("Name"); ok {
	fmt.Printf("Name is exported %t\n", nameField.IsExported())
}

//也可以根据FieldByIndex获取Field
thirdField := typeUser.FieldByIndex([]int{2}) //参数是个slice，因为有struct嵌套的情况
fmt.Printf("third field name %s\n", thirdField.Name)
```

获取 struct 成员方法的信息
```go
typeUser := reflect.TypeOf(User{})
methodNum := typeUser.NumMethod() //成员方法的个数。接收者为指针的方法【不】包含在内
for i := 0; i < methodNum; i++ {
	method := typeUser.Method(i)
	fmt.Printf("method name:%s ,type:%s, exported:%t\n", method.Name, method.Type, method.IsExported())
}
fmt.Println()

typeUser2 := reflect.TypeOf(&User{})
methodNum = typeUser2.NumMethod() //成员方法的个数。接收者为指针或值的方法【都】包含在内，也就是说值实现的方法指针也实现了(反之不成立)
for i := 0; i < methodNum; i++ {
	method := typeUser2.Method(i)
	fmt.Printf("method name:%s ,type:%s, exported:%t\n", method.Name, method.Type, method.IsExported())
}
```

判断类型是否实现了某接口
```go
//通过reflect.TypeOf((*<interface>)(nil)).Elem()获得接口类型。因为People是个接口不能创建实例，所以把nil强制转为*common.People类型
typeOfPeople := reflect.TypeOf((*common.People)(nil)).Elem()
fmt.Printf("typeOfPeople kind is interface %t\n", typeOfPeople.Kind() == reflect.Interface)
t1 := reflect.TypeOf(User{})
t2 := reflect.TypeOf(&User{})
//如果值类型实现了接口，则指针类型也实现了接口；反之不成立
fmt.Printf("t1 implements People interface %t\n", t1.Implements(typeOfPeople))
```

创建 struct
```go
t := reflect.TypeOf(User{})
value := reflect.New(t) //根据reflect.Type创建一个对象，得到该对象的指针，再根据指针提到reflect.Value
value.Elem().FieldByName("Id").SetInt(10)
user := value.Interface().(*User) //把反射类型转成go原始数据类型Call([]reflect.Value{})
```

创建 slice
```go
var slice []User
sliceType := reflect.TypeOf(slice)
sliceValue := reflect.MakeSlice(sliceType, 1, 3)
sliceValue.Index(0).Set(reflect.ValueOf(User{
	Id:     8,
	Name:   "李达",
	Weight: 80,
	Height: 180,
}))
users := sliceValue.Interface().([]User)
fmt.Printf("1st user name %s\n", users[0].Name)
```

创建 map
```go
var userMap map[int]*User
mapType := reflect.TypeOf(userMap)
// mapValue:=reflect.MakeMap(mapType)
mapValue := reflect.MakeMapWithSize(mapType, 10)

user := &User{
	Id:     7,
	Name:   "杰克逊",
	Weight: 65.5,
	Height: 1.68,
}
key := reflect.ValueOf(user.Id)
mapValue.SetMapIndex(key, reflect.ValueOf(user))                    //SetMapIndex 往map里添加一个key-value对
mapValue.MapIndex(key).Elem().FieldByName("Name").SetString("令狐一刀") //MapIndex 根据Key取出对应的map
userMap = mapValue.Interface().(map[int]*User)
fmt.Printf("user name %s %s\n", userMap[7].Name, user.Name)
```
reflect 包里除了 MakeSlice()和 MakeMap()，还有 MakeChan()和 MakeFunc()
