\# 背景



ReactiveX的思想对于处理客户端上的异步操作有很强的思维优势。你只需要表述清楚自己想做什么，而不需要说该怎么做。实际应用场景很多，我这里就只说一个很典型的实际例子。以使用为主，并无原理分析。



\# 例子



现在有这么一个功能要实现：



1. 开屏界面有最少3秒钟时间。

2. 请求接口A

3. 用A的结果请求接口B

4. 请求接口C

4. 以上四条都达成，跳转下一个界面

5. 接口请求失败则使用默认值，之后也要跳转界面

6. 如果在跳转界面前，用户点击了返回按钮，则取消一切操作



这个例子就用到了，并行、串行、组合、异常处理、生命周期绑定。是一个很综合的例子。



            //创建接口调用

        Observable&lt;String&gt; ob1 = Observable.create\(new Observable.OnSubscribe&lt;String&gt;\(\) {

            //创建一个请求城市数据的Observable

            @Override

            public void call\(Subscriber&lt;? super String&gt; subscriber\) {

                String city = httpClient.getCityByLoc\(334, 445\);

                System.out.println\("get city " + city\);

                if \(city == null\) {

                    //如果请求失败，则抛出异常

                    subscriber.onError\(new IllegalArgumentException\("city is null"\)\);

                } else {

                    //请求成功的话，直接调用下一步和完成

                    subscriber.onNext\(city\);

                    subscriber.onCompleted\(\);

                }

            }

        }\).flatMap\(new Func1&lt;String, Observable&lt;String&gt;&gt;\(\) {

            //通过上一步得到的城市信息进一步取得天气信息

            @Override

            public Observable&lt;String&gt; call\(String city\) {

                String temp = httpClient.getTemp\(city\);

                System.out.println\("get temp " + temp\);

                return Observable.create\(new Observable.OnSubscribe&lt;String&gt;\(\) {

                    @Override

                    public void call\(Subscriber&lt;? super String&gt; subscriber\) {

                        if \(temp == null\) {

                            //失败则抛出异常

                            subscriber.onError\(new IllegalArgumentException\("temp is null"\)\);

                        } else {

                            //成功则调用下一步和完成

                            subscriber.onNext\(temp\);

                            subscriber.onCompleted\(\);

                        }

                    }

                }\);

            }

        }\).doOnNext\(new Action1&lt;String&gt;\(\) {

            @Override

            public void call\(String s\) {

                //在执行成功之后进行保存数据的操作

                saveData\(s\);

            }

        }\).onErrorReturn\(new Func1&lt;Throwable, String&gt;\(\) {

            @Override

            public String call\(Throwable throwable\) {

                //如果过程中出现问题，在此处进行处理，不会在最终的subscriber当中出现

                System.out.println\(throwable.toString\(\)\);

                String defaultTmp = "0c";

                saveData\(defaultTmp\);

                return defaultTmp;

            }

        }\).subscribeOn\(Schedulers.io\(\)\);



        Observable&lt;String&gt; ob2 = Observable.create\(new Observable.OnSubscribe&lt;String&gt;\(\) {

            @Override

            public void call\(Subscriber&lt;? super String&gt; subscriber\) {

                String avatar = httpClient.getAvatar\(222\);

                System.out.println\("get avatar " + avatar\);

                if \(avatar == null\) {

                    subscriber.onError\(new IllegalArgumentException\("avatar is null"\)\);

                } else {

                    subscriber.onNext\(avatar\);

                    subscriber.onCompleted\(\);

                }

            }

        }\).doOnNext\(new Action1&lt;String&gt;\(\) {

            @Override

            public void call\(String s\) {

                saveData\(s\);

            }

        }\).onErrorReturn\(new Func1&lt;Throwable, String&gt;\(\) {

            @Override

            public String call\(Throwable throwable\) {

                System.out.println\(throwable.toString\(\)\);

                String defaultAddress = "baidu.com";

                saveData\(defaultAddress\);

                return defaultAddress;

            }

        }\).subscribeOn\(Schedulers.io\(\)\);



        //创建倒计时的Observable

        Observable&lt;Long&gt; ob3 = Observable.timer\(5, TimeUnit.SECONDS\);



        Observable.zip\(ob1, ob2, ob3, new Func3&lt;String, String, Long, String&gt;\(\) {

            //将以上操作进行zip操作，此方法在以上所有操作完成后执行

            @Override

            public String call\(String s, String s2, Long aLong\) {

                return s + s2 + aLong;

            }

        }\)

                .observeOn\(Schedulers.newThread\(\)\)

                .subscribe\(new Action1&lt;String&gt;\(\) {

                               @Override

                               public void call\(String s\) {

                                   //最终处理事件

                                   startNew\(s\);

                               }

                           },

                        new Action1&lt;Throwable&gt;\(\) {

                            @Override

                            public void call\(Throwable throwable\) {

                                System.out.println\("error " + throwable\);

                            }

                        },

                        new Action0\(\) {

                            @Override

                            public void call\(\) {

                                System.out.println\("finished"\);

                            }

                        }

                \);



    }

    

