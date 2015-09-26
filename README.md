[![Latest Stable Version](https://poser.pugx.org/arrilot/bitrix-models/v/stable.svg)](https://packagist.org/packages/arrilot/bitrix-models/)
[![Total Downloads](https://img.shields.io/packagist/dt/arrilot/bitrix-models.svg?style=flat)](https://packagist.org/packages/Arrilot/bitrix-models)
[![Build Status](https://img.shields.io/travis/arrilot/bitrix-models/master.svg?style=flat)](https://travis-ci.org/arrilot/bitrix-models)
[![Scrutinizer Quality Score](https://scrutinizer-ci.com/g/arrilot/bitrix-models/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/arrilot/bitrix-models/)

#Bitrix models (in development)

*Данный пакет представляет собой надстройку над традиционным API Битрикса для работы с элементами инфоблоков и пользователями. Достигается это при помощи создания моделей.*

## Установка

1)```composer require arrilot/bitrix-models```

2) подключаем composer к Битриксу.

Теперь можно создавать свои модели, наследуя их либо от 
```php
Arrilot\BitrixModels\Models\ElementModel
``` 
либо от
```php
Arrilot\BitrixModels\Models\UserModel
```

## Использование

Везде будем рассматривать модель для элемента инфоблока. Для пользователей отличия минимальны.

Создадим, модель для инфоблока Товары.
```php
<?php

use Arrilot\BitrixModels\Models\ElementModel;

class Product extends ElementModel
{
    /**
     * Corresponding iblock id.
     *
     * @return int
     */
    public static function iblockId()
    {
        return PRODUCT_IB_ID;
    }
}
```

Для работы модели необходимо лишь реализовать `iblockId()` возвращающий ID информационного блока.
Для юзеров не требуется и этого.

Рассмотрим примеры работы API моделей.

1) Получение элемента инфоблока из базы.
```php
$product = Product::getById($productId);
// $product  - объект модели Product (как в примере 1), однако он реализует ArrayAccess
// и поэтому с ним во многом можно работать как с массивом, полученным из битриксового getById();
if ($product['CODE'] === 'test') {
    $product->deactivate();
}

// элемент выбирается из базы вместе со свойствами.
echo $product['PROPERTY_VALUES']['GUID'];
```

2) Инстанцирование модели без запросов к базе.
Зачастую нет необходимости в получении информации из БД, достаточно лишь ID объекта.
В этом случае можно просто инстанцировать объект модели.
```php

$product = new Product($productId);
//теперь есть возможно работать с моделью, допустим
$product->deactivate();

//объект для текущего пользователя можно получить так:
$user = User::current();
```

3) Если поля нужны, их потом дополучить
```php
$product = new Product($productId);
// метод `get` обращается к базе, только если информация еще не была получена.
$arProduct = $product->get(); // и то и то

// последующие методы принудительно перегружают информацию из 
// базы даже если она есть
$arProps = $product->refreshProps(); // только свойства
$arFields = $product->refreshFields(); // только поля
$arProduct = $product->refresh(); // и то и то
```

4) Преобразование модели в чистый массив.
```php
$product = Product::getById($productId);
$arProduct = $product->toArray();
$json = $product->toJson();
```

5) Обновление элемента инфоблока.
```php
$product = Product::getById($productId);

// вариант 1
$product['NAME'] = 'Новое имя продукта';
$product->save();

// вариант 2
$product['NAME'] = 'Новое имя продукта';
$product->update(['NAME' => 'Новое имя продукта']);
```

6) Добавление элемента инфоблока.
```php
// $fields - массив аналогичный передаваемому в CIblockElement::Add()
$product = Product::create($fields);
```

7) Подсчёт количества элементов инфоблока с учётом фильтра
```php
$count = Product::count(['ACTIVE'=>'Y']);
```

8) Получения списка элементов
```php
$products = Product::getList([
    'filter' => ['ACTIVE'=>'N'],
    'select' => ['ID', 'NAME', 'CODE'],
    'sort => ['NAME' => 'ASC'],
    'keyBy' => 'ID'
]);
foreach ($products as $id => $product) {
    if ($product['CODE'] === 'test') {
        $product->deactivate();
    }
}
```

`getList` возвращает инстанс `Illuminate\Support\Collection` с котором как видно можно во многом обращатся как с массивом
```php
$products->toArray()
```
преобразует коллекцию в обычный массив. При этом для каждого элемента коллекции будет тоже вызван `->toArray()`

9) Получение первого элемента удовлетворяющего параметрам запроса
```php
$product = Product::first([
    'filter' => ['CODE'=>'30-2515'],
    'select' => ['ID', 'NAME', 'CODE']
]);
```

### "Fluent API" для запросов.

Для `getById`, `getList`, `first` и `count` можно также использовать fluent API.

10) Пример 7 можно также реализовать и так:
```php
$count = Product::query()->filter(['ACTIVE'=>'Y'])->count();
```

11) Получения списка элементов через query
```php
$products = Product::query()
                    ->filter($filter)
                    ->navigation(['nTopCount'=>100])
                    ->select('ID','NAME')
                    ->getList();
```

Опущенные модификаторы будут заполнены дефолтными значениями.
В значения модификаторов можно передавать прямо такие же значения как и в соответсвующие параметры CIblockElement::getList
Некоторые дополнительные моменты:

