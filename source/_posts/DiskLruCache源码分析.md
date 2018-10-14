---
title: DiskLruCache源码分析
---
## 前提
　　前文分析Glide的时候，说到了DiskLruCache磁盘缓存方案。缓存对于提升一个App的性能至关重要，Android官方推荐使用LruCache+DiskLruCache对图片进行两级缓存。DiskLruCache的作者为JakeWharton大神，本文将对其进行学习分析。

## 基本概念
　　DiskLruCache以key-value的形式在磁盘上缓存，支持一对多，线程安全，不支持多进程同时访问一个缓存目录。一般App在SD卡的缓存目录在/sdcard/Android/data/<your packagename>/cache/。在生成磁盘缓存的时候还会生成一个以journal命名的记录所有对缓存操作的文件。数据文件命名以key.index方式存储在缓存目录，index指key-value对应关系中一对多的情况value的下标,如果key-value是一对一的情况，那么一般数据文件名称以key.0的方式存储。如下图所示为一对一的情况：

![](http://orvoldspw.bkt.clouddn.com/DiskLruCache1.png)

　　Journal文件，如下图所示：

![](http://orvoldspw.bkt.clouddn.com/DiskLruCache2.png)

　　文件前五行组成了一个完整的Journal头　

1、第一行是一个字符串常量libcore.io.DiskLruCache，标识版权信息

2、第二行代表DiskLruCache缓存的版本号1，由我们设置

3、第三行代表App的版本号也就是versionCode为1，由我们设置

4、第三行代表key-value是1对N的关系，此例中N=1

5、第五行是空行，与日志内容分隔

　　日志内容一行代表一条操作记录，格式为status key value[0].length ... value[N-1].length,用空格分隔,status代表文件状态，如下

1、DIRTY 脏数据，代表正在对此条数据进行操作,每条DIRTY日志后面都应该是一条CLEAN或者REMOVE日志，否则将删除此key对应的文件

2、CLEAN 代表已经成功写入缓存的数据，可供读取

3、READ 代表读取一条数据

4、REMOVE 代表删除一条数据 （此处我有疑惑，因为我删除缓存并没有产生REMOVE的写入）

## 创建缓存对象
### open()
　　  DiskLruCache的构造方法为私有的，创建DiskLruCache通过静态方法open()方法创建。
	
	public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
      throws IOException {
    	if (maxSize <= 0) {
      		throw new IllegalArgumentException("maxSize <= 0");
    	}
    	if (valueCount <= 0) {
      		throw new IllegalArgumentException("valueCount <= 0");
    	}

    	//判断journal.bkp备份文件是否存在
    	File backupFile = new File(directory, JOURNAL_FILE_BACKUP);
    	if (backupFile.exists()) {
      		File journalFile = new File(directory, JOURNAL_FILE);
      		// 如果journal文件存在，则删除bkp备份文件
      		if (journalFile.exists()) {
        		backupFile.delete();
      		} else {
        		renameTo(backupFile, journalFile, false);
      		}
    	}

    	//创建DiskLruCache对象，参数分别为路径、app版本号、key-value关系和缓存大小
    	DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
		//如果journalFile存在则调用readJournal()和processJournal()对已经存在的缓存数据进行初始化返回DiskLruCache对象
   		if (cache.journalFile.exists()) {
      	try {
        	cache.readJournal();
        	cache.processJournal();
        	cache.journalWriter = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(cache.journalFile, true), Util.US_ASCII));
        	return cache;
      	} catch (IOException journalIsCorrupt) {
        	System.out
            	.println("DiskLruCache "
                + directory
                + " is corrupt: "
                + journalIsCorrupt.getMessage()
                + ", removing");
        	cache.delete();
      		}
    	}

    	//如果不存在journalFile文件，则创建并写入journal头，调用rebuildJournal()方法
    	directory.mkdirs();
    	cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
    	cache.rebuildJournal();
    	return cache;
  	}	

	/**
	 *DiskLruCache的私有构造方法
	 */	
	private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
    	this.directory = directory;
    	this.appVersion = appVersion;
    	this.journalFile = new File(directory, JOURNAL_FILE);
    	this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
    	this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
    	this.valueCount = valueCount;
    	this.maxSize = maxSize;
  	}
