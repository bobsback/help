---

template:         article
reviewed:         2016-02-20
title:            Install Laravel 5
naviTitle:        Laravel
lead:             Laravel is the most PHPopular framework. Learn how to install and tune Laravel 5 on fortrabbit.
group:            Install_guides

websiteLink:      http://laravel.com?utm_source=fortrabbit
websiteLinkText:  laravel.com
category:         framework
image:            laravel-mark.png
version:          5.1

linkincontent:    1

otherVersionLinks:
    - install-laravel-5-old-app
    - install-laravel-4
    - install-laravel-4-old-app

tags:
    - php
    - install
    - laravel

---


## Get ready

We assume you've already created a New App with fortrabbit. You also need a local [Laravel](http://laravel.com/docs/5.1/installation) installation. 


## Install

You can either use an existing one or initialize a new one. For a new one execute locally:

```bash
$ cd ~/Projects
$ composer create-project laravel/laravel --prefer-dist MyApp
```

In any case: change into your local app directory, make sure it is initialized as a Git repo, everything is added and add your App's Git remote:

```bash
$ cd ~/Projects/MyApp
$ git init .
$ git add -A
$ git commit -m 'Initial'
$ git remote add fortrabbit git@deploy.eu2.frbit.com:your-app.git
```

**Note**: Replace `your-app` with the name of your fortrabbit App.

If you haven't already created a key for your Laravel installation, you can do it now by running locally:

```bash
$ php artisan key:generate
```

This will print out the key and write it to your local `.env` file. You can now either create another key or use that one and set is as an [environment variable](/env-vars) to your App, via the Dashboard.

Once that is done, you can just push your local repo to the App remote:

```bash
$ git push -u fortrabbit master
```

