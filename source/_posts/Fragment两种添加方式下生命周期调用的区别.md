---
title: Fragment两种添加方式下生命周期调用的区别
date: 2017/6/20 
---
 
  ## 1.在replace的情况下
	 
  FragmentA显示，正常调用生命周期运行如下：	

  ![](http://img.blog.csdn.net/20160926231337197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)	 
  
  点击按钮2，切换到FragmentB,Fragment A 和 B  生命周期运行如下：

  ![](http://img.blog.csdn.net/20160926231652425?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)	

  点击按钮1，再次切换回FragmentA，Fragment A 和 B 生命周期运行如下： 

  ![](http://img.blog.csdn.net/20160926231839067?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)  
  
  **总结：repalce情况下，切换Fragment时候，原Fragment会完全销毁，再次显示的时候会重新创建**

  ## 2.在add的情况下
  
  FragmentA显示，正常调用生命周期如下：
  
  ![](http://img.blog.csdn.net/20160926231337197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  点击按钮2，切换到Fragment B,Fragment A 和 B 的生命周期如下：
  
  ![](http://img.blog.csdn.net/20160926232511026?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  此时，由于仅仅是add了第二个Fragment 所以Fragment B 和Fragment A 重影

  ## 3.在add和hide/show的情况下
	
  FragmentA显示，正常调用生命周期如下：

  ![](http://img.blog.csdn.net/20160926231337197?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  点击按钮2，切换到Fragment B, Fragment A 和 B 的生命周期如下：

  ![](http://img.blog.csdn.net/20160926234428188?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  点击按钮1，切换到Fragment A,Fragment A 和 B 的生命周期如下（此时 A 和 B 重影）：
  
  ![](http://img.blog.csdn.net/20160926235036184?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  **总结：在add和hide/show的情况下，切换Fragment A不会调用任何生命周期,只因为Fragment A 的hide 和 show 调用了 onHiddenChanged方法，所有View都一直保存在内存中。**