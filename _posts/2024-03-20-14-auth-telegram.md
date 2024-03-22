---
layout: post
title: "Авторизация на сайте через телегу"
preview_image: 14-auth-telegram.jpeg
published_at: "20 марта 2024 г."
tags: ["debug"]
source:
  author:
    link: "https://unsplash.com/@onurbinay?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash"
    name: "Onur Binay"
  image:
    link: "https://unsplash.com/photos/a-person-holding-a-phone-Uw_8vSroCSc?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash"
---

Ну, как обычно, в общем довелось делать авторизацию на сайте через telegram. Как это было...  
Хорошо, что мне не пришлось проксировать запросы на локальный сайт, как [при работе с ботом](/10-debug-telegram-bot/), виджет работает и на локальном окружении.

Первое, что я сделал, создал бота через BotFather с именем MySiteAuthBot. После этого вам нужно вызвать команду в botFather /setdomain. После выбираем бота и указываем наш локальный сайт.
Если у вас локальный сайт имеет зону .ddk, поменяйте на .biz, у меня .ddk телега не распознавал, как нормальный домен. 
Потом зашел на сайте телеги для [создания виджета авторизации](https://core.telegram.org/widgets/login). 
По сути это тег скрипт с необходимым атрибутами, который я решил вынести в виджет yii2:

```php
<?php

declare(strict_types=1);

namespace frontend\modules\Teacher\widgets\TelegramSocialAuth;

use yii\base\Widget;
use yii\helpers\Html;

final class TelegramSocialAuthWidget extends Widget
{
    public function run()
    {
        return Html::tag('script', '', [
            'async' => true,
            'src' => 'https://telegram.org/js/telegram-widget.js?22',
            'data-telegram-login' => 'MySiteAuthBot',
            'data-size' => 'large',
            'data-auth-url' => 'http://mysite.biz/teacher/auth/telegram',
        ]);
    }
}
```

Ок, скрипт есть и при посещении страницы я вижу красивую кнопочку. Стилизация мне нравится, поэтому я не стал изменять какие-либо стили кнопки.
После авторизации, телега перенаправляет пользователя на мой auth-url с дополнительными get-параметрами в адресе.
Это были следующие параметры:
  - id
  - first_name
  - username
  - photo_url
  - auth_date
  - hash

Увы, я не могу просто так взять id и сохранить его у себя, потому что мне нужно удостовериться, что данные пришли от телеграма.
Для этого мне нужно создать хеш из полученных данных по заданному алгоритму и сравнить с их переданным хешом. Они пишут об этом на свой странице создания виджета в секции "Checking authorization". Так же там есть ссылка [с примером проверки на php](https://gist.github.com/anonymous/6516521b1fb3b464534fbc30ea3573c2), за что им большое спасибо.

Я сделал валидатор на их примере.
Dto, которое подается на вход валидатора:
```php
<?php

declare(strict_types=1);

namespace common\components\SocialAuth;

final class TelegramIncomeHashValidatorDto
{
    public function __construct(
        public readonly int $id,
        public readonly ?string $firstName,
        public readonly ?string $userName,
        public readonly ?string $photoUrl,
        public readonly ?int $authDate,
        public readonly ?string $incomeHash,
    ) {
    }
}
```
Сам валидатор:
```php
<?php

declare(strict_types=1);

namespace common\components\SocialAuth;

use DateTimeImmutable;
use Psr\Clock\ClockInterface;

final class TelegramIncomeHashValidator
{
    public function __construct(
        private readonly string $botToken,
        private readonly ClockInterface $clock,
    ) {
    }

    public function validate(TelegramIncomeHashValidatorDto $dto): array
    {
        $timestampDiff = time() - $dto->authDate;
        if ($timestampDiff >= 86400) {
            return ['time is invalid'];
        }

        $attributesWithValues = $this->getAttributesWithValues($dto);
        $attributesWithValuesPairQuery = [];
        foreach ($attributesWithValues as $key => $value) {
            $attributesWithValuesPairQuery[] = "$key=$value";
        }
        $attributesWithValuesPairQuerySorted = $attributesWithValuesPairQuery;
        sort($attributesWithValuesPairQuerySorted);
        $attributesWithValuesPairQuerySortedStr = implode("\n", $attributesWithValuesPairQuerySorted);

        $secretKey = hash('sha256', $this->botToken, true);
        $hash = hash_hmac('sha256', $attributesWithValuesPairQuerySortedStr, $secretKey);
        if (strcmp($hash, $dto->incomeHash) !== 0) {
            return ['hash is invalid'];
        }

        return [];
    }

    private function getAttributesWithValues(TelegramIncomeHashValidatorDto $dto): array
    {
        return [
            'id' => $dto->id,
            'first_name' => $dto->firstName,
            'username' => $dto->userName,
            'photo_url' => $dto->photoUrl,
            'auth_date' => $dto->authDate,
        ];
    }
}
```

Добавляю прокидывание токена через di:

```php
<?php

declare(strict_types=1);

use common\components\SocialAuth\TelegramIncomeHashValidator;
use yii\di\Container;
use yii\helpers\ArrayHelper;
use yii\rbac\ManagerInterface;

return [
        'definitions' => [
            ManagerInterface::class => function (Container $container) {
                return Yii::$app->authManager;
            },
            TelegramIncomeHashValidator::class => function (Container $container) {
                $telegramToken = Yii::$app->params['telegram']['socialAuthBot']['token'];
                return new TelegramIncomeHashValidator($telegramToken);
            },
        ]
];
```

Но, что-то мне было лениво тестировать валидатор руками через обновление страницы и я решил написать unit-тест, к тому же это нужно сделать один раз. Я сохранил полученные параметры от телеграма, считая, что это будет наш эталонный пример.

Так как yii2 мне предоставляет codeception для тестирования, я создал тест для валидатор с таким же расположением, как и основной, только от папки test (так мне будет проще искать его):
```bash
root@9c8d3bcab292:/var/www/common# ../vendor/bin/codecept generate:test unit "components/SocialAuth/TelegramIncomeHashValidator"
Test was created in /var/www/common/tests/unit/components/SocialAuth/TelegramIncomeHashValidatorTest.php
```

Дополняю файл теста:

```php
<?php

declare(strict_types=1);

namespace common\tests\components\SocialAuth;

use Codeception\Test\Unit;
use common\components\SocialAuth\TelegramIncomeHashValidator;
use common\components\SocialAuth\TelegramIncomeHashValidatorDto;
use Yii;

class TelegramIncomeHashValidatorTest extends Unit
{
    public function testValidateSuccess()
    {
        $this->assertEmpty(
            $this->createValidator()->validate(
                new TelegramIncomeHashValidatorDto(
                    274738597,
                    'Artem ©',
                    'artemvolt',
                    'https://t.me/i/userpic/320/Tjq4NbALQ_2m--HVfstnR6PjpPhRgVduEZmFSuw5eWg.jpg',
                    1710969404,
                    'd61f440f5c059ea7e4bd9b03a64a256d7b3f5867c656368159e4bd68e160e6c8'
                )
            )
        );
    }
    
    // решил создать варианты, когда в каждом наборе хотя бы один параметр неверный и пятый элементы, когда все неверны
    // к тому же это не так сложно можно сделать
    public function errorDataProvider(): array
    {
        $initialData = [
            274738597,
            'Artem123',
            'artemvolt',
            'https://t.me/i/userpic/320/Tjq4NbALQ_2m--HVfstnR6PjpPhRgVduEZmFSuw5eWg.jpg',
        ];

        $result = [];

        $allWrongParamsItem = [];

        foreach ($initialData as $key => $row) {
            $dataForTest = $initialData;
            $rowForChange = $dataForTest[$key];
            if (is_int($rowForChange)) {
                $rowForChange += 1;
            } else {
                $rowForChange .= 'Test';
            }

            $dataForTest[$key] = $rowForChange;
            $allWrongParamsItem[$key] = $rowForChange;
            $result[] = $dataForTest;
        }

        $result[] = $allWrongParamsItem;

        return $result;
    }

    /**
     * @dataProvider errorDataProvider
     */
    public function testValidateErrorData($id, $firstName, $username, $photoUrl)
    {
        $this->assertEqual(
            ['hash is invalid'],
            $this->createValidator()->validate(
                new TelegramIncomeHashValidatorDto(
                    $id,
                    $firstName,
                    $username,
                    $photoUrl,
                    1710969404,
                    'd61f440f5c059ea7e4bd9b03a64a256d7b3f5867c656368159e4bd68e160e6c8'
                )
            )
        );
    }

    private function createValidator(): TelegramIncomeHashValidator
    {
        return new TelegramIncomeHashValidator('my_token'); // здесь мой токен от тестового бота
    }
}
```

Вроде бы все хорошо, подумал я, но надо бы еще добавить тест на дату и вот тут возникает вопрос, как подменить текущую дату.
Функция time() с каждым запуском теста будет все увеличиваться и в какой-то момент тест упадет.
Для этого случая, я установил библиотеку psr/clock, чтобы использовать PSR интерфейс:
```bash
composer require psr/clock
```

Потом создал Clock-класс:
```php
<?php

declare(strict_types=1);

namespace common\components\Clock;

use DateTimeImmutable;
use Psr\Clock\ClockInterface;

class Clock implements ClockInterface
{
    public function now(): DateTimeImmutable
    {
        return new DateTimeImmutable();
    }
}
```

Теперь немного изменю свой валидатор:

```php
<?php

declare(strict_types=1);

namespace common\components\SocialAuth;

use DateTimeImmutable;
use Psr\Clock\ClockInterface;

final class TelegramIncomeHashValidator
{
    public function __construct(
        private readonly string $botToken,
        private readonly ClockInterface $clock,
    ) {
    }

    public function validate(TelegramIncomeHashValidatorDto $dto): array
    {
        $authDate = date('Y-m-d H:i:s', $dto->authDate);
        $authDateTime = new DateTimeImmutable($authDate);
        $now = $this->clock->now();
        $timestampDiff = $now->getTimestamp() - $authDateTime->getTimestamp();
        if ($timestampDiff >= 86400) {
            return ['time is invalid'];
        }

        //...
    }

    //...
}
```

В данном случае, я добавил зависимость в конструктор класса, чтобы подменить класс в тесте для выставления определенной даты.
Теперь в di изменю инициализацию класса валидатора:

```php
        TelegramIncomeHashValidator::class => function (Container $container) {
            $telegramToken = Yii::$app->params['telegram']['socialAuthBot']['token'];
            return new TelegramIncomeHashValidator(
                botToken: $telegramToken,
                clock: $container->get(ClockInterface::class),
            );
        },
```

Расширим наш тест класс:
```php
<?php

declare(strict_types=1);

namespace common\tests\components\SocialAuth;

use Codeception\Test\Unit;
use common\components\SocialAuth\TelegramIncomeHashValidator;
use common\components\SocialAuth\TelegramIncomeHashValidatorDto;
use DateTimeImmutable;
use Psr\Clock\ClockInterface;
use Yii;

class TelegramIncomeHashValidatorTest extends Unit
{
    // здесь идут прошлые методы из примера выше

    public function authDatesDataProvider(): array
    {
        return [
            ["2024-03-19 01:00:01", false],
            ["2024-03-19 00:00:01", false],
            ["2024-03-19 00:00:00", true],
            ["2024-03-18 00:00:00", true],
            ["2024-03-18 01:00:00", true],
        ];
    }

    /**
     * @dataProvider authDatesDataProvider
     */
    public function testValidateDateError($authDate, $isErrorDate)
    {
        $this->assertEquals(
            $isErrorDate ? ['time is invalid'] : ['hash is invalid'],
            $this->createValidator()->validate(
                new TelegramIncomeHashValidatorDto(
                    274738597,
                    'Artem ©',
                    'artemvolt',
                    'https://t.me/i/userpic/320/Tjq4NbALQ_2m--HVfstnR6PjpPhRgVduEZmFSuw5eWg.jpg',
                    (new DateTimeImmutable($authDate))->getTimestamp(),
                    'd61f440f5c059ea7e4bd9b03a64a256d7b3f5867c656368159e4bd68e160e6c8'
                )
            )
        );
    }

    // здесь я выставил опредленную дату, чтобы разница по времени всегда была одинакова для теста
    private function createValidator(): TelegramIncomeHashValidator
    {
        $clock = $this->createMock(ClockInterface::class);
        $clock->method('now')->willReturn(new DateTimeImmutable("2024-03-20 00:00:00"));

        return new TelegramIncomeHashValidator(
            'my_token', // здесь токен от тестового бота
            $clock
        );
    }
}
```

Тест работает, ура:
```bash
root@9c8d3bcab292:/var/www/common# ../vendor/bin/codecept run unit tests/unit/components/SocialAuth/TelegramIncomeHashValidatorTest.php           

Common\tests.unit Tests (11) ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
✔ TelegramIncomeHashValidatorTest: Validate success (0.11s)
✔ TelegramIncomeHashValidatorTest: Validate error data | #0 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate error data | #1 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate error data | #2 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate error data | #3 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate error data | #4 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate date error | #0 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate date error | #1 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate date error | #2 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate date error | #3 (0.00s)
✔ TelegramIncomeHashValidatorTest: Validate date error | #4 (0.00s)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Time: 00:00.435, Memory: 14.00 MB

OK (11 tests, 11 assertions)
root@9c8d3bcab292:/var/www/common# 
```

Теперь можно двигаться дальше : )

Перед тем, как передать параметры, полученные от телеграмма, в валидатор, мне, для приличия, нужно бы их проверить на корректность типов. Создал форму:

```php
<?php

declare(strict_types=1);

namespace frontend\modules\Teacher\forms;

use yii\base\Model;

final class TelegramSocialAuthForm extends Model
{
    public mixed $id;
    public mixed $first_name;
    public mixed $username;
    public mixed $photo_url;
    public mixed $auth_date;
    public mixed $hash;

    public function rules(): array
    {
        return [
            [['id', 'auth_date', 'hash'], 'required'],
            [['id', 'auth_date'], 'integer'],
            [['first_name', 'username', 'hash'], 'string'],
            [['photo_url'], 'url'],
        ];
    }
}
```
Потом я делаю маппер из формы в dto валидатора:
```php
<?php

declare(strict_types=1);

namespace frontend\modules\Teacher\mappers;

use common\components\SocialAuth\TelegramIncomeHashValidatorDto;
use frontend\modules\Teacher\forms\TelegramSocialAuthForm;

final class TelegramSocialAuthFormToValidatorDtoMapper
{
    public function map(TelegramSocialAuthForm $telegramSocialAuthForm): TelegramIncomeHashValidatorDto
    {
        return new TelegramIncomeHashValidatorDto(
            id: (int) $telegramSocialAuthForm->id,
            firstName: $telegramSocialAuthForm->first_name,
            userName: $telegramSocialAuthForm->username,
            photoUrl: $telegramSocialAuthForm->photo_url,
            authDate: (int) $telegramSocialAuthForm->auth_date,
            incomeHash: $telegramSocialAuthForm->hash,
        );
    }
}
```

Теперь делаю action для авторизации через телегу:

```php
<?php

declare(strict_types=1);

namespace frontend\modules\Teacher\controllers;

// ...

class AuthController extends Controller
{
    // ...

    public function actionTelegram()
    {
        $telegramSocialAuthForm = new TelegramSocialAuthForm();
        $telegramSocialAuthForm->load($this->request->get(), '');
        if (!$telegramSocialAuthForm->validate()) {
            $this->error("Income params from telegram were incorrect...");
            // на случай, если вдруг поменяется тип данных со стороны телеги, я хотя бы узнаю об этом из логов
            $this->logger->error(implode(', ', $telegramSocialAuthForm->getFirstErrors()));
            return $this->redirect($this->authUrl->index());
        }

        $telegramIncomeHashValidationDto = $this->telegramSocialAuthFormToValidatorDtoMapper->map($telegramSocialAuthForm);
        $hashValidationErrors = $this->telegramIncomeHashValidator->validate($telegramIncomeHashValidationDto);
        if (!empty($hashValidationErrors)) {
            $this->error("Telegram's hash isn't valid. Something wrong...");
            // на случай, если вдруг поменяется валидация хеша со стороны телеги, я хотя бы узнаю об этом из логов
            $this->logger->error(implode(', ', $hashValidationErrors));
            return $this->redirect($this->authUrl->index());
        }
        
        // дальше идет моя внутренняя кухня по сохранению данных и авторизации
        $user = $this->login('telegram', (string) $telegramIncomeHashValidationDto->id);
        return $this->redirect($this->profileUrl->index($user->relatedTeacher->id));
    }
    
    // ...
}
```

Авторизация завелась. 
Получил данные, загрузил в форму, провалидировал, передал в валидатор хеша, снова валидация и потом уже произвожу авторизацию.
Валидатор я мог бы обернуть в yii-ый валидатор и использовать его через форму, но это можно и потом сделать :)

 


