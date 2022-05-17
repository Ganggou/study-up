## defer
* 函数返回的过程
  * 给返回值赋值
  * 调用defer表达式
  * 返回到调用函数中
* defer执行顺序满足先进先出特性
* defer将语句放入到栈中时，也会将相关的值拷贝同时入栈
下例中defer不受num++的影响
```
func P(num int) int {
    defer fmt.Println("num:", num)
    
    num++
    
    return num
}
```
闭包是通过指针传递的,所以对num的修改相当于修改指针指向的内容
```
func P(num int) int {
    defer func() {
      fmt.Println("num1:", num)
      num++
    }()
    
    defer {
      fmt.Println("num2:", num)
      num++
    }()
    
    num++
    
    return num
}
// P(0)
// num2: 1
// num1: 2
```
当defer类似闭包使用时候，访问的总是循环中最后一个值
```
func main() {
    for i:=0; i<5; i++ {
        defer func() {
           fmt.Println(i) 
        }()
    }
}
// 连续输出5个5
```
