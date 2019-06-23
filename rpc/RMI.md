# RMI

https://ipcrs.pbccrc.org.cn

RMI （Remote Method Invocation） – 远程方法调用，一种用于远程过程调用的应用程序编程接口，是纯java 的网络分布式应用系统的核心解决方案之一。RMI 使用Java 远程消息交换协议 JRMP（Java Remote Messageing Protocol） 进行通信，由于JRMP 是专为Java对象制定的，是分布式应用系统的百分之百纯java 解决方案，所以用Java RMI 开发的应用系统可以部署在任何支持JRE的平台上；缺点是，由于JRMP 是专门为java 对象指定的，因此RMI 对于非JAVA 语言开发的应用系统的支持不足，不能与非JAVA 语言书写的对象进行通信。

## 一、示例

### 1、单进程（本地）调用

```java
// service 
public interface IHelloService {
    void sayHello(String message);
}

public class HelloServiceImpl implements IHelloService {
    @Override
    public void sayHello(String message) {
        System.out.println("Hello " + message);
    }
}
```

```java
// client
public class Client {
    public static void main(String[] args) {
        IHelloService service = new HelloServiceImpl();
        service.sayHello("world");
    }
}
```

### 2、使用RMI

```java
// service
// 须继承 java.rmi.Remote 接口
public interface IHelloService extends Remote {
	// 远程调用的方法必须抛出此异常，否则报错
	void sayHello(String message) throws RemoteException; 
}

// 实现类须继承 java.rmi.server.UnicastRemoteObject 类
public class HelloServiceImpl extends UnicastRemoteObject implements IHelloService {
    // 必须覆盖父构造器；
    // 若此处声明构造器为 public，在编译过程中会有警告；
    // 构造器必须抛出 RemoteException
	protected HelloServiceImpl() throws RemoteException {
        super();
    }

    public void sayHello(String message) throws RemoteException{
        System.out.println("Hello " + msg);
    }
}
```

```java
/* 此时需要一个类将远程调用方法发布（暴露）给调用者 */
// server
import java.rmi.registry.LocateRegistry;
public class Server {
    public static void main(String args[]) {
        try {   
            // 创建实例
            IHelloService hello = new HelloServiceImpl(); 
            // 使用 LocateRegistry 注册服务（类似分布式系统中的注册中心）
            LocateRegistry.createRegistry(1099); 
		   // 使用命名服务进行绑定
            Naming.bind("rmi:/127.0.0.1/hello", hello);
            System.out.println("service started..."); 
        } catch (RemoteException e) {
            e.printStackTrace();
        } catch (AlreadyBoundException e) {
            e.printStackTrace();
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
    }
}
```

```java
/* 客户端调用 */
// client
public class Client {
    public static void main(String[] args) 
        throws RemoteException, NotBoundException, MalformedURLException {
        
        // 使用命名服务在在注册中心查找具有相同名字的服务
        IHelloService hello = (IHelloService) Naming.lookup("rmi:/127.0.0.1/hello");
        hello.sayHello("world");
    }
}
```

在生产环境中，需要将通用的类或接口单独做成依赖，以便客户端调用。

## 二、RMI原理分析

### 1、远程对象发布

在使用过程中，我们发现，远程对象必须实现`java.rmi.server.UnicastRemoteObject`，这样才能保证客户端访问获得远程对象。在上面的示例中，`HelloServiceImpl`类需要继承`java.rmi.server.UnicastRemoteObject`并重写构造器，所以，我们从构造器深入源码观察调用过程。

#### 2.1.1 UnicastRemoteObject

`java.rmi.server.UnicastRemoteObject`的构造器：

```java
// java.rmi.server.UnicastRemoteObject#UnicastRemoteObject()
/* 创建一个使用匿名端口的 UnicastRemoteObject 对象 */
protected UnicastRemoteObject() throws RemoteException {
	this(0);
}

// java.rmi.server.UnicastRemoteObject#UnicastRemoteObject(int)
/* 创建一个使用特定端口的 UnicastRemoteObject 对象 */
protected UnicastRemoteObject(int port) throws RemoteException {
    this.port = port;
    exportObject((Remote) this, port); // exportObject()
}
```

追溯源码过程中可以发现，在`UnicastRemoteObject`的构造器中调用了`exportObject()`方法，向下追溯到此方法：

```java
// java.rmi.server.UnicastRemoteObject#exportObject(Remote, int)
/* 发布远程对象，使其能够接收即将到达的调用, 使用了特定端口. */
public static Remote exportObject(Remote obj, int port) throws RemoteException
{
	// 注意此处封装了一个 UnicastServerRef 对象
	return exportObject(obj, new UnicastServerRef(port)); 
}

```

```java
// java.rmi.server.UnicastRemoteObject#exportObject(Remote, UnicastServerRef)
/* 利用指定的 服务引用 发布指定的对象 */
private static Remote exportObject(Remote obj, UnicastServerRef sref) throws RemoteException {
    // 判断服务的实现是不是 `UnicastRemoteObject `的子类，
    // 如果是，则直接赋值其`ref（RemoteRef）`对象为传入的`UnicastServerRef `对象。
    // 反之则调用`sun.rmi.server.UnicastServerRef`类的`exportObject()`方法。
    if (obj instanceof UnicastRemoteObject) {
    	((UnicastRemoteObject) obj).ref = sref;
    }
    
    // 调用 sun.rmi.server.UnicastServerRef#exportObject方法
    return sref.exportObject(obj, null, false); 
}
```

