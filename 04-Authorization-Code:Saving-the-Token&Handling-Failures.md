# 授权码：保存令牌&处理失败

如果之后我们要在Brent这边处理别的API请求会怎么样？我们应该在哪里存储访问令牌？

## 将访问令牌存储在某处

有些令牌持续一个或两个小时，这种的适合被存储在会话中。另外的一些是长期的令牌，例如，Facebook提供一个60天有效期的令牌，这种更适合存储在数据库中。总之，存储令牌将会避免我们要求用户必须重新授权。

在我们的应用中，我们将把它存储在数据库中：
```javascript
// src/OAuth2Demo/Client/Controllers/CoopOAuthController.php

public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...
    $meData = json_decode($response->getBody(), true);

    $user = $this->getLoggedInUser();
    $user->coopAccessToken = $accessToken;
    $user->coopUserId = $meData['id'];
    $this->saveUser($user);

    // ...
}
```

这段代码特指我的应用，但结尾的结果是我为当前已经授权的用户在用户表上更新了`coopAccessToken`列，同时我也保存了`coopUserId`，因为大多数API调用会在URI中需要它。

## 记录过期时间

我们也可以存储令牌即将过期的时间，我会创建一个`DateTime`对象来代表过期时间，我们可以稍后在发起API请求之前检查这个，如果令牌过期了，我们需要再次把用户发向授权处理程序：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...
    $expiresIn = $responseArr['expires_in'];
    $expiresAt = new \DateTime('+'.$expiresIn.' seconds');
    // ...

    $user = $this->getLoggedInUser();
    $user->coopAccessToken = $accessToken;
    $user->coopUserId = $meData['id'];
    $user->coopAccessExpiresAt = $expiresAt;
    $this->saveUser($user);

    // ...
}
```

再一次地，这段代码特指我的应用，但是末尾结果就是为当前用户更新数据库中的列。当我们尝试它的时候，它运行到了我们的`die`声明。但是如果你前往主页，用户下拉列表显示COOP用户ID已经保存了！

## 当授权失败时

但当用户拒绝授权我们的应用时该怎么办？如果那个发生了，OAuth服务器会将用户导回我们的`redirect_uri`。如果我们从首页再次开始，但是在COOP上拒绝了访问，我们就会遇到这个情况。页面爆炸了，因为我们的针对`/token`的请求没有返回一个访问令牌。实际上，COOP没有在跳转的URL中包含`code`查询参数。

这就是取消授权的样子：没有授权码。

很不幸，我们不能只设想用户会授权我们的应用。当这个事情发生时，`code`参数会消失，但是OAuth服务器应当包含少量额外的参数来解释什么出错了。这些通常被称为`error`和`error_description`。让我们抓取这些并将它们传递到一个我已经准备好的模版中：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // equivalent to $_GET['code']
    $code = $request->get('code');

    if (!$code) {
        $error = $request->get('error');
        $errorDescription = $request->get('error_description');

        return $this->render('failed_authorization.twig', array(
            'response' => array(
                'error' => $error,
                'error_description' => $errorDescription
            )
        ));
    }

    // ...
}
```

当我们再次尝试那个流程，我们会看到一个漂亮的消息。你可以在你的应用中做任何你想做的，只需要确保你已经处理了用户会拒绝你的应用请求的可能。

这些错误应当被OAuth服务器归档，标准的集合包括"temporarily_unavailable"，"server_error"，"access_denied"。

## 当拉去访问令牌失败时

