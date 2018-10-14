---
title: Glide源码分析（一）
date: 2017/6/27
---

    Glide
    	.with(this)
    	.load(url)
    	.placeholder(R.drawable.loading_spinner)
    	.into(myImageView);
   
　　众所周知，以上是Glide最简单的使用方法。对于Glide，我们先简单的跟随代码走一遍，看看他如何去将图片加载。由于本文不是我完全阅读完代码总结而写，而是边看别做笔记，所以顺序或许较为杂乱，仅作为我的学习笔记。

## Glide.with(this)获得RequestManager对象
Glide.with()有六种重载的静态方法：

		public static RequestManager with(Context context) {
    		return getRetriever(context).get(context);
  	  	}
	  
	  	public static RequestManager with(Activity activity) {
    		return getRetriever(activity).get(activity);
  	  	}

	  	public static RequestManager with(FragmentActivity activity) {
    		return getRetriever(activity).get(activity);
  	  	}

	  	public static RequestManager with(android.app.Fragment fragment) {
    	    return getRetriever(fragment.getActivity()).get(fragment);
      	}

      	public static RequestManager with(Fragment fragment) {
    	    return getRetriever(fragment.getActivity()).get(fragment);
      	}

	  	public static RequestManager with(View view) {
    	    return getRetriever(view.getContext()).get(view);
      	}
　　需要注意第六种重载方法，参数View必须是Fragment和Activity包含的View，如果这个View没有attached即没有context，这个方法将会调用失败，所以官方更推荐非特殊情况不要使用这个方法。

　　所有方法都是先调用了getRetriever(),获得RequestManagerRetriever对象，再get()得到RequestManager对象


### getRetriever()获得RequestManagerRetriever对象

    	private static RequestManagerRetriever  getRetriever{

		Preconditions.checkNotNull(
        "You cannot start a load on a not yet attached View or a  Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
      
		  return Glide.get(context).getRequestManagerRetriever();
  		}

　　Preconditions.checkNotNull()只是检测context是否为空，Preconditions类的使用值得我学习，大量的判空代码太丑陋。

　　我们可以看到getRetriever()方法最终调用的Glide.get(context)再调用getRequestManagerRetriever()来获得RequestManagerRetriever
#### get(Context context)获得Glide对象 
　　Glide的get方法非常眼熟，单例模式嘛。当Glide对象未创建的时候，调用initGlide(context)方法创建Glide对象。

	 	 public static Glide get(Context context) {
    		if (glide == null) {
      			synchronized (Glide.class) {
        	if (glide == null) {
          		initGlide(context);
        		  }
      			}
    		 }
    		return glide;
		 }	
##### initGlide(context)初始化Glide
	
　　这个方法有将近50行代码，就不贴全部代码了。

　　通过反射获取GeneratedAppGlideModule对象，然后获得RequestManagerRetriever.RequestManagerFactory对象等。这之前有很多涉及集合对象等操作，目前没有看懂作用，还要后续研究。

　　获得RequestManagerRetriever.RequestManagerFactory对象后，传入GlideBuildr类中，此类采用了builder模式来创建Glide真正的对象。

    GlideBuilder builder = new GlideBuilder()
        .setRequestManagerFactory(factory);
    	...
    glide = builder.build(applicationContext);

　　builder.build()方法里

      public Glide build(Context context) {
    	...
		
		if (engine == null) {
      		engine = new Engine(memoryCache, diskCacheFactory, diskCacheExecutor, sourceExecutor,
          	GlideExecutor.newUnlimitedSourceExecutor());
    	}
    	RequestManagerRetriever requestManagerRetriever = new RequestManagerRetriever(
        requestManagerFactory);

    	return new Glide(
        	context,
        	engine,
        	memoryCache,
        	bitmapPool,
        	arrayPool,
        	requestManagerRetriever,
        	connectivityMonitorFactory,
        	logLevel,
        	defaultRequestOptions.lock());
  		}

　　此时，真正的调用了Glide的构造方法

　　其中Engine类是一个任务创建，发起，回调，管理存活和缓存的资源的类，以上代码也可看出内存和本地缓存对象等传入Engine。以后会专门分析下这个类。

　　到现在为止，我们拿到了真正的Glide对象。并且在build（）方法里，创建了RequestManagerRetriever对象，从而在上面getRetriever()方法里继续调用getRequestManagerRetriever()得到了RequestManagerRetriever对象，从而可以获取关键的RequestManager。