在示例中，因为`HelloServiceImpl `继承了`UnicastRemoteObject`，所以在服务启动的时候，会通过`UnicastRemoteObject`的构造方法把该对象进行发布。

再观察`sun.rmi.server.UnicastServerRef`类中的`exportObject`方法，由于Oracle Java并不是完全开源的，所以需要借助工具查看。我们先简要看一下代码， 不做深入。

#### 2.1.2 UnicastServerRef

sun.rmi.server.UnicastServerRef

```java
// sun.rmi.server.UnicastServerRef#exportObject()
public Remote exportObject(Remote var1, Object var2, boolean var3) throws RemoteException {
    Class var4 = var1.getClass(); 
    Remote var5;
    try {
        // 创建代理, this time forceStubUse=false
        var5 = Util.createProxy(var4, this.getClientRef(), this.forceStubUse); // Remote
    } catch (IllegalArgumentException var7) {
        throw new ExportException("remote object implements illegal remote interface", var7);
    }

    //  Stub 是什么？ 
    if (var5 instanceof RemoteStub) { // RemoteStub.class
        // skeleton 是什么?
        this.setSkeleton(var1);  //  create skeleton with reflection api
    }

    // Target负责包装实际对象
    Target var6 = new Target(var1, this, var5, this.ref.getObjID(), var3);
    // 这里的ref指的是sun.rmi.transport.LiveRef， 在下面有分析这段代码
    this.ref.exportObject(var6); // protected sun.rmi.transport.LiveRef#exportObject(); 
    this.hashToMethod_Map = (Map)hashToMethod_Maps.get(var4); // ???
    return var5;
}


```

#### 2.1.2 Util - 1

第一次观察 `sun.rmi.server.Util#createProxy(Class, RemoteRef, boolean)`

```java
// sun.rmi.server.Util#createProxy(Class, RemoteRef, boolean)
// var0=Remote ; var1= getClientRef(); var2=forceStubUse=false here
public static Remote createProxy(Class<?> var0, RemoteRef var1, boolean var2) throws
    StubNotFoundException {
    Class var3;
    try {
        var3 = getRemoteClass(var0); // 下面有分析
    } catch (ClassNotFoundException var9) {
		// code ignored
    }

    // var2 ->forceStubUse OR exists StubClass... --> createStub
    // downbelow subClassExists ..
    if (var2 || !ignoreStubClasses && stubClassExists(var3)) {
        return createStub(var3, var1); 
    } else {
        // 使用动态代理
        final ClassLoader var4 = var0.getClassLoader();
        final Class[] var5 = getRemoteInterfaces(var0);
        // java.rmi.server.RemoteObjectInvocationHandler
        final RemoteObjectInvocationHandler var6 = new RemoteObjectInvocationHandler(var1);

        try {
            return (Remote)AccessController.doPrivileged(new PrivilegedAction<Remote>() {
                public Remote run() {
                    return (Remote)Proxy.newProxyInstance(var4, var5, var6);
                }
            });
        } catch (IllegalArgumentException var8) {
            throw new StubNotFoundException("unable to create proxy", var8);
        }
    }
}

```

sun.rmi.server.Util#getRemoteClass(Class)

```java
// sun.rmi.server.Util#getRemoteClass(Class)
private static Class<?> getRemoteClass(Class<?> var0) throws ClassNotFoundException {
    while(var0 != null) {
        Class[] var1 = var0.getInterfaces();

        for(int var2 = var1.length - 1; var2 >= 0; --var2) {
            // class1.isAssignableFrom(class2):
            // 判定此 Class 对象所表示的类或接口与指定的 Class 参数所表示的类或接口是否相同，
            // 或是否是其超类或超接口。如果是则返回 true； 否则返回 false。
            // 如果该 Class 表示一个基本类型，且指定的 Class 参数正是该 Class 对象，
            // 则该方法返回 true；否则返回 false。
            if (Remote.class.isAssignableFrom(var1[var2])) {
                return var0;
            }
        }

        var0 = var0.getSuperclass(); // 注意这里，使用了superclass, 返回直接继承的父类
    }

    throw new ClassNotFoundException("class does not implement java.rmi.Remote");
}

```

sun.rmi.server.Util#createStub(Class, RemoteRef)

```java
// sun.rmi.server.Util#createStub(Class, RemoteRef)
private static RemoteStub createStub(Class<?> var0, RemoteRef var1) throws StubNotFoundException {
    String var2 = var0.getName() + "_Stub";

    try {
        Class var3 = Class.forName(var2, false, var0.getClassLoader());
        Constructor var4 = var3.getConstructor(stubConsParamTypes);
        return (RemoteStub)var4.newInstance(var1);
    } catch (ClassNotFoundException var5) {
       // code ignored...
    }
}

```

 sun.rmi.server.Util#stubClassExists(Class)

```java
// sun.rmi.server.Util#stubClassExists(Class)
private static boolean stubClassExists(Class<?> var0) {
    if (!withoutStubs.containsKey(var0)) {
        try {
            // if XXX_Stub.class exsits
            Class.forName(var0.getName() + "_Stub", false, var0.getClassLoader());
            return true;
        } catch (ClassNotFoundException var2) {
            // no exists
            withoutStubs.put(var0, (Object)null);
        }
    }

    return false;
}

```

java.rmi.server.RemoteObjectInvocationHandler#invoke(Object, Method, Object[])