### rebuildJournal()
　　open()方法主要就是创建了DiskLruCache对象和初始化了journal文件。我们看下如果没有journal文件，调用的rebuildJournal()方法创建并写入journal头。

	  private synchronized void rebuildJournal() throws IOException {
    	if (journalWriter != null) {
      		journalWriter.close();
    	}

    	Writer writer = new BufferedWriter(
        	new OutputStreamWriter(new FileOutputStream(journalFileTmp), Util.US_ASCII));
    	try {
      		writer.write(MAGIC);
      		writer.write("\n");
      		writer.write(VERSION_1);
      		writer.write("\n");
      		writer.write(Integer.toString(appVersion));
      		writer.write("\n");
      		writer.write(Integer.toString(valueCount));
      		writer.write("\n");
      		writer.write("\n");

      		for (Entry entry : lruEntries.values()) {
        		if (entry.currentEditor != null) {
          			writer.write(DIRTY + ' ' + entry.key + '\n');
        		} else {
          			writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
        		}
      		  }
    		} finally {
                writer.close();
            }

    	if (journalFile.exists()) {
      		renameTo(journalFile, journalFileBackup, true);
    	}
    	renameTo(journalFileTmp, journalFile, false);
    	journalFileBackup.delete();

    	journalWriter = new BufferedWriter(
        	new OutputStreamWriter(new FileOutputStream(journalFile, true), Util.US_ASCII));
  	  }
### readJournal()
　　如果已经存在journal文件，则会调用readJournal()和processJournal()对已经存在的缓存数据进行初始化，如下：

	  private void readJournal() throws IOException {
    	StrictLineReader reader = new StrictLineReader(new FileInputStream(journalFile), Util.US_ASCII);
    	try {
			//读取journal日志文件的头信息
      		String magic = reader.readLine();
      		String version = reader.readLine();
      		String appVersionString = reader.readLine();
      		String valueCountString = reader.readLine();
      		String blank = reader.readLine();
      		if (!MAGIC.equals(magic)
          		|| !VERSION_1.equals(version)
          		|| !Integer.toString(appVersion).equals(appVersionString)
          		|| !Integer.toString(valueCount).equals(valueCountString)
          		|| !"".equals(blank)) {
        			throw new IOException("unexpected journal header: [" + magic + ", " + version + ", "
            			+ valueCountString + ", " + blank + "]");
      		}
		//读取journal日志文件信息
      	int lineCount = 0;
      	while (true) {
        	try {
				//按行读取每条日志信息
          		readJournalLine(reader.readLine());
          		lineCount++;
       		 } catch (EOFException endOfJournal) {
          		break;
        	 }
      	}
      	redundantOpCount = lineCount - lruEntries.size();
      } finally {
        Util.closeQuietly(reader);
      }
    }
#### readJournalLine()
　　我们在看readJournalLine()方法

	  private void readJournalLine(String line) throws IOException {
    	int firstSpace = line.indexOf(' ');
		//判断是否为有效的Journal操作行信息
    	if (firstSpace == -1) {
      		throw new IOException("unexpected journal line: " + line);
    	}

    	int keyBegin = firstSpace + 1;
    	int secondSpace = line.indexOf(' ', keyBegin);
    	final String key;
    	if (secondSpace == -1) {
      		key = line.substring(keyBegin);
			// 如果是REMOVE操作，移除内存中对应的失效缓存
      		if (firstSpace == REMOVE.length() && 	line.startsWith(REMOVE)) {
        		lruEntries.remove(key);
        		return;
      			}
    		} else {
      			key = line.substring(keyBegin, secondSpace);
    		}
		// 读取内存中的Entry，如果没有则创建新的Entry并添加到LinkHashMap中，lruEntries为LinkedHashMap<String, Entry>
    	Entry entry = lruEntries.get(key);
    	if (entry == null) {
      		entry = new Entry(key);
      		lruEntries.put(key, entry);
    	}
		//如果为Clean操作的情况
    	if (secondSpace != -1 && firstSpace == CLEAN.length() && line.startsWith(CLEAN)) {
      		String[] parts = line.substring(secondSpace + 1).split(" ");
      		entry.readable = true;
      		entry.currentEditor = null;
      		entry.setLengths(parts);
		//如果为Dirty数据，赋值给Editor待后续操作
    	} else if (secondSpace == -1 && firstSpace == DIRTY.length() && line.startsWith(DIRTY)) {
      		entry.currentEditor = new Editor(entry);
    	} else if (secondSpace == -1 && firstSpace == READ.length() && line.startsWith(READ)) {
      		// This work was already done by calling lruEntries.get().
    	} else {
      		throw new IOException("unexpected journal line: " + line);
    	}
  	}
