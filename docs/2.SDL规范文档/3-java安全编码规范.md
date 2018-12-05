
# 03.java安全编码规范


作者：108haili

协作：Lost Maniac

-------

## 1输入验证和数据合法性校验
程序接受数据可能来源于未经验证的用户，网络连接和其他不受信任的来源，如果未对程序接受数据进行校验，则可能会引发安全问题。

### 1.1避免SQL注入
使用PreparedStatement预编译SQL,解决SQL注入问题，传递给PreparedStatement对象的参数可以被强制进行类型转换，确保在插入或查询数据时与底层的数据库格式匹配。 

```java
String sqlString = "select * from db_user where username=? and password=?";
PreparedStatement stmt = connection.prepareStatement(sqlString);
stmt.setString(1, username);
stmt.setString(2, pwd);
ResultSet rs = stmt.executeQuery();
```

---

### 1.2避免XML注入

通过StringBulider 或 StringBuffer 拼接XML文件时，需对输入数据进行合法性校验。
对数量quantity 进行合法性校验，控制只能传入0-9的数字：

```java
if (!Pattern.matches("[0-9]+", quantity)) {
    // Format violation
  }
  String xmlString = "<item>\n<description>Widget</description>\n" +
                     "<price>500</price>\n" +
                     "<quantity>" + quantity + "</quantity></item>";
  outStream.write(xmlString.getBytes());
  outStream.flush();
```

---

### 1.3避免跨站点脚本（XSS）

对产生跨站的参数进行严格过滤，禁止传入`<SCRIPT>`标签

//定义需过滤的字段串`<script>`

`String s = "\uFE64" + "script" + "\uFE65";`

// 过滤字符串标准化

`s = Normalizer.normalize(s, Form.NFKC);`

// 使用正则表达式匹配inputStr是否存在`<script>`

```java
Pattern pattern = Pattern.compile(inputStr);
Matcher matcher = pattern.matcher(s);
if (matcher.find()) {
  // Found black listed tag
  throw new IllegalStateException();
} else {
  // ...
}
```

---

## 2声明和初始化

### 2.1避免类初始化的相互依赖

例：

错误的写法：

```java
public class Cycle {
  private final int balance;
  private static final Cycle c = new Cycle();
  private static final int deposit = (int) (Math.random() * 100); // Random deposit
  public Cycle() {
    balance = deposit - 10; // Subtract processing fee
  }
  public static void main(String[] args) {
    System.out.println("The account balance is: " + c.balance);
  }
}
```

类加载时初始化指向Cycle类的静态变量c，而类Cycle的无参构造方法又依赖静态变量deposit，导致无法预期的结果。
正确的写法：

```java
public class Cycle {
  private final int balance;
  private static final int deposit = (int) (Math.random() * 100); // Random deposit
  private static final Cycle c = new Cycle();  // Inserted after initialization of required fields
  public Cycle() {
    balance = deposit - 10; // Subtract processing fee
  }
 
  public static void main(String[] args) {
    System.out.println("The account balance is: " + c.balance);
  }
}
```

---

## 3表达式

### 3.1不可忽略方法的返回值

忽略方法的放回值可能会导致无法预料的结果。

错误的写法：

```java
public void deleteFile(){
  File someFile = new File("someFileName.txt");
   someFile.delete();
}
```

正确的写法：

```java
public void deleteFile(){
  File someFile = new File("someFileName.txt");
   if (!someFile.delete()) {
    // handle failure to delete the file
  }
}
```

### 3.2不要引用空指针

当一个变量指向一个NULL值，使用这个变量的时候又没有检查，这时会导致。NullPointerException。

在使用变量前一定要做是否为NULL值的校验。

### 3.3使用Arrays.equals()来比较数组的内容

数组没有覆盖的`Object. equals()`方法，调用`Object. equals()`方法实际上是比较数组的引用，而不是他们的内容。程序必须使用两个参数`Arrays.equals()`方法来比较两个数组的内容

```java
public void arrayEqualsExample() {
  int[] arr1 = new int[20]; // initialized to 0
  int[] arr2 = new int[20]; // initialized to 0
  Arrays.equals(arr1, arr2); // true
}
```

---

## 4数字类型和操作

### 4.1防止整数溢出

使用java.lang.Number. BigInteger类进行整数运算，防止整数溢出。

