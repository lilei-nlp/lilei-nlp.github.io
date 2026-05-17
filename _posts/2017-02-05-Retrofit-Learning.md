---
title: "Retrofit学习和源码解读"
date: 2017-02-05
categories:
  - blog
tags: []
---

在小蓝项目里用Retrofit + OkHttp作为网络的框架

来尝试着解读一下源码并且看能不能自己实现一个 低配版Retrofit

使用方面 主要步骤就是

1、添加依赖 Retrofit和OkHttp

2、定义接口和对应的Response类

3、在需要调用的时候生成对应的service 并且调用call方法

4、在Callback方法里处理OnResponse的相关逻辑

使用方法见RetrofitTest里面利用和风天气Api做的一个天气查询demo

<span id="more"></span>

接下来主要探讨Retrofit实现的机制

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4</code></pre></td>
<td class="code"><pre><code>Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(API_URL)
              .addConverterFactory(GsonConverterFactory.create())
                .build();</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这里Retrofit实例retrofit的生成是通过 Builder Pattern 来实现的

> The **builder pattern** is an <a href="https://en.wikipedia.org/wiki/Creational_pattern" target="_blank" rel="noopener">object creation</a> software <a href="https://en.wikipedia.org/wiki/Design_pattern_(computer_science" target="_blank" rel="noopener">design pattern</a>). Unlike the <a href="https://en.wikipedia.org/wiki/Abstract_factory_pattern" target="_blank" rel="noopener">abstract factory pattern</a> and the <a href="https://en.wikipedia.org/wiki/Factory_method_pattern" target="_blank" rel="noopener">factory method pattern</a> whose intention is to enable <a href="https://en.wikipedia.org/wiki/Polymorphism_(computer_science" target="_blank" rel="noopener">polymorphism</a>), the intention of the builder pattern is to find a solution to the telescoping constructor <a href="https://en.wikipedia.org/wiki/Anti-pattern" target="_blank" rel="noopener">anti-pattern</a>\[<a href="https://en.wikipedia.org/wiki/Wikipedia:Citation_needed" target="_blank" rel="noopener"><em>citation needed</em></a>\]. The telescoping constructor anti-pattern occurs when the increase of object constructor parameter combination leads to an exponential list of constructors. Instead of using numerous constructors, the builder pattern uses another object, a builder, that receives each initialization parameter step by step and then returns the resulting constructed object at once.

这是Wikipedia上对于建造者模式的描述

使用建造者的主要原因就是 :

构造函数的参数太多，如果重载很多个构造函数，很不优雅。所以就有了 builder 这样的媒介出现，来接收各个参数，然后再通过retrofit里面的各个函数来设定参数值。使用起来能够更加灵活。

然后通过builder.build()方法返回一个Retrofit的实例 就实现了对应参数下的retrofit实例的生成。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21</code></pre></td>
<td class="code"><pre><code>public interface HeWeatherApi {
    @GET(&quot;now&quot;)
    Call&lt;WeatherSearchResponse&gt; getResponse(@Query(&quot;city&quot;) String city,@Query(&quot;key&quot;)   
                                            String key);
}

HeWeatherApi weather = retrofit.create(HeWeatherApi.class);
Call&lt;WeatherSearchResponse&gt; call =            
        weather.getResponse(city_name.getText().toString(),API_KEY);
call.enqueue(new Callback&lt;WeatherSearchResponse&gt;() {
            @Override
            public void onResponse(Call&lt;WeatherSearchResponse&gt; call,
                                   Response&lt;WeatherSearchResponse&gt; response) {
                 //业务逻辑
            }

            @Override
            public void onFailure(Call&lt;WeatherSearchResponse&gt; call, Throwable t) {
              //失败处理
            }
        });</code></pre></td>
</tr>
</tbody>
</table>
</figure>

retrofit.create()方法

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56</code></pre></td>
<td class="code"><pre><code>public &lt;T&gt; T create(final Class&lt;T&gt; service) {
  //判断是否是一个service接口
    Utils.validateServiceInterface(service);
  // validateEagerly为True 我们就直接调用这个service的方法
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
  //否则 用Proxy生成一个新的代理类 并且必须实现 InvocationHandler这个接口 
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class&lt;?&gt;[] { service },
        //实现InvocationHandler接口 可以用lambda 更优雅一点 2333
        new InvocationHandler() {
          // 拿到平台 Java Android 或者是 iOS
          private final Platform platform = Platform.get();
           /* invoke方法 三个参数 代理类实例proxy 被调用的方法对象 method 以及 方法的参数 args 因为数量是不确定的 所以是 Object... 也可以是 Object[]
            */
          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              //如果这个方法所属的类是Object  这里是我们自己定义的接口 结果为false
              return method.invoke(this, args);
            }
            // Android.isDefaultMethod 结果为false
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 最后通过loadServiceMethod来调用方法
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall&lt;&gt;(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
  private void eagerlyValidateMethods(Class&lt;?&gt; service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);
      }
    }
  }
    // 通过loadServiceMethod来调用接口的方法
  ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    // synchronized 锁住方法 避免因为片段打断造成的错误
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        
        result = new ServiceMethod.Builder(this, method).build();
        //用一个LinkedHashMap serviceMethodCache作为method的缓存  
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }</code></pre></td>
</tr>
</tbody>
</table>
</figure>

