# 經典問題 - 指引你「這該怎麼弄？」

這裡是從問題指引你到哪裡可以找到答案。或者曾經有人問過我的問題，這是我的備忘錄。

## gradle 如何只測試單項?

```bash
./gradlew testDebug --tests='*.<testname>'
```

## 利用回傳空列表 `Collections.emptyList()` 來減少不必要的 null `List` 檢查

* 不再 `return new ArrayList<>();` 改用 `Collections.emptyList()`
* 不再 `return new HashSet<>();` 改用 `Collections.emptySet()`

不用檢查：

```java
List<ResolveInfo> infos = packageManager.queryIntentActivities(intent, flags);
// infos == null? 像是官方文件就寫清楚肯定會回傳一個空列表，所以我們可以不用檢查 null
for (info : infos) {
  System.out.println(info);
}
```

## 要怎麼寫非同步？

* Thread
* HandlerThread
* AsyncTask

## 要怎麼避免 Callback Hell

使用 RxJava 或者 Bolts 等具 promise 架構來整平

## 按鈕不想讓別人連按，要怎麼寫 debounce？

## Restful client 要怎麼寫？用哪套？

* Retrofit
* Volley

See Also: ...

## ImageLoader 要用哪套？

每套有其特性，一般短小精幹可用 picasso.

* 大量客製調整 AUIL
* Glide 效能考量

See Also: ...

## Json 轉物件 要用哪套？

* LoganSquare 目前是非 reflection 方式，所以應有較高的效能表現。
* Jackson
* Gson

See Also: ...

## 我該從哪得知 Android App 開發的資訊？

前端 UI 、美工 Art 、平面字體挑選、換場動畫設計。

App 工程開發。

函式庫資訊

## 沒有回傳值的 Callback 用 RxJava 要怎麼寫？

> 想要問一下，我有一個 init() function 可能會做很久，所以我裡面開了一個 thread，然後 ui 傳入一個 callback 當我 init 完成之後我 callback 他。

> 如果想要改成 RxJava 的架構，所以我 init 回傳了 Observable<Boolean>，這裡 Boolean 其實有點多餘，因為我只是要單純的通知 ui 說我做完了，感覺有點硬是要用 Observable 來做。


如果你用 `AsyncTask<I, N, O>` 也會有類似的問題。

```java
Observable<Void> initAsyncObs = Observable.defer(() -> Observable.just(initSync()))
  .subscribeOn(Schedulers.io())
  .observeOn(AndroidSchedulers.mainThread());

Void initSync() {
  // ...
  return null;
}
```

## Background Thread API 用 RxJava `.observeOn(AndroidSchedulers.mainThread());` 好嗎？

> api 回傳 Observable ，然後由 ui subscribe 來取得資料更新畫面
> 而 Observable 先在 api 層設定好要在 background thread 執行，讓 ui 不用去管這段
不知道這樣的觀念是否正確，或是有更好的架構？


Android Threading/Scheduling 很多不只是 background 的問題，還有 lifecycle 問題，LoaderManager 其實也有在處理 lifecycle 問題。

所以 Ui 還是要自己善用 `AppObservable.bindActivity(activity, obs)`, `AppObservable.bindFragment(fragment, obs)`, `ViewObservable.bindView(view, obs)` 這些就是負責處理在 Android 上的 LifeCycle 以及 Threading 問題。 (rxandroid)


## Ui Adapter 要用的，RxJava 回傳 `Observable<List<E>>` 還是 `Observable<E>`

> 感覺回傳 List 就很鳥，但是如果 ui 就是要收集到全部的資料在一次刷新，回傳 list 對 ui 來說好像比較方便

為了 UI adapater 方便，是否傳遞 `Observable<List<E>>`？

首先在函式庫角度來看，它應該只做到單純性、重用性、通用性、彈性、操作性的最大化。

* 傳遞 `Observable<List<E>>` 的劣勢，很明顯的是大部分的 Rx Operator 都無法直接使用，也就是我說的操作性很低。所以一般狀況下，不會傳遞無操作性的介面。

在這個前提下，自然會導出 -> 函式庫不提供 `Observable<Collection/Iteratable> `。

接著是下一層的問題 ，那 Ui 該怎麼辦？很簡單，請服用： `Observable.toList()`, `Observable.buffer()`, etc.

* See Also: https://kaif.io/z/compiling/debates/drg0Cf6oGj/dsCeOiO6Gz

## multidex 跟 proguard 擇一執行哪一個速度比較快？

先不管題目的合理性。
答案是 proguard 比較快。

1. 理論上，經過 proguard 之後，code size 的縮減，所以啟動程式碼載入時間會縮短。
2. multidex 還會變慢，因為 65k 超出去的部份是啟動之後，動態載入的，你知道的，就是那個 reflection ，想當然爾會更慢。

