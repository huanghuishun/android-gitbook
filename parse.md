# Parse

一種 BaaS 。

## RxParse

* https://github.com/yongjhih/RxParse

「找出我留言過的貼文」：

原本的 FindCallback 寫法：

```java
ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()).findInBackground(new FindCallback<ParseComment> {
    @Override
    public done(List<ParseComment> comments, ParseException e) {
        if (e != null) return;

        ParsePost.getQuery().whereContainedIn("comments", comments).findInBackground(new FindCallback<ParsePost>() {
            @Override
            public done(List<ParsePost> posts, ParseException e2) {
                if (e2 != null) return;

                // ...
            }
        });
    }
});
```

FindCallback 拆解寫法：

```java
getMyCommentedPosts(new FindCallback<ParsePost>() {
        @Override
        public done(List<ParsePost> posts, ParseException e) {
            if (e != null) return;

            // ...
        }
    });

public static void getMyCommentedPosts(FindCallback<ParsePost> findCallback) {
    getMyComments(new FindCallback<ParseComment> {
        @Override
        public done(List<ParseComment> comments, ParseException e) {
            if (e != null) {
                findCallback.done(null, e);
                return;
            }

            ParsePost.getQuery().whereContainedIn("comments", comments).findInBackground(findCallback);
        }
    });
}

public static void getMyComments(FindCallback<ParseComment> findCallback) {
    ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()).findInBackground(findCallback);
}

```

改成 Bolts Promise 寫法：

```java
ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()).findInBackground()
    .continueWithTask(new Continuation<List<ParseComment>, Task<List<ParsePost>>>() {
        public Task<List<ParsePost>> then(Task<List<ParseComment>> task) throws Exception {
            if (task.isFaulted()) return null;

            return ParsePost.getQuery().whereContainedIn("comments", task.getResult()).findInBackground();
        }
    }).onSuccess(new Continuation<List<ParsePost>>, Void>() {
        public Void then(Task<List<ParsePost>> task) throws Exception {
            List<ParsePost> posts = task.getResult();

            // ...
            return null;
        }
    });
```

Bolts Promise 拆解寫法：

```java
getMyCommentedPostsTask().onSuccess(new Continuation<List<ParsePost>>, Void>() {
    public Void then(Task<List<ParsePost>> task) throws Exception {
        List<ParsePost> posts = task.getResult();

        // ...
        return null;
    }
});

public static Task<List<ParsePost>> getMyCommentedPostsTask() {
    return getMyCommentsTask().continueWithTask(new Continuation<List<ParseComment>, Task<List<ParsePost>>>() {
        public Task<List<ParsePost>> then(Task<List<ParseComment>> task) throws Exception {
            if (task.isFaulted()) return null;

            return ParsePost.getQuery().whereContainedIn("comments", task.getResult()).findInBackground();
        }
    });
}

public static Task<List<ParseComment>> getMyCommentsTask() {
    return ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()).findInBackground();
}

```

RxParse 寫法：

```java
ParseObservable.find(ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()))
    .toList()
    .flatMap(comments -> ParseObservable.find(ParsePost.getQuery().whereContainedIn("comments", comments)))
    .subscribe(posts -> {});
```

RxParse 拆解寫法：

```java
getMyCommentedPosts().subscribe(posts -> {});

public static Observable<ParsePost> getMyCommentedPosts() {
    return getMyComments().toList().flatMap(comments -> ParseObservable.find(ParsePost.getQuery().whereContainedIn("comments", comments)));
}

public static Observable<ParseComment> getMyComments() {
    return ParseObservable.find(ParseComment.getQuery().whereEqualTo("from", ParseUser.getCurrentUser()));
}

```

## RxParse 測項

