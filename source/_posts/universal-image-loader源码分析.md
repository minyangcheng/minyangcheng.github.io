---
title: universal-image-loader源码分析
date: 2016-11-19 17:42:39
tags: android
---
这几天又重新把UIL的源码细读了，发现其中有太多值得我们借鉴学习的地方。
UIL官方文档网址：<https://github.com/nostra13/Android-Universal-Image-Loader/wiki>
<!-- more -->

## UIL工作流程分析

UIL很好的帮助我们处理了图片加载、变换、缓存、显示，这其中涉及到的情况比较多，我就在这里主要分析一种工作流程：将一张图片(该图片是位于服务器上并且没有被加载过)显示在控件上。

首先我们简单的分析下，将一张图片显示在控件上，需要包括的几个动作：
1. 通过http协议将服务器上的图片下载到本地
2. 放入硬盘缓存
3. 放入内存缓存
4. 在UI线程上回调显示图片

其实UIL的主要工作流程就是这样的，只不过它封装和扩展的比较完善，而且它考虑到了各种极限条件。上一张UIL工作流程的介绍图

![](/images/uil_1.png)

## UIL是怎样判定控件被回收或被复用的

### 回收判定

一般需要在ImageView上显示图片会调用到`displayImage(uri, new ImageViewAware(imageView), null, null, null)`,也就是说ImageView会被包装成ImageViewAeare，而ImageViewViewAware中保存的是WeakReference。判定是否被回收也就是通过`viewRef.get() == null`。

```java
//ImageViewAware父类构造器
public ViewAware(View view, boolean checkActualViewSize) {
		if (view == null) throw new IllegalArgumentException("view must not be null");

		this.viewRef = new WeakReference<View>(view);
		this.checkActualViewSize = checkActualViewSize;
	}
//是否被回收
@Override
	public boolean isCollected() {
		return viewRef.get() == null;
	}
```

### 复用判定

当ImageView被包装成ImageViewAware，ImageViewAware内部会有一个唯一标示Id。当每次调用displayImage时，会根据显示控件的大小产生出memoryCacheKey。ImageLoaderEngine中的Map<Integer, String> cacheKeysForImageAware，存放的就是ImageViewAware对应的id和memoryCacheKey。在需要判定是否被复用的时候，根据id从cacheKeysForImageAware取出memoryCacheKey,判断是否和当前的memoryCacheKey相等，如果不相等则被回收。

```java
private boolean isViewReused() {
		String currentCacheKey = engine.getLoadingUriForView(imageAware);
		// Check whether memory cache key (image URI) for current ImageAware is actual.
		// If ImageAware is reused for another task then current task should be cancelled.
		boolean imageAwareWasReused = !memoryCacheKey.equals(currentCacheKey);
		if (imageAwareWasReused) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_REUSED, memoryCacheKey);
			return true;
		}
		return false;
	}
```

当UIL开始加载图片显示在控件时，一旦发现显示控件被回收和被复用，就会停止任务。

## UIL如何处理加载显示同一个url

UIL会为每个url生成一个ReentrantLock，而后通过ReentrantLock实现多线程互斥操作，看代码

```java
@Override
public void run() {
	if (waitIfPaused()) return;
	if (delayIfNeed()) return;

	//url关联的ReentrantLock
	ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
	L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
	if (loadFromUriLock.isLocked()) {
		L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
	}
	//加锁
	loadFromUriLock.lock();
	Bitmap bmp;
	try {
		checkTaskNotActual();

		bmp = configuration.memoryCache.get(memoryCacheKey);
		if (bmp == null || bmp.isRecycled()) {
			bmp = tryLoadBitmap();
			if (bmp == null) return; // listener callback already was fired

			checkTaskNotActual();
			checkTaskInterrupted();

			if (options.shouldPreProcess()) {
				L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
				bmp = options.getPreProcessor().process(bmp);
				if (bmp == null) {
					L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
				}
			}

			if (bmp != null && options.isCacheInMemory()) {
				L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
				configuration.memoryCache.put(memoryCacheKey, bmp);
			}
		} else {
			loadedFrom = LoadedFrom.MEMORY_CACHE;
			L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
		}

		if (bmp != null && options.shouldPostProcess()) {
			L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
			bmp = options.getPostProcessor().process(bmp);
			if (bmp == null) {
				L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
			}
		}
		checkTaskNotActual();
		checkTaskInterrupted();
	} catch (TaskCancelledException e) {
		fireCancelEvent();
		return;
	} finally {
		//解锁
		loadFromUriLock.unlock();
	}

	DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
	runTask(displayBitmapTask, syncLoading, handler, engine);
}
```