```java
// java.rmi.server.RemoteObjectInvocationHandler#invoke(Object, Method, Object[])
public Object invoke(Object proxy, Method method, Object[] args)  throws Throwable
{
    if (! Proxy.isProxyClass(proxy.getClass())) {
        throw new IllegalArgumentException("not a proxy");
    }

    if (Proxy.getInvocationHandler(proxy) != this) {
        throw new IllegalArgumentException("handler mismatch");
    }

	// 得到目标方法所在类对应的Class对象
    if (method.getDeclaringClass() == Object.class) {
        return invokeObjectMethod(proxy, method, args); // 
    } 
    else if ("finalize".equals(method.getName()) && 
    					method.getParameterCount() == 0 && !allowFinalizeInvocation) {
        return null; // ignore
    } else {
        return invokeRemoteMethod(proxy, method, args);
    }
}

```

// java.rmi.server.RemoteObjectInvocationHandler#invokeObjectMethod(Object, Method, Object[])

```java
//  java.rmi.server.RemoteObjectInvocationHandler#invokeObjectMethod(Object, Method, Object[])
private Object invokeObjectMethod(Object proxy,  Method method, Object[] args)
{
    String name = method.getName();

    if (name.equals("hashCode")) {
        return hashCode();

    } else if (name.equals("equals")) {
        Object obj = args[0];
        InvocationHandler hdlr;
        return
            proxy == obj ||
            (obj != null &&
             Proxy.isProxyClass(obj.getClass()) &&
             (hdlr = Proxy.getInvocationHandler(obj)) instanceof RemoteObjectInvocationHandler &&
             this.equals(hdlr));

    } else if (name.equals("toString")) {
        return proxyToString(proxy);

    } else {
        throw new IllegalArgumentException(
            "unexpected Object method: " + method);
    }
}

```

 java.rmi.server.RemoteObjectInvocationHandler#invokeRemoteMethod(Object, Method, Object[])

```java
// java.rmi.server.RemoteObjectInvocationHandler#invokeRemoteMethod(Object, Method, Object[])
private Object invokeRemoteMethod(Object proxy, Method method, Object[] args) throws Exception
{
    try {
        if (!(proxy instanceof Remote)) {
        	// MUST BE Remote Instance
            throw new IllegalArgumentException("proxy not Remote instance"); 
        }
        // ref = java.rmi.server.RemoteRef <- sun.rmi.server.UnicastRef
        return ref.invoke((Remote) proxy, method, args, getMethodHash(method));
    } catch (Exception e) {
        // code ignored, Exception process...
    }
}

```

sun.rmi.server.UnicastRef

```java
// sun.rmi.server.UnicastRef#invoke(Remote, Method, Object[], long)
public Object invoke(Remote var1, Method var2, Object[] var3, long var4) throws Exception {

    // connection: sun.rmi.transport.Connection <- sun.rmi.transport.tcp.TCPConnection
    Connection var6 = this.ref.getChannel().newConnection(); 
    StreamRemoteCall var7 = null;
    boolean var8 = true;
    boolean var9 = false;

    Object var13;
    try {
        var7 = new StreamRemoteCall(var6, this.ref.getObjID(), -1, var4);

        Object var11;
        try {
            ObjectOutput var10 = var7.getOutputStream();
            this.marshalCustomCallData(var10);
            var11 = var2.getParameterTypes();

            for(int var12 = 0; var12 < ((Object[])var11).length; ++var12) {
                marshalValue((Class)((Object[])var11)[var12], var3[var12], var10);
            }
        } catch (IOException var39) {
            throw new MarshalException("error marshalling arguments", var39);
        }

        var7.executeCall(); //execute call

        try {
            Class var46 = var2.getReturnType();
            if (var46 == Void.TYPE) {
                var11 = null;
                return var11;
            }

            var11 = var7.getInputStream();
            Object var47 = unmarshalValue(var46, (ObjectInput)var11);
            var9 = true;
            this.ref.getChannel().free(var6, true);
            var13 = var47;
        } catch (ClassNotFoundException | IOException var40) {
            ((StreamRemoteCall)var7).discardPendingRefs();
            throw new UnmarshalException("error unmarshalling return", var40);
        } finally {
            try {
                var7.done();
            } catch (IOException var38) {
                var8 = false;
            }
        }
    } catch (RuntimeException var42) {
		// code ignored...
    } finally {
        if (!var9) {
            this.ref.getChannel().free(var6, var8);
        }
    }
    return var13;
}

```

sun.rmi.transport.StreamRemoteCall

```java
// sun.rmi.transport.StreamRemoteCall
public void executeCall() throws Exception {
    DGCAckHandler var2 = null;
    byte var1;
    try {
        if (this.out != null) {
            var2 = this.out.getDGCAckHandler();
        }

        this.releaseOutputStream();
        DataInputStream var3 = new DataInputStream(this.conn.getInputStream());
        byte var4 = var3.readByte();
        if (var4 != 81) {
            throw new UnmarshalException("Transport return code invalid");
        }

        this.getInputStream();
        var1 = this.in.readByte();
        this.in.readID();
    } catch (UnmarshalException var11) {
     	// ...
    } finally {
        if (var2 != null) {
            var2.release();
        }
    }

    switch(var1) {
    case 1:
        return;
    case 2:
        Object var14;
        try {
            var14 = this.in.readObject(); // readObject
        } catch (Exception var10) {
            throw new UnmarshalException("Error unmarshaling return", var10);
        }

        if (!(var14 instanceof Exception)) {
            throw new UnmarshalException("Return type not Exception");
        } else {
            this.exceptionReceivedFromServer((Exception)var14);
        }
    default:
        if (Transport.transportLog.isLoggable(Log.BRIEF)) {
            Transport.transportLog.log(Log.BRIEF, "return code invalid: " + var1);
        }
        throw new UnmarshalException("Return code invalid");
    }
}

```



