lambda
======

lambda表达式在C++里面可以认为是一个匿名的class。然后重载了operator () 。如果是在需要函数指针的地方，可以直接将lambda认为是一个函数指针，不需要再加&。如果加上的话会报错！
如 对于Singleton的一个资源初始化(陈硕：muduo多线程服务器编程的例子）：

    template <typename T>
    class Singleton
    {
    public:
        static T& instance()
        {
            pthread_once(&ponce,&init);
            return *p;
        }
    private:
        static init()
        {
            p = new T();
        }
        Singleton();
        ~Singleton();
        static T * p;
        static pthread_once_t ponce;
    };
    
    template <typename T>
    T * Singleton<T>::p = NULL;
    
    template <typename  T>
    pthread_once_t Singleton<T>::ponce = PTHREAD_ONCE_INIT;

其中7行如果使用lambda的话，可以改为phread_once(&ponce,[]()->void {p = new T()});（这里面的[]不可省略）不需要添加&符号。因为它不一个真正的函数，是一个类的实例。
关于这个lambda的类，结合它的特性特性如：值捕获中的变量不可修改，引用捕获中的变量可修改，mutable的值捕获可以修改{}代码块中的变量但不会影响调用者中的变量，捕获列表中的变量在lambda函数定义的时候会“保存”下来，调用时才会用到()中的参数。

**我理解为 ： 这个匿名的类的实例通过捕获列表[]中的变量完成构造，并将捕获列表中的变量复制为该类的成员，而参数列表()中的参数则为operator()时的参数。也就是说出现lambda表达式的时候，就已经用捕获列表的值初始化自己的成员了，然后等到调用时，才会使用重载的operator()。初始化成员的时候又有以下几种情况：值捕获的时候，类的成员初始化为cnostTv=v;因此会把值保存下来并且不可更改，加了mutable就去掉了const属性；引用捕获的时候初始化为T&v=v。由于调用lambda时才会调用operator，因此要保证operator的参数的生命周期。**

静态变量
====

首先静态变量在类里面是声明，这一点没什么可说的。原来我认为的类静态变量在定义的地方才会分配内存（C语言的静态变量在使用时才会分配内存），但是今天写代码发现如果在定义变量之前有该类的对象生成时，那么此时该类的静态变量也是定义好的。如：

    class a_test
    {
    public:
        static int a;
        int c = a;
    };
    a_test abc;
    int a_test::a = 10;

在7行处，a已经定义好了，否则的话会报错。
当静态变量在private：中时，无法在类外直接使用（私有成员，尽管它不是某个对象的成员，但是可以认为是该类的成员）如cout << Singleton<int>::p是错误的。那为什么第8行没有问题？因为当类出现具体的定义时，它是通过类内找到定义完成初始化的，因此没有private的限制(因此会正常的初始化）。
如上所说，**类的静态变量在类出现具体的定义时（如普通的类定义或者模板类的实例化时Singleton<int>），它的定义会被找到并按照定义的方式初始化**，也就是说当类的定义出现时，第4行就已经初始化好了。而模板类生成对象时，会先有具体的类的定义如Singleton<int>,此时p和ponce已经初始化了，然后才会生成对象。
