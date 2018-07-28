# AsyncLocal使用指南
相关词：异步共享数据，异步上下文

[toc]

# 起源

最近在业务中涉及到登陆，想把登陆后的账户数据放在IOC中，这样非Web层就可以直接获取数据。尽管这种用法不是太好，后面也修改为其它方式了，但发现一个新东西。。

## 如何发现

在以前我是不知道这个东西的。在设计中，业务层代码是单例，我们知道控制器、HttpContext是作用域（每次请求都会新建一个，相当于一次请求就新开一个区域）。

然后我的数据是也是作用域，就发生了在控制器这层能获取到数据，业务性获取的是新New的数据。（有一个中间件，会从IOC中获取数据引用，然后写入数据）。

就发现同样的用法，为什么HttContext可以跨区域调用，我自己的不行呢？查询HttContext[源代码](https://github.com/aspnet/HttpAbstractions/blob/07d115400e4f8c7a66ba239f230805f03a14ee3d/src/Microsoft.AspNetCore.Http/HttpContextAccessor.cs)，发现别人使用了AsyncLocal。至此发现了这两个东西。涉及框架多了肯定要把多线程\异步之间的东西搞清楚，就研究了下。

# 什么是AsyncLocal

在多线程/异步开发中，头疼的除了调用逻辑、线程安全，还有一个就是线程间共享数据的问题。在以前，需要通过执行上下文来共享数据，开发是比较麻烦的。

在.Net 4.6开始引入了[AsyncLocal< T >](https://msdn.microsoft.com/en-us/library/dn906268%28v=vs.110%29.aspx)、[ThreadLocal< T >](https://msdn.microsoft.com/en-us/library/dd642243(v=vs.110).aspx).使得在不同线程之间共享数据变得更加容易。

AsyncLocal用于异步方法之间的共享数据
ThreadLocal用于线程之间的共享数据，相当于单个线程中的staic数据。

## 使用

emmm，原谅我偷懒。。基础的使用看这个[老周的博客](https://www.cnblogs.com/tcjiaan/p/5007737.html)行了。 下面会讲一点心得。


## 陷阱-只读

AsyncLocal是只读的。

我们知道，**async/await**这种**TAP异步模式**中，当遇到**await**时，会离开当前的上下文（不一定是离开线程）。而**AsyncLocal**只能在同是一个上下文中赋值或修改，其它上下文修改值是没有效果的，会还原之前的值。

看一个例子：
``` csharp
   internal class Program
    {
        public static AsyncLocal<int> v;
        public static AsyncLocal<Student> stu = new AsyncLocal<Student>();
        private static void Main(string[] args)
        {
			var task = Task.Run(async () =>
            {
                v = new AsyncLocal<int>
                {
                    Value = 123
                };

                Console.WriteLine("In Task  " + v.Value);
            });
            task.Wait();
            Console.WriteLine(v.Value);
            
            Console.ReadLine();
        }
    }

```

猜想下结果是什么呢？
``` 
In Task  123
0
```

原因就是在里面的上下文中被赋值了，然后外面调用时，只会取到默认的0；微软这样设计我觉得是没问题的，因为一般这种需要跨线程的任务，比较担心的就是不知道什么时间、什么代码把数据修改了，导致追踪很困难。而这种只读的方式，可以保证数据**尽量干净**.




## 陷阱-引用类型的只读

但如果 **AsyncLocal &lt;T>**  中的T是只读的话，就有点特殊了。

先说**结论**：引用类型的引用本身还是只读，但属性却不是只读。

首先，定义一个测试用的实体类：
``` csharp
    public class Student
    {
        public string Name { get; set; }
        public int Age { get; set; }
        public List<StudentDetail> DetailList { get; set; }
    }
```
包括了值类型、特殊引用类型、普通引用类型。

然后是测试代码
```
          stu.Value = new Student()
            {
                Age = 100 ,
                DetailList = new List<StudentDetail>()
                {
                    new StudentDetail (){Address="aaaaaa"}
                } ,
                Name = "Tom"
            };
            var task2 = Task.Run(async () =>
            {
                Program.stu.Value.Age = 200;
                Program.stu.Value.DetailList = new List<StudentDetail>()
                {
                    new StudentDetail ()
                    {
                        Address ="bbbbbb"
                    }
                };
                Program.stu.Value.Name = "Tony";

                await Intercept.InvokeStudentAsync();
               
            });
            task2.Wait();

            Console.WriteLine("Age  " + Program.stu.Value.Age);
            Console.WriteLine("Address  " + Program.stu.Value.DetailList[0].Address);
            Console.WriteLine("Name  " + Program.stu.Value.Name);

```

结果：
```
Age  200
Address  bbbbbb
Name  Tony
```

但如果你在里面是写的 **Program.stu.Value = new Student();**,你会发现：
```
//Task Run里面第一行加上这行代码
//Program.stu.Value = new Student();

//结果
Age  100
Address  aaaaaa
Name  Tom
```

既然微软设计的是只读，为什么在里面重新赋值 不会报错呢？我想是因为跨线程收集异常信息很麻烦，而设计成**“线程安全”**却成本低一点。只是一般人直接就向着**只读**去用了，等到需要跨线程写值时才发现问题。。

# 总结

关于AsyncLocal有几点;
1.可以跨线程/异步共享数据
2.如果T是值类型，则数据是只读的
3.如果T是引用类型，引用本身可以看成“值”类型，所以也是只读的。
4.引用类型不可变，但引用类型下面的属性、字段，却是**可变**的。

关于AsyncLocal原理，查阅资料后有一点信息，但因为我对这些还不是那么熟悉，说不到点子上。

AsyncLocal或是说任务调度器发现AsyncLocal类的数据要跨上下文时，会把数据**浅拷贝**一份到新的上下文中。在任务执行完成，返回自己的上下文时，又重新把数据**覆盖**上去。也间接说明上面的奇怪情况。

当然，还是尽量**只读**比较好哦！



# The End
