# c++ shared_ptr & weak_ptr & enable_shared_from_this

c++ 智能指针非常好用，个人感觉只要有指针的地方就应该用智能指针。shared_ptr为一个模板类，但是有个细节需要注意，指针p指向对象时(shared_ptr< T > p(new T) )，应该保持从一开始就使用智能指针的习惯，如果用raw pointer初始化shared_ptr，那么shared_ptr释放时，会把指向的对象释放掉。如果想用**weak_ptr（不管对象的生命周期）时，只能从智能指针初始化，不可以用raw pointer初始化**。



enable_shared_from_this是干什么的？

简而言之就是把this 变为shared_ptr。

为什么需要它？当需要从一个成员函数返回对象的指针的时候：

1：可以直接返回this，但是如刚才所说，尽量任何都用智能指针，（而且将对象a的this 返回给b时，b并不知道以后使用this时，a是否还存在，这是非常危险的）。

2：另一个情况，如果成员函数返回shared< T > (this)，就会造成  **两个shared_ptr被同一个raw pointer初始化，析构的时候会被析构两次**。

现在可以用< memory >头文件的模板类 std::enable_shared_from_this来搞定这种情况，简而言之，这个模板类可以通过成员函数shared_from_this() 将this作为shared_ptr返回。同时不会发生上面 第2种情况，也就是说不管返回多少次shared_from_this()，都可以正确的引用计数。前提是想使用这个成员函数shared_from_this()的类T必须继承enable_shared_from_this。这个是很好理解的。但是为什么继承它就可以做到？

1：weak_ptr不管所指向的对象的生命周期，就是说它销毁了，也不会调用对象的析构函数。而且它还会**共享shared_ptr的计数器**。所以从同一个weak_ptr复制的多个shared_ptr也会正确计数。

2：有了weak_ptr的特性，所以每次需要返回this时，我们想办法返回一个指向该对象的weak_ptr就可以了。template < T > enable_shared_from_this就有一个成员weak_ptr< T > wp。所以只要让wp指向本对象就可以。但是weak_ptr是不能用this(raw pointer)初始化的。所以当对象创建时，这个wp为空指针，想要让wp指向this，就必须换个方式给他赋值，这个方式就是使用shared_ptr< T >指向该对象。也就是说shared_ptr< T > p (new T) 时，这个weak_ptr才会指向p。因此使用时要在上述的**智能指针初始化的情况下才可以使用shared_from_this**。之后就可以随便shared_from_this()了，因为之后返回的shared_ptr是由wp构造的（共享计数器）。这也刚好符合任何对象都由智能指针管理的风格。

3：因为wp类型为weak_ptr< T >，所以要使用T为enable_shared_from_this的模板参数。因此使用该模板的方式为

class T : enable_shared_from_this< T >:{}; 将派生类作为基类的模板参数。