## UIL对图片的处理

当UIL将服务器上的图片下载到本地后，需要将图片文件转换为Bitmap对象，在转换时就需要根据图片本身的大小和目标控件的大小进行缩放图片，有时候还需要对图片的角度进行变换。

```java
@Override
public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
	Bitmap decodedBitmap;
	ImageFileInfo imageInfo;

	InputStream imageStream = getImageStream(decodingInfo);
	if (imageStream == null) {
		L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
		return null;
	}
	try {
		imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
		imageStream = resetStream(imageStream, decodingInfo);
		Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
		decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
	} finally {
		IoUtils.closeSilently(imageStream);
	}

	if (decodedBitmap == null) {
		L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
	} else {
		decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
				imageInfo.exif.flipHorizontal);
	}
	return decodedBitmap;
}
```

计算图片缩放比例:主要是根据目标大小和源大小、控件ScaleType类型，计算scale用于BitmapOption

```java
public static int computeImageSampleSize(ImageSize srcSize, ImageSize targetSize, ViewScaleType viewScaleType,
			boolean powerOf2Scale) {
	final int srcWidth = srcSize.getWidth();
	final int srcHeight = srcSize.getHeight();
	final int targetWidth = targetSize.getWidth();
	final int targetHeight = targetSize.getHeight();

	int scale = 1;

	switch (viewScaleType) {
		case FIT_INSIDE:
			if (powerOf2Scale) {
				final int halfWidth = srcWidth / 2;
				final int halfHeight = srcHeight / 2;
				while ((halfWidth / scale) > targetWidth || (halfHeight / scale) > targetHeight) { // ||
					scale *= 2;
				}
			} else {
				scale = Math.max(srcWidth / targetWidth, srcHeight / targetHeight); // max
			}
			break;
		case CROP:
			if (powerOf2Scale) {
				final int halfWidth = srcWidth / 2;
				final int halfHeight = srcHeight / 2;
				while ((halfWidth / scale) > targetWidth && (halfHeight / scale) > targetHeight) { // &&
					scale *= 2;
				}
			} else {
				scale = Math.min(srcWidth / targetWidth, srcHeight / targetHeight); // min
			}
			break;
	}

	if (scale < 1) {
		scale = 1;
	}
	scale = considerMaxTextureSize(srcWidth, srcHeight, scale, powerOf2Scale);

	return scale;
}
```

## UIL默认的硬盘缓存

UIL默认用的硬盘缓存为LruDiskCache，其主要对DiskLruCache做了一层封装。

