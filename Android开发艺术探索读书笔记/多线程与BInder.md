# Binder
* Binder是Android中的一个类,它实现了IBinder接口.从IPC角度来说Binder是Android中的一种跨进程通信方式,Binder还可以理解为一种虚拟的物理设备,它的设备驱动是/dev/binder,该通信在Linux中没有;从Android FrameWork的角度来说Binder是ServiceManger连接各种Manager(ActivityManager,WindowManager,等等)和相应的ManagerService的桥梁;从Android应用层来说,Binder是客户端和服务端进行通信的媒介,当bindservice的时候,服务端会返回一个包含了服务端业务调用的Binder对象,通过这个Binder对象,客户端就可以获取服务端提供的服务或者数据,这里的服务包括普通服务和基于AIDL的服务.

* AIDL分析Binder机制:

```//Book.java
public class Book implements Parcelable{
	public int bookId;
	public String bookName;
	
	public Book(int bookId,String bookName){
		this.bookId = bookId;
		this.bookName = bookName;
	}
	
	public int describeContents(){
		return 0;
	}
	
	public void writeToParcel(Parcel out, int flags){
		out.writeInt(bookId);
		out.writeString(bookName);
	}
	
	public static final Parcelable.Creator<Book> 	CREATOR = new Parcelable.Creator<Book>(){
		public Book createFromParcel(Parcel in){
			return new Book(in);
		}
		
		public Book[] new Array(int size){
			return new Book[size];
		}
	};
	
	private Book(Parcel in){
		bookId = in.readInt();
		bookName = in.readString();
	}
}
```

```//Book.aidl
parcelable Book;
```

```//IBookManager.aidl
interface IBookManager{
	List<Book> getBookList();
	void addBook(in Book book);
}
```

onTransact
这个方法运行在服务端中的Binder线程池中,当客户端发起跨进程请求时,远程请求会通过系统底层封装后交由此方法来处理.该方法的原型为
`public Boolean onTransact(int code,android.os.Parcel data,android.os.Parcel reply,int flags)`服务端通过code可以确定客户端锁请求的目标方法是什么,接着从data中取出目标方法所需的参数(如果目标方法中有参数的话),然后执行目标方法,当目标方法执行完毕后,就向reply中写入返回值,如果onTransact方法返回false,那么客户端的请求会失败,可以利用这个特性来做权限验证,毕竟不希望每一个进程都能调用我们的服务.

Proxy#getBookList
这个方法运行在客户端,当客户端远程调用此方法时,首先创建该方法所需要的输入性Parcel对象_data,输出型Parcel对象_reply和返回值对象List,然后把该方法的参数信息写入_data中,接着调用transact方法来发起RPC(远程调用)请求,同时当前线程挂起;然后服务端的onTransact方法会被调用,知道RPC过程返回后,当前线程继续执行,并从_reply中取出RPC过程的返回结果;最后返回_reply中的数据

![Binder](http://oidd437cq.bkt.clouddn.com/image/notesBinder%20%281%29.png)

* IPC方式
例:A进程正在进行一个计算，计算完成后它要启动B进程的一个组件把计算结果传递给B进程,可是遗憾的是这个计算结果并不支持放入Bundle中,因此无法通过Intent来传输,这个时候如果用其它IPC方式又略显复杂,可以考虑如下方式:通过Intent启动B进程的一个Service组件(比如IntentService),让Service在后台进行计算,计算完毕后再启动B进程真正要启动的目标组件,由于Service也在B进程中,所以目标组件就可以直接获取计算结果,这样一来就解决了跨进程问题,这种方式的核心思想在于将原本需要在A进程的计算任务转移到B进程的后台Service中去执行,这样就成功避免了进程间通信问题,而且只用了很小的代价.

```
1.使用文件共享
两个进程通过读/写同一个文件来交换数据  适合在对数据同步要求不高的进程之间进行通信,并且要妥善处理并发读/写的问题

2.使用Messenger
Messenger译为信使,通过它可以在不同进程中传递Message对象,在Message中传递数据就可以实现进程间的数据传递了,Messenger是一种轻量级的IPC方案,它的底层实现是AIDL.
实现一个Messenger有如下几部:
服务端进程:
首先,我们需要在服务端创建一个Service来处理客户端的连接请求,同事创建一个Handler并通过它来创建一个Messenger对象,然后在Service的onBind中返回这个Messenger对象底层的Binder即可.
```





 