还有另外一个可能的失败点：当请求离开`/token`的时候，如果响应中没有一个`access_token`域会怎么样？在正常环境中，这的确不可能发生，但是让我们渲染一个不同的错误模版以防它发生了。不要担心我传递给模版的变量，我只是想要传递更多的信息以便于我们可以看到问题是什么：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...
    $request = $http->post('/token', null, array(
        // ...
    ));

    $response = $request->send();
    $responseBody = $response->getBody(true);
    $responseArr = json_decode($responseBody, true);

    // if there is no access_token, we have a problem!!!
    if (!isset($responseArr['access_token'])) {
        return $this->render('failed_token_request.twig', array(
            'response' => $responseArr ? $responseArr : $response
        ));
    }
    // ...
}
```

再次尝试整个流程，但是这次通过应用的授权，在第一次，它当然会正常工作，但是如果你刷新一下，你会看到错误，`code`参数存在，但是它失效了，向`/token`的请求失败了。

## 成功后跳转

直到现在，我们在代码底部有一个丑陋的`die`声明来处理OAuth跳转，你在这里真正应该做的事是将用户跳转到一些别的页面，我们的工作到此已经完成了，所以我们想要帮助用户继续在我们的网站上浏览：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...

    // redirect back to the homepage
    return $this->redirect($this->generateUrl('home'));
}
```

在我们的应用中，这个代码简单地将我们导向了首页，就像那一样，我们做到了！这就是授权认证类型，它有不同的两个步骤：
1.  首先，使用`authorize`端点，你的应用的`client_id`，一个`redirect_uri`，和你想要的权限范围将用户导向OAuth服务器。URL和参数的外观可能有所不同，但是思路是相同的。

2. 在授权了我们的应用后，OAuth服务器带着一个`code`参数跳转回我们的站点上的一个URL，我们可以使用这个，和我们的`client_id`和`client_secret`一起来向`/token`端点发起请求，现在我们就得到访问令牌了。

最后，让我们使用它来数鸡蛋！

## 数鸡蛋

在首页，仍旧有"Authorize"按钮，但是现在我们有了一个用于用户的访问令牌，我们现在不需要它了。显示这个页面的模版在`views/dashboard.twig`，我已经传递了一个当前认证过的用户对象：`user`变量到这里。如果用户有一个`coopUserId`存储在数据库中，让我们隐藏掉"Authorize"链接：

```javascript
{# views/dashboard.twig #}
{# ... #}

{% if user.coopUserId %}

{% else %}
    <a class="btn btn-primary btn-lg" href="{{ path('coop_authorize_start') }}">Authorize</a>
{% endif %}
```

如果我们没有`coopUserId`，让我们添加一个链接以便用户可以点击它来统计每日蛋量。不要担心你对这里的代码不熟悉，我仅仅生成了一个跳转到一个我已经设置好的新页面的链接：

```javascript
{# views/dashboard.twig #}
{# ... #}

{% if user.coopUserId %}
    <a class="btn btn-primary btn-lg" href="{{ path('count_eggs') }}">Count Eggs</a>
{% else %}
    <a class="btn btn-primary btn-lg" href="{{ path('coop_authorize_start') }}">Authorize</a>
{% endif %}
```

当我们刷新时，我们会看到一个新链接，点击它会给我们一个新的todo消息。打开`src/OAuth2Demo/Client/Controllers/CountEggs.php`，它是新页面背后的代码。

## 发起蛋量统计API的请求

从`CoopOAuthController`拷贝`/api/me`代码开始，由于`eggs-count`端点需要POST，所以将方法从`get`改为`post`：
```javascript
// src/OAuth2Demo/Client/Controllers/CountEggs.php
// ...

class CountEggs extends BaseController
{
    // ...
    public function countEggs()
    {
        $http = new Client('http://coop.apps.knpuniversity.com', array(
            'request.options' => array(
                'exceptions' => false,
            )
        ));

        $request = $http->post('/api/me');
        $request->addHeader('Authorization', 'Bearer '.$accessToken);
        $response = $request->send();
        $meData = json_decode($response->getBody(), true);

        die('Implement this in CountEggs::countEggs');

        return $this->redirect($this->generateUrl('home'));
    }
}
```

现在我们要访问的端点是`/api/USER_ID/eggs-count`。幸运的是，我们已经在数据库中保存了当前登录用户的COOP用户id和访问令牌。通过使用我们的应用的`$this->geLoggedInUser()`方法来获取那个数据并更新URL：

```javascript
public function countEggs()
{
    $user = $this->getLoggedInUser();

    $http = new Client('http://coop.apps.knpuniversity.com', array(
        'request.options' => array(
            'exceptions' => false,
        )
    ));

    $request = $http->post('/api/'.$user->coopUserId.'/eggs-count');
    $request->addHeader('Authorization', 'Bearer '.$user->coopAccessToken);
    // ...
}
```