而提問者其實後來要問的是「編譯」而不是「執行」，如果是編譯，multidex 會快些。而我們本來在開發上就是 release build 才會跑 proguard minify ，所以倒不用擔心。

## abstract class 與 interface 有什麼不同？

坊間很多地方可以看到這個問題，
不推薦給初學者從這個問題著手去了解 abstract class 與 interface 存在的意義，因為這個問題誤導初學者。
但是作為一個面試題目，卻不失誘導應試者發揮看法。

* abstract class 是一個功能不完整的半成品
* interface 是待完成功能的集合體

讓我們舉個例子對照來促進思考：

Before:

```java
abstract class BaseButton {
    public abstract onClick(View view);
}
```

After:

```java
interface OnClickListener {
    public onClick(View view);
}

abstract class BaseButton2 implements OnClickListener {
}
```

也就是任何 abstract class 你都可以抽出一個 interface ，這樣就排除了一個誤導：abstract class 與 interface 不應該是對等的比較，也就是這是個偽命題。應該要問的是「哪時候使用 abstract class 還是 interface + class」。

* *題外話: 這也就是筆者常拿來質疑 abstract class 修飾子的存在意義*
* *解決多重繼承問題：如果你不知道什麼是「多重繼承」，就不需要去了解 interface 是如何解決的，因為對你來說是不存在的問題，很有可能因此混淆*
* *「abstract class 不能 new 」並不是一個事實，如果是一個事實，那麼 interface 也不能 new ，所以並不是不同之處。而任何物件在建構時，都務必實現所有功能才能建構，abstract class 你可以：`new AsyncTask() { @Override xxx() ... }`，interface 也可以 `new OnClickListener() { @Override xxx() ... }`*
* *「interface 不能繼承，只有 abstract class 可以繼承」這也不是事實，這應該是想要說 abstract class 通常被預期來拿來繼承使用，否則只能建構時馬上實做來使用*

abstract class 被繼承或者建構時，必須實做所有未完成的方法，否則依然還是一個 abstract class。class + interface 在建構時，不需要立即實現，通常是以事後賦予的方式呈現。

abstract class:

```java
abstract class BaseButton {
    public abstract onClick(View view);
}

BaseButton button = new BaseButton() {
    @Override public onClick(View view) {
        System.out.println("click: " + view);
    }
};
```

class + interface:

```java
class Button {
    OnClickListener mOnClickListener;

    public void setOnClickListener(OnClickListener onClickListener) {
        mOnClickListener = onClickListener;
    }

    private void click(View view) {
        mOnClickListener.onClick(view);
    }
}

Button button = new Button();

button.setOnClickListener(new OnClickListener() {
    @Override public onClick(View view) {
        System.out.println("click: " + view);
    }
});
```

abstract class 特性：

* 依賴性：使用時必須實做，類似建構子的參數，建構依賴
* 流程引用性：abstract method 是否都要被 parent 引用？

習性：

* interface + class: 跨部門的功能，你就應該抽出 interface 來請大家實現。對老手來說，是一個拉平繼承樹或者倒裝繼承樹的手法
* 當你寫了一個類別希望給大家繼承來用，但是建構時，怕你忘記做一些前置作業，就會使用 abstract class

以客觀來說，abstract class 用於分段實做。

以慣例來說，parent 制定大部分流程，必要流程請子嗣實現。

```java
class DownloadTask extends AsyncTask<String, Long, File> {
    @Override protected File doInBackground(String... urls) {
        try {
            HttpRequest request =  HttpRequest.get(urls[0]);
            File file = null;
            if (request.ok()) {
                file = File.createTempFile("download", ".tmp");
                request.receive(file);
                publishProgress(file.length());
            }
            return file;
        } catch (HttpRequestException exception) {
            return null;
        }
    }

    @Override protected void onProgressUpdate(Long... progress) {
        Log.d("DownloadTask", "Downloaded bytes: " + progress[0]);
    }

    @Override protected void onPostExecute(File file) {
        if (file != null) {
            Log.d("DownloadTask", "Downloaded file to: " + file.getAbsolutePath());
        } else {
            Log.d("DownloadTask", "Download failed");
        }
    }
}

new DownloadTask().execute("http://google.com");
```

以另一個慣例來說，parent 制定流程，操作的實體由子嗣實現。

```java
class SimpleActivity extends BaseActivity {
    @Override public int getContentView() {
        return R.layout.main;
    }
}

abstract class BaseActivity extends Activity {
    public abstract int getContentView();

    @Override public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getContentView());
    }
}
```


## 什麼是泛形(generic) 以及 `<? super XX>` 與 `<? extends XX>` 的使用時機