```java
  @Test
  public void testParseObservableAllNextAfterCompleted() {
    ParseUser user = mock(ParseUser.class);
    ParseUser user2 = mock(ParseUser.class);
    ParseUser user3 = mock(ParseUser.class);
    List<ParseUser> users = new ArrayList<>();
    when(user.getObjectId()).thenReturn("" + user.hashCode());
    users.add(user);
    when(user2.getObjectId()).thenReturn("" + user2.hashCode());
    users.add(user2);
    when(user3.getObjectId()).thenReturn("" + user3.hashCode());
    users.add(user3);
    ParseQueryController queryController = mock(ParseQueryController.class);
    ParseCorePlugins.getInstance().registerQueryController(queryController);

    Task<List<ParseUser>> task = Task.forResult(users);
    when(queryController.findAsync(
            any(ParseQuery.State.class),
            any(ParseUser.class),
            any(Task.class))
    ).thenReturn(task);
    when(queryController.countAsync(
      any(ParseQuery.State.class),
      any(ParseUser.class),
      any(Task.class))).thenReturn(Task.<Integer>forResult(users.size()));

    ParseQuery<ParseUser> query = ParseQuery.getQuery(ParseUser.class);
    query.setUser(new ParseUser());

    final AtomicBoolean completed = new AtomicBoolean(false);
    rx.parse.ParseObservable.all(query)
        //.observeOn(Schedulers.newThread())
        //.subscribeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<ParseObject>() {
        @Override public void call(ParseObject it) {
            System.out.println("onNext: " + it.getObjectId());
            if (completed.get()) {
                fail("Should've onNext after completed.");
            }
        }
    }, new Action1<Throwable>() {
        @Override public void call(Throwable e) {
            System.out.println("onError: " + e);
        }
    }, new Action0() {
        @Override public void call() {
            System.out.println("onCompleted");
            completed.set(true);
        }
    });

    try {
        ParseTaskUtils.wait(task);
    } catch (Exception e) {
        // do nothing
    }
  }

  @Test
  public void testParseObservableFindNextAfterCompleted() {
    ParseUser user = mock(ParseUser.class);
    ParseUser user2 = mock(ParseUser.class);
    ParseUser user3 = mock(ParseUser.class);
    List<ParseUser> users = new ArrayList<>();
    users.add(user);
    users.add(user2);
    users.add(user3);
    ParseQueryController queryController = mock(ParseQueryController.class);
    ParseCorePlugins.getInstance().registerQueryController(queryController);

    Task<List<ParseUser>> task = Task.forResult(users);
    when(queryController.findAsync(
            any(ParseQuery.State.class),
            any(ParseUser.class),
            any(Task.class))
    ).thenReturn(task);

    ParseQuery<ParseUser> query = ParseQuery.getQuery(ParseUser.class);
    query.setUser(new ParseUser());

    final AtomicBoolean completed = new AtomicBoolean(false);
    rx.parse.ParseObservable.find(query)
        //.observeOn(Schedulers.newThread())
        //.subscribeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<ParseObject>() {
        @Override public void call(ParseObject it) {
            System.out.println("onNext: " + it);
            if (completed.get()) {
                fail("Should've onNext after completed.");
            }
        }
    }, new Action1<Throwable>() {
        @Override public void call(Throwable e) {
            System.out.println("onError: " + e);
        }
    }, new Action0() {
        @Override public void call() {
            System.out.println("onCompleted");
            completed.set(true);
        }
    });

    try {
        ParseTaskUtils.wait(task);
    } catch (Exception e) {
        // do nothing
    }
  }
```

* https://github.com/yongjhih/RxParse/blob/master/rxparse/src/test/java/com/parse/ParseObservableTest.java

## Cloud Coding

Before, cloud code 原本的寫法：

```js
Parse.Cloud.define("signInWithWeibo", function (request, response)) { // 註冊 RPC 名稱
  //console.log(request.user + request.params.accessToken); // 取參數，對應 android 手機端 ParseCloud.callFunctionInBackground("signInWithWeibo", Map<K, V>);
  // if (where) response.success(obj); // 回傳資料
  // else response.error(error); // 回報錯誤
}
```

After, 1. 改善註冊 RPC 的方法：

