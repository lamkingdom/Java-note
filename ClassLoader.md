# ClassLoader

## 基本概念

JVM通过不同的类加载器(ClassLoader)加载类到方法区。



## getResource()

### 方法定义

获取传入名称的文件资源，返回为URL对象。

### 使用示例

```java
package cn.invain.classLoader;

public class Test {
    public static void main(String[] args) {
        // 使用getClassLoader()
	      System.out.println(ClassA.class.getClassLoader());
        System.out.println(ClassA.class.getClassLoader().getResource(""));
        System.out.println(ClassA.class.getClassLoader().getResource("/"));

        // 直接通过类获取
        System.out.println(ClassA.class.getResource("/"));
        System.out.println(ClassA.class.getResource(""));
    }
}

// 输出
sun.misc.Launcher$AppClassLoader@18b4aac2
file:/Users/invain/Documents/idea-workspace/first/out/production/Study/
null
file:/Users/invain/Documents/idea-workspace/first/out/production/Study/
file:/Users/invain/Documents/idea-workspace/first/out/production/Study/cn/invain/classLoader/
```

#### 使用类加载器来获取

可以看到`ClassA`是我们自定义的类，故使用`AppClassLoader`加载，而如果使用类加载的`getResource()`方法获取URL，传入空字符串`""`默认是输出根目录。

而传入`"/"`则是null，是因为不支持'/'开头的方式来获取根路径，默认就是项目的根路径，这个很好理解，因为类加载器的工作是加载ClassPath上的类的，是以根目录为基础的。

**注意：**使用类加载器获取是自带`"/"`的，所以填写路径不需要在添加`"/"`，否则会找不到文件。

#### 使用类来获取

直接使用class的`getResource()`，传入`"/"`获取到的是根目录，而传入`""`是打印出当前类所在包。

即第一种是以项目根路径为初始路径，使用时必须指明从根路径开始，即必须加上`"/"`，而对于第二种相对路径默认时自带`"/"`，不需要再加上斜杆，例如访问当前类上一级文件夹只需要`./`即可。