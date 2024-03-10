---
layout: post
title: "Rate Limiter"
preview_image: 9-rate-limiter.jpeg
published_at: "18 февраля 2024 г."
telegramPostIdForComments: 20
tags: ["research", "yii2"]
---

Во многих фреймворках, таких как Laravel, Symfony, yii2, yii3 существуют механизмы для управления кол-вом запросов к тем или иным разделам сайта. Лимиты используются для контроля над количеством объектов или действий, которые пользователь может выполнить за определенный период времени.  
Это может быть полезно для предотвращения DDoS-атак или для ограничения использования ресурсов одним пользователем. Например, можно ограничить количество записей, которые пользователь может создать за определенный период времени. Ограничить кол-во запросов к вашему api, если вдруг соседний отдел, который с вами интегрируется, забыл в своем коде выйти из цикла и начинает вас бомбить запросами : )

Я хотел бы рассмотреть примеры на фреймворке yii2.

Yii2 предоставляет встроенную поддержку rate limiting через фильтр yii\filters\RateLimiter.

Для реализации лимитов в Yii2, обычно используются фильтры (filters) или поведения (behaviors), которые могут быть прикреплены к контроллерам или моделям.
Я изначально всегда думал, что мы можем ограничить контроллер, но к приятному моему удивлению, что можно еще добавить это поведение и к модели. Стоит ли это добавлять в модель, свое мнение скажу чуть позже.

Если мы посмотрим [пример из документации](https://www.yiiframework.com/doc/guide/2.0/ru/rest-rate-limiting), то поймем, что нам нужен класс User identity, который должен реализовывать yii\filters\RateLimitInterface.

Ок, я попробую это сделать.

Если мы зайдем в сам класс vendor/yiisoft/yii2/filters/RateLimiter.php, то увидим описание к классу и собственно такой код в beforeAction:

```php
    /**
     * {@inheritdoc}
     */
    public function beforeAction($action)
    {
        if ($this->user === null && Yii::$app->getUser()) {
            $this->user = Yii::$app->getUser()->getIdentity(false);
        }

        if ($this->user instanceof Closure) {
            $this->user = call_user_func($this->user, $action);
        }

        if ($this->user instanceof RateLimitInterface) {
            Yii::debug('Check rate limit', __METHOD__);
            $this->checkRateLimit($this->user, $this->request, $this->response, $action);
        } elseif ($this->user) {
            Yii::info('Rate limit skipped: "user" does not implement RateLimitInterface.', __METHOD__);
        } else {
            Yii::info('Rate limit skipped: user not logged in.', __METHOD__);
        }

        return true;
    }
```
Теперь я понимаю, что если не будет класса user, то ограничение работать не будет. Об этом так же говорится и в описании к классу.
Хорошо, первый вариант, который я хочу создать, это ограничить кол-во запросов для любых пользователей.
Создам тестовый класс для примера в app\models, который будет хранить информацию о кол-во запросов в кеше. Пример будет без учета конкурентных запросов (race condition).
```php
<?php

namespace app\models;

use yii\caching\CacheInterface;
use yii\filters\RateLimitInterface;

class IpRateLimiter implements RateLimitInterface
{
    public function __construct(
        private readonly int $limitRequests,
        private readonly int $limitTimeSeconds,
        private readonly int $rememberTimeSeconds,
        private readonly CacheInterface $cache,
    ) {
    }

    public function getRateLimit($request, $action)
    {
        return [$this->limitRequests, $this->limitTimeSeconds];
    }

    public function loadAllowance($request, $action)
    {
        $ip = $request->getRemoteIP();

        return [
            $this->cache->get("rate_limiter_{$ip}.value"),
            $this->cache->get("rate_limiter_{$ip}.timestamp"),
        ];
    }

    public function saveAllowance($request, $action, $allowance, $timestamp)
    {
        $ip = $request->getRemoteIP();
        $this->cache->set("rate_limiter_{$ip}.value", $allowance, $this->rememberTimeSeconds);
        $this->cache->set("rate_limiter_{$ip}.timestamp", $timestamp, $this->rememberTimeSeconds);
    }
}
```
Добавляю класс в конфиг приложения 'app/config/web.php':

```php
    'container' => [
        'definitions' => [
            IpRateLimiter::class => function (Container $container) {
                return new IpRateLimiter(
                    3,
                    60,
                    60,
                    Yii::$app->cache
                );
            },
        ]
    ],
```

Подключу теперь его в свой контроллер:

```php
<?php

namespace app\controllers;

use app\models\IpRateLimiter;
use Yii;
use yii\filters\AccessControl;
use yii\filters\RateLimiter;
use yii\web\Controller;
use yii\web\Response;
use yii\filters\VerbFilter;
use app\models\LoginForm;
use app\models\ContactForm;

class SiteController extends Controller
{
    public function __construct(
        $id, 
        $module, 
        private readonly IpRateLimiter $ipRateLimiter,
        $config = []
    ) {
        parent::__construct($id, $module, $config);
    }

    /**
     * {@inheritdoc}
     */
    public function behaviors()
    {
        return [
            'access' => [
                'class' => AccessControl::class,
                'only' => ['logout'],
                'rules' => [
                    [
                        'actions' => ['logout'],
                        'allow' => true,
                        'roles' => ['@'],
                    ],
                ],
            ],
            'verbs' => [
                'class' => VerbFilter::class,
                'actions' => [
                    'logout' => ['post'],
                ],
            ],
            // добавил новую секцию
            'rateLimiter' => [
                'class' => RateLimiter::class,
                'user' => $this->ipRateLimiter,
                'only' => ['index'],
            ],
        ];
    }
    // продолжение контроллера
```

Из примера выше для теста я сделал лимит в 3 запроса за 60 секунд, чтобы это можно было руками проверить.

Три раза подряд обновив страницу я вижу ожидаемое поведение:

![](/examples/2024-02-18-9-rate-limiter/web-too-many-request.png)

Что, если я хочу протестировать больше запросов... Перезагружать страницу, допустим 100 раз будет утомительно.
Есть отличная утилита apache benchmark. Попробую выполнить в один поток 5 запросов:

```bash
ab -c1 -n5 http://127.0.0.1:8101/site/index
```

После выполнения команды я вижу следующее:

```bash
Complete requests:      5
Failed requests:        2
   (Connect: 0, Receive: 0, Length: 2, Exceptions: 0)
Non-2xx responses:      2
```

Что же, отлично! Ограничитель работает. Так же рекомендую вам данную утилиту для тестирования параллельных запросов. На одном собеседовании мне нужно было выполнить задание на подсчет кол-во обращений к странице. Счетчик нужно было записывать в файл. Утилита в этом мне отлично помогла для отладки кода.

Мы так же можем добавить нашей модели User данный интерфейс и реализовать эти три метода, но из своей практики я бы советовал от этого воздержаться. Это позволит не перегружать вашу модель. Ограничение на кол-во запросов относится к моменту входа в наше приложение, а модель user относится к таблице user.

Если в каком-то контроллере происходит сохранение модели user - лучше добавить в поведение контроллера ограничение на кол-во запросов, чем в саму модель. Предположим, ваша модель сохраняется через веб-форму на сайте и через метод в апи. Нам тогда придется править модель User - для апи одно, для веба другое. А так мы будем кастомизировать только контроллеры, не боясь, что сломаем модель user.

Также, для реализации более сложных лимитов, вы можете создать свои собственные фильтры или поведения, которые будут соответствовать вашим требованиям. Вы можете их хранить в бд или еще где-либо.