```java
public class BigIntegerUtil {

    private static final BigInteger bigMaxInt = BigInteger.valueOf(Integer.MAX_VALUE);
    private static final BigInteger bigMinInt = BigInteger.valueOf(Integer.MIN_VALUE);

    public static BigInteger intRangeCheck(BigInteger val) throws ArithmeticException {
        if (val.compareTo(bigMaxInt) == 1 || val.compareTo(bigMinInt) == -1) {
            throw new ArithmeticException("Integer overflow");
        }
        return val;
    }

    public static int addInt(int v1, int v2) throws ArithmeticException {
        BigInteger b1 = BigInteger.valueOf(v1);
        BigInteger b2 = BigInteger.valueOf(v2);
        BigInteger res = intRangeCheck(b1.add(b2));
        return res.intValue(); 
    }
    
    public static int subInt(int v1, int v2) throws ArithmeticException {
        BigInteger b1 = BigInteger.valueOf(v1);
        BigInteger b2 = BigInteger.valueOf(v2);
        BigInteger res = intRangeCheck(b1.subtract(b2));
        return res.intValue(); 
    }
    
    public static int multiplyInt(int v1, int v2) throws ArithmeticException {
        BigInteger b1 = BigInteger.valueOf(v1);
        BigInteger b2 = BigInteger.valueOf(v2);
        BigInteger res = intRangeCheck(b1.multiply(b2));
        return res.intValue(); 
    }
    
    public static int divideInt(int v1, int v2) throws ArithmeticException {
        BigInteger b1 = BigInteger.valueOf(v1);
        BigInteger b2 = BigInteger.valueOf(v2);
        BigInteger res = intRangeCheck(b1.divide(b2));
        return res.intValue(); 
    }
}
```

### 4.2避免除法和取模运算分母为零

要避免因为分母为零而导致除法和取模运算出现异常。

```java
if (num2 == 0) {
  // handle error
} else {
 result1= num1 /num2;
  result2= num1 % num2;
}
```

## 5类和方法操作

### 5.1数据成员声明为私有，提供可访问的包装方法

攻击者可以用意想不到的方式操纵public或protected的数据成员，所以需要将数据成员为private，对外提供可控的包装方法访问数据成员。

### 5.2敏感类不允许复制

包含私人的，机密或其他敏感数据的类是不允许被复制的，解决的方法有两种：

1、类声明为final

```java
final class SensitiveClass {
  // ...
}
```

2、Clone 方法抛出CloneNotSupportedException异常

```java
class SensitiveClass {
  // ...
  public final SensitiveClass clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
  }
}
```

### 5.3比较类的正确做法

如果由同一个类装载器装载，它们具有相同的完全限定名称，则它们是两个相同的类。
不正确写法：

```java
// Determine whether object auth has required/expected class object
 if (auth.getClass().getName().equals(
      "com.application.auth.DefaultAuthenticationHandler")) {
   // ...
}
正确写法：
// Determine whether object auth has required/expected class name
 if (auth.getClass() == com.application.auth.DefaultAuthenticationHandler.class) {
   // ...
}
```

### 5.4不要硬编码敏感信息

硬编码的敏感信息，如密码，服务器IP地址和加密密钥，可能会泄露给攻击者。

敏感信息均必须存在在配置文件或数据库中。

### 5.5验证方法参数

验证方法的参数，可确保操作方法的参数产生有效的结果。不验证方法的参数可能会导致不正确的计算，运行时异常，违反类的不变量，对象的状态不一致。
对于跨信任边界接收参数的方法，必须进行参数合法性校验

```java
private Object myState = null;
//对于修改myState 方法的入参，进行非空和合法性校验
void setState(Object state) {
  if (state == null) {
    // Handle null state
  }
  if (isInvalidState(state)) {
    // Handle invalid state
  }
  myState = state;
}
```

### 5.6不要使用过时、陈旧或低效的方法

在程序代码中使用过时的、陈旧的或低效的类或方法可能会导致错误的行为。

### 5.7数组引用问题

某个方法返回一个对敏感对象的内部数组的引用，假定该方法的调用程序不改变这些对象。即使数组对象本身是不可改变的，也可以在数组对象以外操作数组的内容，这种操作将反映在返回该数组的对象中。如果该方法返回可改变的对象，外部实体可以改变在那个类中声明的 public 变量，这种改变将反映在实际对象中。

不正确的写法：

```java
public class XXX {
	private String[] xxxx;
	public String[] getXXX() {
			return xxxx;
	}
}
```

正确的写法：

```java
public class XXX {
	private String[] xxxx;
	public String[] getXXX() {
			String temp[] = Arrays.copyof(…);  // 或其他数组复制方法
			return temp;
	}
}
```

### 5.8不要产生内存泄露

垃圾收集器只收集不可达的对象，因此，存在未使用的可到达的对象，仍然表示内存管理不善。过度的内存泄漏可能会导致内存耗尽，拒绝服务（DoS）。

---

## 6异常处理

### 6.1不要忽略捕获的异常