我会添加一些调试代码以便于我们能够看到这个是否工作了：

```javascript
public function countEggs()
{
    // ...

    $request = $http->post('/api/'.$user->coopUserId.'/eggs-count');
    $request->addHeader('Authorization', 'Bearer '.$user->coopAccessToken);
    $response = $request->send();
    echo ($response->getBody(true));die;
    // ...
}
```

当我们刷新的时候，你会看到一个漂亮的JSON响应，对，我们统计到了鸡蛋的数量！那同样也会显示农场主Scott的！

由于顶级咯咯的意图是保持追踪每天每家收集的鸡蛋量，让我们把新的统计数据存入数据库。像以前一样，我已经完成了所有的麻烦的工作，以便于我们仅仅专注于OAuth片段，只需要调用`setTodaysEggCountForUser`并给它传递一个当前用户和蛋量统计就可以了。在这里，我们可以移除`die`声明并在我们完成（工作）后将用户导回首页：

```javascript
public function countEggs()
{
    // ...

    $response = $request->send();
    $countEggsData = json_decode($response->getBody(), true);

    $eggCount = $countEggsData['data'];
    $this->setTodaysEggCountForUser($this->getLoggedInUser(), $eggCount);

    return $this->redirect($this->generateUrl('home'));
}
```

当我们刷新后，我们应当导回至首页，但是（on the right），农场主Brent的鸡蛋统计没有上升，让我们前往COOP然后手动收集一些鸡蛋。回到FCL，如果我们再次统计鸡蛋，我们会发现蛋量更新了。非常好！

## 所有可能出错的事情

我们创建的“统计鸡蛋”页面工作得非常好，但是我们没有处理任何可能发生错误的情况，首先，当我们隐藏了它的链接，但是如果某人因为没有`coopUserId`和`coopAccessToken`而在页面上卡住了，该怎么办？让我们给这种情况编码：

```javascript
public function countEggs()
{
    $user = $this->getLoggedInUser();

    if (!$user->coopAccessToken || !$user->coopUserId) {
        throw new \Exception('Somehow you got here, but without a valid COOP access token! Re-authorize!');
    }

    // ...
}
```

我抛出了一个异常消息，但我们也可以以不同的方式处理这个，就像将用户重定向到"授权"页来开始OAuth流程。

另外一个我们要检查的是，令牌是否过期。有可能我们存储在数据库中的是已经过期的数据，我已经创建了一个简单地助手方法来检查这个。如果这个发生了，让我们将用户重定向到重新授权，就像他点击了“授权”链接：

```javascript
public function countEggs()
{
    $user = $this->getLoggedInUser();

    if (!$user->coopAccessToken || !$user->coopUserId) {
        throw new \Exception('Somehow you got here, but without a valid COOP access token! Re-authorize!');
    }

    if ($user->hasCoopAccessTokenExpired()) {
        return $this->redirect($this->generateUrl('coop_authorize_start'));
    }

    // ...
}
```

最后，如果API自身出错了，该怎么？下面是一个简单的处理方式：

```javascript
public function countEggs()
{
    // ...

    $request = $http->post('/api/'.$user->coopUserId.'/eggs-count');
    $request->addHeader('Authorization', 'Bearer '.$user->coopAccessToken);
    $response = $request->send();

    if ($response->isError()) {
        throw new \Exception($response->getBody(true));
    }

    // ...
}
```

当然，你可能想要做得更成熟一些，这个想一个包含有一些错误信息，你可以利用一下。对于OAuth，这是挺重要的，因为调用可能会因为`access_token`失效而失败。什么，我觉得我们已经检查过它了？在真实生活中，没有守卫来确保令牌在它的进程时间之前不会失效。另外，用户可能想要废除你的令牌－真流氓（what a bully）。谨慎地按情况处理。再强调一下，OAuth服务器应当在"error"和"error_description"参数中题工作错误信息。

你现在处于危险总，所以让我们继续前进让我们的农场主真正通过COOP登入FCL。
