##All About BroadCastReceiver
####编写广播接收器
继承BroadCastReceiver类，覆写其中的onReceive()方法

	class MyBroadcastReceiver extends BroadcastReceiver {  
     //接收到广播会被自动调用    
    @Override  
    public void onReceive (Context context, Intent intent) {  
        //从Intent中获取action  
        …your code here…  
    	}  
	}  

然后向系统注册，让系统知道这个接收器可以处理哪些消息

* 静态注册
* 动态注册

常见方式是采用**静态注册**，在application标签中添加receiver标签

	<application>  
    	<activity name=""/>  
    	<receiver android:name=".MyBroadcastReceiver">  
        	<!-- intent过滤器,指定可以匹配哪些intent, 一般需要定义action 可以是自定义的也可是系统的 -->   
        	<intent-filter>  
            	<action android:name="com.app.bc.test"/>  
        	</intent-filter>  
    	</receiver>  
	</application> 
	
此时，发送一个广播出去，代码如下：
	
	Intent intent = new Intent(“com.app.bc.test”);  
	sendBroadcast(intent);//发送广播事件 
	
在Activity中，我们可以用代码实现动态注册：

	//生成一个BroadcastReceiver对象  
	SMSReceiver  smsReceiver = new SMSReceiver();  
	//生成一个IntentFilter对象  
	IntentFilter filter = new IntentFilter();         
	filter.addAction(“android.provider.Telephony.SMS_RECEIVED”);  
	//将BroadcastReceiver对象注册到系统当中  
	//此处表示该接收器会处理短信事件  
	TestBC1Activity.this.registerReceiver(smsReceiver, filter); 
	
####静态注册和动态注册的区别

* 静态注册：在AndroidManifest.xml注册，android不能自动销毁广播接收器，也就是说当应用程序关闭后，还是会接收广播
* 动态注册：在代码中通过registerReceiver()手工注册。当程序关闭时，该接收器也会随之销毁。当然，也可手工调用unregisterReceiver()进行销毁


####BroadCastReceiver常见概念
广播接收器和时间处理机制类似，但是事件处理机制是程序组件级别的（比如，按钮的点击时间），而广播事件处理机制是系统级别的。我们可以用Intent来启动一个组件，可以用sendBroadcast()方法发起一个系统级别的事件广播来传递消息。我们同样可以在自己的应用程序中实现BroadcastReceiver来监听和响应广播的Intent。

事件的广播通过创建Intent对象并调用sendBroadcast()方法将广播发出。事件的接收是通过定义一个继承BroadcastReceiver的类来实现的，继承该类后覆盖onReceive,在该方法中设置响应时间~
下面是android系统中定义了很多标准的Broadcast Action来响应系统的广播事件

* ACTION_TIME_CHANGED	(时间改变时触发)
* ACTION_BOOT_COMPLETED	（系统启动完成后触发），有些程序开机启动就是用这种方式实现的
* ACTION_PACKAGE_ADDED （添加包时触发）
* ACTION_BATTERY_CHANGED （电量低时触发）

静态注册中，intent-filter标签中的priority是设置广播接收器的优先级。当priority为Integer的最大值时，才是优先级最高。仅限于静态注册









