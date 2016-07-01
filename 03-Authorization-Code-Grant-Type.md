# 认证码授权类型

突然，下午的时候，Brent被农场主Scott的尖叫声颠醒（Scott正带着他的鸡蛋前往超市）：“哈哈，Brent!我的鸡产的蛋比你的多！”但在现实中，Brent知道他的鸡比Scott的产蛋量更高...但是，怎样提高产量呢？

然后冒出来了一个想法！COOP API有一个端点可以查看每天一个用户的农场收集了多少鸡蛋。Brent决定创建一个新网站来使用这个端点来统计一个用户的农场的总的产蛋量。他把它叫做：顶级咯咯！幻鸡联盟，或者缩写成FCL。要在每个用户那边调用`/api/eggs-count`端点，这个站点会使用OAuth来收集每个注册进来的农场主的访问令牌。

再一次的，问题是：每个用户要怎样给FCL一个访问令牌来让它统计他们那边的蛋量呢？

## 启动FCL

让我们查看一下FCL应用，Brent已经在构建了，它就躺在下载的代码中的`client/`目录中，我将会使用内置的PHP网络服务器来运行这个站点：

```terminal
cd client/web
php -S localhost:9000
```

那个代码启动了一个内置的PHP网络服务器，它就会一直呆在那直到我们将它关闭。这个项目依旧使用Composer，所以我们需要使用之前安装好的Composer来安装这个项目使用的一些依赖项。

> 注意

> 如果这不起作用，然后PHP只是简单地提示了命令行选项，请检查你的PHP版本，内置的PHP网络服务器至少需要PHP5.4以上。

将指向站点的URL输入浏览器中然后载入它。欢迎来到顶级咯咯！我们已经有了一个排行榜和基本的注册数据。去注册一个账号，这将自动将我们登入系统。

这个站点是功能齐全的，数据库已经准备好了保持追踪每个农场收集的蛋量。唯一缺的是OAuth：取得每个用户的访问令牌以便于我们可以发起请求来统计他们的蛋量。

## 跳转向认证

在顶级咯咯可以向COOP发起API请求来统计我的蛋量之前，我需要认证它。在主页上，已经有了一个认证链接，它现在只会打印出一条消息。

这个链接后面的代码位于`src/OAuth2Demo/Client/Controllers/CoopOAuthController.php`文件中，你甚至不需要理解它是如何工作的，只需要直到无论我们在这里做什么，都会出场（just know that what ever we do here, shows up）:

```javascript
// src/OAuth2Demo/Client/Controllers/CoopOAuthController.php
// ...

public function redirectToAuthorization(Request $request)
{
    die('Hallo world!');
}
```