This first push can take a bit, since all the Composer packages need to be installed. While that is happening, change back to the Dashboard and [set the document root](/domains#toc-set-a-custom-root-path) of your App's domains to `public`.

When the push is done you can visit your App URL in the browser and see the Laravel welcome screen! Any subsequent push will be much faster and you can leave you the `-u fortrabbit master`.

Tuning
------

The above will give you an up and running App. However, to make the most of Laravel on fortabbit, it needs some tuning.

### Logging

Per default Laravel writes all logs to `storage/log/..`. Since you don't have direct file access, you need to configure Laravel to write to the PHP `error_log` method instead. That's easily done: open `boostrap/app.php` and add the following just before the `return $app` statement at the bottom:

```php
$app->configureMonologUsing(function($monolog) {
    // chose the error_log handler
    $handler = new \Monolog\Handler\ErrorLogHandler();
    // time will already be logged -> change default format
    $handler->setFormatter(new \Monolog\Formatter\LineFormatter('%channel%.%level_name%: %message% %context% %extra%'));
    $monolog->pushHandler($handler);
});
```

You can now use the regular [log access](logging) to view the stream.

### MySQL

You can use the [App secrets](secrets) to attain your database credentials. Modify the `config/database.php` like so:

```php
$mysql = [
    'driver'    => 'mysql',
    'host'      => env('DB_HOST', 'localhost'),
    'database'  => env('DB_DATABASE', 'forge'),
    'username'  => env('DB_USERNAME', 'forge'),
    'password'  => env('DB_PASSWORD', ''),
    'charset'   => 'utf8',
    'collation' => 'utf8_unicode_ci',
    'prefix'    => '',
    'strict'    => false,
    'engine'    => null,
];

if (isset($_SERVER['APP_SECRETS'])) {
    $secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);
    $mysql = [
        'driver'    => 'mysql',
        'host'      => $secrets['MYSQL']['HOST'],
        'port'      => $secrets['MYSQL']['PORT'],
        'database'  => $secrets['MYSQL']['DATABASE'],
        'username'  => $secrets['MYSQL']['USER'],
        'password'  => $secrets['MYSQL']['PASSWORD'],
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci',
        'prefix'    => '',
        'strict'    => false,
    ];
}

// ..
return [
    // ..
    'connections' => [
        // ..
        'mysql' => $mysql,
        // ..
    ],
    // ..
];
```


#### Setting time zone in Laravel

As Eloquent uses `PDO`, you can use the `PDO::MYSQL_ATTR_INIT_COMMAND` option. Extend your `mysql` configuration array in `app/config/database.php` or your specific environment `database.php` file:

```php
return [
    // ..
    'mysql' => [
        // ..
        'options'   => [
            \PDO::MYSQL_ATTR_INIT_COMMAND => 'SET time_zone = \'+02:00\''
        ]
    ],
    // ..
];
```


### Memcache

Make sure you enabled the [Memcache](memcache) component. Then you can use the [App secrets](app-secrets) to attain your credentials. Modify the `memcached` connection in your `config/cache.php` like so:

```php
$servers = [[
    'host' => env('MEMCACHED_HOST', '127.0.0.1'),
    'port' => env('MEMCACHED_PORT', 11211),
    'weight' => 100,
]];
if (isset($_SERVER['APP_SECRETS'])) {
    $secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);
    $servers = [[
        'host' => $secrets['MEMCACHE']['HOST1'],
        'port' => $secrets['MEMCACHE']['PORT1'],
        'weight' => 100
    ]];
    if ($secrets['MEMCACHE']['COUNT'] > 1) {
        $servers []= [
            'host' => $secrets['MEMCACHE']['HOST2'],
            'port' => $secrets['MEMCACHE']['PORT2'],
            'weight' => 100
        ];
    }
}

// ..
return [
    // ..
    'stores' => [
        // ..
        'memcached' => [
            'driver'  => 'memcached',
            'servers' => $servers
        ],
        // ..
    ],
    // ..
];
```

In addition, set the `CACHE_DRIVER` [environment variable](env-vars) so that you can use `memcached` in your production App on fortrabbit and `apc` or `array` on your local machine, via `.env` file.


### Redis

Redis can be used in Laravel as a cache or a queue or both. First integrate with [Redis Cloud](redis-cloud) then configure the redis database connection in `config/database.php`:

```php
$redis = [
    'host'     => env('REDIS_HOST', 'localhost'),
    'password' => env('REDIS_PASSWORD', null),
    'port'     => env('REDIS_PORT', 6379),
    'database' => 0,
];

if (isset($_SERVER['APP_SECRETS'])) {
    $secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);
    $redis = [
        'host'     => $secrets['CUSTOM']['REDIS_HOST'],
        'port'     => $secrets['CUSTOM']['REDIS_PORT'],
        'password' => $secrets['CUSTOM']['REDIS_PASSWORD']
    ];
}

return [
    // ...
    'redis' => [
        'cluster' => false,
        'default' => $redis
    ],
    // ...
];
```

If you plan on using Redis as a cache, then open `config/cache.php` and set `default` to `redis` (or set the `CACHE_DRIVER` [environment variable](env-vars) to `redis` in the Dashboard).

For [queue](#toc-queue) usage see below.

### Queue

Laravel supports multiple queue drivers. One which can be used with fortrabbit out of the box is `database`, which simple uses your database connection as a queue. That's great for small use-cases and tinkering, but if your App handles very many queue messages you should consider [Redis](#toc-redis).

Once you've decided the queue you want to use just open `config/queue.php` and set `default` to either `redis`, `database`, `iron` or `amqp` - or even better: set the `QUEUE_DRIVER` [environment variable](env-vars) accordingly in the Dashboard.


#### CloudAMQP

CloudAMQP  can be used in Laravel as a queue driver. First integrate with [CloudAMQP](cloudamqp), install the composer package `fhteam/laravel-amqp` then configure the queue connection in `config/queue.php`:

```php
$amqp = [];
if (isset($_SERVER['APP_SECRETS'])) {
    $secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);
    $amqp = [
        'driver'         => 'amqp',
        'host'           => $secrets['CUSTOM']['CLOUD_AMQP_HOST'],
        'port'           => $secrets['CUSTOM']['CLOUD_AMQP_PORT'],
        'user'           => $secrets['CUSTOM']['CLOUD_AMQP_USER'],
        'password'       => $secrets['CUSTOM']['CLOUD_AMQP_PASSWORD'],
        'vhost'          => $secrets['CUSTOM']['CLOUD_AMQP_VHOST'],
        'queue'          => null,
        'channel_id'     => null,
        'exchange_name'  => null,
        'exchange_type'  => null,
        'exchange_flags' => null,
        
         //Durable queue (survives server crash)
        'queue_flags'=> ['durable' => true, 'routing_key' => null],
        
        //Persistent messages (survives server crash)
        'message_properties' => ['delivery_mode' => 2],
    ];
}

return [
    // ...
    'connections' => [
        // ...
        'amqp' => $amqp,
        // ...
    ],
    // ...
];
```

Make sure you have added `Forumhouse\LaravelAmqp\ServiceProvider\LaravelAmqpServiceProvider::class` to `providers` in `config/app.php`.

Lastly set the `QUEUE_DRIVER` [environment variable](env-vars) in the Dashboard to `amqp`.

#### IronMQ

IronMQ can be used in Laravel as a queue driver. First integrate with [IronMQ](ironmq) then configure the queue connection in `config/queue.php`:

```php
$iron = [
    'driver'  => 'iron',
    'host'    => env('IRON_MQ_HOST'),
    'token'   => env('IRON_MQ_TOKEN'),
    'project' => env('IRON_MQ_PROJECT_ID'),
    'queue'   => env('IRON_MQ_QUEUE'),
    'encrypt' => true,
];
if (isset($_SERVER['APP_SECRETS'])) {
    $secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);
    $iron = [
        'driver'  => 'iron',
        'host'    => $secrets['CUSTOM']['IRON_MQ_HOST'],
        'token'   => $secrets['CUSTOM']['IRON_MQ_TOKEN'],
        'project' => $secrets['CUSTOM']['IRON_MQ_PROJECT_ID'],
        'queue'   => $secrets['CUSTOM']['IRON_MQ_QUEUE'],
        'encrypt' => true,
    ];
}

return [
    // ...
    'connections' => [
        // ...
        'iron' => $iron,
        // ...
    ],
    // ...
];
```

**Note for Laravel 5.2:** The `iron`-block does not exist and needs to be created. Also the `laravelcollective/iron-queue` composer package must be installed manually (mind adding `Collective\IronQueue\IronQueueServiceProvider::class` into `providers` in `app.php`).

Lastly set the `QUEUE_DRIVER` [environment variable](env-vars) in the Dashboard to `iron`.

### Migrate & other database commands

Open `config/database.php` and add a new connection:

```php
// ..
'connections' => [
    // ..
    'mysql-tunnel' => [
        'driver'    => 'mysql',
        'host'      => '127.0.0.1',
        'port'      => '13306',
        'database'  => 'your-app-name',
        'username'  => 'your-app-name',
        'password'  => env('DB_PASSWORD', ''),
        'charset'   => 'utf8',
        'collation' => 'utf8_unicode_ci',
        'prefix'    => '',
        'strict'    => false,
    ],
    // ..
],
```

Fetch your current database password using the `secrets` command:

```bash
$ ssh git@deploy.eu2.frbit.com secrets your-app mysql.password
```

Now open up a [tunnel](/mysql#toc-shell-tunnel-mysql) and run in another terminal window your migration or seed commands:

```bash
$ DB_PASSWORD="your database password" php artisan migrate --database="mysql-tunnel"
$ DB_PASSWORD="your database password" php artisan db:seed --database="mysql-tunnel"
```

### Persistent storage

If you require a persistent storage, for uploaded media or anything else, you can use a cloud storage. We recommend Amazon's S3 and have written up a BLOG[guide to get your started](new-app-cloud-storage-s3).

For the sake of an example, let's say you are using S3. First setup the `S3_*` [custom App secrets](/secrets#toc-adding-custom-app-secrets). Then you can open up `config/filesystems.php` and modify it akin to the following:

```php
$secrets = json_decode(file_get_contents($_SERVER['APP_SECRETS']), true);

return [
    // ..
    'disks' => [
        // ..
        's3' => [
            'driver' => 's3',
            'key'    => $secrets['CUSTOM']['S3_KEY'],
            'secret' => $secrets['CUSTOM']['S3_SECRET'],
            'region' => $secrets['CUSTOM']['S3_REGION'],
            'bucket' => $secrets['CUSTOM']['S3_BUCKET'],
        ],
        // ..
    ],
    // ..
];
```

If you want to use S3 remotely and a local storage locally, then replace the "default" value. For example like so:

```php
// ...
'default' => env('FS_TYPE', 'local'),
// ..
```

Now set `FS_TYPE` in your local `.env` file and the [environment variables](/env-vars) in the Dashboard.

### Sending mail

You can not use [sendmail](quirks#toc-mailing) on fortrabbit but Laravel provides a API over the popular SwiftMailer library. The mail configuration file is `app/config/mail.php`, and contains options allowing you to change your SMTP host, port, and credentials, as well as set a global form address for all messages delivered by the library.