可以认为在上一段代码执行完毕时，就已发布了一个远程对象。具体的方法在下面有分析。

我们看代码 `Server.java` 中的下一行：注册

```java
// Server.java   -- createRegistry
LocateRegistry.createRegistry(1099);  // java.rmi.registry.Registry

```



```java
// java.rmi.registry.Registry#createRegistry(int)
public static Registry createRegistry(int port) throws RemoteException {
    // call RegistryImpl
    return new RegistryImpl(port);
}

```



```java
// sun.rmi.registry.RegistryImpl#RegistryImpl(int)
public RegistryImpl(final int var1) throws RemoteException {
    this.bindings = new Hashtable(101);
   
    // try 块中，判断如果端口是 1099 并且开启了权限管理， 则绕过权限
    if (var1 == 1099 && System.getSecurityManager() != null) {
        try {
            AccessController.doPrivileged(new PrivilegedExceptionAction<Void>() {
                public Void run() throws RemoteException {
                    LiveRef var1x = new LiveRef(RegistryImpl.id, var1);
                    // 调用了 setup()
                    RegistryImpl.this.setup(new UnicastServerRef(var1x, (var0) -> {
                        return RegistryImpl.registryFilter(var0);
                    }));
                    return null;
                }
            }, 
           (AccessControlContext)null, new SocketPermission("localhost:" + var1, "listen,accept"));
        } catch (PrivilegedActionException var3) {
            throw (RemoteException)var3.getException();
        }
    } else {
        LiveRef var2 = new LiveRef(id, var1);
        // 调用了 setup
        this.setup(new UnicastServerRef(var2, RegistryImpl::registryFilter));
    }
}

```

上面的代码中，有 `AccessController`，它是一个权限控制类，这里我们不做探讨，另外，我们看到上面有两处调用了setup方法，所以进下一步分析setup()方法。

```java
// sun.rmi.registry.RegistryImpl#setup(UnicastServerRef)
// 注意在这也封装了一个 UnicastServerRef 对象 
private void setup(UnicastServerRef var1) throws RemoteException {
    this.ref = var1;
    var1.exportObject(this, (Object)null, true);
}

```

至此，可以认为远程对象注册完毕。

在setup方法中，又调用了 上面提到的 `sun.rmi.server.UnicastServerRef`类的 `exportObject()` 方法，我们再回过头深入其中。先观察 `sun.rmi.server.Util#createProxy()` 。

```java
// sun.rmi.server.Util#createProxy
// var0=Remote ; var1= getClientRef(); var2=forceStubUse
public static Remote createProxy(Class<?> var0, RemoteRef var1, boolean var2)  
    throws StubNotFoundException {
    Class var3;
    try {
        var3 = getRemoteClass(var0); // 获取远程对象的复制
    } catch (ClassNotFoundException var9) {
        throw new StubNotFoundException("...."");
    }

    // 判断是否使用Stub, 如果是就创建一个Stub否则创建一个代理类
    if (var2 || !ignoreStubClasses && stubClassExists(var3)) {
        // “stub”, 下面有分析这个方法
        return createStub(var3, var1);  // return var1's proxy
    } else {
        // 使用代理
        final ClassLoader var4 = var0.getClassLoader();
        final Class[] var5 = getRemoteInterfaces(var0);
        final RemoteObjectInvocationHandler var6 = new RemoteObjectInvocationHandler(var1);

        try {
            return (Remote)AccessController.doPrivileged(new PrivilegedAction<Remote>() {
                public Remote run() {
                    return (Remote)Proxy.newProxyInstance(var4, var5, var6); // Proxy
                }
            });
        } catch (IllegalArgumentException var8) {
            throw new StubNotFoundException("unable to create proxy", var8);
        }
    }
}

```

看完下面这段代码，再回看上面的代码。

```java
// sun.rmi.server.Util#createStub
// 使用反射创建了一个代理类
private static RemoteStub createStub(Class<?> var0, RemoteRef var1) throws StubNotFoundException {
	// 生成的代理类会以_Stub结尾, HelloServiceImpl_Stub.class
    String var2 = var0.getName() + "_Stub"; 
	try {
		Class var3 = Class.forName(var2, false, var0.getClassLoader());
		Constructor var4 = var3.getConstructor(stubConsParamTypes);
		return (RemoteStub)var4.newInstance(var1); // cast to RemoteStub.class
	} catch (ClassNotFoundException var5) {
		// code ignored , throw some exceptions
	}
}

```

sun.rmi.server.UnicastServerRef#exportObject()

```java
// sun.rmi.server.UnicastServerRef#getClientRef()
protected RemoteRef getClientRef() {
    // 在RemoteRef中使用了UnicastRef
    // UnicastRef构造器的 this.ref 指向 sun.rmi.transport.LiveRef
	return new UnicastRef(this.ref); 
}

```

在 `sun.rmi.server.UnicastServerRef`类的 `exportObject()` 方法中，下面包装实际类的部分，使用了`sun.rmi.transport.LiveRef#exportObject()` 。

```java
// sun.rmi.transport.LiveRef#exportObject()
public void exportObject(Target var1) throws RemoteException {
	// 这里的ep指向的是  sun.rmi.transport.Endpoint
    this.ep.exportObject(var1); 
}

```