### processJournal()
　　processJournal()方法对lruEntries中的数据进行加工，删除脏数据，计算可用缓存数据的总长度

	  private void processJournal() throws IOException {
    	deleteIfExists(journalFileTmp);
    	for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
      		Entry entry = i.next();
			//前文说到的Dirty数据，如果不为空删除脏数据
      		if (entry.currentEditor == null) {
        		for (int t = 0; t < valueCount; t++) {
          		size += entry.lengths[t];
        		}	
      		} else {
        		entry.currentEditor = null;
        		for (int t = 0; t < valueCount; t++) {
          			deleteIfExists(entry.getCleanFile(t));
         		    deleteIfExists(entry.getDirtyFile(t));
        		}
        		i.remove();
      		 }
   		   }
 		}
### LinkedHasLinkedHashMaphMap lruEntries和Entry entry 
　　lruEntries为一个全局的LinkedHashMap<String, Entry>，用来保存所有的缓存对象Entry，排序方式以访问频次为权重，从少到多。Entry的主要参数如下:	

    private final String key;

    //每个文件的长度
    private final long[] lengths;

    /** 曾经被发布过 那他的值就是true */
    private boolean readable;

    /** 正在被编辑的条目或者是没有被编辑的条目，即Dirty数据 */
    private Editor currentEditor;

    /** 最近提交的编辑条目. */
    private long sequenceNumber;

## 写入缓存
　　缓存的写入需要通过内部类Editor类,调用edit(String key)获取此key对应的Editor对象。代码浅显易懂，如下所示：

	  private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
		//检测journalWriter是否为空
    	checkNotClosed();
		//key值不能有[a-z0-9_-]{1,64}以外的值
    	validateKey(key);
    	Entry entry = lruEntries.get(key);
    	if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
        	|| entry.sequenceNumber != expectedSequenceNumber)) {
      		return null; // Snapshot is stale.
    	}
    	if (entry == null) {
      		entry = new Entry(key);
      		lruEntries.put(key, entry);
    	} else if (entry.currentEditor != null) {
      		return null; // Another edit is in progress.
    	}

    	Editor editor = new Editor(entry);
    	entry.currentEditor = editor;

    	// Flush the journal before creating files to prevent file leaks.
    	journalWriter.write(DIRTY + ' ' + key + '\n');
    	journalWriter.flush();
    	return editor;
  	}
　　	edit()方法获取Editor对象，Editor为内部类，主要方法如下：

	1.newInputStream()方法获取对应index的输入流，对应最后一次写入的数据，如果没有数据返回null

	2.newOutputStream()方法获取一个输出流，将需要缓存的数据写入此OutputStream。如果在写入的过程中需要错误，Editor将会中止，返回的OutputStream不会抛出IOException

	3.commit()提交此次操作

	4.abort()放弃此次操作