对于捕获的异常要进行相应的处理，不能忽略已捕获的异常

不正确写法：

```java
class Foo implements Runnable {
  public void run() {
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      // 此处InterruptedException被忽略
    }
  }
}
```

正确写法：

```java
class Foo implements Runnable {
  public void run() {
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt(); // Reset interrupted status
    }
  }
}
```

### 6.2不允许暴露异常的敏感信息

没有过滤敏感信息的异常堆栈往往会导致信息泄漏，

不正确的写法：

```java
try {
  FileInputStream fis =
      new FileInputStream(System.getenv("APPDATA") + args[0]);
} catch (FileNotFoundException e) {
  // Log the exception
  throw new IOException("Unable to retrieve file", e);
}
```

正确的写法：

```java
class ExceptionExample {
  public static void main(String[] args) {
    File file = null;
    try {
      file = new File(System.getenv("APPDATA") +
             args[0]).getCanonicalFile();
      if (!file.getPath().startsWith("c:\\homepath")) {
        log.error("Invalid file");
        return;
      }
    } catch (IOException x) {
     log.error("Invalid file");
      return;
    }
    try {
      FileInputStream fis = new FileInputStream(file);
    } catch (FileNotFoundException x) {
      log.error("Invalid file");
      return;
    }
  }
}
```

### 6.3不允许抛出RuntimeException, Exception,Throwable

不正确的写法：

```java
boolean isCapitalized(String s) {
  if (s == null) {
    throw new RuntimeException("Null String");
  }
}

private void doSomething() throws Exception {
  //...
}
```

正确写法：

```java
boolean isCapitalized(String s) {
  if (s == null) {
    throw new NullPointerException();
  }
}

private void doSomething() throws IOException {
  //...
}
```

### 6.4不要捕获NullPointerException或其他父类异常

不正确的写法：

```java
boolean isName(String s) {
  try {
    String names[] = s.split(" ");
    if (names.length != 2) {
      return false;
    }
    return (isCapitalized(names[0]) && isCapitalized(names[1]));
  } catch (NullPointerException e) {
    return false;
  }
}
```

正确的写法：

```java
boolean isName(String s) /* throws NullPointerException */ {
  String names[] = s.split(" ");
  if (names.length != 2) {
    return false;
  }
  return (isCapitalized(names[0]) && isCapitalized(names[1]));
}
```

## 7多线程编程

### 7.1确保共享变量的可见性

对于共享变量，要确保一个线程对它的改动对其他线程是可见的。
线程可能会看到一个陈旧的共享变量的值。为了共享变量是最新的，可以将变量声明为`volatile`或同步读取和写入操作。
将共享变量声明为`volatile`：

```java
final class ControlledStop implements Runnable {
  private volatile boolean done = false;
  @Override public void run() {
    while (!done) {
      try {
        // ...
        Thread.currentThread().sleep(1000); // Do something
      } catch(InterruptedException ie) { 
        Thread.currentThread().interrupt(); // Reset interrupted status
      } 
    }    
  }
  public void shutdown() {
    done = true;
  }
}
```

同步读取和写入操作：

```java
final class ControlledStop implements Runnable {
  private boolean done = false;
  @Override public void run() {
    while (!isDone()) {
      try {
        // ...
        Thread.currentThread().sleep(1000); // Do something
      } catch(InterruptedException ie) { 
        Thread.currentThread().interrupt(); // Reset interrupted status
      } 
    }    
  }
  public synchronized boolean isDone() {
    return done;
  }
  public synchronized void shutdown() {
    done = true;
  }
}
```

### 7.2确保共享变量的操作是原子的

除了要确保共享变量的更新对其他线程可见的，还需要确保对共享变量的操作是原子的，这时将共享变量声明为volatile往往是不够的。需要使用同步机制或Lock
同步读取和写入操作：

```java
final class Flag {
  private volatile boolean flag = true;
  public synchronized void toggle() {
    flag ^= true; // Same as flag = !flag;
  }
  public boolean getFlag() {
    return flag;
  }
}
```

//使用读取锁确保读取和写入操作的原子性

```java
final class Flag {
  private boolean flag = true;
  private final ReadWriteLock lock = new ReentrantReadWriteLock();
  private final Lock readLock = lock.readLock();
  private final Lock writeLock = lock.writeLock();
  public void toggle() {
    writeLock.lock();
    try {
      flag ^= true; // Same as flag = !flag;
    } finally {
      writeLock.unlock();
    }
  }
  public boolean getFlag() {
    readLock.lock();
    try {
      return flag;
    } finally {
      readLock.unlock();
    }
  }
}
```

### 7.3不要调用Thread.run()，不要使用Thread.stop()以终止线程