```js
defineCloud(signInWithWeibo);

function signInWithWeibo(request, response) {
  // ...
}

function defineCloud(func) {
  Parse.Cloud.define(func.name, func); // func.name 可以取得 func 的函式名稱
}
```

After, 2. 將 response 機制隱藏，轉成對應的 Promise ：

```js
function promiseResponse(promise, response) {
  promise.then(function (o) {
    response.success(o);
  }, function (error) {
    response.error(error);
  })
}

/**
 * Returns the session token of available parse user via weibo access token within `request.params.accessToken`.
 *
 * @param {Object} request Require request.params.accessToken
 * @param {Object} response
 * @returns {String} sessionToken
 */
function signInWithWeibo(request, response) {
  promiseResponse(signInWithWeiboPromise(request.user, request.params.accessToken, request.params.expiresTime), response);
}

/**
 * Returns the session token of available parse user via weibo access token.
 *
 * @param {Parse.User} user
 * @param {String} accessToken
 * @param {Number} expiresTime
 * @returns {Promise<String>} sessionToken
 */
function signInWithWeiboPromise(user, accessToken, expiresTime) {
  // ...
}
```

## Parse.Cloud.httpRequest

回傳 `{Promise<HTTPResponse>}` ，所可以接龍：

```js
/** @returns {Promise<String>} email */
function getEmail(accessToken) {
  // GET https://api.weibo.com/2/account/profile/email.json?access_token={access_token}
  // 這裡嚴格分離的 params 方式, 好處是未來改成 POST 也統一寫法
  return Parse.Cloud.httpRequest({
    url: "https://api.weibo.com/2/account/profile/email.json",
    params: {
      access_token: accessToken
    }
  }).then(function (httpResponse) {
    return JSON.parse(httpResponse.text)[0].email; // [ { email: "abc@example.com" } ]
  });
}
```

## Promise

* `Parse.Promise.as("Hello")`

```js
Parse.Promise.as("Hello").then(function (hello) {
  console.log(hello);
});
```

* `Parse.Promise.when(helloPromise, worldPromise)`

```js
var helloPromise = Parse.Promise.as("Hello");
var worldPromise = Parse.Promise.as(", world!");
Parse.Promise.when(helloPromise, worldPromise).then(function (hello, world) {
  console.log(hello + world);
});
```

flat and zip:

```js
var helloPromise = Parse.Promise.as("Hello");
var worldPromise = Parse.Promise.as(", world!");
helloPromise.then(function (hello) {
  return Parse.Promise.when(Parse.Promise.as(hello), worldPromise);
}).then(function (hello, world) {
  console.log(hello + world);
});
```

Error handling:

```js
/**
 * Returns email.
 *
 * @param {String} accessToken
 * @returns {Promise<String>} email
 */
function getEmailAlternative(accessToken) {
    return getEmail(accessToken).then(function (email) {
        if (!email) return Parse.Promise.error("Invalid email");

        return Parse.Promise.as(email);
    }, function (error) {
        return getUid(accessToken).then(function (uid) {
            return Parse.Promise.as(uid + "@weibo.com");
        });
    });
}

/**
 * Returns email
 *
 * @param {String} accessToken
 * @returns {Promise<String>} email
 */
function getEmail(accessToken) {
    return Parse.Cloud.httpRequest({
        url: "https://api.weibo.com/2/account/profile/email.json",
        params: {
            access_token: accessToken
        }
    }).then(function (httpResponse) {
        return JSON.parse(httpResponse.text)[0].email; // [ { email: "abc@example.com" } ]
    });
};

/**
 * Returns uid.
 *
 * @param {String} accessToken
 * @returns {Promise<String>} uid
 */
function getUid(accessToken) {
    return Parse.Cloud.httpRequest({
        url: "https://api.weibo.com/2/account/get_uid.json",
        params: {
            access_token: accessToken
        }
    }).then(function (httpResponse) {
        return JSON.parse(httpResponse.text).uid; // { uid: 5647447265 }
    });
}
```

ref.

* https://gist.github.com/yongjhih/e196e01fc7da9c03ce7e
