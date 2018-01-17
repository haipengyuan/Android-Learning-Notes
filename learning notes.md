### Retrofit结合RxJava

```Java
// 创建接口
public interface GetRequest_Interface {
    @GET("url地址")
    Obserable<Bean> getCall();
}

Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("http://www.baidu.com")
    .addConverterFactory(GsonConverterFactory.create())  // 使用Gson解析
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())  // 支持RxJava
    .build();

// 创建接口实例
GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

Observable<Bean> observable = request.getCall();

// 发送网络请求
observable.subscribeOn(Schedulers.io())  // 在IO线程请求网络
    .observeOn(AndroidSchedulers.mainThread())  // 在主线程处理请求结果
    .subscribe(new Observer<Bean> {
			  
        // 发送请求后调用（无论请求是否成功）
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe");
        }
					
        // 发送请求成功后调用
        @Override
        public void onNext(Bean result) {
            // TODO 对result进行处理
        }

        // 请求成功后，onNext()后调用
        @Override 
        public void onComplete() {
            Log.d(TAG, "请求成功");
        }
        
        // 请求失败后调用
        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "请求失败");
        }
    });
```
