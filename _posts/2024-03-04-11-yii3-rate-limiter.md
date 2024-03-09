---
layout: post
title: "Yii3 Rate limiter"
preview_image: 11-rate-limiter.jpeg
published_at: "4 марта 2024 г."
tags: ["research", "yii3"]
---
 
Недавно я писал про [rate limiter для yii2](https://artemvolt.ru/9-rate-limiter/) и решил сделать то же самое только на yii3.
Время идет и пора выходить из зоны комфорта.  
С учетом, что я 3-ей версией еще не пользовался, я решил просто попробовать. Да, возможно, я не буду сразу изучать всю документацию и часть оставлю на исследовательский энтузиазм. Сердцу не прикажешь : )  
Первым делом я склонировал [репозиторий докера для yii](https://github.com/yiisoft/yii-docker), настроил, залез в контейнер и установил фреймворк.

Сперва я посмотрел [пример его использования](https://github.com/yiisoft/rate-limiter):
```php
use Psr\Http\Message\ServerRequestInterface;
use Yiisoft\Yii\RateLimiter\LimitRequestsMiddleware;
use Yiisoft\Yii\RateLimiter\Counter;
use Nyholm\Psr7\Factory\Psr17Factory;
use Yiisoft\Yii\RateLimiter\Policy\LimitAlways;
use Yiisoft\Yii\RateLimiter\Policy\LimitPerIp;
use Yiisoft\Yii\RateLimiter\Policy\LimitCallback;
use Yiisoft\Yii\RateLimiter\Storage\StorageInterface;
use Yiisoft\Yii\RateLimiter\Storage\SimpleCacheStorage;

/** @var StorageInterface $storage */
$storage = new SimpleCacheStorage($cache);

$counter = new Counter($storage, 2, 5);
$responseFactory = new Psr17Factory();

$middleware = new LimitRequestsMiddleware($counter, $responseFactory); // LimitPerIp by default
```

С учетом того, что обычно зависимости добавляются через di-контейнер, то я собственно полез в него config/web/di/application.php и добавил следующее:

```php
<?php

declare(strict_types=1);

use App\Handler\NotFoundHandler;
use Nyholm\Psr7\Factory\Psr17Factory;
use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Yiisoft\Cache\File\FileCache;
use Yiisoft\Definitions\DynamicReference;
use Yiisoft\Definitions\Reference;
use Yiisoft\Injector\Injector;
use Yiisoft\Middleware\Dispatcher\MiddlewareDispatcher;
use Yiisoft\Yii\RateLimiter\Counter;
use Yiisoft\Yii\RateLimiter\LimitRequestsMiddleware;
use Yiisoft\Yii\RateLimiter\Storage\SimpleCacheStorage;
use Yiisoft\Yii\RateLimiter\Storage\StorageInterface;

/** @var array $params */

return [
    Yiisoft\Yii\Http\Application::class => [
        '__construct()' => [
            'dispatcher' => DynamicReference::to(static function (Injector $injector) use ($params) {
                return $injector->make(MiddlewareDispatcher::class)
                    ->withMiddlewares($params['middlewares']);
            }),
            'fallbackHandler' => Reference::to(NotFoundHandler::class),
        ],
    ],
    \Yiisoft\Yii\Middleware\Locale::class => [
        '__construct()' => [
            'supportedLocales' => $params['locale']['locales'],
            'ignoredRequestUrlPatterns' => $params['locale']['ignoredRequests'],
        ],
    ],
    LimitRequestsMiddleware::class => function (ContainerInterface $container) {
        $cache = $container->get(CacheInterface::class);
        $storage = new SimpleCacheStorage($cache);
        $counter = new Counter(storage: $storage, limit: 2, periodInSeconds: 60);
        $responseFactory = $container->get(Psr17Factory::class);
        return new LimitRequestsMiddleware($counter, $responseFactory);
    },
];

```

Разница с yii2 в том, что здесь rate limiter реализован как независимая библиотека, совместимая с PSR-интерфейсами, то есть мы можем его подключить в любой другой проект вне зависимости от способа его реализации. 

В документации есть примеры более детальной его настройки, но я остановился пока на типовом использовании. 
После добавления настройки в di, я иду в config/common/routes.php и изменяю готовые пример из:
```php
Route::get('/')
    ->action([SiteController::class, 'index'])
    ->name('home')
```
на
```php
Route::get('/')
    ->action([SiteController::class, 'index'])
    ->prependMiddleware(LimitRequestsMiddleware::class)
    ->name('home')
```

Обновляю главную страницу несколько раз подряд и ничего не происходит. Интересно, почему. Я делаю дамп переменной (xdebug я настрою позднее : ):
```php
...
LimitRequestsMiddleware::class => function (ContainerInterface $container) {
        $cache = $container->get(CacheInterface::class);
        dump($cache);
        ....
```

Вижу, что фреймворк уже подставляет Yiisoft\Cache\ArrayCache для интерфейса Psr\SimpleCache\CacheInterface. ArrayCache кеш хранит кол-во в памяти, поэтому при каждой перезагрузке кеш обнуляется.

Я хочу, чтобы запоминалось и интуитивно в PhpStorm'е начинаю вводить что-то FileCa.. и получаю Yiisoft\Cache\File\FileCache. Отлично, но я хочу, чтобы теперь для любой библиотеки у меня использовался файловых кеш по-умолчанию. 

Добавляю следующую строчку в config/web/di/application.php:
```php
<?php

declare(strict_types=1);

use App\Handler\NotFoundHandler;
use Nyholm\Psr7\Factory\Psr17Factory;
use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Yiisoft\Cache\File\FileCache;
use Yiisoft\Definitions\DynamicReference;
use Yiisoft\Definitions\Reference;
use Yiisoft\Injector\Injector;
use Yiisoft\Middleware\Dispatcher\MiddlewareDispatcher;
use Yiisoft\Yii\RateLimiter\Counter;
use Yiisoft\Yii\RateLimiter\LimitRequestsMiddleware;
use Yiisoft\Yii\RateLimiter\Storage\SimpleCacheStorage;
use Yiisoft\Yii\RateLimiter\Storage\StorageInterface;

/** @var array $params */

return [
    ...
    CacheInterface::class => function (ContainerInterface $container) {
        return $container->get(FileCache::class);
    },
    LimitRequestsMiddleware::class => function (ContainerInterface $container) {
        $cache = $container->get(CacheInterface::class);
        $storage = new SimpleCacheStorage($cache);
        $counter = new Counter(storage: $storage, limit: 2, periodInSeconds: 60);
        $responseFactory = $container->get(Psr17Factory::class);
        return new LimitRequestsMiddleware($counter, $responseFactory);
    },
];
```

Вот, теперь после нескольких попыток обновить страницу я вижу сообщение Too Many Requests.
Отлично, работает! Но у меня возник вопрос, с какими настройками вернулся FileCache. То есть у файла FileCache есть параметры в конструкторе:
```php
    public function __construct(
        private string $cachePath,
        private int $directoryMode = 0775,
    ) {
        if (!$this->createDirectoryIfNotExists($cachePath)) {
            throw new CacheException("Failed to create cache directory \"$cachePath\".");
        }
    }
```
Если бы класса FileCache не было бы в di до моего кода, сейчас я бы получил ошибку. Сделав дамп переменной:
```php
CacheInterface::class => function (ContainerInterface $container) {
   $cache = $container->get(FileCache::class);
   dump($cache);
   return $cache;
},
```
Вижу следующее:
```php
Yiisoft\Cache\File\FileCache#560
(
    [Yiisoft\Cache\File\FileCache:fileSuffix] => '.bin'
    [Yiisoft\Cache\File\FileCache:fileMode] => null
    [Yiisoft\Cache\File\FileCache:directoryLevel] => 1
    [Yiisoft\Cache\File\FileCache:gcProbability] => 10
    [Yiisoft\Cache\File\FileCache:cachePath] => '/app/runtime/cache'
    [Yiisoft\Cache\File\FileCache:directoryMode] => 509
)
```
Как видно, cachePath уже удачно предустановлен. В конфигах установленного приложения я не нашел настройки для FileCache.
У yii3 в описании [настройки конфигурации](https://yiisoft.github.io/docs/guide/en/concept/configuration.html) говорится про config/packages/merge_plan.php.

В самом файле я вижу:
```php
<?php

declare(strict_types=1);

// Do not edit. Content will be replaced.
return [
    '/' => [
        'di' => [
            'yiisoft/cache-file' => [
                'config/di.php',
            ],
        // ... остальной код
```
Если я закомментирую строчку с 'config/di.php', то получу ошибку на сайте:
```php
No definition or class found or resolvable for "Yiisoft\Cache\File\FileCache" while building "Yiisoft\Cache\File\FileCache".
```

Иду в папку, где лежит наш класс FileCache, а именно в app/vendor/yiisoft/cache-file и нахожу config/di.php:
```php
<?php

declare(strict_types=1);

use Yiisoft\Aliases\Aliases;
use Yiisoft\Cache\File\FileCache;

/* @var $params array */

return [
    FileCache::class => static fn (Aliases $aliases) => new FileCache(
        $aliases->get($params['yiisoft/cache-file']['fileCache']['path'])
    ),
];
```
Собственно, вот и ответ откуда берется предустановленный кеш.
Довольно удобный инструмент с учетом того, что разрабатывая пакет для приложения, можно подгружать дефолтные конфиги. Это можно так же встретить в laravel и symfony.

В конечном итоге мой файл config/web/di/application.php:
```php
<?php

declare(strict_types=1);

use App\Handler\NotFoundHandler;
use Nyholm\Psr7\Factory\Psr17Factory;
use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Yiisoft\Cache\File\FileCache;
use Yiisoft\Definitions\DynamicReference;
use Yiisoft\Definitions\Reference;
use Yiisoft\Injector\Injector;
use Yiisoft\Middleware\Dispatcher\MiddlewareDispatcher;
use Yiisoft\Yii\RateLimiter\Counter;
use Yiisoft\Yii\RateLimiter\LimitRequestsMiddleware;
use Yiisoft\Yii\RateLimiter\Storage\SimpleCacheStorage;
use Yiisoft\Yii\RateLimiter\Storage\StorageInterface;

/** @var array $params */

return [
    Yiisoft\Yii\Http\Application::class => [
        '__construct()' => [
            'dispatcher' => DynamicReference::to(static function (Injector $injector) use ($params) {
                return $injector->make(MiddlewareDispatcher::class)
                    ->withMiddlewares($params['middlewares']);
            }),
            'fallbackHandler' => Reference::to(NotFoundHandler::class),
        ],
    ],
    \Yiisoft\Yii\Middleware\Locale::class => [
        '__construct()' => [
            'supportedLocales' => $params['locale']['locales'],
            'ignoredRequestUrlPatterns' => $params['locale']['ignoredRequests'],
        ],
    ],
    CacheInterface::class => function (ContainerInterface $container) {
        return $container->get(FileCache::class);
    },
    LimitRequestsMiddleware::class => function (ContainerInterface $container) {
        $cache = $container->get(CacheInterface::class);
        $storage = new SimpleCacheStorage($cache);
        $counter = new Counter(storage: $storage, limit: 2, periodInSeconds: 60);
        $responseFactory = $container->get(Psr17Factory::class);
        return new LimitRequestsMiddleware($counter, $responseFactory);
    },
];
```

В документации сказано, что так же можно использовать сервис провайдеры, но с этим я ознакомлюсь позднее : )