　　每次操作完成后，必须要调用commit()或者abort()，他们最终都会调用completeEdit()方法

	  private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
    	Entry entry = editor.entry;
    	if (entry.currentEditor != editor) {
      		throw new IllegalStateException();
    	}

    	// If this edit is creating the entry for the first time, every index must have a value.
    	if (success && !entry.readable) {
      		for (int i = 0; i < valueCount; i++) {
        		if (!editor.written[i]) {
          			editor.abort();
         			 throw new IllegalStateException("Newly created entry didn't create value for index " + i);
        		}
        if (!entry.getDirtyFile(i).exists()) {
          editor.abort();
          return;
        	}
      	   }
    	  }
		// 将写入成功的临时文件重命名为正式的缓存文件
    	for (int i = 0; i < valueCount; i++) {
      		File dirty = entry.getDirtyFile(i);
      		if (success) {
        		if (dirty.exists()) {
          			File clean = entry.getCleanFile(i);
          			dirty.renameTo(clean);
          			long oldLength = entry.lengths[i];
          			long newLength = clean.length();
          			entry.lengths[i] = newLength;
          			size = size - oldLength + newLength;
        		}
      		} else {
        		deleteIfExists(dirty);
      		}
   		 }
		 // 多余的操作数量+1，用来判断是否需要重建 journal 文件用的
   		 redundantOpCount++;
    	 entry.currentEditor = null;
    	 if (entry.readable | success) {
      		entry.readable = true;
      		journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
      	 if (success) {
			 // 如果在此之前有获取过该 Entry 的 Snapshot ，这一步操作将使 Snapshot 过期
        	entry.sequenceNumber = nextSequenceNumber++;
           }
    	 } else {// 写入 REMOVE 记录，便于下次移除临时文件
      	    lruEntries.remove(entry.key);
      		journalWriter.write(REMOVE + ' ' + entry.key + '\n');
    	 }
    	journalWriter.flush();
		 // journal 文件清理
    	if (size > maxSize || journalRebuildRequired()) {
      		executorService.submit(cleanupCallable);
    	  }
  	   }
## 读取缓存
　　读取缓存主要依靠get()方法，拿到key所对应的缓存，然后打开所有关联的文件输入流，同时将操作记录写入journal文件或重建journal文件，最终返回Snapshot对象。我们读取缓存通过Snapshot对象获取输入流。

	  public synchronized Snapshot get(String key) throws IOException {
    	checkNotClosed();
    	validateKey(key);
    	Entry entry = lruEntries.get(key);
    	if (entry == null) {
      		return null;
    	}

    	if (!entry.readable) {
      		return null;
    	}

    	// Open all streams eagerly to guarantee that we see a single published
    	// snapshot. If we opened streams lazily then the streams could come
    	// from different edits.
    	InputStream[] ins = new InputStream[valueCount];
    	try {
      		for (int i = 0; i < valueCount; i++) {
        		ins[i] = new FileInputStream(entry.getCleanFile(i));
      		}
    	} catch (FileNotFoundException e) {
      	// A file must have been deleted manually!
      		for (int i = 0; i < valueCount; i++) {
        		if (ins[i] != null) {
          			Util.closeQuietly(ins[i]);
        		} else {
          			break;
        		}
      		}
      		return null;
    	}

    	redundantOpCount++;
    	journalWriter.append(READ + ' ' + key + '\n');
		//判断是否需要重建Journal文件
    	if (journalRebuildRequired()) {
      		executorService.submit(cleanupCallable);
    	}

    	return new Snapshot(key, entry.sequenceNumber, ins, entry.lengths);
  	}
## 清理空间
　　删除缓存的remove()方法我们就不说了，具体变化不大。但我们要说说所有操作缓存的方法里面的executorService.submit(cleanupCallable)，采用了线程池异步操作，具体实现如下：

	final ThreadPoolExecutor executorService =
      new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
  	
	private final Callable<Void> cleanupCallable = new Callable<Void>() {
    public Void call() throws Exception {
      synchronized (DiskLruCache.this) {
        if (journalWriter == null) {
          return null; // Closed.
        }
        trimToSize();
        if (journalRebuildRequired()) {
          rebuildJournal();
          redundantOpCount = 0;
        }
      }
      return null;
     }
    };
	  //trimeToSize基于LinkedHashMap将eldest项目移除,如同LRU算法
	  private void trimToSize() throws IOException {
    	while (size > maxSize) {
      	Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
      	remove(toEvict.getKey());
    	}
  	}

## 总结
　　1.DiskLruCache使用了两个内部类，一个是DiskLruCache.Editor用来写入缓存信息，一个是DiskLruCache.Snapshot用来读取缓存文件。
	
　　2.缓存文件主要分为CLEAN文件和DIRTY文件，CLEAN文件为缓存完成的文件，DIRTY文件用来临时标记正在修改，READ文件代表读取一条数据，REMOVE文件代表删除一条数据

　　3.journal文件记录了哪些关键字已经被缓存、哪些正在修改、哪些正在读取。

　　4.缓存入口同LruCache一样使用LinkedHashMap，省去了LRU算法的实现。

　　5.整理空间和重写journal操作留给了线程池单线程异步处理，一是防止了主线程阻塞问题，二是解决了同步问题。