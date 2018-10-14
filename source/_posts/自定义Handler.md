---
title: 自定义Handler
date: 2017/12/6
---

　　说是自定义Handlr其实吹流弊了，因为还是按照Android原来的Handler写的。为什么要自己再来一遍呢？只是为了自己加深理解，脑子笨那就敲一遍呗。当然，这里剩下的都是Handler的核心代码，也可以练习练习用作面试时候别人问你能不能手撸一套Handler哦~注释就不写了，这是非常简单的版本了。

## Handler类
	
	package userdefindedhandler;

	public class Handler {
	
		final Looper mLooper;
		final MessageQueue mQueue;
	
		public Handler(){
		
			mLooper = Looper.myLooper();
		
			if(mLooper == null){
				throw new RuntimeException(
					"Can't create handler inside thread that has not called Looper.prepare()");
			}
		
			mQueue = mLooper.mQueue;
		}
	
		public void sendMessage(Message msg){
			sendMessage(msg,0);
		}
	
		public void sendMessage(Message msg,long uptimeMillis){
			MessageQueue queue = mQueue;
		
			if(queue == null){
				RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
			}
		
			msg.target = this;
		
			queue.enqueueMessage(msg,uptimeMillis);
		
		
		}
	
	/**
     * Subclasses must implement this to receive messages.
     */
    	public void handleMessage(Message msg) {
    	}
    
    /**
     * Handle system messages here.
     */
    	public void dispatchMessage(Message msg) {
    		handleMessage(msg);
    	}
    }

## Looper类

	package userdefindedhandler;

	public class Looper {
	
		final MessageQueue mQueue;
	
		static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>(); 
	
	
		private Looper(){
			mQueue = new MessageQueue();
		}
	
		public static Looper myLooper(){
			return sThreadLocal.get();
		}
	
		public static void prepare(){
		 	if (sThreadLocal.get() != null) {
	            throw new RuntimeException(
	            		"Only one Looper may be created per thread");
	        	}
	        	sThreadLocal.set(new Looper());
		}
	
		public static void loop(){
			final Looper me = myLooper();
		
			if(me == null){
				throw new RuntimeException(
					"No Looper; Looper.prepare() wasn't called on this thread.");
			}
		
			final MessageQueue queue = me.mQueue;
		
			for(;;){
				Message msg = queue.next();
				if(msg == null){
					return;
				}
				msg.target.dispatchMessage(msg);
			}
		}

	}

## MessageQueue类
	package userdefindedhandler;

	public class MessageQueue {
	
	Message mMessages;
	
	/**
	 * MessageQueue
	 * @param msg
	 * @param when
	 */
		void enqueueMessage(Message msg, long when){
			if(msg.target == null){
				throw new IllegalArgumentException("Message must have a target.");
			}
		
			synchronized(this){
			
				msg.when = when;
				Message p = mMessages;
			
				if(p == null || when == 0 || when <p.when){
				
					msg.next = p;
					mMessages = msg;
				
				}else{
					Message prev;
				
					for(;;){
						prev = p;
						p = p.next;
					
						if (p == null || when < p.when) {
                        	break;
                    	}
					}
				
					msg.next = p;
					prev.next = msg;
				}
			}
		}
	
		Message next(){
	
			for(;;){
			
				synchronized(this){
					Message prevMsg = null;
					Message msg = mMessages;
				
					if(msg != null){
						if (prevMsg != null) {
                        	prevMsg.next = msg.next;
                    	} else {
                        	mMessages = msg.next;
                    	}
                    	msg.next = null;
                    
                    	return msg;
					}
				}
			}
		}

	}

## Message类
	package userdefindedhandler;

	public final class Message {
	
		Handler target;
	
		long when;
	
		Message next;
	
		int what;
	
		Object obj;
	
		@Override
		public String toString() {
			// TODO Auto-generated method stub
			return obj.toString();
		}
	}
## Main函数
package userdefindedhandler;

import java.util.UUID;

public class MainTestDemo {
	
	public static void main(String[] args) {
		
		Looper.prepare();
		
		final Handler handler = new Handler(){
			public void handleMessage(Message msg) {
				System.out.println(Thread.currentThread().getName() + "--receiver--" + msg.toString());
				
		    }
		};
		
		
		 for (int i = 0; i < 10; i++) {
	            new Thread(new Runnable() {
	                public void run() {
	                    while (true) {
	                        Message msg = new Message();
	                        msg.what = 0;
	                        synchronized (UUID.class) {
	                            msg.obj = Thread.currentThread().getName()+"--send---"+UUID.randomUUID().toString();
	                        }
	                        System.out.println(msg);
	                        handler.sendMessage(msg);
	                        try {
	                            Thread.sleep(1000);
	                        } catch (InterruptedException e) {
	                            e.printStackTrace();
	                        }
	                    }
	                }
	            }).start();
	        }
	        //开始消息循环
	        Looper.loop();
	    }
	}