我們可以用來簡化轉型步驟：

Before:

```java
interface OnClickListener {
    void onClick(View view);
}

MyActivity extends Activity {
    // ...
    imageView.setOnClickListener(new OnClickListener() {
        @Override public void onClick(View view) {
            ImageView iv = (ImageView) view;
            iv.setImageResource(R.drawable.clicked);
        }
    });
}
```

After:

```java
interface OnClickListener<T extends View> {
    void onClick(T view);
}


class ImageView extends View {
    OnClickListener<ImageView> mOnClickListener;

    public void setOnClickListener(OnClickListener<ImageView> onClickListener) {
        mOnClickListener = onClickListener;
    }
}

class MyActivity extends Activity {
    // ...
    imageView.setOnClickListener(new OnClickListener<ImageView>() {
        @Override public void onClick(ImageView iv) {
            iv.setImageResource(R.drawable.clicked);
        }
    });
}
```

`<? extends T>` 的使用時機:

```java
interface Collection<E> ... {
    E get(int index);
    void addAll(Colletion<? extend E> items);
}
```

`<? super T>` 的使用時機:

```java
```


<!--
## `<? super T>` 的使用時機

## `<? extends T>` 的使用時機
-->

## 沒有 Http Library 直接操作 Socket 會怎麼做？

或許要問的是，對 Socket 本身的看法吧，我把 Socket 當作一種檔案流，實際上在 POSIX 系統就真的是檔案(其實什麼都檔案)，你先建立一個 listening socket file by ip/port 去聽 (bind and listen)，當然人要丟資料的時候，會開啟一個獨立的 connection socket file (accept)，然後就可以開始讀檔了 `while (inputStream)`。


但是萬一後面還有人要進來，就沒法聽到了怎麼辦？

```java
var binder = bind(address);
while (binder.listen()) { // blocking until requested
    new Thread(() -> { // 馬上開一個 thread 去處理讀資料流
        var socket = binder.accept();
        read(socket);
        // business logic
        write(socket);
        }));
    // 得以繼續等待下一個 request
}
```

下一個問題，如果很多 reuqest 進來，會不會來不及 `new Thread` 然後下一位的 request 沒聽到怎麼辦？

或許可以一開始就開 multithread 去 listent and accept 額定一個 thread 量，為了要重用 thread 還要開個 thread pool 。

而客戶端的部份，一樣主要 Socket to In/OutputStream ，只是因為網路存取通常是長時間存取，所以通常會開一個 Thread 出去避免 blocking main-thread ，那麼這種 io/net thread 為了要重用，也有 io/net thread pool 。

## Multi-Process 與 Multi-Thread 選擇

如果是只為了 non-blocking：

像是單純的網路服務，process 初始化的低消較大，如果用 fork 更開了副本，也有損耗問題，不過好處是 process management 獨立性/隔離性，萬一發生什麼問題不會影響到其他人。

而如果有需要交換資料且頻繁，就用 Multi-Thread 吧，否則 Multi-Process 你交換資料就只能 IPC 基本上都是透過 socket ，不然你就跟 Android 一樣做一個 amem 做 binder 幫你 marshall/unmarshall/de/serialize 來減輕低消。

像是 Android Service 宣告，會幫裝在 Process ，透過 AIDL ，generating Stub ，讓 binder 做資料交換，只要宣告 interface 即可。

系統應該要有一個機制去降低 fork process 低消，例如 COW 技術，在開啟新的 process 的時候，並沒有即時產生副本，頂多產生副本 refs ，當有人寫入的時候才進行實體副本。

## Android Observable 與 Button

Observable 表示可被觀測，所以你可以塞一個觀察者 Observer 進去，聆聽一些變動。而 Button 可以 binding 一些變動行為。

一般 Android 實務都是你的某個 ContentProvider 實現了 Observable ，所以你當然可以註冊 Observer 進去，一般也繼承 ContentObsever，為什麼要繼承 ContentObserver ？因為可以給 key 進去，讓 ContentProvider calling back by key 阿。

以 RxJava 撰寫風格來說：

```
getProviderSubject(key).asObservable().subscribe(changed -> changed ? button.on() : button.off());
```

稍微接近一點 Android 的寫法，但是我習慣接龍(Fluent)：

```java
ContentResolvers.select(key).subscribe(changed -> changed ? button.on() : button.off());
```

ref. https://github.com/yongjhih/content-observables

## Activity Lifecycle 的排定是為什麼？

為了應用以及不傷身體，為了節省資源回收的時候狀態要告知，才能夠不傷身體。應用則是一些狀態告知可以應用 onPause/onResume ，較多的排定也是為了靈活手機的應用以及硬體限制需要回收的告知對策。

## 隨便 kill 無所謂？