sun.rmi.transport.Endpoint#exportObject()

```java
// sun.rmi.transport.Endpoint#exportObject() ,
// Endpoint 是一个接口，它只有一个实现类 sun.rmi.transport.tcp.TCPEndpoint
public interface Endpoint {
    Channel getChannel();
    void exportObject(Target var1)	 throws RemoteException;
    Transport getInboundTransport();
    Transport getOutboundTransport();
}

```

sun.rmi.transport.tcp.TCPEndpoint#exportObject()

```java
// sun.rmi.transport.tcp.TCPEndpoint#exportObject() 实现了  Endpoint
public void exportObject(Target var1) throws RemoteException {
    // 这里的 transport 指向的是sun.rmi.transport.tcp.TCPTransport
	this.transport.exportObject(var1);
}

```

sun.rmi.transport.tcp.TCPTransport#exportObject()

```java
// sun.rmi.transport.tcp.TCPTransport#exportObject()
public void exportObject(Target var1) throws RemoteException {
    synchronized(this) {
        this.listen(); // listen()里做了什么？
        ++this.exportCount;
    }

    boolean var2 = false;
    boolean var12 = false;

    try {
        var12 = true;
        // 调用了父类的exportObject，父类是 sun.rmi.transport.Transport
        super.exportObject(var1);  // 注意这里
        var2 = true;
        var12 = false;
    } finally {
        // code ignored
    }
	// ...
}

```



```java
// sun.rmi.transport.tcp.TCPTransport#listen()
private void listen() throws RemoteException {
    TCPEndpoint var1 = this.getEndpoint();
    int var2 = var1.getPort();
    if (this.server == null) {
        try {
            // 这里使用了ServerSocket -- create server socket 在TCP 协议层发起socket 监听，
            // 由此知道，发布远程对象是使用了TPC 的 SOCKET
            // 该远程对象会把自身的一个拷贝以Socket 形式传输给客户端，
            // 客户端获得的拷贝称为“stub” ， 
            // 而服务器端本身已经存在的远程对象成为“skeleton”，
            // 此时客户端的stub 是客户端的一个代理，用于与服务器端进行通信，
            // 而skeleton 是服务端的一个代理，用于接收客户端的请求之后调用远程方法来响应客户端的请求
            this.server = var1.newServerSocket();
           
            // 并采用多线程循环接收请求
            // 如果服务 端指定的端口是1099 
            // 并且系统开启了安全管理器，那么就 可以在限定的权限集内绕过系统的安全校验
            Thread var3 = (Thread)AccessController.
                doPrivileged( new NewThreadAction(new TCPTransport.AcceptLoop(this.server), 
                              "TCP Accept-" + var2, 
                              true));
            var3.start();
        } catch (BindException var4) {
        	// code ignored, throw some exceptions
        }
    } else {
        SecurityManager var6 = System.getSecurityManager();
        if (var6 != null) {
            var6.checkListen(var2);
        }
    }
}

```

`sun.rmi.runtime.NewThreadAction`是一个线程，我们看其构造器的 `sun.rmi.transport.tcp.TCPTransport.AcceptLoop` 参数做了什么：

```java
// sun.rmi.transport.tcp.TCPTransport.AcceptLoop.class
AcceptLoop(ServerSocket var2) {
    this.serverSocket = var2;
}

public void run() {
    try {
        this.executeAcceptLoop(); // 执行 executeAcceptLoop()
    } finally {
        try {
            this.serverSocket.close();
        } catch (IOException var7) {
            ;
        }
    }
}

```



```java
// sun.rmi.transport.tcp.TCPTransport.AcceptLoop#executeAcceptLoop()
private void executeAcceptLoop() {
    while(true) {
        Socket var1 = null;
        try {
            var1 = this.serverSocket.accept(); // 使用了server socket
            InetAddress var16 = var1.getInetAddress();
            String var3 = var16 != null ? var16.getHostAddress() : "0.0.0.0";
            try {
                // TCP connection
                TCPTransport.connectionThreadPool.execute(
                    TCPTransport.this.new ConnectionHandler(var1, var3) );
            } catch (RejectedExecutionException var11) {
                // ...
            }
        } catch (Throwable var15) {
            // ...
        }
    }
}

```



```java
private class ConnectionHandler implements Runnable {
	//...
}

```



```java
//sun.rmi.transport.tcp.TCPTransport.ConnectionHandler#run()
public void run() {
    Thread var1 = Thread.currentThread();
    String var2 = var1.getName();

    try {
        var1.setName("RMI TCP Connection(" 
                     + TCPTransport.connectionCount.incrementAndGet() 
                     + ")-" 
                     + this.remoteHost);
        AccessController.doPrivileged(() -> {
            this.run0(); // run0()
            return null;
        }, TCPTransport.NOPERMS_ACC);
    } finally {
        var1.setName(var2);
    }
}

```