```java
public class LruDiskCache implements DiskCache {
	/** {@value */
	public static final int DEFAULT_BUFFER_SIZE = 32 * 1024; // 32 Kb
	/** {@value */
	public static final Bitmap.CompressFormat DEFAULT_COMPRESS_FORMAT = Bitmap.CompressFormat.PNG;
	/** {@value */
	public static final int DEFAULT_COMPRESS_QUALITY = 100;

	private static final String ERROR_ARG_NULL = " argument must be not null";
	private static final String ERROR_ARG_NEGATIVE = " argument must be positive number";

	protected DiskLruCache cache;
	private File reserveCacheDir;

	protected final FileNameGenerator fileNameGenerator;

	protected int bufferSize = DEFAULT_BUFFER_SIZE;

	protected Bitmap.CompressFormat compressFormat = DEFAULT_COMPRESS_FORMAT;
	protected int compressQuality = DEFAULT_COMPRESS_QUALITY;

	/**
	 * @param cacheDir          Directory for file caching
	 * @param fileNameGenerator {@linkplain com.nostra13.universalimageloader.cache.disc.naming.FileNameGenerator
	 *                          Name generator} for cached files. Generated names must match the regex
	 *                          <strong>[a-z0-9_-]{1,64}</strong>
	 * @param cacheMaxSize      Max cache size in bytes. <b>0</b> means cache size is unlimited.
	 * @throws IOException if cache can't be initialized (e.g. "No space left on device")
	 */
	public LruDiskCache(File cacheDir, FileNameGenerator fileNameGenerator, long cacheMaxSize) throws IOException {
		this(cacheDir, null, fileNameGenerator, cacheMaxSize, 0);
	}

	/**
	 * @param cacheDir          Directory for file caching
	 * @param reserveCacheDir   null-ok; Reserve directory for file caching. It's used when the primary directory isn't available.
	 * @param fileNameGenerator {@linkplain com.nostra13.universalimageloader.cache.disc.naming.FileNameGenerator
	 *                          Name generator} for cached files. Generated names must match the regex
	 *                          <strong>[a-z0-9_-]{1,64}</strong>
	 * @param cacheMaxSize      Max cache size in bytes. <b>0</b> means cache size is unlimited.
	 * @param cacheMaxFileCount Max file count in cache. <b>0</b> means file count is unlimited.
	 * @throws IOException if cache can't be initialized (e.g. "No space left on device")
	 */
	public LruDiskCache(File cacheDir, File reserveCacheDir, FileNameGenerator fileNameGenerator, long cacheMaxSize,
			int cacheMaxFileCount) throws IOException {
		if (cacheDir == null) {
			throw new IllegalArgumentException("cacheDir" + ERROR_ARG_NULL);
		}
		if (cacheMaxSize < 0) {
			throw new IllegalArgumentException("cacheMaxSize" + ERROR_ARG_NEGATIVE);
		}
		if (cacheMaxFileCount < 0) {
			throw new IllegalArgumentException("cacheMaxFileCount" + ERROR_ARG_NEGATIVE);
		}
		if (fileNameGenerator == null) {
			throw new IllegalArgumentException("fileNameGenerator" + ERROR_ARG_NULL);
		}

		if (cacheMaxSize == 0) {
			cacheMaxSize = Long.MAX_VALUE;
		}
		if (cacheMaxFileCount == 0) {
			cacheMaxFileCount = Integer.MAX_VALUE;
		}

		this.reserveCacheDir = reserveCacheDir;
		this.fileNameGenerator = fileNameGenerator;
		initCache(cacheDir, reserveCacheDir, cacheMaxSize, cacheMaxFileCount);
	}

	private void initCache(File cacheDir, File reserveCacheDir, long cacheMaxSize, int cacheMaxFileCount)
			throws IOException {
		try {
			cache = DiskLruCache.open(cacheDir, 1, 1, cacheMaxSize, cacheMaxFileCount);
		} catch (IOException e) {
			L.e(e);
			if (reserveCacheDir != null) {
				initCache(reserveCacheDir, null, cacheMaxSize, cacheMaxFileCount);
			}
			if (cache == null) {
				throw e; //new RuntimeException("Can't initialize disk cache", e);
			}
		}
	}

	@Override
	public File getDirectory() {
		return cache.getDirectory();
	}

	@Override
	public File get(String imageUri) {
		DiskLruCache.Snapshot snapshot = null;
		try {
			snapshot = cache.get(getKey(imageUri));
			return snapshot == null ? null : snapshot.getFile(0);
		} catch (IOException e) {
			L.e(e);
			return null;
		} finally {
			if (snapshot != null) {
				snapshot.close();
			}
		}
	}

	@Override
	public boolean save(String imageUri, InputStream imageStream, IoUtils.CopyListener listener) throws IOException {
		DiskLruCache.Editor editor = cache.edit(getKey(imageUri));
		if (editor == null) {
			return false;
		}

		OutputStream os = new BufferedOutputStream(editor.newOutputStream(0), bufferSize);
		boolean copied = false;
		try {
			copied = IoUtils.copyStream(imageStream, os, listener, bufferSize);
		} finally {
			IoUtils.closeSilently(os);
			if (copied) {
				editor.commit();
			} else {
				editor.abort();
			}
		}
		return copied;
	}

	@Override
	public boolean save(String imageUri, Bitmap bitmap) throws IOException {
		DiskLruCache.Editor editor = cache.edit(getKey(imageUri));
		if (editor == null) {
			return false;
		}

		OutputStream os = new BufferedOutputStream(editor.newOutputStream(0), bufferSize);
		boolean savedSuccessfully = false;
		try {
			savedSuccessfully = bitmap.compress(compressFormat, compressQuality, os);
		} finally {
			IoUtils.closeSilently(os);
		}
		if (savedSuccessfully) {
			editor.commit();
		} else {
			editor.abort();
		}
		return savedSuccessfully;
	}

	@Override
	public boolean remove(String imageUri) {
		try {
			return cache.remove(getKey(imageUri));
		} catch (IOException e) {
			L.e(e);
			return false;
		}
	}

	@Override
	public void close() {
		try {
			cache.close();
		} catch (IOException e) {
			L.e(e);
		}
		cache = null;
	}

	@Override
	public void clear() {
		try {
			cache.delete();
		} catch (IOException e) {
			L.e(e);
		}
		try {
			initCache(cache.getDirectory(), reserveCacheDir, cache.getMaxSize(), cache.getMaxFileCount());
		} catch (IOException e) {
			L.e(e);
		}
	}

	private String getKey(String imageUri) {
		return fileNameGenerator.generate(imageUri);
	}

	public void setBufferSize(int bufferSize) {
		this.bufferSize = bufferSize;
	}

	public void setCompressFormat(Bitmap.CompressFormat compressFormat) {
		this.compressFormat = compressFormat;
	}

	public void setCompressQuality(int compressQuality) {
		this.compressQuality = compressQuality;
	}
}
```

