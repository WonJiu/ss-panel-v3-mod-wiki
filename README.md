### 配合后端 https://github.com/010ada/ss-panel-v3-mod-wiki/blob/master/%E5%90%8E%E7%AB%AF%E4%BF%AE%E6%94%B9.md

## 一、mysql 修改

#### 给ss_node表添加 port_group min_port max_port 字段
```mysql
ALTER TABLE `ss_node` ADD column `port_group` int(3) NOT NULL DEFAULT '0';
ALTER TABLE `ss_node` ADD column `min_port` int(11) NOT NULL DEFAULT '1025';
ALTER TABLE `ss_node` ADD column `max_port` int(11) NOT NULL DEFAULT '65500';
```

#### 添加新的表
存储自定义端口段的用户数据
```mysql
CREATE TABLE `user_method` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `user_id` int(11) NOT NULL,
    `node_id` int(11) NOT NULL,
    `port` int(11) NOT NULL,
    `passwd` varchar(16) NOT NULL,
    `method` varchar(64) NOT NULL DEFAULT 'chacha-20',
    `protocol` varchar(128) DEFAULT 'origin',
    `protocol_param` varchar(128) DEFAULT '',
    `obfs` varchar(128) DEFAULT 'plain',
    `obfs_param` varchar(128) DEFAULT '',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

## 二、app/Models/UserMethod.php
创建文件，并添加以下内容
```php
<?php
namespace App\Models;
class UserMethod extends Model
{
    protected $connection = "default";
    protected $table = "user_method";
}
```

## app/Utils/Tools.php
添加以下函数，根据已有端口和端口段范围，随机获取端口

用于重置（更换端口段）或获取端口（新增自定义端口的节点）
```php
    public static function getAvPort_ForPortGroup($port_group_array, $node_id)
    {
        $det = UserMethod::where('node_id',$node_id)->pluck('port')->toArray();
        $port = array_diff(range($port_group_array['min_port'], $port_group_array['max_port']), $det);
        shuffle($port);

        return $port[0];
    }