讲解：

1. doOnNext这个方法在一些教程上讲的不多，实际上很实用，使用这个方法，可以把Observable产生的数据进行处理，而不用非要到subscribe之后再处理。

2. onErrorReturen这个是处理异常的方法。通过中间把异常处理掉，可以防止出现一点错误，整个调用链都断裂的情况，出现小问题，自己搞个默认值处理掉，整个链条还是继续走。

3. zip操作符，通过这个操作符可以把很多的observable看成一组来对待。当这一组都完成的时候才继续往下走，而且流出了一个处理数据的方法。对于等待多个操作完成才能执行的调用，非常实用。

    

Android项目中的例子



    WebU51ServiceFactory webU51ServiceFactory = new WebU51ServiceFactory\(\);

        Activated activated = webU51ServiceFactory.createRetrofitService\(Activated.class\);



        Observable.zip\(

                activated.getActivated\("inviteFriend"\)

                        .doOnNext\(actBean -&gt; {

                            LogUtils.i\(tag, "get act bean " + actBean\);

                            int status = actBean.status != 0 && actBean.status != 1 ? 0 : actBean.status;

                            SharedPreferUtil.putInt\(SplashActivity.this,

                                    Constants.H5\_PREF\_FILE\_NAME, Constants.PREF\_KEY\_ENABLE\_FRIEND\_INVITE,

                                    status\);

                        }\)

                        .onErrorReturn\(throwable -&gt; {

                            LogUtils.e\(tag, "error " + throwable\);

                            return null;

                        }\),

                Observable.timer\(3, TimeUnit.SECONDS\),

                \(activatedBean, aLong\) -&gt; activatedBean

        \)

                .compose\(lifecycleProvider.bindUntilEvent\(ActivityEvent.DESTROY\)\)

                .subscribe\(activatedBean -&gt; {

                    LogUtils.i\(tag, "sub bean is " + activatedBean\);

                    U51SplashSDK.start\(SplashActivity.this\);

                }, throwable -&gt; LogUtils.e\(tag, "sub bean is " + throwable\), \(\) -&gt; LogUtils.i\(tag, "finished"\)\);

                

                

这里存在一个关键点：



\`compose\(lifecycleProvider.bindUntilEvent\(ActivityEvent.DESTROY\)\)\`



通过这一行调用，可以做到当Activity的destroy回调发生时，把整个链条的订阅者都取消关注。这样可以方式内存泄露。原理是这样的。



compose是通过一个转换，你给我一个observable我根据规则返回你转换后的另一个observable。一般可以用来给原有的observable搞一点通用操作，比如都设置成运行在io线程之类的。



这里的操作涉及到RxNavi库的原理。



\# switchMap的用法



    RxTextView.textChanges\(editText\).

                filter\(\(text\) -&gt; text.length\(\) &gt; 3\)

                .debounce\(300, TimeUnit.MILLISECONDS\)

                .switchMap\(\(text\) -&gt; Observable.create\(\(observer\) -&gt; {

                    if \(observer.isUnsubscribed\(\)\) {

                        return;

                    }

                    String response = null;

                    try {

                        response = httpUtil.get\(text.toString\(\)\);

                        observer.onNext\(response\);

                        observer.onCompleted\(\);

                    } catch \(InterruptedException e\) {

                        e.printStackTrace\(\);

                        observer.onError\(e\);

                    }

                }\).subscribeOn\(Schedulers.io\(\)\)\)

                .observeOn\(AndroidSchedulers.mainThread\(\)\)

                .compose\(provider.bindUntilEvent\(ActivityEvent.DESTROY\)\)

                .subscribe\(\(text\) -&gt; textView.setText\(text + ""\)\);

                



这是一个在搜索框进行搜索提示的实现，就是用户一遍输入，下边同时就提示出搜索关键字推荐的条目。

这个功能呢有个特点，就是用户输入很快，每输入一个字符都要进行一次http请求，但是在请求还没返回时，用户又输入了字符，那么上一次请求的返回已经没意义了。那么就扔掉。这就是switchMap的意思，就是产生了新的事件之后，之前还没完成的都扔掉。

