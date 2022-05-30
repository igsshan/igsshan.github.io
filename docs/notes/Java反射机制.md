# Java反射机制

> #### Java反射获取对象的三种方法

- Object--> getClass();
- 任何数据类型(包括基本数据类型)都有一个'`静态`'的class属性
- 通过Class类的静态方法: forName(String className) (常用)

> class类源码

```java
package java.lang;

public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
    private static final int ANNOTATION= 0x00002000;
    private static final int ENUM      = 0x00004000;
    private static final int SYNTHETIC = 0x00001000;

    private static native void registerNatives();
    static {
        registerNatives();
    }

    
    private Class(ClassLoader loader) {
        // Initialize final field for classLoader.  The initialization value of non-null
        // prevents future JIT optimizations from assuming this final field is null.
        classLoader = loader;
    }

    public String toString() {
        return (isInterface() ? "interface " : (isPrimitive() ? "" : "class "))
            + getName();
    }

    public String toGenericString() {
        if (isPrimitive()) {
            return toString();
        } else {
            StringBuilder sb = new StringBuilder();

            // Class modifiers are a superset of interface modifiers
            int modifiers = getModifiers() & Modifier.classModifiers();
            if (modifiers != 0) {
                sb.append(Modifier.toString(modifiers));
                sb.append(' ');
            }

            if (isAnnotation()) {
                sb.append('@');
            }
            if (isInterface()) { // Note: all annotation types are interfaces
                sb.append("interface");
            } else {
                if (isEnum())
                    sb.append("enum");
                else
                    sb.append("class");
            }
            sb.append(' ');
            sb.append(getName());

            TypeVariable<?>[] typeparms = getTypeParameters();
            if (typeparms.length > 0) {
                boolean first = true;
                sb.append('<');
                for(TypeVariable<?> typeparm: typeparms) {
                    if (!first)
                        sb.append(',');
                    sb.append(typeparm.getTypeName());
                    first = false;
                }
                sb.append('>');
            }

            return sb.toString();
        }
    }

    
    @CallerSensitive
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }

    @CallerSensitive
    public static Class<?> forName(String name, boolean initialize,
                                   ClassLoader loader)
        throws ClassNotFoundException
    {
        Class<?> caller = null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // Reflective call to get caller class is only needed if a security manager
            // is present.  Avoid the overhead of making this call otherwise.
            caller = Reflection.getCallerClass();
            if (sun.misc.VM.isSystemDomainLoader(loader)) {
                ClassLoader ccl = ClassLoader.getClassLoader(caller);
                if (!sun.misc.VM.isSystemDomainLoader(ccl)) {
                    sm.checkPermission(
                        SecurityConstants.GET_CLASSLOADER_PERMISSION);
                }
            }
        }
        return forName0(name, initialize, loader, caller);
    }

    // 生成新的实例
    @CallerSensitive
    public T newInstance()
        throws InstantiationException, IllegalAccessException
    {
        if (System.getSecurityManager() != null) {
            checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
        }

        // NOTE: the following code may not be strictly correct under
        // the current Java memory model.

        // Constructor lookup
        if (cachedConstructor == null) {
            if (this == Class.class) {
                throw new IllegalAccessException(
                    "Can not call newInstance() on the Class for java.lang.Class"
                );
            }
            try {
                Class<?>[] empty = {};
                final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
                // Disable accessibility checks on the constructor
                // since we have to do the security check here anyway
                // (the stack depth is wrong for the Constructor's
                // security check to work)
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedAction<Void>() {
                        public Void run() {
                                c.setAccessible(true);
                                return null;
                            }
                        });
                cachedConstructor = c;
            } catch (NoSuchMethodException e) {
                throw (InstantiationException)
                    new InstantiationException(getName()).initCause(e);
            }
        }
        Constructor<T> tmpConstructor = cachedConstructor;
        // Security check (same as in java.lang.reflect.Constructor)
        int modifiers = tmpConstructor.getModifiers();
        if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            if (newInstanceCallerCache != caller) {
                Reflection.ensureMemberAccess(caller, this, null, modifiers);
                newInstanceCallerCache = caller;
            }
        }
        // Run constructor
        try {
            return tmpConstructor.newInstance((Object[])null);
        } catch (InvocationTargetException e) {
            Unsafe.getUnsafe().throwException(e.getTargetException());
            // Not reached
            return null;
        }
    }             
}
// 更多高级用法可以在使用时去查看api
// 包括校验类型,生成实例,获取类型(实例类,接口,注解...)等等方法
```

> api使用

```java
public class DemoClazz {
public static void main(String[] args) {
		//第一种方式获取Class对象  
		Student stu1 = new Student();
		Class stuClass = stu1.getClass();//获取Class对象
		System.out.println(stuClass.getName());
		
    
    
		//第二种方式获取Class对象
		Student stu2 = new Student();
	    Student.class.getName();
		System.out.println(Student.class.getName());
		
    
    
		//第三种方式获取Class对象
		String className = "com.igsshan.Student";
		try {
			Class stuClass3 = Class.forName(className);
		} catch (ClassNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println(className);
	}
}
```

注意 :  在运行期间,一个类只生成一个class对象

