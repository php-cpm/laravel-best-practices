![Laravel best practices](/images/logo-english.png?raw=true)

切换语言:

+ [English](https://github.com/alexeymezenin/laravel-best-practices)
+ [Русский](russian.md)

我们这里要讨论的并不是 Laravel 版的 SOLID 原则（想要了解更多 SOLID 原则细节<a href="https://www.jianshu.com/p/21573a0b2ad9" target="_blank" rel="noopener">查看这篇文章</a>）亦或是设计模式，而是 Laravel 实际开发中容易被忽略的最佳实践。
<h3 id="toc_0">内容概览</h3>
<ul>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_1">单一职责原则</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_2">胖模型，瘦控制器</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_3">验证</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_4">业务逻辑应该放到服务类</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_5">DRY（Don't Repeat Yourself，不要重复造轮子）</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_6">优先使用 Eloquent 而不是查询构建器和原生 SQL 查询，优先使用集合而不是数组</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_7">批量赋值</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_8">不要在 Blade 模板中执行查询 &amp; 使用渴求式加载（避免 N+1 问题）</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_9">注释代码</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_10">不要把 JS 和 CSS 代码放到 Blade 模板里面，不要在 PHP 类中写 HTML 代码</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_11">使用配置、语言文件、常量而不是在代码中写死</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_12">使用社区接受的标准 Laravel 工具</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_13">遵循 Laravel 命名约定</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_14">使用更短的、可读性更好的语法</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_15">使用 IoC 容器或门面而不是创建新类</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_16">不要直接从 <code>.env</code> 文件获取数据</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_17">以标准格式存储日期，使用访问器和修改器来编辑日期格式</a></li>
 	<li><a href="http://laravelacademy.org/post/8464.html#toc_18">其他好的实践</a></li>
</ul>
<h3 id="toc_1">单一职责原则</h3>
一个类和方法只负责一项职责。

坏代码：
<pre><code>public function getFullNameAttribute()
{
    if (auth()-&gt;user() &amp;&amp; auth()-&gt;user()-&gt;hasRole('client') &amp;&amp; auth()-&gt;user()-&gt;isVerified()) {
        return 'Mr. ' . $this-&gt;first_name . ' ' . $this-&gt;middle_name . ' ' $this-&gt;last_name;
    } else {
        return $this-&gt;first_name[0] . '. ' . $this-&gt;last_name;
    }
}
</code></pre>
好代码：
<pre><code>public function getFullNameAttribute()
{
    return $this-&gt;isVerifiedClient() ? $this-&gt;getFullNameLong() : $this-&gt;getFullNameShort();
}

public function isVerfiedClient()
{
    return auth()-&gt;user() &amp;&amp; auth()-&gt;user()-&gt;hasRole('client') &amp;&amp; auth()-&gt;user()-&gt;isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this-&gt;first_name . ' ' . $this-&gt;middle_name . ' ' . $this-&gt;last_name;
}

public function getFullNameShort()
{
    return $this-&gt;first_name[0] . '. ' . $this-&gt;last_name;
}
</code></pre>
<h3 id="toc_2">胖模型、瘦控制器</h3>
如果你使用的是查询构建器或原生 SQL 查询的话将所有 DB 相关逻辑都放到 Eloquent 模型或 Repository 类。

坏代码：
<pre><code>public function index()
{
    $clients = Client::verified()
        -&gt;with(['orders' =&gt; function ($q) {
            $q-&gt;where('created_at', '&gt;', Carbon::today()-&gt;subWeek());
        }])
        -&gt;get();

    return view('index', ['clients' =&gt; $clients]);
}
</code></pre>
好代码：
<pre><code>public function index()
{
    return view('index', ['clients' =&gt; $this-&gt;client-&gt;getWithNewOrders()]);
}

Class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this-&gt;verified()
            -&gt;with(['orders' =&gt; function ($q) {
                $q-&gt;where('created_at', '&gt;', Carbon::today()-&gt;subWeek());
            }])
            -&gt;get();
    }
}
</code></pre>
<h3 id="toc_3">验证</h3>
将验证逻辑从控制器转移到请求类。