认证码批准类型的第一步是将用户转向一个COOP上特定的URL。在这里，用户将给我们的app授权。根据[COOP的API认证页](http://coop.apps.knpuniversity.com/api#authentication)，我们需要将用户导向`/authorize`并发送一些查询参数。

在我们的代码中，让我们开始构建那个URL：

```javascript
// src/OAuth2Demo/Client/Controllers/CoopOAuthController.php
// ...

public function redirectToAuthorization(Request $request)
{
    $url = 'http://coop.apps.knpuniversity.com/authorize?'.http_build_query(array(
        'response_type' => 'code',
        'client_id' => '?',
        'redirect_uri' => '?',
        'scope' => 'eggs-count profile'
    ));

    var_dump($url);die;
}
```

`response_type`类型是`code`是因为我们要使用认证码流。另外一个有效的值是`token`，它用于被称为模糊流的一个批准类型。我们将在后面看到它。

对于`scopes`，我们使用了`profile`和`eggs-count`，以便于一旦我们认证通过了，我们就可以得到一些COOP用户的轮廓数据，还有，当然，统计他们的蛋量。

要获得一个`client_id`，让我们前往COOP然后创建一个新的应用来代表顶级咯咯。最重要的事情是检查轮廓信息和收集蛋量这两个，我很快就会为你展示为什么。

> 提示

> 如果已经存在了一个你想要的应用名，只需呀哦选择另外一个并将它作为你的`client_id`。

将`client_id`复制到你的URL中，好了！最后一步是`redirect_uri`，它是我们的网站上的一个URL，COOP会在授权或拒绝我们的应用访问之后将用户发给它。一旦那个发生了，我们将要做很多重要的事情。

让我们将那个URL设置为`/coop/oauth/handle`，它就是一个打印信息的别的页面。这个的代码就在同一个文件中，在下面一点点：

```javascript
// src/OAuth2Demo/Client/Controllers/CoopOAuthController.php
// ...

public function receiveAuthorizationCode(Application $app, Request $request)
{
    // equivalent to $_GET['code']
    $code = $request->get('code');

    die('Implement this in CoopOAuthController::receiveAuthorizationCode');
}

```

相比硬编码的URL，我将使用URL生成器：

```javascript
public function redirectToAuthorization(Request $request)
{
    $redirectUrl = $this->generateUrl('coop_authorize_redirect', array(), true);

    $url = 'http://coop.apps.knpuniversity.com/authorize?'.http_build_query(array(
        'response_type' => 'code',
        'client_id' => 'TopCluck',
        'redirect_uri' => $redirectUrl,
        'scope' => 'eggs-count profile'
    ));
    // ...
}
```

在你构建URL的时候，要将它构建成绝对地址。好了，我们已经构建好了用于COOP的授权URL，让我们将用户导向它：

```javascript
public function redirectToAuthorization(Request $request)
{
    // ...

    return $this->redirect($url);
}
```

`redirect`方法是我的应用特有的，所以你的代码可能有所不同，只要以某种方式将用户进行了重导（重新将用户导向某个地址），就可以。

> 提示

> 由于我们使用Silex，`redirect`方法确实是一个我创建的用来创建新的`RedirectResponse`对象的快捷方式。

## 在COOP上认证

让我们试一下它！返回到首页，然后点击"Authorize"链接。这将把我们带到我们的代码那里，然后这会将我们导向COOP。我们已经登录进去了，所以它直接询问我们来授权这个应用，注意，我们在URL中包含的范围（权限范围）已经清晰地传达到了。让我们授权这个应用，稍后我们将看到发生的事情。

当我们点击了授权按钮后，我们被发回了顶级咯咯上的`redirect_uri`！没什么实际的事情发生，COOP没有设置任何cookie或者别的任何事情。但是URL中确实包含了一个`code`参数。

## 交换授权码以得到一个访问令牌

这个查询参数被称为授权码，它对于这种批准类型来说是唯一的，他不是我们真正想要的令牌，但是它是得到令牌的关键，授权码是用户说我们可以有访问令牌的临时证明。

让我们把授权码从`collect_eggs.php`脚本中复制粘贴到这里，继续并将`client_id`和`client_secret`更改为来自新客户端或我们创建的用于顶级咯咯的：

```javascript
// src/OAuth2Demo/Client/Controllers/CoopOAuthController.php
// ...

public function receiveAuthorizationCode(Application $app, Request $request)
{
    // equivalent to $_GET['code']
    $code = $request->get('code');

    $http = new Client('http://coop.apps.knpuniversity.com', array(
        'request.options' => array(
            'exceptions' => false,
        )
    ));

    $request = $http->post('/token', null, array(
        'client_id'     => 'TopCluck',
        'client_secret' => '2e2dfd645da38940b1ff694733cc6be6',
        'grant_type'    => 'authorization_code',
    ));

    // make a request to the token url
    $response = $request->send();
    $responseBody = $response->getBody(true);
    var_dump($responseBody);die;
}
```

如果我们回去查看COOP的API认证文档，我们会看到`/token`有两个别的和认证授权类型一起使用的参数：`code`和`redirect_uri`。我已经接收了`code`查询参数，所以让我们把这些填入。确保按照文档中描述的那样更改`grant_type`为`authorization_code`。最终，倒出`$responseBody`来看这个请求是否工作了：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // equivalent to $_GET['code']
    $code = $request->get('code');
    // ...

    $request = $http->post('/token', null, array(
        'client_id'     => 'TopCluck',
        'client_secret' => '2e2dfd645da38940b1ff694733cc6be6',
        'grant_type'    => 'authorization_code',
        'code'          => $code,
        'redirect_uri'  => $this->generateUrl('coop_authorize_redirect', array(), true),
    ));

    // ...
}
```

这个流程中的关键是`code`参数。当COOP接收了我们的请求，它会检查那个授权码是否有效，它还知道这个授权码属于哪个用户，它必须精确匹配原始的我们用于重导用户的`redirect_uri`。

好了，我们试一下！当我们刷新后，API实际上给了我们一个错误：

```javascript
{
    "error": "invalid_grant",
    "error_description": "The authorization code has expired"
}
```

授权码有一个非常段的生命，通常以秒计，我们通常立即用它来交换访问令牌，所以，那好！让我们重新从首页开始整个处理。

> 注意

> 通常，OAuth服务器会记住一个用户已经授权过这个应用了，并立即将用户导回你的应用，但COOP不这么做是为了让流程更容易理解。

这次，发向`/token`的请求得到了`access_token`。我们赢了！让我们同时设置`expires_in`到一个变量中，它是一个这个访问令牌过期的秒数：

```
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...

    $request = $http->post('/token', null, array(
        'client_id'     => 'TopCluck',
        'client_secret' => '2e2dfd645da38940b1ff694733cc6be6',
        'grant_type'    => 'authorization_code',
        'code'          => $code,
        'redirect_uri'  => $this->generateUrl('coop_authorize_redirect', array(), true),
    ));

    // make a request to the token url
    $response = $request->send();
    $responseBody = $response->getBody(true);
    $responseArr = json_decode($responseBody, true);

    $accessToken = $responseArr['access_token'];
    $expiresIn = $responseArr['expires_in'];
}
```

## 使用访问令牌

就像我们的计划任务脚本一样，让我们使用访问令牌来发起API请求！一个端点是`/api/me`，它会返回用户的绑定到这个访问令牌上的信息，让我们向这个端点发起一个GET请求，在`Authorization`头部设置访问令牌，就像前面做的一样：

```javascript
public function receiveAuthorizationCode(Application $app, Request $request)
{
    // ...

    $accessToken = $responseArr['access_token'];
    $expiresIn = $responseArr['expires_in'];

    $request = $http->get('/api/me');
    $request->addHeader('Authorization', 'Bearer '.$accessToken);
    $response = $request->send();
    echo ($response->getBody(true));die;
}
```

返回首页并点击"Authorize"来尝试它。由于认证码将已经过期，简单地刷新页面是不行的。带着任意的幸运，你会看到一个带有用户信息的JSON响应：

```
{
    id: "2",
    email: "brent@knpuniversity.com",
    firstName: "Brent",
    lastName: "Shaffer"
}
```

这个当然会工作，是因为我们发送了一个绑定到Brent的账户上的访问令牌，这同样是因为当我们转到用户时，我们请求了`profile`环境。

随着这个，我们看到了认证码授权方式中的关键部分，以及如何在我们的应用中使用访问令牌。但是我们应该把令牌存储在哪里，以及如果用户觉觉了我们的应用访问，该怎么办？我们将在后面处理这些。