> #### 反射处理运行型注解

- 判断注解是否存在`Class类`、`Field类`、`Method类`、`Constructor类`

  - Class.isAnnotationPresent(Class)
  - Field.isAnnotationPresent(Class)
  - Method.isAnnotationPresent(Class)

  - Constructor.isAnnotationPresent(Class)

- 读取注解

  - Class.getAnnotation(Class)
  - Field.getAnnotation(Class)
  - Method.getAnnotation(Class)
  - Constructor.getAnnotation(Class)

注意:  注解也是一种`类Class`,所有注解均继承自`java.lang.annotation.Annotation`,因此可以使用反射

> #### `常用方法`

- getName()，一个Class对象描述了一个特定类的特定属性，而这个方法就是返回String形式的该类的简要描述。
- newInstance()，可以根据某个Class对象产生其对应类的实例。需要强调的是，它调用的是此类的默认构造方法。例如：MyObject x = new MyObject();
  MyObject y = x.getClass().nwInstance();
- getClassLoader()，返回该Class对象对应的类的类加载器。
- getSuperClass()，返回某子类所对应的直接父类所对应的Class对象
- isArray()，判定此Class对象所对应的是否是一个数组对象

> #### `基本方法`

- getClassLoader() 
  获取该类的类装载器。 
- getComponentType() 
  如果当前类表示一个数组，则返回表示该数组组件的 Class 对象，否则返回 null。 
- getConstructor(Class[]) 
  返回当前 Class 对象表示的类的指定的公有构造子对象。 
- getConstructors() 
  返回当前 Class 对象表示的类的所有公有构造子对象数组。 
- getDeclaredConstructor(Class[]) 
  返回当前 Class 对象表示的类的指定已说明的一个构造子对象。 
- getDeclaredConstructors() 
  返回当前 Class 对象表示的类的所有已说明的构造子对象数组。 
- getDeclaredField(String) 
  返回当前 Class 对象表示的类或接口的指定已说明的一个域对象。 
- getDeclaredFields() 
  返回当前 Class 对象表示的类或接口的所有已说明的域对象数组。 
- getDeclaredMethod(String, Class[]) 
  返回当前 Class 对象表示的类或接口的指定已说明的一个方法对象。 
- getDeclaredMethods() 
  返回 Class 对象表示的类或接口的所有已说明的方法数组。 
- getField(String) 
  返回当前 Class 对象表示的类或接口的指定的公有成员域对象。 
- getFields() 
  返回当前 Class 对象表示的类或接口的所有可访问的公有域对象数组。 
- getInterfaces() 
  返回当前对象表示的类或接口实现的接口。 
- getMethod(String, Class[]) 
  返回当前 Class 对象表示的类或接口的指定的公有成员方法对象。 
- getMethods() 
  返回当前 Class 对象表示的类或接口的所有公有成员方法对象数组，包括已声明的和从父类继承的方法。 
- getModifiers() 
  返回该类或接口的 Java 语言修改器代码。 
- getName() 
  返回 Class 对象表示的类型(类、接口、数组或基类型)的完整路径名字符串。 
- getResource(String) 
  按指定名查找资源。 
- getResourceAsStream(String) 
  用给定名查找资源。 
- getSigners() 
  获取类标记。 
- getSuperclass() 
  如果此对象表示除 Object 外的任一类, 那么返回此对象的父类对象。 
- isArray() 
  如果 Class 对象表示一个数组则返回 true, 否则返回 false。 
- isAssignableFrom(Class) 
  判定 Class 对象表示的类或接口是否同参数指定的 Class 表示的类或接口相同，或是其父类。 
- isInstance(Object) 
  此方法是 Java 语言 instanceof 操作的动态等价方法。 
- isInterface() 
  判定指定的 Class 对象是否表示一个接口类型。 
- isPrimitive() 
  判定指定的 Class 对象是否表示一个 Java 的基类型。 
- newInstance() 
  创建类的新实例。 
- toString() 
  将对象转换为字符串。

```java

public class DemoClazz {
    public static void main(String[] args) throws IllegalAccessException, ClassNotFoundException, NoSuchFieldException {
        Student stu1 = new Student();
        stu1.setAge(18);
        stu1.setName("zhansgan");

        Class<? extends Student> aClass = stu1.getClass();
        Field[] declaredFields = aClass.getDeclaredFields();
        for (Field field : declaredFields) {
            field.setAccessible(true);
            System.out.println(field.getName());
            System.out.println(field.get(stu1));
        }
        System.out.println("============");
        Class stuClass = stu1.getClass();
        System.out.println(stuClass.getName());
        System.out.println(stuClass.getField("name").get(stu1));

        System.out.println("============");
        Student.class.getName();
        System.out.println(Student.class.getName());

        System.out.println("手动分割===========");
        String str1="abc";
        Class cls1=str1.getClass();
        Class cls2=String.class;
        Class cls3=Class.forName("java.lang.String");
        System.out.println(cls1==cls2);
        System.out.println(cls1==cls3);
    }
}


// 运行结果

age
18
name
zhansgan
============
com.igsshan.web.Student
zhansgan
============
com.igsshan.web.Student
手动分割===========
true
true
```

