# 客户端凭证

Meet Brent是一位努力工作，络腮胡子，喜欢大声咀嚼狗尾巴草的大叔，它在密西西比河边上有一个养着一群漂亮的，小巧的，产蛋量高的母鸡的鸡舍。但是喂鸡和处理农场周边的事情总是会耗费他大量的时间。

但是好消息来了！新的“母鸡监管操作平台”，或称为COOP网站，刚刚启动了！利用COOP，你可以登录到网站里面并收集你的鸡蛋、解锁库房、做任何别的事情，只需要点击一下按钮。

注意，COOP有一个API，Brent在想，他能否编写一个小的脚本来自动收集鸡蛋。是的，如果他有一个从他这边发送API请求的小脚本，他就可以以CRON（计划任务）工作的方式日常运行他然后睡自己的觉！

COOP是真实存在的。你可以访问<http://coop.apps.knpuniversity.com>，找到这个假装的网站，去创建一个账户，然后控制你的虚拟农场。这就是未来！

## 开始我们的命令行脚本

COOP的API比较简单，只需要少量的端点（endpoints），包括要用于我们的小命令行脚本的那个：eggs-collect。

我已经创建了一个包含有一个叫做`collect_eggs.php`的目录`cron/`，我们可以从这里开始：

```javascript
// collect_eggs.php
include __DIR__.'/vendor/autoload.php';
use Guzzle\Http\Client;

// create our http client (Guzzle)
$http = new Client('http://coop.apps.knpuniversity.com', array(
    'request.options' => array(
        'exceptions' => false,
    )
));
```

> 提示

> 代码跟随着我们！你可以点击页面上的下载按钮来获得起始项目，然后跟随说明的指引将它设置好。（译者注：我在GitHub上找到了一它他们的起始项目：https://github.com/knpuniversity/oauth， 当然，如果你是他们的订阅者，你可以在网站页面的右上角找到下载按钮。）

