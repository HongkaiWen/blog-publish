---
title: 利用 RxJava 按时间窗缓存输出日志
date: 2018-06-24 16:55:50
tags: 雕虫小技
---


最近有一个小场景，利用websocket想browser实时输出日志，后端利用 [Apache Tailer](https://commons.apache.org/proper/commons-io/javadocs/api-2.4/org/apache/commons/io/input/Tailer.html) 监控文件变化，当有新内容时，实时想前端推送，由于推送过于频繁, 浏览器端出现假死的现象。模拟代码如下，[Apache Tailer](https://commons.apache.org/proper/commons-io/javadocs/api-2.4/org/apache/commons/io/input/Tailer.html) 每读取一行，就会调用一次MyListener的handle方法。



```java
public class TailFilePlay {

    public static void main (String args[]) throws InterruptedException {
        Tailer.create(new File("/tmp/test.log"), new MyListener());
        TimeUnit.SECONDS.sleep(3000);
    }

}

class MyListener extends TailerListenerAdapter {

    @Override
    public void handle(String line) {
        //send log to browser
        System.out.println(line);
    }
}
```



 考虑做一个buffer，每隔一段时间，积累一些日之后，向前端输出日志。



```java
public class TailFilePlay {

    private static final String LOG_FILE_PATH = "/tmp/test.log";

    public static void main (String args[]) throws InterruptedException {

        // 三秒钟输出一次日志
        Observable.create(emitter ->
            Tailer.create(new File(LOG_FILE_PATH), new MyListener(emitter))
        )
                .buffer(3, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .subscribe(TailFilePlay::print);



        // 模拟每隔一秒钟生成一行日志
        Observable.interval(1, TimeUnit.SECONDS).map(String::valueOf).map(log -> String.format(">>>   %s\n", log)).subscribe(TailFilePlay::monitorLog);


        TimeUnit.SECONDS.sleep(300);
    }

    private static void print(List<Object> logs){
        logs.forEach(System.out::println);
    }

    private static void monitorLog(String log) throws IOException {
        FileUtils.writeStringToFile(new File(LOG_FILE_PATH), log, "utf-8", true);
    }

}

class MyListener extends TailerListenerAdapter {

    ObservableEmitter<Object> emitter;

    public MyListener(ObservableEmitter<Object> emitter) {
        this.emitter = emitter;
    }

    @Override
    public void handle(String line) {
        emitter.onNext(line);
    }
}
```



虽然写了一坨代码，真正核心的代码就如下一句：

```java
// 三秒钟输出一次日志
Observable.create(emitter ->
                  Tailer.create(new File(LOG_FILE_PATH), new MyListener(emitter))
                 )
  .buffer(3, TimeUnit.SECONDS)
  .subscribeOn(Schedulers.io())
  .subscribe(TailFilePlay::print);
```

