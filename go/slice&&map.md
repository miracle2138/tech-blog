## slice
- 创建一个slice有两三方式，一种是字面常量，另一种是make函数，还有一种是基于其他的slice或者数组
- slice有三个变量，一个是底层数组的指针，一个len，一个cap。slice的概念可以类比于java的arraylist，不同的点在于，arraylist没有
   把cap这个变量暴露给使用者，个人感觉slice其实也没有必要。访问一个slice，只能访问len大小的数据，否则会越界
- 相比于数组，slice的优势是可以动态扩容。可以调用append函数给slice增加元素。语法上append会返回一个slice，可能和之前的slice一样
   可能和之前的不一样，取决于被扩容的slice的cap是否够。append可以扩容多个元素，也可扩容另一个slice，但是需要加...
- 可以使用`a[i,j]`的方式基于其他的slice或者数组构建slice。i和j代表开始和结束索引，前闭后开。那新的slice的len和cap怎么计算？
   ```
    lenNew = j - i  
    capNew = cap - i       
   ```
   len好理解，cap则取决于起始位置，起始位置之后的元素都算新slice的cap
   **需要注意的是，新旧slice的底层数组是共享的**，这意味着修改其中一个slice的元素，可能会影响另外一个slice。当然如果其中一个slice由于
   append进行了扩容，那么就持有了一个新的底层数组，两个slice就没有关联了。如果新的slice的cap还够，append之后就会影响旧的slice
- 话说回来，基于slice构建slice，总是需要担心新老slice共享数组的问题。那么有一个好办法就是使用三个参数的构建方式`a[i,j,k]`
    i和j含义不变，k指定新的slice的cap为下标，当然k不能比原有slice的cap大
   ```
    lenNew = j - i
    capNew = k - i
   ```
    为了避免共享，我们可以设置k = j，也即cap = len，那么新append元素时会触发扩容
- go语言是值拷贝，所以slice的赋值，也会导致新旧slice共享底层数组

## if 条件优先级
- &&与||是支持短路的，并且如果同时出现且没有括号，那么&&优先级比||高，但是一般建议加（），更好理解

## map
- 创建一个map有两种方式，一种是字面常量，另一种是make函数
- 获取值时，go语言里的map肯定会返回一个值，如果不存在会返回一个零值。所以最好的办法是，获取多返回值，荣国第二个bool类型判存
