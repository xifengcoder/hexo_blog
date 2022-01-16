---
title: OKhttp的理解-CacheInterceptor
urlname: okhttp_interceptor
date: 2021-12-26 17:29:13
tags: Http
categories: Network
description: OKHttp中的CacheInterceptor的理解...
---

### 一、用法
#### 1. noCache
```java
Request request = new Request.Builder()
        .cacheControl(new CacheControl.Builder().noCache().build())
        .url("http://publicobject.com/helloworld.txt")
        .build();
```

#### 2. maxAge
```
Request request = new Request.Builder()
    .cacheControl(new CacheControl.Builder().maxAge(0, TimeUnit.SECONDS).build())
    .url("http://publicobject.com/helloworld.txt")
    .build();
```

#### 3. Only-If-Cached

```
Request request = new Request.Builder()
      .cacheControl(new CacheControl.Builder()
      .onlyIfCached()
      .build())
      .url("http://publicobject.com/helloworld.txt")
      .build();
Response forceCacheResponse = client.newCall(request).execute();
if (forceCacheResponse.code() != 504) {
    // The resource was cached! Show it.
} else {
   // The resource was not cached.
}
```

#### 4. MaxStale
```
Request request = new Request.Builder()
        .cacheControl(new CacheControl.Builder()
        .maxStale(365, TimeUnit.DAYS)
        .build())
        .url("http://publicobject.com/helloworld.txt")
        .build();
```
### 二、Cache的设计
CacheInterceptor负责对缓存的处理。
根据当前时间戳、源request和该request对应的response缓存（cacheCandidate变量， 存储在cache中（InternalCache实例））三个因素来确定缓存逻辑。
1. 如果request中禁用了网络、同时cacheResponse为空，则直接返回504错误；
2. 如果request中禁用了网络，同时cacheResponse不为空，则直接返回cacheResponse;
3. 如果以上2条不满足，则将request传递给下一个责任链处理，即会请求网络。继续后面的处理。
4. 如果cacheResponse不为空，且responseCode为304（HTTP_NOT_MODIFIED），则基于cacheResponse和networkResponse构建新的response，并更新本地缓存。
5. 如果cacheResponse为空：
如果该response中含有body、并且该response是可缓存的，则将该response存入cache（InternalCache实例）中。
6. 如果该request非GET请求，则从cache中移除该请求。
```java
public final class CacheInterceptor implements Interceptor {
    final @Nullable InternalCache cache;

    public CacheInterceptor(@Nullable InternalCache cache) {
        this.cache = cache;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Response cacheCandidate = cache != null
                ? cache.get(chain.request())
                : null;

        long now = System.currentTimeMillis();

        CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
        Request networkRequest = strategy.networkRequest;
        Response cacheResponse = strategy.cacheResponse;

        if (cache != null) {
            cache.trackResponse(strategy);
        }

        if (cacheCandidate != null && cacheResponse == null) {
            closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
        }

        // If we're forbidden from using the network and the cache is insufficient, fail.
        if (networkRequest == null && cacheResponse == null) {
            return new Response.Builder()
                    .request(chain.request())
                    .protocol(Protocol.HTTP_1_1)
                    .code(504)
                    .message("Unsatisfiable Request (only-if-cached)")
                    .body(Util.EMPTY_RESPONSE)
                    .sentRequestAtMillis(-1L)
                    .receivedResponseAtMillis(System.currentTimeMillis())
                    .build();
        }

        // If we don't need the network, we're done.
        if (networkRequest == null) {
            return cacheResponse.newBuilder()
                    .cacheResponse(stripBody(cacheResponse))
                    .build();
        }

        Response networkResponse = null;
        try {
            networkResponse = chain.proceed(networkRequest);
        } finally {
            // If we're crashing on I/O or otherwise, don't leak the cache body.
            if (networkResponse == null && cacheCandidate != null) {
                closeQuietly(cacheCandidate.body());
            }
        }

        // If we have a cache response too, then we're doing a conditional get.
        if (cacheResponse != null) {
            if (networkResponse.code() == HTTP_NOT_MODIFIED) {
                Response response = cacheResponse.newBuilder()
                        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                        .cacheResponse(stripBody(cacheResponse))
                        .networkResponse(stripBody(networkResponse))
                        .build();
                networkResponse.body().close();

                // Update the cache after combining headers but before stripping the
                // Content-Encoding header (as performed by initContentStream()).
                cache.trackConditionalCacheHit();
                cache.update(cacheResponse, response);
                return response;
            } else {
                closeQuietly(cacheResponse.body());
            }
        }

        Response response = networkResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();

        if (cache != null) {
            if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
                // Offer this request to the cache.
                CacheRequest cacheRequest = cache.put(response);
                return cacheWritingResponse(cacheRequest, response);
            }

            if (HttpMethod.invalidatesCache(networkRequest.method())) {
                try {
                    cache.remove(networkRequest);
                } catch (IOException ignored) {
                    // The cache cannot be written.
                }
            }
        }

        return response;
    }
}
```
注意点：
CacheRequest put(Response response)
put操作的缓存逻辑：
1. 如果是非GET请求，则不作缓存；
2. 如果response的headers中含有Vary指令、且值包含“*”，则不作缓存；
3. 其他情况下可以缓存，put方法返回CacheRequestImpl实例（CacheRequest接口）。