坏代码：
<pre><code>public function store(Request $request)
{
    $request-&gt;validate([
        'title' =&gt; 'required|unique:posts|max:255',
        'body' =&gt; 'required',
        'publish_at' =&gt; 'nullable|date',
    ]);

    ....
}
</code></pre>
好代码：
<pre><code>public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' =&gt; 'required|unique:posts|max:255',
            'body' =&gt; 'required',
            'publish_at' =&gt; 'nullable|date',
        ];
    }
}
</code></pre>
<h3 id="toc_4">业务逻辑需要放到服务类</h3>
一个控制器只负责一项职责，所以需要把业务逻辑都转移到服务类中。

坏代码：
<pre><code>public function store(Request $request)
{
    if ($request-&gt;hasFile('image')) {
        $request-&gt;file('image')-&gt;move(public_path('images') . 'temp');
    }

    ....
}
</code></pre>
好代码：
<pre><code>public function store(Request $request)
{
    $this-&gt;articleService-&gt;handleUploadedImage($request-&gt;file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image-&gt;move(public_path('images') . 'temp');
        }
    }
}
</code></pre>
<h3 id="toc_5">DRY</h3>
尽可能复用代码，单一职责原则可以帮助你避免重复，此外，尽可能复用 Blade 模板，使用 Eloquent 作用域。

坏代码：
<pre><code>public function getActive()
{
    return $this-&gt;where('verified', 1)-&gt;whereNotNull('deleted_at')-&gt;get();
}

public function getArticles()
{
    return $this-&gt;whereHas('user', function ($q) {
            $q-&gt;where('verified', 1)-&gt;whereNotNull('deleted_at');
        })-&gt;get();
}
</code></pre>
好代码：
<pre><code>public function scopeActive($q)
{
    return $q-&gt;where('verified', 1)-&gt;whereNotNull('deleted_at');
}

public function getActive()
{
    return $this-&gt;active()-&gt;get();
}

public function getArticles()
{
    return $this-&gt;whereHas('user', function ($q) {
            $q-&gt;active();
        })-&gt;get();
}
</code></pre>
<h3 id="toc_6">优先使用 Eloquent 和 集合</h3>
通过 Eloquent 可以编写出可读性和可维护性更好的代码，此外，Eloquent 还提供了强大的内置工具如软删除、事件、作用域等。

坏代码：
<pre><code>SELECT *
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
</code></pre>
好代码：
<pre><code> Article::has('user.profile')-&gt;verified()-&gt;latest()-&gt;get();
</code></pre>
<h3 id="toc_7">批量赋值</h3>
关于批量赋值细节可查看<a href="http://laravelacademy.org/post/8194.html#toc_11" target="_blank" rel="noopener">对应文档</a>。

坏代码：
<pre><code>$article = new Article;
$article-&gt;title = $request-&gt;title;
$article-&gt;content = $request-&gt;content;
$article-&gt;verified = $request-&gt;verified;
// Add category to article
$article-&gt;category_id = $category-&gt;id;
$article-&gt;save();
</code></pre>
好代码：
<pre><code>$category-&gt;article()-&gt;create($request-&gt;all());
</code></pre>
<h3 id="toc_8">不要在 Blade 执行查询 &amp; 使用渴求式加载</h3>
坏代码：
<pre><code>@foreach (User::all() as $user)
    {{ $user-&gt;profile-&gt;name }}
@endforeach
</code></pre>
好代码：
<pre><code>$users = User::with('profile')-&gt;get();

...

@foreach ($users as $user)
    {{ $user-&gt;profile-&gt;name }}
@endforeach
</code></pre>
<h3 id="toc_9">注释你的代码</h3>
坏代码：
<pre><code>if (count((array) $builder-&gt;getQuery()-&gt;joins) &gt; 0)
</code></pre>
好代码：
<pre><code>// Determine if there are any joins.
if (count((array) $builder-&gt;getQuery()-&gt;joins) &gt; 0)
</code></pre>
最佳：
<pre><code>if ($this-&gt;hasJoins())
</code></pre>
<h3 id="toc_10">将前端代码和 PHP 代码分离：</h3>
不要把 JS 和 CSS 代码写到 Blade 模板里，也不要在 PHP 类中编写 HTML 代码。