1. В фильтр автоматически подставляется ID инфоблока.

2. Как видно из примера `select()` поддерживает не только массив но и произвольное число аргументов
`select('ID', 'NAME')`  равнозначно `select(['ID', 'NAME'])` 

3. `select()` поддерживает два дополнительных значения - `'FIELDS'` (выбрать все поля), `'PROPS'` (выбрать все свойства).
Для пользователей также можно указать `'GROUPS'` (добавить группы пользователя в выборку)
Значение по-умолчанию - `['FIELDS', 'PROPS']`

4.  Есть возможность указать `keyBy()` - именно указанное тут поле станет ключом в списке-результате.
Значение по-умолчанию - `false`, используется обычный автоинкриментирующийся integer.

5. Для ограничения выборки добавлены алиасы `limit($value)` (соответсвует `nPageSize`) и `page($num)` (соответсвует `iNumPage`)

5. В некоторых местах API более дружелюбный чем в самом Битриксе. Допустим в фильтре по пользователям не обязательно использовать
`'GROUP_IDS'`. При передаче `'GROUP_ID'` (именно такой ключ требует Битрикс допустим при создании пользователя) или `'GROUPS'` 
результат будет аналогичен.



### Query Scopes

Построитель запросов можно расширять добавляя "query scopes" в модель.
Для этого необходимо создать публичный метод начинаюзищися со `scope`.

Пример "query scope"-a уже присутсвующего в пакете.
```php

    /**
     * Scope to get only active items.
     *
     * @param BaseQuery $query
     *
     * @return BaseQuery
     */
    public function scopeActive($query)
    {
        $query->filter['ACTIVE'] = 'Y';
    
        return $query;
    }

...

$products = Product::query()
                    ->filter(['SECTION_ID' => $secId])
                    ->active()
                    ->navigation(['nTopCount'=>100])
                    ->getList();
```

В "query scopes" можно также передавать дополнительные параметры.
```php

    /**
     * @param ElementQuery $query
     * @param string|array $category
     *
     * @return ElementQuery
     */
    public function scopeFromCategory($query, $category)
    {
        $query->filter['SECTION_CODE'] = $category;

        return $query;
    }

...
$users = User::query()->sort(["ID" => "ASC"])->filter(['NAME'=>'John'])->fromGroup(7)->getList();
```

#### Остановка действия

Иногда требуется остановить выборку из базы из query scope. 
Для этого достаточно вернуть false.
Пример:
```php
    public function scopeFromCategory($query, $category)
    {
        if (!$category) {
            return false;
        }
        
        $query->filter['SECTION_CODE'] = $category;

        return $query;
    }
...
В результате запрос в базу не будет сделан - getList вернёт пустую коллекцию, getById - false, а count - 0.
Того же самого эффекта можно добиться вызвав вручную метод `stopQuery()` 
```php 
$users = User::query()->stopQuery()->getList();
```

### Accessors

Временами возникает потребность как то модифицировать данные между выборкой их из базы и получением их из модели.
Для этого используются акссессоры.
Также как и для "query scopes", для добавления аксессора необходимо добавить метод в соответсвующую модель.

Правило именования метода - $methodName = "get".camelCase($field)."field".
Пример акссессора который принудительно делает первую букву имени заглавной:
```php

    public function getNameField($value)
    {
        return ucfirst($value);  
    }
    
    // теперь в $product['NAME'] будет модифицированное значение
    
```

Аксессоры можно создавать также и для несуществущих полей, например:
```php
    public function getFullNameField()
    {
        return $this['NAME']." ".$this['LAST_NAME'];
    }
    
    ...
    
    echo $user['NAME']; // John
    echo $user['LAST_NAME']; // Doe
    echo $user['FULL_NAME']; // John Doe
```

Для того чтобы такие аксессоры отображались в toArray() и toJson() их указать в поле $appends модели.
```php
    protected $appends = ['FULL_NAME'];
```

### События моделей (Model Events)

События позволяют вклиниваться в разлчичные точки жизненного цикла модели и выполнять в них произвольный код.
Например - автоматически проставлять символьный код при создании элемента.
Обработчик события задается переопределением соответсвующего метода в классе-модели.

```php
class News extends ElementModel
{
    /**
     * Hook into before item create or update.
     *
     * @return mixed
     */
    protected function onBeforeSave()
    {
        $this['CODE'] = CUtil::translit($this['NAME'], "ru");
    }

    /**
     * Hook into after item create or update.
     *
     * @param bool $result
     *
     * @return void
     */
    protected function onAfterSave($result)
    {
        //
    }
}
```

Сигнатуры обработчиков других событий совпадают с приведенными выше.

Список доступных эвентов:

1. `onBeforeCreate` - перед добавлением записи
2. `onAfterCreate` - после добавления записи
3. `onBeforeUpdate` - перед обновлением записи
4. `onAfterUpdate` - после обновления записи
5. `onBeforeSave` - перед добавлением или обновлением записи
6. `onAfterSave` - после добавления или обновления записи
7. `onBeforeDelete` - перед удалением записи
8. `onAfterDelete` - после удаления записи

Если сделать `return false;` из обработчика `onBefore...` то последующее действие будет отменено.