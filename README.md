![Laravel best practices](/images/logo-english.png?raw=true)

注： 对照英文版和学院君版本修改而来

切换语言:

+ [English](https://github.com/alexeymezenin/laravel-best-practices)
+ [Русский](russian.md)

我们这里要讨论的并不是 Laravel 版的 SOLID 原则（想要了解更多 SOLID 原则细节<a href="https://www.jianshu.com/p/21573a0b2ad9" target="_blank" rel="noopener">查看这篇文章</a>）亦或是设计模式，而是 Laravel 实际开发中容易被忽略的最佳实践。

## 目录
 - [单一职责原则](#单一职责原则)
 - [胖模型，瘦控制器](#胖模型瘦控制器)
 - [验证](#验证)
 - [业务逻辑应该放到服务类](#业务逻辑应该放到服务类)
 - [不要重复造轮子(DRY)](#不要重复造轮子dry)
 - [优先使用 Eloquent 而不是查询构建器和原生 SQL 查询，优先使用集合而不是数组](#优先使用-eloquent-而不是查询构建器和原生-sql-查询优先使用集合而不是数组)
 - [批量赋值](#批量赋值)
 - [不要在 Blade 模板中执行查询，使用渴求式加载（避免 N+1 问题）](#不要在-blade-模板中执行查询使用渴求式加载避免-n1-问题)
 - [注释代码建议使用描述性的方法和变量名](#注释代码建议使用描述性的方法和变量名)
 - [不要把 JS 和 CSS 代码放到 Blade 模板里面，不要在 PHP 类中写 HTML 代码](#不要把-js-和-css-代码放到-blade-模板里面不要在-php-类中写-html-代码)
 - [使用配置、语言文件、常量而不是在代码中写死](#使用配置语言文件常量而不是在代码中写死)
 - [使用社区认可的标准 Laravel 工具](#使用社区认可的标准-laravel-工具)
 - [遵循 Laravel 命名约定](#遵循-laravel-命名约定)
 - [使用更短的、可读性更好的语法](#使用更短的可读性更好的语法)
 - [使用 IoC 容器或门面而不是创建新类](#使用-ioc-容器或门面而不是创建新类)
 - [不要直接从 `.env` 文件获取数据](#不要直接从-env-文件获取数据)
 - [以标准格式存储日期，使用访问器和修改器来编辑日期格式](#以标准格式存储日期使用访问器和修改器来编辑日期格式)
 - [其他好的实践](#其他好的实践)

## **单一职责原则**

一个类和方法只负责一项职责。

坏代码：

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```
好代码：

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerfiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 返回目录](#目录)

## **胖模型，瘦控制器**

如果你使用的是查询构建器或原生 SQL 查询的话将所有 DB 相关逻辑都放到 Eloquent 模型或 Repository 类。

坏代码：

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

好代码：

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

Class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 返回目录](#目录)

### **验证**

将验证逻辑从控制器转移到请求类。


坏代码：

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

好代码：

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 返回目录](#目录)

### **业务逻辑应该放到服务类**

一个控制器只负责一项职责，所以需要把业务逻辑都转移到服务类中。


坏代码：

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

好代码：

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 返回目录](#目录)

### **不要重复造轮子(DRY)**

尽可能复用代码，单一职责原则可以帮助你避免重复，此外，尽可能复用 Blade 模板，使用 Eloquent 作用域。

坏代码：

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

好代码：

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 返回目录](#目录)

### **优先使用 Eloquent 而不是查询构建器和原生 SQL 查询，优先使用集合而不是数组**

通过 Eloquent 可以编写出可读性和可维护性更好的代码，此外，Eloquent 还提供了强大的内置工具如软删除、事件、作用域等。

坏代码：

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

好代码：

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 返回目录](#目录)

### **批量赋值**

关于批量赋值细节可查看<a href="http://laravelacademy.org/post/8194.html#toc_11" target="_blank" rel="noopener">对应文档</a>。

坏代码：

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

好代码：

```php
$category->article()->create($request->all());
```

[🔝 返回目录](#目录)

### **不要在 Blade 模板中执行查询，使用渴求式加载（避免 N+1 问题）**

Bad (for 100 users, 101 DB queries will be executed):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Good (for 100 users, 2 DB queries will be executed):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 返回目录](#目录)

### **注释代码建议使用描述性的方法和变量名**

坏代码：

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Better:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

好代码：

```php
if ($this->hasJoins())
```

[🔝 返回目录](#目录)

### **不要把 JS 和 CSS 代码放到 Blade 模板里面，不要在 PHP 类中写 HTML 代码**

坏代码：

```php
let article = `{{ json_encode($article) }}`;
```

Better:

```php
<input id="article" type="hidden" value="{{ json_encode($article) }}">

或者

<button class="js-fav-article" data-article="{{ json_encode($article) }}">{{ $article->name }}<button>
```

在 JavaScript 文件里：

```php
let article = $('#article').val();
```

最好是使用指定的 php 到 js 的包来传递数据

[🔝 返回目录](#目录)

### **使用配置、语言文件、常量而不是在代码中写死**

坏代码：

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

好代码：

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 返回目录](#目录)

### **使用社区认可的标准 Laravel 工具**

优先使用 Laravel 内置功能和社区版扩展包，其次才是第三方扩展包和工具。这样做的好处是降低以后的学习和维护成本。

任务 | 标准工具 | 第三方工具
------------ | ------------- | -------------
授权 | Policies | Entrust, Sentinel 等
编译资源文件(assets) | Laravel Mix | Grunt, Gulp, 3rd party packages
开发环境 | Homestead | Docker
部署 | Laravel Forge | Deployer and other solutions
单元测试 | PHPUnit, Mockery | Phpspec
浏览器测试 | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
模板 | Blade | Twig
处理数据 | Laravel collections | 数组
表单验证 | Request classes | 第三方扩展包、控制器中验证
认证 | 内置功能 | 第三方扩展包、你自己的解决方案
API认证 | Laravel Passport | 第三方 JWT 和 OAuth 扩展包
创建API	 | 内置功能 | Dingo API和类似扩展包
处理DB结构 | Migrations | 	直接操作DB
本地化 | 内置功能 | 第三方工具
实时用户接口 | Laravel Echo, Pusher | 第三方直接处理 WebSocket的扩展包
生成测试数据 | Seeder classes, Model Factories, Faker | 手动创建测试数据
任务调度 | Laravel Task Scheduler | 脚本或第三方扩展包
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝 返回目录](#目录)

### **遵循 Laravel 命名约定**

遵循 [PSR 标准](http://www.php-fig.org/psr/psr-2/)。

此外，还要遵循 Laravel 社区版的命名约定：

什么 | 怎么做 | 好 | 坏
------------ | ------------- | ------------- | -------------
控制器 | 单数 | ArticleController | ~~ArticlesController~~
路由 | 复数 | articles/1 | ~~article/1~~
命名路由 | 下划线+'.'号分隔 | users.show_active | ~~users.show-active, show-active-users~~
模型 | 单数 | User | ~~Users~~
一对一关联 | 单数 | articleComment | ~~articleComments, article_comment~~
其他关联关系 | 复数 | articleComments | ~~articleComment, article_comments~~
表 | 复数 | article_comments | ~~article_comment, articleComments~~
中间表 | 按字母表排序的单数格式 | article_user | ~~user_article, articles_users~~
表字段 | 下划线，不带模型名 | meta_title | ~~MetaTitle; article_meta_title~~
外键 | 单数、带_id后缀 | article_id | ~~ArticleId, id_article, articles_id~~
主键 | - | id | ~~custom_id~~
迁移 | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
方法 | 驼峰 | getAll | ~~get_all~~
RestFul资源控制器方法 | [文档](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
测试类方法 | 驼峰 | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
变量 | 驼峰 | $articlesWithAuthor | ~~$articles_with_author~~
集合 | descriptive, 复数 | $activeUsers = User::active()->get() | ~~$active, $data~~
对象 | descriptive, 单数 | $activeUser = User::active()->first() | ~~$users, $obj~~
配置和语言文件索引 | 下划线 | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | 下划线 | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Config | 下划线 | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
契约（接口） | 形容词或名词 | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | 形容词 | Notifiable | ~~NotificationTrait~~

[🔝 返回目录](#目录)

### **使用更短的、可读性更好的语法**

坏代码：

```php
$request->session()->get('cart');
$request->input('name');
```

好代码：

```php
session('cart');
$request->name;
```

更多示例：

通用语法 | 可读性更好的
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? $object->relation->id : null }` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝 返回目录](#目录)

### **使用 IoC 容器或门面而不是创建新类**

自己创建新的类会导致代码耦合度高，且难于测试，取而代之地，我们可以使用 IoC 容器或门面。

坏代码：

```php
$user = new User;
$user->create($request->all());
```

好代码：

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->all());
```

[🔝 返回目录](#目录)

### **不要直接从 `.env` 文件获取数据**

通过配置文件传递数据，然后使用 `config()` 辅助函数获取数据。


坏代码：

```php
$apiKey = env('API_KEY');
```

好代码：

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[🔝 返回目录](#目录)

### **以标准格式存储日期，使用访问器和修改器来编辑日期格式**

坏代码：

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

好代码：

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at']
public function getMonthDayAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->monthDay }}
```

[🔝 返回目录](#目录)

## 其他好的实践

不要在路由文件中写任何业务逻辑。

在 Blade 模板中尽量不要写原生 PHP。

[🔝 返回目录](#目录)