坏代码：
<pre><code>let article = `{{ json_encode($article) }}`;
</code></pre>
好代码：
<pre><code>&lt;input id="article" type="hidden" value="{{ json_encode($article) }}"&gt;

或者

&lt;button class="js-fav-article" data-article="{{ json_encode($article) }}"&gt;{{ $article-&gt;name }}&lt;button&gt;
</code></pre>
在 JavaScript 文件里：
<pre><code>let article = $('#article').val();
</code></pre>
<h3 id="toc_11">使用配置、语言文件和常量取代硬编码</h3>
坏代码：
<pre><code>public function isNormal()
{
    return $article-&gt;type === 'normal';
}

return back()-&gt;with('message', 'Your article has been added!');
</code></pre>
好代码：
<pre><code>public function isNormal()
{
    return $article-&gt;type === Article::TYPE_NORMAL;
}

return back()-&gt;with('message', __('app.article_added'));
</code></pre>
<h3 id="toc_12">使用被社区接受的标准 Laravel 工具</h3>
优先使用 Laravel 内置功能和社区版扩展包，其次才是第三方扩展包和工具。这样做的好处是降低以后的学习和维护成本。
<table>
<thead>
<tr>
<th>任务</th>
<th>标准工具</th>
<th>第三方工具</th>
</tr>
</thead>
<tbody>
<tr>
<td>授权</td>
<td>策略类</td>
<td>Entrust、Sentinel等</td>
</tr>
<tr>
<td>编译资源</td>
<td>Laravel Mix</td>
<td>Grunt、Gulp等</td>
</tr>
<tr>
<td>开发环境</td>
<td>Homestead</td>
<td>Docker</td>
</tr>
<tr>
<td>部署</td>
<td>Laravel Forge</td>
<td>Deployer等</td>
</tr>
<tr>
<td>单元测试</td>
<td>PHPUnit、Mockery</td>
<td>Phpspec</td>
</tr>
<tr>
<td>浏览器测试</td>
<td>Laravel Dusk</td>
<td>Codeception</td>
</tr>
<tr>
<td>DB</td>
<td>Eloquent</td>
<td>SQL、Doctrine</td>
</tr>
<tr>
<td>模板</td>
<td>Blade</td>
<td>Twig</td>
</tr>
<tr>
<td>处理数据</td>
<td>Laravel集合</td>
<td>数组</td>
</tr>
<tr>
<td>表单验证</td>
<td>请求类</td>
<td>第三方扩展包、控制器中验证</td>
</tr>
<tr>
<td>认证</td>
<td>内置功能</td>
<td>第三方扩展包、你自己的解决方案</td>
</tr>
<tr>
<td>API认证</td>
<td>Laravel Passport</td>
<td>第三方 JWT 和 OAuth 扩展包</td>
</tr>
<tr>
<td>创建API</td>
<td>内置功能</td>
<td>Dingo API和类似扩展包</td>
</tr>
<tr>
<td>处理DB结构</td>
<td>迁移</td>
<td>直接操作DB</td>
</tr>
<tr>
<td>本地化</td>
<td>内置功能</td>
<td>第三方工具</td>
</tr>
<tr>
<td>实时用户接口</td>
<td>Laravel Echo、Pusher</td>
<td>第三方直接处理 WebSocket的扩展包</td>
</tr>
<tr>
<td>生成测试数据</td>
<td>填充类、模型工厂、Faker</td>
<td>手动创建测试数据</td>
</tr>
<tr>
<td>任务调度</td>
<td>Laravel Task Scheduler</td>
<td>脚本或第三方扩展包</td>
</tr>
<tr>
<td>DB</td>
<td>MySQL、PostgreSQL、SQLite、SQL Server</td>
<td>MongoDB</td>
</tr>
</tbody>
</table>
<h3 id="toc_13">遵循 Laravel 命名约定</h3>
遵循 <a href="http://www.php-fig.org/psr/psr-2/" target="_blank" rel="noopener">PSR 标准</a>。此外，还要遵循 Laravel 社区版的命名约定：
<table>
<thead>
<tr>
<th>What</th>
<th>How</th>
<th>Good</th>
<th>Bad</th>
</tr>
</thead>
<tbody>
<tr>
<td>控制器</td>
<td>单数</td>
<td>ArticleController</td>
<td><del>ArticlesController</del></td>
</tr>
<tr>
<td>路由</td>
<td>复数</td>
<td>articles/1</td>
<td><del>article/1</del></td>
</tr>
<tr>
<td>命名路由</td>
<td>下划线+'.'号分隔</td>
<td>users.show_active</td>
<td><del>users.show-active,show-active-users</del></td>
</tr>
<tr>
<td>模型</td>
<td>单数</td>
<td>User</td>
<td><del>Users</del></td>
</tr>
<tr>
<td>一对一关联</td>
<td>单数</td>
<td>articleComment</td>
<td><del>articleComments,article_comment</del></td>
</tr>
<tr>
<td>其他关联关系</td>
<td>复数</td>
<td>articleComments</td>
<td><del>articleComment,article_comments</del></td>
</tr>
<tr>
<td>数据表</td>
<td>复数</td>
<td>article_comments</td>
<td><del>article_comment,articleComments</del></td>
</tr>
<tr>
<td>中间表</td>
<td>按字母表排序的单数格式</td>
<td>article_user</td>
<td><del>user_article,article_users</del></td>
</tr>
<tr>
<td>表字段</td>
<td>下划线，不带模型名</td>
<td>meta_title</td>
<td><del>MetaTitle; article_meta_title</del></td>
</tr>
<tr>
<td>外键</td>
<td>单数、带_id后缀</td>
<td>article_id</td>
<td><del>ArticleId, id_article, articles_id</del></td>
</tr>
<tr>
<td>主键</td>
<td>-</td>
<td>id</td>
<td><del>custom_id</del></td>
</tr>
<tr>
<td>迁移</td>
<td>-</td>
<td>2017_01_01_000000_create_articles_table</td>
<td><del>2017_01_01_000000_articles</del></td>
</tr>
<tr>
<td>方法</td>
<td>驼峰</td>
<td>getAll</td>
<td><del>get_all</del></td>
</tr>
<tr>
<td>资源类方法</td>
<td><a href="http://laravelacademy.org/post/7836.html#toc_6" target="_blank" rel="noopener">文档</a></td>
<td>store</td>
<td><del>saveArticle</del></td>
</tr>
<tr>
<td>测试类方法</td>
<td>驼峰</td>
<td>testGuestCannotSeeArticle</td>
<td><del>test_guest_cannot_see_article</del></td>
</tr>
<tr>
<td>变量</td>
<td>驼峰</td>
<td>$articlesWithAuthor</td>
<td><del>$articles_with_author</del></td>
</tr>
<tr>
<td>集合</td>
<td>复数</td>
<td>$activeUsers = User::active()-&gt;get()</td>
<td><del>$active, $data</del></td>
</tr>
<tr>
<td>对象</td>
<td>单数</td>
<td>$activeUser = User::active()-&gt;first()</td>
<td><del>$users, $obj</del></td>
</tr>
<tr>
<td>配置和语言文件索引</td>
<td>下划线</td>
<td>articles_enabled</td>
<td><del>ArticlesEnabled; articles-enabled</del></td>
</tr>
<tr>
<td>视图</td>
<td>下划线</td>
<td>show_filtered.blade.php</td>
<td><del>showFiltered.blade.php, show-filtered.blade.php</del></td>
</tr>
<tr>
<td>配置</td>
<td>下划线</td>
<td>google_calendar.php</td>
<td><del>googleCalendar.php, google-calendar.php</del></td>
</tr>
<tr>
<td>契约（接口）</td>
<td>形容词或名词</td>
<td>Authenticatable</td>
<td><del>AuthenticationInterface, IAuthentication</del></td>
</tr>
<tr>
<td>Trait</td>
<td>形容词</td>
<td>Notifiable</td>
<td><del>NotificationTrait</del></td>
</tr>
</tbody>
</table>
<h3 id="toc_14">使用缩写或可读性更好的语法</h3>
坏代码：
<pre><code>$request-&gt;session()-&gt;get('cart');
$request-&gt;input('name');
</code></pre>
好代码：
<pre><code>session('cart');
$request-&gt;name;
</code></pre>
更多示例：
<table>
<thead>
<tr>
<th>通用语法</th>
<th>可读性更好的</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>Session::get('cart')</code></td>
<td><code>session('cart')</code></td>
</tr>
<tr>
<td><code>$request-&gt;session()-&gt;get('cart')</code></td>
<td><code>session('cart')</code></td>
</tr>
<tr>
<td><code>Session::put('cart', $data)</code></td>
<td><code>session(['cart' =&gt; $data])</code></td>
</tr>
<tr>
<td><code>$request-&gt;input('name'), Request::get('name')</code></td>
<td><code>$request-&gt;name, request('name')</code></td>
</tr>
<tr>
<td><code>return Redirect::back()</code></td>
<td><code>return back()</code></td>
</tr>
<tr>
<td><code>is_null($object-&gt;relation) ? $object-&gt;relation-&gt;id : null }</code></td>
<td><code>optional($object-&gt;relation)-&gt;id</code></td>
</tr>
<tr>
<td><code>return view('index')-&gt;with('title', $title)-&gt;with('client', $client)</code></td>
<td><code>return view('index', compact('title', 'client'))</code></td>
</tr>
<tr>
<td><code>$request-&gt;has('value') ? $request-&gt;value : 'default';</code></td>
<td><code>$request-&gt;get('value', 'default')</code></td>
</tr>
<tr>
<td><code>Carbon::now(), Carbon::today()</code></td>
<td><code>now(), today()</code></td>
</tr>
<tr>
<td><code>App::make('Class')</code></td>
<td><code>app('Class')</code></td>
</tr>
<tr>
<td><code>-&gt;where('column', '=', 1)</code></td>
<td><code>-&gt;where('column', 1)</code></td>
</tr>
<tr>
<td><code>-&gt;orderBy('created_at', 'desc')</code></td>
<td><code>-&gt;latest()</code></td>
</tr>
<tr>
<td><code>-&gt;orderBy('age', 'desc')</code></td>
<td><code>-&gt;latest('age')</code></td>
</tr>
<tr>
<td><code>-&gt;orderBy('created_at', 'asc')</code></td>
<td><code>-&gt;oldest()</code></td>
</tr>
<tr>
<td><code>-&gt;select('id', 'name')-&gt;get()</code></td>
<td><code>-&gt;get(['id', 'name'])</code></td>
</tr>
<tr>
<td><code>-&gt;first()-&gt;name</code></td>
<td><code>-&gt;value('name')</code></td>
</tr>
</tbody>
</table>
<h3 id="toc_15">使用 IoC 容器或门面</h3>
自己创建新的类会导致代码耦合度高，且难于测试，取而代之地，我们可以使用 IoC 容器或门面。

