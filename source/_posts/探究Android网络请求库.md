---
title: 探究Android网络请求库
date: 2017-08-31 12:45:09
tags: android
---
android网络请求从最初的手写HttpUrlConnection，到谷歌开源的Volley框架，再到Square开源的Retrofit框架，使得现在开发中请求网络数据越来越简单，代码也变得越来越简洁。

<!-- more -->

### Volley

使用Volley框架，我们就可以不用每次做网络请求的时候需要手写HttpUrlConnenct请求数据，手写AsyncTask或者Handler进行线程切换更新视图，其内部的封装了缓存机制和请求队列机制，可以使我们更好的完成网络请求。

#### 实现原理

Volley框架内部有两个请求队列：一个用于处理从缓存中读取数据的阻塞队列和一个用于处理网络请求的阻塞队列，与之匹配的就存在一个用于处理缓存读取的任务线程和四个用于处理网络数据读取的任务线程，这些线程去执行任务队列中的request，数据请求完成后再用转发器将对应的数据回调到Request对应的监听器上。

#### 工作流程分析

* 启动RequestQueue，此后任务线程不断的从各自的请求队列中执行request
```
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```

* 网络请求任务线程NetworkDispatcher,请求完成后用`mDelivery.postResponse(request, response)`，用转发器将结果回调到request的监听器上
```
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
```

### Retrofit

在使用Volley的时候，每次需要做网络请求的时候都需要new一个request出来，或者在你可以封装出一个自定义的Request，让一个实体对应一个请求，这样做在请求接口不多的情况下还好，但是一旦项目的接口一多，就会让这个工程多出很多这些无用的实体对象。这时候采用Retrofit就解决了这个这个问题。
此外Retrofit还有好多特性，比如可以结合rxjava一起使用等。

#### 实现原理

Retrofit主要是通过包装OkHttp来实现数据请求的，内部通过java动态代理实现对Api接口的实现，当我们在接口上写的注解，retrofit就回通过注解得到此次请求的Url、请求的参数、请求类型等，然后在包装这些请求数据，通过okhttp发出去，得到数据后再就行一层封装交给调用方。

#### 工作流程分析

* 创建接口代理对象
```
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //ServiceMethod包装了请求的url、paramter、CallAdapter、Converter等
            ServiceMethod serviceMethod = loadServiceMethod(method);
            //OkHttpCall将请求数据代理给okhttp进行网络请求
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //返回method定义的返回类型
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

* 返接口方法定义的类型对象，这一步就返回了一个Observable对象，一旦这个Observable被订阅，就会触发`Response<T> response = call.execute();`
```
@Override public <R> Observable<Response<R>> adapt(Call<R> call) {
      Observable<Response<R>> observable = Observable.create(new CallOnSubscribe<>(call));
      if (scheduler != null) {
        return observable.subscribeOn(scheduler);
      }
      return observable;
    }

  static final class CallOnSubscribe<T> implements Observable.OnSubscribe<Response<T>> {
    private final Call<T> originalCall;

    CallOnSubscribe(Call<T> originalCall) {
      this.originalCall = originalCall;
    }

    @Override public void call(final Subscriber<? super Response<T>> subscriber) {
      // Since Call is a one-shot type, clone it for each new subscriber.
      Call<T> call = originalCall.clone();

      // Wrap the call in a helper which handles both unsubscription and backpressure.
      RequestArbiter<T> requestArbiter = new RequestArbiter<>(call, subscriber);
      subscriber.add(requestArbiter);
      subscriber.setProducer(requestArbiter);
    }
  }
```

* 进行网络请求，在获取到了请求结果后，调用GsonResponseBodyConverter对http响应体里面字符串进行反序列化。此步骤完成后就会将数据发送到Observable的订阅对象
```
  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;
	//...省略
    return parseResponse(call.execute());
  }

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    //...省略
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

  T toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }

  @Override public T convert(ResponseBody value) throws IOException {
    JsonReader jsonReader = gson.newJsonReader(value.charStream());
    try {
      return adapter.read(jsonReader);
    } finally {
      value.close();
    }
  }
```

### 在Volley基本上封装一个简单的Retrofit

公司的项目之前用的是Volley，并且自定义了一个Request，可以让接口定义在实体对象中，但是这样做在项目后期出现了大量的请求实体对象。考虑到之前的用的是Volley，所以自己在volley基础上，写了一个带有基本功能的Retrofit。

#### 实现技术点

1. 动态代理
2. 定义注解 
3. 反射读取请求参数
4. Rxjava
5. 采用Volley的StringRequest进行真正的数据请求

#### 部分代码

```
public class EasyRetrofit {

    private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();

    private String baseUrl;
    private RequestQueue requestQueue;

    public EasyRetrofit(Builder builder) {
        baseUrl = builder.baseUrl;
        requestQueue = builder.requestQueue;
    }

