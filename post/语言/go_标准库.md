## reflect



### interface 与 反射

* go的变量由`(type, value)`两部分组成
* `type`包括`static type`（`int`、`float`、`string`等）和`concrete type`（`runtime`看见的类型，也叫`dynamic type`）
* 类型断言能否成功，取决于`concrete type`（一个` reader`变量如果它的`concrete type`也实现了`write`方法，它也可以被类型断言为`writer`）



**static/concrete type**:



如果有一个`interface`

```go
type I interface{ F() }
```

以及上述接口的两个实现

```go
type S struct{}
func (S) F(){}

type T struct{}
func (T) F(){}
```

那么

```go
var x I
x = S{}
```

`x`的`static type`是`I`，`dynamic type`是`S`









反射主要与`interface`类型相关（它的type是`concrete type`），**只有`interface`类型才有反射一说**。



每个`interface`变量都可以看作一个`pair`

```go
(value, type)
```

`value`是实际变量的值，`type`是实际变量的类型，一个`interface{}`类型的变量包含了2个指针，一个指针指向值的类型（对应`concrete type`），另外一个指针指向实际的值（对应`value`）



这个pair在接口变量的连续赋值过程中是**不变**的:

```go
tty, _ := os.Openfile("/dev/tty", os.O_RDWR, 0)

var r io.Reader
r = tty // r 的 pair 是 (tty, *os.File)

var w io.Writer
w = r.(io.Writer) // w 的 pair 还是 (tty, *os.File)
```



**反射就是用来检测存储在接口变量内部 `pair`对的一种机制。**



### 反射

**反射就是用来检测存储在接口变量内部 `pair(value, type)`对的一种机制。**



所以反射包有基本的`reflect.TypeOf`、`reflect.ValueOf`

```go
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}

// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	escapes(i)

	return unpackEface(i)
}
```



```go
func main() {
	var num float64 = 3.14
	fmt.Println("type:", reflect.TypeOf(num))
	fmt.Println("value:", reflect.ValueOf(num))
}
/*
type: float64
value: 3.14
*/
```

也就是说明反射可以将“**接口类型变量**”转换为“**反射类型对象**”，反射类型指的是`reflect.Type`和`reflect.Value`



#### 强制转换

```go
func main() {
	var num float64 = 3.14

	pointer := reflect.ValueOf(&num)
	value := reflect.ValueOf(num)
	
    // 已知类型后转换为其对应的类型的做法如下，直接通过Interface方法然后强制转换
    // realValue := value.Interface().(已知的类型)
	convertPoint := pointer.Interface().(*float64)
	convertValue := value.Interface().(float64)

	fmt.Println(convertPoint)
	fmt.Print(convertValue)
}
```

也就是说反射可以将“**反射类型对象**”再重新转换为“**接口类型变量**”



#### 遍历字段/方法调用

很多情况下，我们可能并不知道其具体类型，需要我们进行遍历探测其Filed来得知

```go
type User struct {
	Id int			`json:"id",xml:"id"`
	Name string		`json:"name",xml:"name"`
	Age int			`json:"age",xml:"age"`
}

func (u User) PrintInfo() {
	fmt.Println(u.Id, u.Name, u.Age)
}


func main() {
	user := User{
		Id:   1,
		Name: "kirito",
		Age:  18,
	}

	DoFiledAndMethod(user)
}

func DoFiledAndMethod (input interface{}) {
	getType := reflect.TypeOf(input)
	fmt.Println("get Type is:", getType.Name())

	getValue := reflect.ValueOf(input)
	fmt.Println("get Value is", getValue)

	// 获取字段
	for i := 0; i < getType.NumField(); i++ {
		field := getType.Field(i)
		value := getValue.Field(i).Interface()
		// tag := field.Tag.Get("json")
		fmt.Printf("%s: %v = %v \t tag = %s\n", field.Name, field.Type, value, field.Tag)
	}

	// 获取方法
	for i := 0; i < getType.NumMethod(); i++ {
		m := getType.Method(i)
		fmt.Printf("%s: %v\n", m.Name, m.Type)
		
		m.Func.Call([]reflect.Value{getValue}) // 这里是知道只有这一个函数，所以写在这里了
	}
}
/*
get Type is: User
get Value is {1 kirito 18}
Id: int = 1 	 tag = json:"id",xml:"id"
Name: string = kirito 	 tag = json:"name",xml:"name"
Age: int = 18 	 tag = json:"age",xml:"age"
PrintInfo: func(main.User)
1 kirito 18
*/
```



#### 设置实际变量的值

`reflect.Value`是通过`reflect.ValueOf(X)`获得的，只有当`X`是指针的时候，才可以通过`reflec.Value`修改实际变量`X`的值

```go
func main() {
	var num float64 = 3.14
	fmt.Println("old value of num", num)

	pointer := reflect.ValueOf(&num)

	// 获取指向的 value
	newValue := pointer.Elem() // Elem returns the value that the interface v contains or that the pointer v points to.

	fmt.Println("type of newValue:", newValue.Type())
	fmt.Println("settability of newValue:", newValue.CanSet())
	fmt.Println("settability of pointer:", pointer.CanSet())

	// 重新赋值
	newValue.SetFloat(77)
	fmt.Println("new value of num:", num)

	// pointer = reflect.ValueOf(num) // 不是指针
	// newValue = pointer.Elem()  // panic
}
```

对于结构体，也是一样的

```go
type User struct {

	Id int
}

func main() {
	user := User{Id: 1}
	fmt.Println("old id", user.Id)
	p := reflect.ValueOf(&user)
	newV := p.Elem()
	fmt.Println("settability newV", newV.CanSet())
	field := newV.FieldByName("Id")
	fmt.Println("settability newV.Id", field.CanSet())
	field.SetInt(88)
	fmt.Println("new id", user.Id)
}

/*
old id 1
settability newV true
settability newV.Id true
new id 88
*/
```