```java
public final class Cache implements Closeable, Flushable {
    final InternalCache internalCache = new InternalCache() {
        @Override
        public @Nullable
        Response get(Request request) throws IOException {
            return Cache.this.get(request);
        }

        @Override
        public @Nullable
        CacheRequest put(Response response) throws IOException {
            return Cache.this.put(response);
        }

        @Override
        public void remove(Request request) throws IOException {
            Cache.this.remove(request);
        }

        @Override
        public void update(Response cached, Response network) {
            Cache.this.update(cached, network);
        }

        @Override
        public void trackConditionalCacheHit() {
            Cache.this.trackConditionalCacheHit();
        }

        @Override
        public void trackResponse(CacheStrategy cacheStrategy) {
            Cache.this.trackResponse(cacheStrategy);
        }
    };
    final DiskLruCache cache;

    public Cache(File directory, long maxSize) {
        this(directory, maxSize, FileSystem.SYSTEM);
    }

    Cache(File directory, long maxSize, FileSystem fileSystem) {
        this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
    }
}
```

### 三、DiskLruCache
一个journal文件的格式：
```
libcore.io.DiskLruCache
1
100
2
CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
```
#### 3.1 关键类SnapShot
```java
public final class DiskLruCache implements Closeable, Flushable {
    private final class Entry {
        final String key;
        final long[] lengths;
        final File[] cleanFiles;
        final File[] dirtyFiles;
        boolean readable; //该Entry是否已经成功发布
        Editor currentEditor; //如果该Entry正在编辑状态时, currentEditor为非空.
        long sequenceNumber; //最近一次提交到该Entry的commit的序列号.

        Entry(String key) {
            this.key = key;
            lengths = new long[valueCount];
            cleanFiles = new File[valueCount];
            dirtyFiles = new File[valueCount];

            // The names are repetitive so re-use the same builder to avoid allocations.
            StringBuilder fileBuilder = new StringBuilder(key).append('.');
            int truncateTo = fileBuilder.length();
            for (int i = 0; i < valueCount; i++) {
                fileBuilder.append(i);
                cleanFiles[i] = new File(directory, fileBuilder.toString());
                fileBuilder.append(".tmp");
                dirtyFiles[i] = new File(directory, fileBuilder.toString());
                fileBuilder.setLength(truncateTo);
            }
        }

        //为lengths[]赋值
        void setLengths(String[] strings) throws IOException {
            ...
        }

        //将lengths[]中的值写入writer中.
        void writeLengths(BufferedSink writer) throws IOException {
            ...
        }

        private IOException invalidLengths(String[] strings) throws IOException {
            throw new IOException("unexpected journal line: " + Arrays.toString(strings));
        }

        //返回该Entry的一个snapshot.
        Snapshot snapshot() {
            ...
        }
    }


     /**
     * A snapshot of the values for an entry.
     */
    public final class Snapshot implements Closeable {
        private final String key;
        private final long sequenceNumber;
        private final Source[] sources;
        private final long[] lengths;

        Snapshot(String key, long sequenceNumber, Source[] sources, long[] lengths) {
            this.key = key;
            this.sequenceNumber = sequenceNumber;
            this.sources = sources;
            this.lengths = lengths;
        }

        public String key() {
            return key;
        }

        /**
         * Returns an editor for this snapshot's entry, or null if either the entry has changed since
         * this snapshot was created or if another edit is in progress.
         */
        public @Nullable Editor edit() throws IOException {
            return DiskLruCache.this.edit(key, sequenceNumber);
        }

        public Source getSource(int index) {
            return sources[index];
        }

        public long getLength(int index) {
            return lengths[index];
        }

        public void close() {
            for (Source in : sources) {
                Util.closeQuietly(in);
            }
        }
    }

    /**
     * Edits the values for an entry.
     */
    public final class Editor {
        final Entry entry;
        final boolean[] written;
        private boolean done;

        Editor(Entry entry) {
            this.entry = entry;
            this.written = (entry.readable) ? null : new boolean[valueCount];
        }

        void detach() {
            if (entry.currentEditor == this) {
                for (int i = 0; i < valueCount; i++) {
                    try {
                        fileSystem.delete(entry.dirtyFiles[i]);
                    } catch (IOException e) {
                        // This file is potentially leaked. Not much we can do about that.
                    }
                }
                entry.currentEditor = null;
            }
        }

        /**
         * Returns an unbuffered input stream to read the last committed value, or null if no value has
         * been committed.
         */
        public Source newSource(int index) {
            synchronized (DiskLruCache.this) {
                ...
            }
        }

        public Sink newSink(int index) {
            synchronized (DiskLruCache.this) {
                ...
            }
        }

        /**
         * Commits this edit so it is visible to readers.  This releases the edit lock so another edit
         * may be started on the same key.
         */
        public void commit() throws IOException {
            synchronized (DiskLruCache.this) {
                if (done) {
                    throw new IllegalStateException();
                }
                if (entry.currentEditor == this) {
                    completeEdit(this, true);
                }
                done = true;
            }
        }

        /**
         * Aborts this edit. This releases the edit lock so another edit may be started on the same
         * key.
         */
        public void abort() throws IOException {
            synchronized (DiskLruCache.this) {
                if (done) {
                    throw new IllegalStateException();
                }
                if (entry.currentEditor == this) {
                    completeEdit(this, false);
                }
                done = true;
            }
        }

        public void abortUnlessCommitted() {
           ...
        }
    }
}
```