# 继承

## 基本概念

- Java中的继承只能是单继承。
- 继承是实现多态的前提条件。
- 构造器不可以是`private`修饰。
- 子类继承超类所有的方法（包括静态方法），超类私有方法只是无法访问，但存在子类中。
- 超类如果使用`private`修饰方法，则自动加上`final`关键字，代表无法重写。
- 超类的静态方法在子类可以直接使用，不能使用`this`或`super`关键字来获取静态方法，因为静态方法属于类而不是对象（但可以使用“类名.”来获取）；同理，无法在子类重写父类的静态方法。
- 如果子类没有重写超类的静态方法，则用“子类.静态方法”调用的还是父类的静态方法。

## 执行顺序

```java
public class A {
    private int value;
    
    public A () {
        // 父类构造器
        // 注意,虽然子类会先加载父类构造器，但调用的方法是根据创建对象来调用的。
        setValue();
    }
    
    public void setValue () {
        
    }
}



public class B extends A {
    
    public B () {
        // 默认super()
        // 子类构造器
    }
    
    public void setValue () {
        
    }
}
```



## 题目

```java
public class Chapter1ApplicationTests {  
    
    public static void main(String[] args) {
        
        System.out.println(new B().getValue());
    }
    
	static class A {
        protected int value;

        public A (int v) {
            setValue(v);	// 题目是new B()，故此处调用的是B的setValue(int v)方法
        }

        public void setValue(int val) {
            this.value = val;
        }

        public int getValue() {
            try {
                value++;
                return value;  // 如果是基本类型,则finally即便是有return value也不会改变返回值
            } finally {
                this.setValue(value);
                System.out.println(value);  // return(保留，等待finally执行完毕) --> finally
            }
        }
    }

    static class B extends A {

        public B () {
            super(5);
            setValue(getValue() - 3);  // 11 - 3 = 8
        }

        public void setValue (int val) {
            super.setValue(2 * val);
        }
    }
}

// 输出
22
34
17
```