```java
// //sun.rmi.transport.tcp.TCPTransport.ConnectionHandler#run0()
private void run0() {
        TCPEndpoint var1 = TCPTransport.this.getEndpoint();
        int var2 = var1.getPort();
        TCPTransport.threadConnectionHandler.set(this);

        try {
            this.socket.setTcpNoDelay(true);
        } catch (Exception var31) {
            ;
        }

        try {
            if (TCPTransport.connectionReadTimeout > 0) {
                this.socket.setSoTimeout(TCPTransport.connectionReadTimeout);
            }
        } catch (Exception var30) {
            ;
        }

        try {
            InputStream var3 = this.socket.getInputStream();
            Object var4 = var3.markSupported() ? var3 : new BufferedInputStream(var3);
            ((InputStream)var4).mark(4);
            DataInputStream var5 = new DataInputStream((InputStream)var4);
            int var6 = var5.readInt();
            if (var6 == 1347375956) {
                if (TCPTransport.disableIncomingHttp) {
                    throw new RemoteException("RMI over HTTP is disabled");
                }

                TCPTransport.tcpLog.log(Log.BRIEF, "decoding HTTP-wrapped call");
                ((InputStream)var4).reset();

                try {
                    this.socket = new HttpReceiveSocket(this.socket, (InputStream)var4, (OutputStream)null);
                    this.remoteHost = "0.0.0.0";
                    var3 = this.socket.getInputStream();
                    var4 = new BufferedInputStream(var3);
                    var5 = new DataInputStream((InputStream)var4);
                    var6 = var5.readInt();
                } catch (IOException var29) {
                    throw new RemoteException("Error HTTP-unwrapping call", var29);
                }
            }

            short var7 = var5.readShort();
            if (var6 == 1246907721 && var7 == 2) {
                OutputStream var8 = this.socket.getOutputStream();
                BufferedOutputStream var9 = new BufferedOutputStream(var8);
                DataOutputStream var10 = new DataOutputStream(var9);
                int var11 = this.socket.getPort();
                if (TCPTransport.tcpLog.isLoggable(Log.BRIEF)) {
                    TCPTransport.tcpLog.log(Log.BRIEF, "accepted socket from [" + this.remoteHost + ":" + var11 + "]");	
                }

                byte var15 = var5.readByte();
                TCPEndpoint var12;
                TCPChannel var13;
                TCPConnection var14;
                switch(var15) {
                case 75:
                    var10.writeByte(78);
                    if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                        TCPTransport.tcpLog.log(Log.VERBOSE, 
                        "(port " + var2 + ") suggesting " + this.remoteHost + ":" + var11);
                    }

                    var10.writeUTF(this.remoteHost);
                    var10.writeInt(var11);
                    var10.flush();
                    String var16 = var5.readUTF();
                    int var17 = var5.readInt();
                    if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                        TCPTransport.tcpLog.log(Log.VERBOSE, 
                        "(port " + var2 + ") client using " + var16 + ":" + var17);
                    }

                    var12 = new TCPEndpoint(this.remoteHost, 
                                            this.socket.getLocalPort(),
                                            var1.getClientSocketFactory(), 
                                            var1.getServerSocketFactory());
                    var13 = new TCPChannel(TCPTransport.this, var12);
                    var14 = new TCPConnection(var13, this.socket, (InputStream)var4, var9);
                    TCPTransport.this.handleMessages(var14, true); // handle message
                    return;
                case 76:
                    var12 = new TCPEndpoint(this.remoteHost,
                                            this.socket.getLocalPort(),
                                            var1.getClientSocketFactory(), 
                                            var1.getServerSocketFactory());
                    var13 = new TCPChannel(TCPTransport.this, var12);
                    var14 = new TCPConnection(var13, this.socket, (InputStream)var4, var9);
                    TCPTransport.this.handleMessages(var14, false); // handle message
                    return;
                case 77:
                    if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                        TCPTransport.tcpLog.log(Log.VERBOSE, "(port " + var2 + ") accepting multiplex protocol");
                    }

                    var10.writeByte(78);

                    var10.writeUTF(this.remoteHost);
                    var10.writeInt(var11);
                    var10.flush();
                    var12 = new TCPEndpoint(var5.readUTF(), 
                                            var5.readInt(), 
                                            var1.getClientSocketFactory(),
                                            var1.getServerSocketFactory());

                    ConnectionMultiplexer var18;
                    // process multi-connections
                    synchronized(TCPTransport.this.channelTable) {
                        var13 = TCPTransport.this.getChannel(var12);
                        var18 = new ConnectionMultiplexer(var13, (InputStream)var4, var8, false);
                        var13.useMultiplexer(var18);
                    }

                    var18.run();
                    return;
                default:
                    var10.writeByte(79);
                    var10.flush();
                    return;
                }
            }

            TCPTransport.closeSocket(this.socket);
        } catch (IOException var32) {
            TCPTransport.tcpLog.log(Log.BRIEF, "terminated with exception:", var32);
            return;
        } finally {
            TCPTransport.closeSocket(this.socket);
        }

    }
}

```

这个run0()方法里做了一些判断，但是它太复杂，而且还有一些特定的数字，估计或者猜测是对不同的通信协议做了不同的处理，但是不论它的具体过程做了什么 ，它最终会执行最后的finally{}代码块，

```java
// sun.rmi.transport.tcp.TCPTransport#handleMessages()
void handleMessages(Connection var1, boolean var2) {
    int var3 = this.getEndpoint().getPort();

    try {
        DataInputStream var4 = new DataInputStream(var1.getInputStream());

        do {
            int var5 = var4.read();
            if (var5 == -1) {
                return;
            }

            switch(var5) {
            case 80:
                StreamRemoteCall var6 = new StreamRemoteCall(var1);
               // service call
                if (!this.serviceCall(var6)) {
                    return; 
                }
                break;
            case 81:
            case 83:
            default:
                throw new IOException("unknown transport op " + var5);
            case 82:
                DataOutputStream var7 = new DataOutputStream(var1.getOutputStream());
                var7.writeByte(83);
                var1.releaseOutputStream();
                break;
            case 84:
                DGCAckHandler.received(UID.read(var4));
            }
        } while(var2);

    } catch (IOException var17) {
        if (tcpLog.isLoggable(Log.BRIEF)) {
            tcpLog.log(Log.BRIEF, "(port " + var3 + ") exception: ", var17);
        }

    } finally {
        try {
            var1.close();
        } catch (IOException var16) {
            ;
        }

    }
}

```