```

## 三、app/Controllers/Admin/NodeController.php

#### index函数 public function index
```$table_config['total_column'] = Array(```

后面，添加 ```"port_group" => "端口段",```

#### add函数 public function add
1、```$node = new Node();```
下一行添加
```php
$node->port_group = $request->getParam('port_group');
$node->min_port = $request->getParam('min_port');
$node->max_port = $request->getParam('max_port');
$port_group_array = [$node->min_port, $node->max_port];
```

3、return 上一行，添加
```php
        $node = Node::where('server',$request->getParam('server'))->where('name',$request->getParam('name'))->first();
        if ( ($node->sort == 0 || $node->sort == 9 || $node->sort == 10) && ($node->port_group != 0) ) {
            $users = User::all();
            foreach ($users as $user) {
    		    $newmethod = new UserMethod();
                $newmethod->user_id = $user->id;
                $newmethod->port = Tools::getAvPort_ForPortGroup($port_group_array, $node->id);
                $newmethod->passwd = $user->passwd;
                $newmethod->node_id = $node->id;
                $newmethod->method = $user->method;
                $newmethod->protocol = $user->protocol;
                $newmethod->protocol_param = $user->protocol_param;
                $newmethod->obfs = $user->obfs;
                $newmethod->obfs_param = $user->obfs_param;
                $newmethod->save();
            }
        }
```

#### update函数 public function update
1、```$node = Node::find($id);```
下一行添加
```php
//  准备一份旧数据
$old_node = $node;

$node->port_group = $request->getParam('port_group');
$node->min_port = $request->getParam('min_port');
$node->max_port = $request->getParam('max_port');
$port_group_array => [
    'min_port' => $node->min_port,
    'max_port' => $node->max_port
];

// 判断是否需要修改user_method内的数据

// 默认端口段则不修改

$change_user_method=false;

// 启用自定义端口段
if ($node->port_group == 1){
    // 对比新旧自定义端口是否更改
    if ( ! ($old_node->min_port == $node->min_port && $old_node->max_port == $node->max_port) ){
        $change_user_method=true;
    }
}


```

2、```if (!$node->save()) {```
上一行添加
```php
        if ( ($node->sort == 0 || $node->sort == 9 || $node->sort == 10) ) {

            // 表中无数据，则新增
            // 有数据，则检查旧端口是否处于端口段内，否，则重置

            if ($change_user_method){
                $users = User::all();
                $log_exist = UserMethod::where('node_id',$node->id)->first();
                if ($log_exist==null) {
                    foreach ($users as $user) {
                        $newmethod = new UserMethod();
                        $newmethod->user_id = $user->id;
                        $newmethod->passwd = $user->passwd;
                        $newmethod->port = Tools::getAvPort_ForPortGroup($port_group_array, $node->id);
                        $newmethod->node_id = $node->id;
                        $newmethod->method = $user->method;
                        $newmethod->protocol = $user->protocol;
                        $newmethod->protocol_param = $user->protocol_param;
                        $newmethod->obfs = $user->obfs;
                        $newmethod->obfs_param = $user->obfs_param;
                        $newmethod->save();
                    }
                }else{
                    foreach ($users as $user) {
                        $log_exist = UserMethod::where('node_id',$node->id)->where('user_id',$user->id)->first();
                        if ($log_exist==null) {
                            $newmethod = new UserMethod();
                            $newmethod->user_id = $user->id;
                            $newmethod->passwd = $user->passwd;
                            $newmethod->port = Tools::getAvPort_ForPortGroup($port_group_array, $node->id);
                            $newmethod->node_id = $node->id;
                            $newmethod->method = $user->method;
                            $newmethod->protocol = $user->protocol;
                            $newmethod->protocol_param = $user->protocol_param;
                            $newmethod->obfs = $user->obfs;
                            $newmethod->obfs_param = $user->obfs_param;
                            $newmethod->save();
                        } else {
                            if ( ! ($log_exist->port <= $port_group_array['max_port'] && $log_exist->port >= $port_group_array['min_port']) ){
                                $log_exist->port=Tools::getAvPort_ForPortGroup($port_group_array, $node->id);
                                $log_exist->save();
                            }
                        }
                    }
                }
            }

        }
```

#### ajax函数 public function ajax
1、```$total_column = Array(```
后面添加
```"port_group" => "端口段","min_port" => "最小端口","max_port"=>"最大端口"```

2、```$body = $response->getBody();```
上一行添加
```php
        $datatables->edit('port_group', function ($data) {
            $port_group = '';
            switch($data['port_group']) {
                case 0:
                  $port_group = '默认';
                  break;
                case 1:
                  $port_group = $data['min_port']." - ".$data['max_port'];
                  break;
            }
            return $port_group;
        });
```

## 四、管理员界面

### resources/views/material/admin/node/create.tpl
1、在以下内容
```html
                                    <div class="form-group form-group-label">
                                        <label class="floating-label" for="group">节点群组（分组为数字，不分组请填0）</label>
                                        <input class="form-control" id="group" type="number" value="0" name="group">
                                    </div>
```
的后面添加这些
```html
                                    <div class="form-group form-group-label">
                                        <div class="checkbox switch">
                                            <label for="port_group">
                                                <input class="access-hide" id="port_group" name="port_group" type="checkbox"><span class="switch-toggle"></span>启用自定义端口段
                                            </label>
                                        </div>
                                    </div>
                                    <div class="form-group form-group-label">
                                        <label class="floating-label" for="min_port">最小端口</label>
                                        <input class="form-control" id="min_port" name="min_port" type="number" value="1025">
                                    </div>
                                    <div class="form-group form-group-label">
                                        <label class="floating-label" for="max_port">最大端口</label>
                                        <input class="form-control" id="max_port" name="max_port" type="number" value="65500">
                                    </div>

```

2、在以下内容
```javascript
            if(document.getElementById('type').checked)
            {
                var type=1;
            }
            else
            {
                var type=0;
            }
```
的后面添加这些
```javascript
            if(document.getElementById('port_group').checked)
            {
                var port_group=1;
            }
            else
            {
                var port_group=0;
            }
```

3、在以下内容
```javascript
                    name: $("#name").val(),
```
的后面添加这些
```javascript
                    min_port: $("#min_port").val(),
                    max_port: $("#max_port").val(),
                    type: port_group,
```

### resources/views/material/admin/node/edit.tpl 修改同上

## 五、用户界面

#### config/routes
```php
// User Center
$app->group('/user', function () {
	$this->get('', 'App\Controllers\UserController:index');
```
后加上这三行
```php
    $this->get('/nodeedit', 'App\Controllers\UserController:nodeedit');
    $this->post('/usermethod', 'App\Controllers\UserController:updateUserMethod');
    $this->post('/usermethodport', 'App\Controllers\UserController:updateUserMethodPort');
```

#### app/Controllers/UserController.php
添加以下函数，函数名称与上一步对应
```php
    public function nodeedit($request, $response, $args)
    {
        $user = Auth::getUser();

        $nodes = Node::where('port_group', 1)->get()->whereIn('sort',[1,9,10]);
        if (count($nodes)==0) {
            return $this->view()->assign('nodes', 0)->display('user/nodeedit.tpl');
        } elseif ($user->is_admin) {
            $nodes = $nodes->sortBy('name');
        } else {
            $nodes = $nodes->whereIn('node_group', [0,$user->node_group])->filter(function ($item) use($user) {
                return $item['node_class'] <= $user->class;
            })->sortBy('name');
        }

        $usermethods = UserMethod::where('user_id',$user->id)->get();
        return $this->view()->registerClass("URL", "App\Utils\URL")->assign('nodes', $nodes)->assign('method_list', Config::getSupportParam('method'))->assign('protocol_list', Config::getSupportParam('protocol'))->assign('obfs_list', Config::getSupportParam('obfs'))->assign('usermethods', $usermethods)->display('user/nodeedit.tpl');
    }
    public function updateUserMethod($request, $response, $args)
    {
        $node_id = $request->getParam('node_id');
        $protocol = $request->getParam('protocol');
        $obfs = $request->getParam('obfs');
        $sspwd = $request->getParam('sspwd');
        $method = $request->getParam('method');
        $method = strtolower($method);

        $user = Auth::getUser();

        if ($node_id == ""||$protocol == ""||$obfs == ""||$sspwd == ""||$method == "") {
            $res['ret'] = 0;
            $res['msg'] = "请填好相应内容";
            return $response->getBody()->write(json_encode($res));
        }
        if (!Tools::is_param_validate('method', $method) || !Tools::is_param_validate('obfs', $obfs) || !Tools::is_param_validate('protocol', $protocol) || !Tools::is_validate($sspwd)) {
            $res['ret'] = 0;
            $res['msg'] = "悟空别闹";
            return $response->getBody()->write(json_encode($res));
        }

        $antiXss = new AntiXSS();

        $user->protocol = $antiXss->xss_clean($protocol);
        $user->obfs = $antiXss->xss_clean($obfs);

        if (!Tools::checkNoneProtocol($user)) {
            $res['ret'] = 0;
            $res['msg'] = "您好，系统检测到您目前的加密方式为 none ，但您将要设置为的协议并不在以下协议<br>".implode(',', Config::getSupportParam('allow_none_protocol')).'<br>之内，请您先修改您的加密方式，再来修改此处设置。';
            return $this->echoJson($response, $res);
        }

        if(!URL::SSCanConnect($user) && !URL::SSRCanConnect($user)) {
            $res['ret'] = 0;
            $res['msg'] = "很抱歉，由于您的设置会导致没有客户端能够连接，系统已拒绝了您的设置！";
            return $this->echoJson($response, $res);
        }

        $usermethod = UserMethod::where('user_id',$user->id)->where('node_id',$node_id)->first();
        $usermethod->passwd = $sspwd;
        $usermethod->method = $method;
        $usermethod->protocol = $protocol;
        $usermethod->obfs = $obfs;
        $usermethod->save();

        if(!URL::SSCanConnect($user)) {
            $res['ret'] = 0;
            $res['msg'] = "设置成功，但您目前的协议、混淆、加密方式设置会导致 SS 原版客户端无法连接，请您自行更换到 SSR 客户端。";
            return $this->echoJson($response, $res);
        }

        if(!URL::SSRCanConnect($user)) {
            $res['ret'] = 0;
            $res['msg'] = "设置成功，但您目前的协议、混淆、加密方式设置会导致 SSR 客户端无法连接，请您自行更换到 SS 客户端。";
            return $this->echoJson($response, $res);
        }

        $res['ret'] = 0;
        $res['msg'] = "设置成功，您可自由选用客户端来连接。";
        return $this->echoJson($response, $res);
    }
    public function updateUserMethodPort($request, $response, $args)
    {
        $user = $this->user;
        $node_id = $request->getParam('node_id');
        $user_method = UserMethod::where('user_id',$user->id)->where('node_id',$node_id)->first();

        $node = Node::where('id',$node_id)->first();

        $old_port = $user_method->port;

        $port_group_array => [
            'min_port' => $node->min_port,
            'max_port' => $node->max_port
        ];

        $user_method->port = Tools::getAvPort_ForPortGroup($port_group, $node_id);
        $user_method->save();

        $relay_rules = Relay::where('user_id', $user->id)->where('port', $old_port)->get();
        foreach ($relay_rules as $rule) {
            $rule->port = $user_method->port;
            $rule->save();
        }

        $res['ret'] = 1;
        $res['msg'] = "设置成功，新端口是".$user_method->port;
        return $response->getBody()->write(json_encode($res));
    }
```

#### resources/views/material/user/nodeedit.tpl
新增文件，内容见下

https://github.com/010ada/ss-panel-v3-mod-wiki/blob/master/nodeedit.tpl

## 六、订阅及其他链接生成
#### app/Utils/URL.php
1、添加函数，在函数 getItem 前
```php
    public static function checkPortGroup($user, $node) {
        $new_user = clone $user;

        if ($node->port_group == 1) {
            $u = UserMethod::where('user_id',$user->id)->where('node_id',$node->id)->first();
            $new_user->port = $u->port;
            $new_user->passwd = $u->passwd;
            $new_user->method = $u->method;
            $new_user->protocol = $u->protocol;
            $new_user->protocol_param = $u->protocol_param;
            $new_user->obfs = $u->obfs;
            $new_user->obfs_param = $u->obfs_param;
        }

        return $new_user;
    }
```
2、修改 getItem 函数
```php
        $node_name = $node->name;
```
后面添加一行
```php
        $user = URL::checkSingleNode($user, $node);
```