### 7.4确保执行阻塞操作的线程可以终止

```java
  public final class SocketReader implements Runnable {
  private final SocketChannel sc;
  private final Object lock = new Object();
  public SocketReader(String host, int port) throws IOException {
    sc = SocketChannel.open(new InetSocketAddress(host, port));
  }
  @Override public void run() {
    ByteBuffer buf = ByteBuffer.allocate(1024);
    try {
      synchronized (lock) {
        while (!Thread.interrupted()) {
          sc.read(buf);
          // ...
        }
      }
    } catch (IOException ie) {
      // Forward to handler
    }
  }
  public static void main(String[] args) 
                          throws IOException, InterruptedException {
    SocketReader reader = new SocketReader("somehost", 25);
    Thread thread = new Thread(reader);
    thread.start();
    Thread.sleep(1000);
    thread.interrupt();
  }
}
```

### 7.5相互依存的任务不要在一个有限的线程池执行

有限线程池指定可以同时执行在线程池中的线程数量的上限。程序不得使用有限线程池线程执行相互依赖的任务。可能会导致线程饥饿死锁，所有的线程池执行的任务正在等待一个可用的线程中执行一个内部队列阻塞

---

## 8输入输出

### 8.1程序终止前删除临时文件

### 8.2检测和处理文件相关的错误

Java的文件操作方法往往有一个返回值，而不是抛出一个异常，表示失败。因此，忽略返回值文件操作的程序，往往无法检测到这些操作是否失败。Java程序必须检查执行文件I / O方法的返回值。

不正确的写法：

```java
File file = new File(args[0]);
file.delete();
正确的写法：
File file = new File("file");
if (!file.delete()) {
  log.error("Deletion failed");
}
```

### 8.3及时释放资源

垃圾收集器无法释放非内存资源，如打开的文件描述符与数据库的连接。因此，不释放资源，可能导致资源耗尽攻击。

```java
try {
  final FileInputStream stream = new FileInputStream(fileName);
  try {
    final BufferedReader bufRead =
        new BufferedReader(new InputStreamReader(stream));
 
    String line;
    while ((line = bufRead.readLine()) != null) {
      sendLine(line);
    }
  } finally {
    if (stream != null) {
      try {
        stream.close();
      } catch (IOException e) {
        // forward to handler
      }
    }
  }
} catch (IOException e) {
  // forward to handler
}
```

## 9序列化

### 9.1不要序列化未加密的敏感数据

序列化允许一个对象的状态被保存为一个字节序列，然后重新在稍后的时间恢复，它没有提供任何机制来保护序列化的数据。敏感的数据不应该被序列化的例子包括加密密钥，数字证书。 
解决方法：

1. 对于数据成员可以使用transient ，声明该数据成员是瞬态的。
2. 重写序列化相关方法writeObject、readObject、readObjectNoData，防止被子类恶意重写

```java
class SensitiveClass extends Number {
  // ...
  protected final Object writeObject(java.io.ObjectOutputStream out) throws NotSerializableException {
    throw new NotSerializableException();
  }
  protected final Object readObject(java.io.ObjectInputStream in) throws NotSerializableException {
    throw new NotSerializableException();
  }
  protected final Object readObjectNoData(java.io.ObjectInputStream in) throws NotSerializableException {
    throw new NotSerializableException();
  }
}
```

### 9.2在序列化过程中避免内存和资源泄漏

不正确的写法：

```java
class SensorData implements Serializable {
  // 1 MB of data per instance!
   public static SensorData readSensorData() {...}
  public static boolean isAvailable() {...}
}
class SerializeSensorData {
  public static void main(String[] args) throws IOException {
    ObjectOutputStream out = null;
    try {
      out = new ObjectOutputStream(
          new BufferedOutputStream(new FileOutputStream("ser.dat")));
      while (SensorData.isAvailable()) {
        // note that each SensorData object is 1 MB in size
        SensorData sd = SensorData.readSensorData();
        out.writeObject(sd);
      }
    } finally {
      if (out != null) {
        out.close();
      }
    }
  }
}
```

正确写法：

```java
class SerializeSensorData {
  public static void main(String[] args) throws IOException {
    ObjectOutputStream out = null;
    try {
      out = new ObjectOutputStream(
          new BufferedOutputStream(new FileOutputStream("ser.dat")));
      while (SensorData.isAvailable()) {
        // note that each SensorData object is 1 MB in size
        SensorData sd = SensorData.readSensorData();
        out.writeObject(sd);
        out.reset(); // reset the stream
      }
    } finally {
      if (out != null) {
        out.close();
      }
    }
  }
}
```

### 9.3反序列化要在程序最小权限的安全环境中


(完成)