```java
// sun.rmi.transport.Transport#serviceCall()
public boolean serviceCall(final RemoteCall var1) {
    try {
        ObjID var39;
        try {
            var39 = ObjID.read(var1.getInputStream());
        } catch (IOException var33) {
            throw new MarshalException("unable to read objID", var33);
        }

        Transport var40 = var39.equals(dgcID) ? null : this;
        // Get From ObjectTable
        Target var5 = ObjectTable.getTarget(new ObjectEndpoint(var39, var40)); 
        final Remote var37;
        if (var5 != null && (var37 = var5.getImpl()) != null) {
            final Dispatcher var6 = var5.getDispatcher();
            var5.incrementCallCount();

            boolean var8;
            try {
                // call dispatcher
                final AccessControlContext var7 = var5.getAccessControlContext();
                ClassLoader var41 = var5.getContextClassLoader();
                ClassLoader var9 = Thread.currentThread().getContextClassLoader();

                try {
                    setContextClassLoader(var41);
                    currentTransport.set(this);

                    try {
                        AccessController.doPrivileged(new PrivilegedExceptionAction<Void>() {
                            public Void run() throws IOException {
                                Transport.this.checkAcceptPermission(var7);
                                var6.dispatch(var37, var1); //DISPATCH
                                return null;
                            }
                        }, var7);
                        return true;
                    } catch (PrivilegedActionException var31) {
                        throw (IOException)var31.getException();
                    }
                } finally {
                    setContextClassLoader(var9);
                    currentTransport.set((Object)null);
                }
            } catch (IOException var34) {
                transportLog.log(Log.BRIEF, "exception thrown by dispatcher: ", var34);
                var8 = false;
            } finally {
                var5.decrementCallCount();
            }

            return var8;
        }

        throw new NoSuchObjectException("no such object in table");
    } catch (RemoteException var36) {
       // ... ...
    }

        try {
            ObjectOutput var38 = var1.getResultStream(false);
            UnicastServerRef.clearStackTraces(var2);
            var38.writeObject(var2);
            var1.releaseOutputStream();
        } catch (IOException var29) {
            transportLog.log(Log.BRIEF, "exception thrown marshalling exception: ", var29);
            return false;
        }
    }

    return true;
}

```





RMI使用了TCP 的 ServerSocket 进行通信。

因为 sun.rmi.transport.tcp.TCPTransport#exportObject() 方法调用了 sun.rmi.transport.Transport#exportObject() 

```java
// super.exportObject(var1)
// sun.rmi.transport.Transport#exportObject()
public void exportObject((Target var1) throws RemoteException {
    var1.setExportedTransport(this);
    // 使用 ObjectTable 来维护远程对象
    // ObjectTable中使用 HashMap 做为对象容器， 具体的就不分析了。
    ObjectTable.putTarget(var1);
}

```

再看`Server.java`中的下一段代码：

```java
// Server.java
Naming.bind("rmi:/127.0.0.1/hello", "hello") // 也可以使用 rebind() 方法

```



```java
// java.rmi.Naming#bind()
public static void bind(String name, Remote obj)
    throws AlreadyBoundException, java.net.MalformedURLException, RemoteException
{
    ParsedNamingURL parsed = parseURL(name); // 私有内部类，只是简单包装了主机IP，端口等信息
    Registry registry = getRegistry(parsed);

    if (obj == null)
        throw new NullPointerException("cannot bind to null");

    registry.bind(parsed.name, obj);
}

```



```java
// java.rmi.Naming#getRegistry()
private static Registry getRegistry(ParsedNamingURL parsed) throws RemoteException
{
    return LocateRegistry.getRegistry(parsed.host, parsed.port);
}

```



```java
// java.rmi.registry.LocateRegistry#getRegistry()
public static Registry getRegistry(String host, int port)  throws RemoteException
{
    return getRegistry(host, port, null);
}

```



```java
// java.rmi.registry.LocateRegistry#getRegistry()
public static Registry getRegistry(String host, int port, RMIClientSocketFactory csf)
    throws RemoteException
{
    Registry registry = null;

    if (port <= 0)  port = Registry.REGISTRY_PORT;

    if (host == null || host.length() == 0) {
        // If host is blank (as returned by "file:" URL in 1.0.2 used in
        // java.rmi.Naming), try to convert to real local host name so
        // that the RegistryImpl's checkAccess will not fail.
        try {
            host = java.net.InetAddress.getLocalHost().getHostAddress();
        } catch (Exception e) {
            // If that failed, at least try "" (localhost) anyway...
            host = "";
        }
    }

    /*
     * Create a proxy for the registry with the given host, port, and
     * client socket factory.  If the supplied client socket factory is
     * null, then the ref type is a UnicastRef, otherwise the ref type
     * is a UnicastRef2.  If the property
     * java.rmi.server.ignoreStubClasses is true, then the proxy
     * returned is an instance of a dynamic proxy class that implements
     * the Registry interface; otherwise the proxy returned is an
     * instance of the pregenerated stub class for RegistryImpl.
     **/
    LiveRef liveRef =
        new LiveRef(new ObjID(ObjID.REGISTRY_ID),
                    new TCPEndpoint(host, port, csf, null),
                    false);
    RemoteRef ref =
        (csf == null) ? new UnicastRef(liveRef) : new UnicastRef2(liveRef);

	// create RegistryImpl_Stub
    return (Registry) Util.createProxy(RegistryImpl.class, ref, false);
}

```



