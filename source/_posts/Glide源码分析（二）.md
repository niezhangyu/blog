---
title: Glide源码分析（二）
date: 2017/6/28
---

	Glide
    	.with(this)
    	.load(url)
    	.placeholder(R.drawable.loading_spinner)
    	.into(myImageView);
　　上文我们跟随代码走过了Glide.with(this)方法得到了RequestManager对象
## RequestManager
　　RequestManager类的作用是Glide的请求管理类，可以根据Activity，Fragment的生命周期来智能管理请求。

　　那么，它是如何根据这个生命周期来只能管理请求的呢？我们可以看到RequestManager类实现了LifecycleListener接口
    
    public interface LifecycleListener {
  		/**
   			* Callback for when {@link android.app.Fragment#onStart()}} or {@link
   			* android.app.Activity#onStart()} is called.
   			*/
  			void onStart();

  		/**
   			* Callback for when {@link android.app.Fragment#onStop()}} or {@link
   			* android.app.Activity#onStop()}} is called.
   			*/
  			void onStop();

  		/**
   			* Callback for when {@link android.app.Fragment#onDestroy()}} or {@link
   			* android.app.Activity#onDestroy()} is called.
   			*/
  			void onDestroy();
	}
　　onStart(),onStop()和onDestroy()，很明显正贴切我们Activity和Fragment的生命周期。我们要问的是，那是如何和我们应用中的Activity和Fragment绑定的呢？关于这一点，我们还得返回去撸Glide.with(this)里面的逻辑。

## Glide.with()内部调用的get()方法
　　前文说到，我们通过Glide.with()方法里的getRetriever()获得了RequestManagerRetriever对象，然后调用它的get()方法获得了RequestManager。那我们，就看一下这个get()方法。

　　get()方法对应前文提到的with()的六种重载方法，也有六种。

### get(Context context)
	public RequestManager get(Context context) {
    	if (context == null) {
      		throw new IllegalArgumentException("You cannot start a load on a null Context");
    	} else if (Util.isOnMainThread() && !(context instanceof Application)) {
      		if (context instanceof FragmentActivity) {
        		return get((FragmentActivity) context);
      	  } else if (context instanceof Activity) {
        		return get((Activity) context);
      	  } else if (context instanceof ContextWrapper) {
        	    return get(((ContextWrapper) context).getBaseContext());
      	  }
       }
    	return getApplicationManager(context);
 	 }
　　第一种方法，如果传入的context为其余的Activity等，会调用他们的get()方法，所以我们要看当context为ApplicationContext或在后台运行时调用的getApplicationManager(context)。
    
	  private RequestManager getApplicationManager(Context context) {
    	if (applicationManager == null) {
      		synchronized (this) {
        		if (applicationManager == null) {
          			Glide glide = Glide.get(context);
          			applicationManager =
              		factory.build(glide, new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
        		}
      		}
    	}
    		return applicationManager;
  	  }
　　如果看了那么多代码后，你还能记得本文开头的时候所说，RequestManager的作用是根据Activity和Fragment生命周期智能管理请求。但是这个Application是不能接受到Activity和Fragment生命周期的，所以Glide在这里用Application的生命周期来管理。也因此，我认为如果大家使用Glide如果想更大化的利用他的机制，还是不用ApplicationContext更好。

### get(Activity activity)
	//第二种方法，针对Activity
	  public RequestManager get(Activity activity) {
		//判断当前是否后台运行，如果是则使用调用ApplicationContext的方法	
    	if (Util.isOnBackgroundThread()) {
      		return get(activity.getApplicationContext());
    	} else {
			//判断Activity是否销毁
      		assertNotDestroyed(activity);
      		android.app.FragmentManager fm = activity.getFragmentManager();
      		return fragmentGet(activity, fm, null /*parentHint*/);
    	}
  	  }
　　以上的代码可以看到获取了FragmentManager后调用了fragmentGet()方法   

      private RequestManager fragmentGet(Context context,android.app.FragmentManager fm,android.app.Fragment parentHint) {
    		RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);
    		RequestManager requestManager = current.getRequestManager();
    		if (requestManager == null) {
      		// TODO(b/27524013): Factor out this Glide.get() call.
      			Glide glide = Glide.get(context);
      			requestManager =
          			factory.build(glide, current.getLifecycle(), current.getRequestManagerTreeNode());
      			current.setRequestManager(requestManager);
    		}
    	return requestManager;
  	　}
　　通过getRequestManagerFragment()方法获得一个空白的fragment。在这个方法里，如果本地管理的缓存里没有Fragment才会去新建并将其添加进缓存中，并删除掉先前的FragmentManager以确保同一个Activity或父Fragment中只会创建一个Fragment。获得这个空白Fragment后如果是新建的Fragment没有RequestManager会创建出一个RequestManager并和Fragment绑定，如果不是新建的Fragment就直接返回这个Fragment的RequestManager。  	

### 其余的get()方法
　　其余的四种get()方法，和以上两种没有什么不同，最终都是获取Fragment的RequestManager。
## 总结
　　综上所述，说到这里，我们已经可以看出来，Glide对于生命周期的管理是依托在给当前Activity和Fragment再添加一个新的空白的Fragment来实现的，通过这个空白的Fragment生命周期的变化回调来绑定Glide的RequestManager对请求智能管理。