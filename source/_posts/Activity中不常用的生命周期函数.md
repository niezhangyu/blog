---
title: Activity中不常用的生命周期函数
date: 2017/7/2 
---

## 前提

　　昨晚在掘金上看到一篇文章[https://juejin.im/post/5948dcd761ff4b006c062ac6](https://juejin.im/post/5948dcd761ff4b006c062ac6 "超详细的生命周期图-你能回答全吗") 觉得很不错，Activity是我们开发中躲不过的四大控件之一。但其中很多方法，也许我们一般用不到的，但看了这篇文章觉得也应该了解一下。


## onNewIntent(Intent intent)

　　网上很多博客说Activity设置为"SingleTask"，在第一启动的时候执行onCreate()---->onStart()---->onResume()等后续生命周期函数，也就时说第一次启动Activity并不会执行到onNewIntent()，而后面如果再启动Activity的时候，那就是执行onNewIntent()---->onResart()------>onStart()----->onResume()。

　　但是根据官方文档的解释来看，**在Activity设置为"SingleTop"启动模式后，当这个Activity再次启动的时候，Activity栈顶的Activity将会重新启动而不会重新创建，onNewIntent()也将会被调用。Activity在被新的Intent启动之前会暂停onPause()，onIntent()将会在onResume()之前被调用。**所以，我对网上很多博客所说表示怀疑，启动模式究竟是"SingleTask"还是"SingleTop"，onRestart()方法到底是否会调用，onRestart()方法如果被调用那对应的还有是否调用了onStop()呢？

　　我重新写了一个简单的demo，代码就不贴了仅仅是将Activity设置为"SingleTop"，启动后点击图标再次启动这个Activity。

　　代码执行结果我发现在重新启动目标Activity后，并没有执行onStop()方法，同理再次onRestart()也没有执行，第一次启动的时候执行onCreate()---->onStart()---->onResume()，点击图标再次启动调用onPause()---->ononNewIntent()---->onResume()。所以，网上千篇一律的博客貌似不对啊。

　　再次对启动模式进行修改，我发现除了默认的standard模式，其余的模式其实都可以调用onNewIntent()方法。

　　官方文档的解释还有一条就是**关于onNewIntent()方法的参数Intent，当我们第二次启动目标Activity时，传递的参数Intent就是新的Intent，但如果我们去getIntent()得到的却是第一次启动的Intent。所以，如果我们想getIntent()得到第二次新的Intent需要我们在onNewIntent()方法中将参数setIntent(Intent)来更新Intent**。说的可能有一点点绕，所以这里稍微贴一点点代码。

	//第一次启动传的参数
	it.putExtra("INTENT","第一个Intent");
	
	//第二次启动传的参数
	it.putExtra("INTENT","第二个Intent");

	//在onNewIntent()方法中
	 @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        
		String str=intent.getStringExtra("INTENT");
        String oldstr=getIntent().getStringExtra("INTENT");
        Log.e("ActivityLife","SecondActivity:  onNewIntent++"+str+"   old:::"+oldstr);
    }

	//输出结果：
	E/ActivityLife: SecondActivity:  onNewIntent++第二个Intent   old:::第一个Intent

　　综上所述，启动模式只要不是standard模式，如果目标Activity没有被销毁onDestroy()，第二次启动目标Activity都会在onResume()之前调用onNewIntent()，如果销毁了就只能老老实实onCreate()。当调用onNewIntent()时，要注意getIntent()得到的会是第一个Intent，所以要在方法里重新setIntent(Intent)更新Intent。
	
## onUserInteraction和onUserLeaveHint
　　这两个方法是对立方法，主要用于Activity处理触摸等事件。

　　实现onUserInteraction()方法可以让你知道当Activity正在运行时，用户是否正在和设备交互。这个回调和onUserLeaveHint()的目的是帮助Activity智能的管理状态栏的通知；尤其是，帮助Activities在合适的时间取消通知。调用onUserLeaveHint()的时候都会同时调用onUserInteraction()，这样确保Activity可以得知例如用户拉下通知面板并点击了通知消息等信息。需要注意的是，onUserInteraction()被touch down的动作所调用，可能不会被touch-moved和touch-up调用。

　　而onUserLeaveHint()方法在用户点击Home键使Activity进入后台时候被调用。但是如果是一个突发的电话事件，onUserLeaveHint()方法不会被调用。onUserLeaveHint()方法会在onPause()之前被调用。 


## onPostCreate和onPostResume

　　这两个方法，我个人认为掘金博客说的测量控件高度并不准确。根据官方文档的解释而言，应用不需要实现这两个方法，他们是用来系统代码确认Activity是否初始化等。

　　对于应用开发工程师只要知道onPostCreate方法在onStart()和onRestoreInstanceState(Bundle)之后调用，即可，onPostResume方法在onResume()方法之后调用即可，至于安卓底层开发是否需要还要进一步了解。

　　Google对这两个方法的使用仅见于以下Navigation Drawer的代码
    
	@Override
	protected void onPostCreate(Bundle savedInstanceState) {
    	super.onPostCreate(savedInstanceState);

    	// Sync the toggle state after onRestoreInstanceState has occurred.
    	mDrawerToggle.syncState();
	}	