    public <T> T create(final Class<T> service) {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object... arg)
                            throws Throwable {
                        ServiceMethod serviceMethod = loadServiceMethod(method);
                        HttpCall httpCall = new HttpCall(baseUrl, requestQueue, serviceMethod, arg);
                        return httpCall.execute();
                    }
                });
    }

    private ServiceMethod loadServiceMethod(Method method) {
        ServiceMethod result;
        synchronized (serviceMethodCache) {
            result = serviceMethodCache.get(method);
            if (result == null) {
                result = new ServiceMethod(method);
                serviceMethodCache.put(method, result);
            }
        }
        return result;
    }

    public static class Builder {

        private String baseUrl;
        private RequestQueue requestQueue;

        public Builder baseUrl(String baseUrl) {
            this.baseUrl = baseUrl;
            return this;
        }

        public Builder requestQueue(RequestQueue requestQueue) {
            this.requestQueue = requestQueue;
            return this;
        }

        public EasyRetrofit build() {
            if (TextUtils.isEmpty(baseUrl)) {
                throw new RetrofitException(RetrofitException.BASE_URL_CAN_NOT_EMPTY);
            }
            if (requestQueue == null) {
                throw new RetrofitException(RetrofitException.REQUEST_QUEUE_NOT_EMPTY);
            }
            return new EasyRetrofit(this);
        }

    }

}
```

```
public class ServiceMethod {

    private Method method;

    public ServiceMethod(Method method) {
        this.method = method;
    }

    public Type getReturnType() {
        Type returnType = method.getGenericReturnType();
        if (returnType instanceof ParameterizedType) {
            Type[] types = ((ParameterizedType) returnType).getActualTypeArguments();
            return types[0];
        }
        return null;
    }

    public String getRequestUrl() {
        POST postAnnotation = method.getAnnotation(POST.class);
        if (postAnnotation != null) {
            return postAnnotation.value();
        } else {
            throw new RetrofitException(RetrofitException.URL_ERROR);
        }
    }

    public Map<String, String> getRequestFileds(Object[] args) {
        Map<String, String> map = new TreeMap<>();
        List<String> fieldNames = getRequestFieldName();
        if (args.length != 0 && args.length == fieldNames.size()) {
            for (int i = 0; i < args.length; i++) {
                putArgToMap(map, fieldNames.get(i), args[i]);
            }
            return map;
        } else {
            throw new RetrofitException(RetrofitException.METHOD_FIELD_ERROR);
        }
    }

    private void putArgToMap(Map<String, String> map, String name, Object value) {
        if (value == null) return;
        Class clazz = value.getClass();
        if (clazz.isPrimitive() || clazz == String.class) {
            map.put(name, value.toString());
        } else {
            map.put(name, JSON.toJSONString(value));
        }
    }

    private List<String> getRequestFieldName() {
        Annotation[][] parameterAnnotations = method.getParameterAnnotations();
        if (parameterAnnotations != null && parameterAnnotations.length != 0) {
            List<String> fieldNames = new ArrayList<>();
            Field field;
            for (Annotation[] annotations : parameterAnnotations) {
                field = (Field) annotations[0];
                fieldNames.add(field.value());
            }
            return fieldNames;
        } else {
            throw new RetrofitException(RetrofitException.METHOD_FIELD_ERROR);
        }
    }

}
```

```
public class HttpCall {

    private String url;
    private Map<String, String> map;
    private Type type;
    private String baseUrl;
    private RequestQueue requestQueue;

    public HttpCall(String baseUrl, RequestQueue requestQueue, ServiceMethod serviceMethod, Object[] args) {
        this.baseUrl = baseUrl;
        this.requestQueue = requestQueue;
        this.url = handleUrl(serviceMethod.getRequestUrl());
        this.map = serviceMethod.getRequestFileds(args);
        type = serviceMethod.getReturnType();
    }

    public Observable<Bean<?>> execute() {
        return Observable.<String>create(subscriber -> {
            StringRequest request = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
                @Override
                public void onResponse(String response) {
                    if (subscriber == null || subscriber.isUnsubscribed()) return;
                    subscriber.onNext(response);
                    subscriber.onCompleted();
                }
            }, new Response.ErrorListener() {
                @Override
                public void onErrorResponse(VolleyError error) {
                    if (subscriber == null || subscriber.isUnsubscribed()) return;
                    subscriber.onError(error);
                }
            }) {
                @Override
                protected Map<String, String> getParams() throws AuthFailureError {
                    Map<String, String> result = ApiHandlerUtil.handleParams(map,url);
                    L.d(AppContants.HTTP_LOG, "url= %s , response=%s", url, JSON.toJSONString(result, true));
                    return result;
                }
            };
            requestQueue.add(request);
        }).map(new Func1<String, Bean<?>>() {
            @Override
            public Bean<?> call(String s) {
                Bean<?> bean = JSON.parseObject(s, type);
                return bean;
            }
        });
    }

    private String handleUrl(String s) {
        String url = s;
        if (!s.startsWith("http")) {
            url = baseUrl + url;
        }
        return url;
    }

}
```