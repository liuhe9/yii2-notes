# yii2笔记

####  也可安装YII官方文档安装： https://www.yiichina.com/doc/guide/2.0/start-installation 
#### 这里开发和线上环境都是 ubuntu（18.04）+php(7.2)+nginx+mysql

## 安装nginx、php、mysql、memcache、redis、apc、imagick、supervisor、composer及php其他扩展，不用的可以去掉
```
apt-get install nginx php7.2 php7.2-fpm php7.2-mysql php7.2-memcache php7.2-redis php7.2-apc php7.2-imagick  php7.2-mbstring php7.2-gd php7.2-curl php7.2-simplexml  php7.2-intl php7.2-pdo-sqlite php7.2-pdo-mysql php7.2-pdo-pgs  php7.2-bcmath php7.2-zip php7.2-curl php7.2-xml php7.2-json php7.2-gd php7.2-mbstring supervisor 
```
## 更新php环境至7.2 (如已安装php7.2 fpm，可跳过此步骤)
```
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update
sudo apt-get -y install php7.2
sudo apt-get install php7.2-fpm
```
## nginx配置切换php版本
```
sudo update-alternatives --config php
```
## 安装composer
```
#进入某个目录下载composer
cd /path/to/composer
#安装composer
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '544e09ee996cdf60ece3804abc52599c22b1f40f4323403c44d44fdfdd586475ca9813a858088ffbc1f233e9b180f061') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
#查看composer版本
composer -v 
#拷贝，可全局使用
sudo mv composer.phar /usr/local/bin/composer
composer global require "fxp/composer-asset-plugin:^1.4.4”
```
## 进入项目目录，初始化yii2 高级模板
```
composer create-project yiisoft/yii2-app-advanced yii2-advanced
```
## 查看本机环境是否缺少扩展，expose_php  warning不用管
```
php requirements.php
```
## 初始化项目  
```
php init
``` 
## nginx配置示例，用一个域名解析到yii2所有子项目上
```
server {
    listen 80;
    server_name api.xxx.com;
    #我使用的是deploy发布的，所以这里我到current
    #set $root_path /data/deploy/api/current;
    set $root_path /data/deploy/api;
    root $root_path;
    #例如我有2个 app1  app2，其实是一样的配置，如有弄清楚泛解析的可以告诉我
    location /app1 {
        set $app_id 'app1';
        alias  $root_path/$app_id/web;
        try_files  $uri $uri/ /index.php$is_args$args;
        if (!-f $request_filename) {
            rewrite ^/([a-z]+)/(.*)$ /$1/index.php?$2 last;
        }
        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            #划重点，下面这句是重点，一定不能错，否则会404
            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        }
    }

    location /app2 {
        set $app_id 'app2';
        alias  $root_path/$app_id/web;
        try_files  $uri $uri/ /index.php$is_args$args;
        if (!-f $request_filename) {
            rewrite ^/([a-z]+)/(.*)$ /$1/index.php?$2 last;
        }
        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        }
    }
}
```