RegistryImpl_Stub implements Registry

```java
// sun.rmi.registry.RegistryImpl_Stub#bind()
public void bind(String var1, Remote var2) 
    throws AccessException, AlreadyBoundException, RemoteException {
    try {
        // sun.rmi.server.UnicastRef#newCall(0)
        RemoteCall var3 = this.ref.newCall(this, operations, 0, 4905912898345647071L);

        try {
            ObjectOutput var4 = var3.getOutputStream(); // output stream
            var4.writeObject(var1);
            var4.writeObject(var2);
        } catch (IOException var5) {
            throw new MarshalException("error marshalling arguments", var5);
        }

        this.ref.invoke(var3); // ?
        this.ref.done(var3);
    } catch (RuntimeException var6) {
       // code ignored ...
    }
}

```





sun.rmi.registry.RegistryImpl#bind()

```java
// sun.rmi.registry.RegistryImpl#bind()
public void bind(String var1, Remote var2) 
	throws RemoteException, AlreadyBoundException, AccessException {
    Hashtable var3 = this.bindings;
    synchronized(this.bindings) {
        Remote var4 = (Remote)this.bindings.get(var1);
        if (var4 != null) { // already bound
            throw new AlreadyBoundException(var1);
        } else {
            this.bindings.put(var1, var2); // put<String,Remote>()
        }
    }
}

```



### 2、远程引用层

java.rmi.Naming#lookup()

```java
// java.rmi.Naming#lookup()
public static Remote lookup(String name)
    throws NotBoundException, java.net.MalformedURLException, RemoteException
{
    ParsedNamingURL parsed = parseURL(name);
    Registry registry = getRegistry(parsed); // RegistryImpl_Stub

    if (parsed.name == null)
        return registry;
    return registry.lookup(parsed.name); // called java.rmi.Registry#lookup()
}

```



sun.rmi.registry.RegistryImpl implements java.rmi.Registry

sun.rmi.registry.RegistryImpl_Stub implements java.rmi.Registry

sun.rmi.registry.RegistryImpl.lookup()

```java
// sun.rmi.registry.RegistryImpl.lookup()
public Remote lookup(String var1) throws RemoteException, NotBoundException {
    Hashtable var2 = this.bindings;
    synchronized(this.bindings) {
        Remote var3 = (Remote)this.bindings.get(var1);
        if (var3 == null) {
            throw new NotBoundException(var1);
        } else {
            return var3;
        }
    }
}

```



```java
// sun.rmi.registry.RegistryImpl_Stub#lookup()
public Remote lookup(String var1) throws AccessException, NotBoundException, RemoteException {
    try {
        // newCall
        RemoteCall var2 = this.ref.newCall(this, operations, 2, 4905912898345647071L);

        try {
            ObjectOutput var3 = var2.getOutputStream();
            var3.writeObject(var1);
        } catch (IOException var17) {
            throw new MarshalException("error marshalling arguments", var17);
        }

        this.ref.invoke(var2); //  transient protected RemoteRef ref;

        Remote var22;
        try {
            ObjectInput var4 = var2.getInputStream();
            var22 = (Remote)var4.readObject();
        } catch (IOException var14) {
           // code ignored...
        } finally {
            this.ref.done(var2);
        }

        return var22;
    } catch (RuntimeException var18) {
		// code ignored...
    }
}

```



sun.rmi.server.UnicastRef#invoke()

```java
// sun.rmi.server.UnicastRef#invoke()
public void invoke(RemoteCall var1) throws Exception {
    try {
        clientRefLog.log(Log.VERBOSE, "execute call");
        var1.executeCall(); // sun.rmi.server.Remote#executeCall()
    } catch (RemoteException var3) {
      // code ignored...
    }
}

```



sun.rmi.transport.StreamRemoteCall implements sun.rmi.server.RemoteCall

```java
public void executeCall() throws Exception {
    DGCAckHandler var2 = null;

    byte var1;
    try {
        if (this.out != null) {
            var2 = this.out.getDGCAckHandler();
        }

        this.releaseOutputStream();
        DataInputStream var3 = new DataInputStream(this.conn.getInputStream());
        byte var4 = var3.readByte();
        if (var4 != 81) { // strange value
            if (Transport.transportLog.isLoggable(Log.BRIEF)) {
                Transport.transportLog.log(Log.BRIEF, "transport return code invalid: " + var4);
            }

            throw new UnmarshalException("Transport return code invalid");
        }

        this.getInputStream();
        var1 = this.in.readByte();
        this.in.readID();
    } catch (UnmarshalException var11) {
       // code ignored...
    } finally {
        if (var2 != null) {
            var2.release();
        }

    }

    switch(var1) {
    case 1:
        return;
    case 2:
        Object var14;
        try {
            var14 = this.in.readObject();
        } catch (Exception var10) {
            throw new UnmarshalException("Error unmarshaling return", var10);
        }

        if (!(var14 instanceof Exception)) {
            throw new UnmarshalException("Return type not Exception");
        } else {
            this.exceptionReceivedFromServer((Exception)var14);
        }
    default:
            throw new UnmarshalException("Return code invalid");
    }
}

```







## 三、类图（UML）



## 四、时序图

## 五、小结



## 六、Create a RPC framework