## UIL默认的内存缓存

UIL默认用的内存缓存为LruMemoryCahce，其内部主要是用LinkedHashMap来实现LRU算法的。

```java
public class LruMemoryCache implements MemoryCache {

	private final LinkedHashMap<String, Bitmap> map;

	private final int maxSize;
	/** Size of this cache in bytes */
	private int size;

	/** @param maxSize Maximum sum of the sizes of the Bitmaps in this cache */
	public LruMemoryCache(int maxSize) {
		if (maxSize <= 0) {
			throw new IllegalArgumentException("maxSize <= 0");
		}
		this.maxSize = maxSize;
		//当LinkedHashMap构造器第三个参数指定为true时，其内部即可实现LRU算法
		this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
	}

	/**
	 * Returns the Bitmap for {@code key} if it exists in the cache. If a Bitmap was returned, it is moved to the head
	 * of the queue. This returns null if a Bitmap is not cached.
	 */
	@Override
	public final Bitmap get(String key) {
		if (key == null) {
			throw new NullPointerException("key == null");
		}

		synchronized (this) {
			return map.get(key);
		}
	}

	/** Caches {@code Bitmap} for {@code key}. The Bitmap is moved to the head of the queue. */
	@Override
	public final boolean put(String key, Bitmap value) {
		if (key == null || value == null) {
			throw new NullPointerException("key == null || value == null");
		}

		synchronized (this) {
			size += sizeOf(key, value);
			Bitmap previous = map.put(key, value);
			if (previous != null) {
				size -= sizeOf(key, previous);
			}
		}

		trimToSize(maxSize);
		return true;
	}

	/**
	 * Remove the eldest entries until the total of remaining entries is at or below the requested size.
	 *
	 * @param maxSize the maximum size of the cache before returning. May be -1 to evict even 0-sized elements.
	 */
	private void trimToSize(int maxSize) {
		while (true) {
			String key;
			Bitmap value;
			synchronized (this) {
				if (size < 0 || (map.isEmpty() && size != 0)) {
					throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
				}

				if (size <= maxSize || map.isEmpty()) {
					break;
				}

				Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
				if (toEvict == null) {
					break;
				}
				key = toEvict.getKey();
				value = toEvict.getValue();
				map.remove(key);
				size -= sizeOf(key, value);
			}
		}
	}

	/** Removes the entry for {@code key} if it exists. */
	@Override
	public final Bitmap remove(String key) {
		if (key == null) {
			throw new NullPointerException("key == null");
		}

		synchronized (this) {
			Bitmap previous = map.remove(key);
			if (previous != null) {
				size -= sizeOf(key, previous);
			}
			return previous;
		}
	}

	@Override
	public Collection<String> keys() {
		synchronized (this) {
			return new HashSet<String>(map.keySet());
		}
	}

	@Override
	public void clear() {
		trimToSize(-1); // -1 will evict 0-sized elements
	}

	/**
	 * Returns the size {@code Bitmap} in bytes.
	 * <p/>
	 * An entry's size must not change while it is in the cache.
	 */
	private int sizeOf(String key, Bitmap value) {
		return value.getRowBytes() * value.getHeight();
	}

	@Override
	public synchronized final String toString() {
		return String.format("LruCache[maxSize=%d]", maxSize);
	}
}
```

## 总结

我这里只是简单的写了分析的几个重要的类和思路，其实源码阅读这个事情，还是需要自己一步一步去读懂才好，源码里有很多东西很难通过文字表达不来。