它没有做任何事情，除了创建了一个指向COOP网站的`Client`对象。由于我们将要向COOP的API发起一个请求，我们将会使用一个真正很棒的PHP库：[Guzzle](http://guzzlephp.org/)。不要担心你从没使用过它，它非常简单。

在我们开始之前，我们需要使用[Composer](http://getcomposer.org/)来下载Guzzle。[下载Composer](http://getcomposer.org/download/)到`cron/`目录中，然后安装vendor库：

```terminal
php composer.phar install
```

> 注意

> 没用过Composer？给你一个免费福利，让你可以精通它：[Composer的奇妙世界](http://knpuniversity.com/screencast/composer)

让我们尝试向`/api/2/eggs-collect`发起我们的第一个API请求。`2`是我们的COOP用户的ID，因为我们要从我们的农场收集鸡蛋。你的ID可能有所不同：

```javascript
// collect_eggs.php
// ...

$request = $http->post('/api/2/eggs-collect');
$response = $request->send();
echo $response->getBody();

echo "\n\n";
```

尝试从命令终端中运行那个脚本：

```terminal
php collect_eggs.php
```

没有出乎意料，这个吹爆了！

```javascript
{
  "error": "access_denied",
  "error_description": "an access token is required"
}
```

## OAuth应用

在我们考虑得到令牌之前，我们需要在COOP上创建一个应用。这个应用代表着我们想要创建的内部的app或网站。在我们的例子中，就是那个小的命令行脚本。在OAuth来讲，就是这个应用会请求访问一个用户的COOP账户。

给它一个类似于"Brent's Lazy CRON Job"的名字，一个描述，然后仅仅检查“收集鸡蛋”“盒子”。这些是“范围”，或者是当你的应用具有一个从COOP授予的令牌后的基本的权限。

当我们完成后，我们就有了一个客户端ID和一个自动生成的“客户端秘文”，这些是一个用于应用的用户名和密码的序列。一个奇怪的事情是“应用”和“客户端”的条款在OAuth中是互换使用的，都可以用来指代我们刚刚注册的应用和你正在构架的实际的app，就像计划任务脚本或你的网站一样，我将试着一路解释清楚。

现在，让我们得到一个访问令牌！

## 客户端凭证授权类型

第一个OAuth授权类型被称为客户端凭证，它是所有类型中最简单的。它只需要两部分，客户端和服务端。对于我们来说，这就是命令行脚本和COOP API。

使用这种授权类型，就没有“用户”，我们得到的访问令牌将只会允许我们访问在这个应用控制下的资源。当我们使用这个访问令牌发起API请求，它就像我们作为应用自身登入一样，不是任何独立的用户。我即将会解释更多的内容。

如果你访问你早先创建的应用，你会看到一个漂亮的"Generate a Token"链接，当点击后，它会拿出一个（令牌）。在这个场景后面，这使用了客户端凭证，我们将会很快地详细了解它：

```terminal
http://coop.apps.knpuniversity.com/token
    ?client_id=Your+Client+Name
    &client_secret=abcdefg
    &grant_type=client_credentials
```

但是现在，我么可以庆祝立即使用这令牌来在应用方面采取行动！

## API中的访问令牌

具体怎么做到这个取决于你所请求的API，一个常用的方法，COOP使用的那个，通过一个认证持有头来发送它。

```terminal
GET /api/barn-unlock HTTP/1.1
Host: coop.apps.knpuniversity.com
Authorization: Bearer ACCESSTOKENHERE
```

更新脚本来发送这个头：

```javascript
// collect-eggs.php
// ...

$accessToken = 'abcd1234def67890';

$request = $http->post('/api/2/eggs-collect');
$request->addHeader('Authorization', 'Bearer '.$accessToken);
$response = $request->send();
echo $response->getBody();

echo "\n\n";
```

当我们再次之行这个脚本，开始庆祝吧，因为它工作了！现在我们有足够的鸡蛋来做蛋卷了:)

```javascript
{
  "action": "eggs-collect",
  "success": true,
  "message": "Hey look at that, 3 eggs have been collected!",
  "data": 3
}
```

## 尝试收集别人的鸡蛋

注意，这里会收集我们的鸡蛋，因为在URL中使用的是我们的用户的ID。当把这个ID改为一个别的用户的ID后会发生什么呢？

> /api/3/eggs-collect

如果你尝试了它，就会发现失败了！

```javascript
{
  "error": "access_denied",
  "error_description": "You do not have access to take this action"
}
```

从技术角度上说，带着来自客户端凭证的令牌，我们不在一个用户的角度发送API请求，却站在应用的角度。这使得客户端凭证完美地用于发送编辑或取得应用自身信息的API请求，就像一个它有多少用户合计的请求。

我们决定创建COOP以便于应用也可以有权利来修改创建了这个应用的用户，这就是为什么我们可以收集我们的用户的鸡蛋，而不是邻居的。

## 根据客户端凭证获取令牌

把香槟酒拿开：我们还没有完成。通常情况下，访问令牌不会永久有效。COOP的令牌持续24个小时，这就意味着，明天到来时，我们的脚本就会中断。

让网站来为我们处理客户端凭证工作来测一测是很好的，但是我们需要自己在脚本里面处理好。每一个OAuth服务都会有一个用于请求访问令牌的API端点。如果我们查看COOP的API认证文档，我们可以看到那个URL和它所需要的参数：

> <http://coop.apps.knpuniversity.com/token>

> **Parameters:**

>     client_id client_secret grant_type

让我们更新一下我们的脚本来首次发起这个API请求，填上`client_id`，`client_secret`，和`grant_type`这些POST参数：

```javascript
// collect-eggs.php
// ...

// run this code *before* requesting the eggs-collect endpoint
$request = $http->post('/token', null, array(
    'client_id'     => 'Brent\'s Lazy CRON Job',
    'client_secret' => 'a2e7f02def711095f83f2fb04ecbc0d3',
    'grant_type'    => 'client_credentials',
));

// make a request to the token url
$response = $request->send();
$responseBody = $response->getBody(true);
var_dump($responseBody);die;
// ...
```

带着幸运，当你运行它的时候，你会看到一个带着一个访问令牌和一些少量其他细节的JSON相应：

```javascript
{
  "access_token": "fa3b4e29d8df9900816547b8e53f87034893d84c",
  "expires_in": 86400,
  "token_type": "Bearer",
  "scope": "chickens-feed"
}
```

让我么使用这个访问令牌来取代我们之前粘贴在那里的那个：

```javascript
// collect-eggs.php
// ...

// step1: request an access token
$request = $http->post('/token', null, array(
    'client_id'     => 'Brent\'s Lazy CRON Job',
    'client_secret' => 'a2e7f02def711095f83f2fb04ecbc0d3',
    'grant_type'    => 'client_credentials',
));

// make a request to the token url
$response = $request->send();
$responseBody = $response->getBody(true);
$responseArr = json_decode($responseBody, true);
$accessToken = $responseArr['access_token'];

// step2: use the token to make an API request
$request = $http->post('/api/2/eggs-collect');
$request->addHeader('Authorization', 'Bearer '.$accessToken);
$response = $request->send();
echo $response->getBody();

echo "\n\n";
```

现在，它仍旧工作，由于我们每次得到一个新鲜的令牌，所以我们永远不会有令牌过期的问题。一旦Brent设置好了一个计划任务工作来运行我们这个脚本，他就可以一直睡到下午！

## Why, What和When：客户端凭证

每个授权类型最终使用`/token`端点来得到一个令牌，但是在那之前的细节不同。客户端凭证是一种直接得到令牌的方式，一个限制就是它需要你的客户端秘文，这目前来说是可以的，因为我们的脚本藏在某个服务器上。

但是在网络上，我们不能暴露客户端秘文，这就是为什么下面的两种授权方式变得重要的原因。
