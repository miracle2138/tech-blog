
我们通过反射创建实例时，需要传入type，如果type是指针类型，需要调用Elem()方法拿到原始类型。
例子：
```go
func Test30(t *testing.T) {
	p := &Person{}
	typeOf := reflect.TypeOf(p)
	fmt.Printf("kind: %s, type: %s \n", typeOf.Kind(), typeOf.Name()) // ptr 空串

	obj := reflect.New(typeOf.Elem()).Interface().(*Person)
	fmt.Printf("%+v\n", obj)

	//typeOf1 := reflect.TypeOf(*p)
	//typeOf1.Elem() // panic here
}
```
Elem()方法：
```go
	// Elem returns a type's element type.
	// It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
	Elem() Type
```
可以看到，对于指针、数组等类型，需要通过Elem方法获取元素类型。使用时可以先通过Kind()方法判断需不需要Elem()。