## common/config/main.php 参考
```
<?php

$schemaCache = false;

// 本机ip
$testip = '192.168.1.1';

if (YII_ENV == 'prod') {
    $schemaCache = true;
} else {
    $gii = [
        'class' => 'yii\gii\Module',
        'allowedIPs' => ['127.0.0.1','192.168.88.1', '::1'], //只允许本地访问gii
    ];

    $debug = [
        'class' => 'yii\debug\Module',
        'panels' => [
            'queue' => yii\queue\debug\Panel::class, //队列
        ],
        // uncomment the following to add your IP if you are not connecting from localhost.
        'allowedIPs' => ['127.0.0.1','192.168.88.1', '::1'],
    ];
}

$config = [
    'timeZone' => 'Asia/Shanghai',
    'language' => 'zh-CN',

    'aliases' => [
        '@bower' => '@vendor/bower-asset',
        '@npm' => '@vendor/npm-asset',
    ],
    'vendorPath' => dirname(dirname(__DIR__)) . '/vendor',
    'components' => [
        // 错误日志
        'log' => [
            'targets' => [
                [
                    'class' => 'yii\log\FileTarget',
                    'levels' => ['error'],
                    'logFile' => '@app/runtime/logs/application-'.date('y-m-d').'.log',
                ],
            ],
        ],


        // 队列专用redis
        'queue_redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => $testip,
            'password' => 'xxxx',
            'port' => 6379,
            'database' => 6,
        ],

        'queue' => [
            'class' => \yii\queue\redis\Queue::class,
            'redis' => 'queue_redis', // 用redis做队列，也可以选择beanstalk、各种mq，我们简单点，业务复杂可以选其他的
            'channel' => 'queue',
        ],

        'queue2' => [
            'class' => \yii\queue\redis\Queue::class,
            'redis' => 'queue_redis', 
            'channel' => 'queue2', 
        ],

        //数据结构缓存
        'schemaCache' => [
            'class' => 'yii\caching\FileCache',
            'cachePath' => '@console/runtime/cache',
        ],

        'urlManager' => [
            'enablePrettyUrl' => true,
            'showScriptName' => false,
        ],


        #数据库配置文件
        'testdb' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host='.$testip.';dbname=testdb',
            'username' => 'test',
            'password' => 'xxxxx',
            'charset' => 'utf8mb4',
            'tablePrefix' => '',
            'enableSchemaCache' => $schemaCache,
            'schemaCacheDuration' => 86400,
            'schemaCache' => 'schemaCache',
        ],
        // 不指定db 默认调用这里
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host='.$testip.';dbname=testdb2',
            'username' => 'test',
            'password' => 'xxxxx',
            'charset' => 'utf8mb4',
            'tablePrefix' => '',
            'enableSchemaCache' => $schemaCache,
            'schemaCacheDuration' => 86400,
            'schemaCache' => 'schemaCache',
        ],
    ]
];

if (YII_ENV_DEV) {
    $config['modules']['gii'] = $gii;
    $config['bootstrap'][] = 'gii';
    $config['modules']['debug'] = $debug;
    $config['bootstrap'][] = 'debug';
}

return $config;

```
+ migration文件指定db，重写该方法即可，上面配置文件指定的db的key值如testdb，不指定即默认用 db,执行迁移的时候如果分库也得指定db
```
    public function init()
    {
        $this->db = 'testdb';
        parent::init();
    }
```
+ model文件指定db，重写该方法即可，上面配置文件指定的db的key值如testdb, 不指定即默认用 db
```
    public static function getDb()
    {
        return Yii::$app->get('testdb');
    }
```

## console/config/main.php 支持队列
```
'bootstrap' => ['log', 'queue', 'queue2'],
```

## 安装supervisor监听queue进程 https://github.com/yiisoft/yii2-queue/blob/master/docs/guide-zh-CN/worker.md
```
sudo supervisord : 启动supervisor
sudo supervisorctl reload :修改完配置文件后重新启动supervisor
sudo supervisorctl status :查看supervisor监管的进程状态
sudo supervisorctl start 进程名 ：启动XXX进程
sudo supervisorctl stop 进程名 ：停止XXX进程
sudo supervisorctl stop all：停止全部进程，注：start、restart、stop都不会载入最新的配置文件。
sudo supervisorctl update：根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启
```
+ supervisor配置文件参考，开多个队列
```
[program:queue]
process_name=%(process_num)02d
command=php /data/deploy/xxx/current/yii queue/listen --verbose=1 --color=0
loglevel=warn
autostart=true
startretries=3
autorestart=true
user=www-data
numprocs=5
redirect_stderr=true
stdout_logfile=/tmp/queue.log

[program:queue2]
process_name=%(process_num)02d
command=php /data/deploy/xxx/current/yii queue2/listen --verbose=1 --color=0
loglevel=warn
autostart=true
startretries=3
autorestart=true
user=www-data
numprocs=5
redirect_stderr=true
stdout_logfile=/tmp/queue2.log
```
