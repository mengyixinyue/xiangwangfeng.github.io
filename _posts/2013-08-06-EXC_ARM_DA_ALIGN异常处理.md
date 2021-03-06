---
layout: post
title:  EXC_ARM_DA_ALIGN 异常处理
---

 前几天无聊晃到 Matt Galloway 的博客，看到一篇 [《ARM Hacking：EXC_ARM_DA_ALIGN  Exception》][1] ，时间是 2010 年，突然有种穿越了的感觉：去年服务器的帐号从字符串改成 uint64 后，ios 版客户端时不时抛 EXC_ARM_DA_ALIGN 异常，而我最终 hack 的方法也和 Matt Galloway 的一样，所以记上一笔，方便中文搜索的童鞋们。我们定义了一种从 buffer 中获取 uint64 的方法:

{% highlight objc %}
 uint64_t pop_uint64() const
 {
     uint64_t i64 = *((uint64_t *)data_);
     i64 = xntohll(i64);
     data_ += 8u;
     size_ -= 8u;
     return i64;
 }
{% endhighlight %}
在大多数情况下这种做法都是对的，但在 pop 完某个单字节数据后，再调用 pop_uint64，编译器就会抛出 EXC_ARM_DA_ALIGN 异常。原因是 ARM 对于内存的访问是基于 WORD(32bit) 对齐，当访问未对齐的内存地址时就会抛出异常。而 hack 的方法就是对相应的内存数据做 memcpy：
{% highlight objc %}
uint64_t pop_uint64() const 
{
    uint64_t i64 = 0; 
    memcpy(&i64,data_,sizeof(i64));
    i64 = xntohll(i64); 
    data_ += 8u; 
    size_ -= 8u;
    return i64; 
}
{% endhighlight %}
不过比较奇怪的是：既然是基于 WORD 对齐，那么我们反序列化的代码里访问非 4 倍数内存地址并强制转换出 int 的代码也应该会抛出异常，而事实却是只有对 double，int64 这种 8 字节数据类型的非对齐访问才会出错，可能和处理器本身是 32 位有关。而且从理论上来说这种硬件层面的异常也不应该抛到 APP 层，系统自己默默消化就好了。

最后吐槽下 Matt Galloway 的新书《Effective Objective-C》真心贵，亚马逊上竟然卖 300 多一本，一个月的零食费啊 =。= 


  [1]: http://www.galloway.me.uk/2010/10/arm-hacking-exc_arm_da_align-exception/