kill 照理說要作到不傷身體，你就該好好告知你的客戶(application)，也有很多 signal 可以 trap ，所以安排了很多告知方法：lifecycle。
除非你是用 -9 signal ，但是萬一因此傷身，這個應該是發起者後果自負。

## LRUCache

像是行車記錄器會把最後的刪除。如果要實現，沒多想，直接用 mod index 。

不過這樣讀取就太慢了，如果仍然使用 Map 來裝，要有順序 order index ， on-put dereference 。

* p.s. *當天路上忽然想到，好像沒人糾正說講錯，不叫做 push/pop ，有點壞心啊，腦袋關聯想左推右彈，嘴巴稱之 push/pop，但是這是錯誤的用詞，也許因為我之前寫過一個 SimpleLruCache 給 [simple-parse/f082efe2](https://github.com/yongjhih/simple-parse/commit/f082efe2ce46f40b8b7cd80a50d3d65652fc4ad7) 用，裡面一堆 map.put() ？不過回想起來這好蠢喔。如果要寫，核心想法就是 mod index ，反正沒 ref 就 gc，不過讀取應該就慢爆了吧*

## Media Server

froyo 以前叫做 opencore ，後來叫做 stagefright 。年代久遠，基本上還是要回去看我 [patches](https://drive.google.com/open?id=0B0Q2kL5yaBdRNmtnREFZWUVvS1k) 才會會想起在幹麻了(froyo 的 patches 似乎已經丟失)。

反倒是 camera layer 有點印象, camera service , jni , libcamera, hal mm-camera, v4l2

## Java 語言使用上的問題

問題挺多的，就看我寫的一些雞肋函式庫就知道了，不然看看 kotlin 那篇介紹也可以，像是 lambda , pojo getter/setter , nullable/optional, return-type overloading, generic infer

## 會用 jenkins 嗎？

* 2011 年開始使用 Jenkins 集合 Gerrit 自動 verify ： https://docs.google.com/document/d/1_H2AW-jrpugcq29h8-Uq4ViNnFPSfY9b1zSy3Cttg6w/
* 2011-2013 on-push/on-upload 自動建置 aosp 與其 cts
* 2013-2016 on-push 自動建置 apk

## 對於 devops 的看法

* 從開發流程到檢驗到佈署的自動化/最佳化
* 在這之前你應該為了彈性抽換，會有虛擬化/隔離化 e.g. lxc
* 到資源管理 scheduling
* 到成本管理

## 測試

優先把重用性高的拆出去成獨立的 lib ，既然已經拆出去，解藕，那麼 Unit Test 就會很好寫了。而商業邏輯在新創公司經常性變動，也就是程式區段生命週期短的部份就不用刻意寫自動化測試，直到穩定成為週期長的程式區段，再進行自動化測試的撰寫。

## 優先做出東西還是優先找最佳解

* 優先做出雛型 - 這要看實際耗時問題，雖然希望優先做出雛型，但是其實往往工程師本位，還是會希望找出正解，否則怎麼會用 RxJava/retrofit/ORM？一開始的 restful client 需求肯定很低的，Model 也少，然後 rx 來做 pipeline 也不多，會什麼要導入 rxjava/retrofit/ORM ? 直接 `db.exec(sql)` 不就好了，直接 `httpclient.get(url)` 取幾個 json 欄位，很快就可以寫好。
* 不過找最佳解的途中，也許會發現那個東西根本有問題，也就是在釐清問題，收斂問題。

## WindowManager

現在我還沒什麼印象，只記得 Phone xx Manager ，跟改過 KeyGuard

## ViewHirarchy TouchEvent

從樹根傳遞，像是 nested scrollview 常需要 Parent Reqesut disallow Inercept

## 攤平 callback hell 的一些想法

在談論 Rx 的時候，常常跟朋友提到 callback hell 問題與解法

Before:

```java
loginFacebook(fbToken -> {
  loginParse(lastFbToken, parseToken -> {
    loginOctory(lastParseToken, octoryToken -> {
    });
  });
});
```

我對於 callback hell 想要攤平，其實比較簡單的概念就是 register 模式：

After:

```java
Tasks<String> tasks = Tasks.create();
tasks.add(task -> loginFacebook(fbToken -> { task.next(fbToken); }));
tasks.add(task -> loginParse(task.get(), parseToken -> { task.next(parseToken); }));
tasks.add(task -> loginOctory(task.get(), octoryToken -> { task.next(octoryToken); }));
tasks.execute();
```

只有型別的部份比較麻煩，所以通常透過 generic infer 來做：

After:

```java
Task.from(() -> loginFacebook()).then(profile -> loginParse(profile)).then(profile -> loginOctory(profile));
```

如果這樣寫完，其實就完成了 promise 規範了。