这段代码精华所在就是 动态代理 又是一种设计模式 代理模式

Retrofit在写的时候 是不知道关于所需要使用的对应的类的任何信息的，我们不可能根据每个人的要求写一个Retrofit出来，所以**动态代理**就出现了。

动态代理，我个人的理解就是，在运行时才知道对应是哪个类，然后通过Java的反射机制，拿到这个类的一些信息（ClassLoader、Methods等)，然后能够生成对应的代理类，然后我们再通过代理类来调用接口或者类的一系列方法。

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32</code></pre></td>
<td class="code"><pre><code> public ServiceMethod build() {
   //在ServiceMethod.build()方法里生成了我们需要的这个CallAdapter
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError(&quot;&#39;&quot;
            + Utils.getRawType(responseType).getName()
            + &quot;&#39; is not a valid response body type. Did you mean ResponseBody?&quot;);
      }
      responseConverter = createResponseConverter();
   
   //后面代码是对于接口中的注释(Annotation)做一些解析和处理
    ...
 }

private CallAdapter&lt;?&gt; createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            &quot;Method return type must not include a type variable or wildcard: %s&quot;, returnType);
      }
      if (returnType == void.class) {
        throw methodError(&quot;Service methods cannot return void.&quot;);
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        //实际上调用retrofit的callAdapter方法
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, &quot;Unable to create call adapter for %s&quot;, returnType);
      }
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

再来看看retrofit的callAdapter()方法

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19</code></pre></td>
<td class="code"><pre><code>public CallAdapter&lt;?&gt; callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}
 
public CallAdapter&lt;?&gt; nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, &quot;returnType == null&quot;);
    checkNotNull(annotations, &quot;annotations == null&quot;);

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i &lt; count; i++) {
      //通过遍历adapterFacories工厂来找到匹配我们的返回类型的adapter
      CallAdapter&lt;?&gt; adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
  ...
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

据说在adapter这里可以结合RxJava用 我不是很清楚 所以这里就是Retrofit默认的callAdapter 是一个call

是一个异步的方法

> /\* Asynchronously send the request and notify {@code callback} of its response or if an error *occurred talking to the server, creating the request, or processing the response.* /
>
> <figure class="highlight plain">
> <table>
> <colgroup>
> <col style="width: 50%" />
> <col style="width: 50%" />
> </colgroup>
> <tbody>
> <tr>
> <td class="gutter"><pre><code>1
> 2</code></pre></td>
> <td class="code"><pre><code>&gt; void enqueue(Callback&lt;T&gt; callback);
> &gt;</code></pre></td>
> </tr>
> </tbody>
> </table>
> </figure>

call.enqueue() 这个操作实际上是调用了ExecutorCallbackCall类里面的enqueue方法

<figure class="highlight java">
<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr>
<td class="gutter"><pre><code>1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64</code></pre></td>
<td class="code"><pre><code>//默认的adapterFactory就是这个
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }
 // 我们在上面通过这个类的get方法 拿到了一个 ExecutorCallbackCall 作为我们的call
  @Override
  public CallAdapter&lt;Call&lt;?&gt;&gt; get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter&lt;Call&lt;?&gt;&gt;() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public &lt;R&gt; Call&lt;R&gt; adapt(Call&lt;R&gt; call) {
        return new ExecutorCallbackCall&lt;&gt;(callbackExecutor, call);
      }
    };
  }

  static final class ExecutorCallbackCall&lt;T&gt; implements Call&lt;T&gt; {
    final Executor callbackExecutor;
    final Call&lt;T&gt; delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call&lt;T&gt; delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
    
  // 调用call.enqueue 是调用了这个方法
    @Override public void enqueue(final Callback&lt;T&gt; callback) {
      if (callback == null) throw new NullPointerException(&quot;callback == null&quot;);

      delegate.enqueue(new Callback&lt;T&gt;() {
        @Override public void onResponse(Call&lt;T&gt; call, final Response&lt;T&gt; response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp&#39;s behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException(&quot;Canceled&quot;));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call&lt;T&gt; call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
  }
  ...
}</code></pre></td>
</tr>
</tbody>
</table>
</figure>

ExecutorCallbackCall又把网络请求委托给了OkHttpCall去实现，然后拿到Response，再根据我们重写的OnResponse来处理请求的数据。

本来还想继续挖一下OkHttpCall enqueue的实现的，不过我觉得Retrofit到这里就差不多了。

顺着这条线，有两个很重要的设计模式(事实上是三个)：Builder Pattern,Proxy Pattern,Factory Pattern (工厂模式出现了 但是没有提)

这也仅仅只是Retrofit的冰山一角，还要继续努力啊，加油！

参考资料：掘金-Retrofit源码剖析
- <a href="javascript:void(0)" class="js-tag article-tag-list-link color4">Retrofit</a>