坏代码：
<pre><code>$user = new User;
$user-&gt;create($request-&gt;all());
</code></pre>
好代码：
<pre><code>public function __construct(User $user)
{
    $this-&gt;user = $user;
}

....

$this-&gt;user-&gt;create($request-&gt;all());   
</code></pre>
<h3 id="toc_16">不要从直接从 .env 获取数据</h3>
传递数据到配置文件然后使用 <code>config</code> 辅助函数获取数据。

坏代码：
<pre><code>$apiKey = env('API_KEY');
</code></pre>
好代码：
<pre><code>// config/api.php
'key' =&gt; env('API_KEY'),

// Use the data
$apiKey = config('api.key');
</code></pre>
<h3 id="toc_17">以标准格式存储日期</h3>
使用访问器和修改器来编辑日期格式。

坏代码：
<pre><code>{{ Carbon::createFromFormat('Y-d-m H-i', $object-&gt;ordered_at)-&gt;toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object-&gt;ordered_at)-&gt;format('m-d') }}
</code></pre>
好代码：
<pre><code>// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at']
public function getMonthDayAttribute($date)
{
    return $date-&gt;format('m-d');
}

// View
{{ $object-&gt;ordered_at-&gt;toDateString() }}
{{ $object-&gt;ordered_at-&gt;monthDay }}
</code></pre>
<h3 id="toc_18">其他好的实践</h3>
不要把任何业务逻辑写到路由文件中。

在 Blade 模板中尽量不要编写原生 PHP。
