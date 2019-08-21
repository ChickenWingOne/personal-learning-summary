### 作用 ###
将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。<br/>
例如一个复杂对象，属性太多时就可使用builder模式赋值，如果用多参数的构造法，则会显得代码难看，而且顺序容易错乱，一直调用set方法也是，不够美观
### 使用 ###
lombok直接可支持builder模式，在类上加入@Builder注解即可(实现原理：如类名是XXX，在此类中添加了一个内部XXXBuilder类，最后的build再赋值回对象)
### demo ###
```
@Builder
public class A{
    private String a;
}
A.ABuilder builder = A.builder();
A a = builder
            .a("hh")
            .build();
```