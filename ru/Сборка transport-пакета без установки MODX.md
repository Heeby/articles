# Сборка transport-пакета без установки MODX

Писать свои пакеты для MODX не просто для новичка, да и опытному разработчику иногда не сладко приходится. Но новичок пугается, а опытный разбирается :).

Эта заметка рассказывает о том, как можно написать и собрать пакет компонента для MODX без установки и настройки самого MODX. Уровень выше среднего, так что возможно придется поломать голову в отдельных случаях, но оно того стоит.

За подробностями прошу под кат.

Когда-то, когда MODX Revolution только только появился, был еще в ранней beta-версии, разработчики еще не знали, как с ним работать и как писать для него плагины. Ну кроме команды, которая корпела над CMS. И команда, надо сказать, отчасти преуспела и предусмотрела в самой системе возможность удобно собирать пакеты, которые потом можно установить через репозиторий, что выглядит логичным. Но с тех пор прошло много лет и требования к пакетам и их сборке немного поменялись.

## Копипаста — зло, хотя и не всегда

Несколько последние месяцев мне не давала покоя мысль, почему, чтобы собрать пакет для MODX, нужно обязательно устанавливать его, создавать базу данных, создавать админа и т.д. Столько лишних действий. Нет, ничего страшного в этом нет, если один раз настроил и потом пользуешься. Многие так и делают. Но как быть, когда хочется сборку поручить скрипту, а самому сходить выпить кофейку?

Так получилось, что создатели MODX привыкли работать с самим MODX и добавили прямо в ядро классы, обеспечивающие сборку пакетов. Они же написали первые компоненты, первые build-скрипты, которые потом использовались как примеры другими разработчиками, которые просто копипастили решение, не всегда особо вникая в суть происходящего. И я так делал.

Но задача — автоматизировать сборку пакета, желательно на сервере, обязательно с минимальным набором требуемого ПО, с минимальными затратами ресурсов и следовательно с большей скоростью. Задача была поставлена и после исследования исходников, тормошения Джейсона в чате решение нашлось.

## И какое же?

Первое, что я выяснил, это что код, отвечающий за сборку пакета непосредственно, лежит в библиотеке xPDO, а в MODX только классы-обертки, предоставляющие более удобное API и с которыми несколько проще работать, но только если MODX установлен. Следовательно, наверное как-то можно использовать только xPDO, но в коде конструктор объекта xPDO требует указывать данные для коннекта к БД.

```php
public function __construct($dsn, $username= '', $password= '', 
                            $options= array(), $driverOptions= null) {}
```

После расспросов Джейсона стало понятно, что хоть параметры и нужно задавать, реальный же физический коннект к базе данных происходит ровно в тот момент, когда это необходимо. Lazy load во всей красе. Вторая проблема была решена.

Третьей проблемой стал вопрос подключения xPDO к проекту. Сразу на ум пришел Composer, но 2.x версия, на которой работает нынешний MODX, не поддерживает Composer, а ветка 3.x использует неймспейсы и имена классов записываются не так, как в 2.x, что приводит к конфликтам и ошибкам. В общем, несовместимы. Тогда пришлось использовать средства git и подключить xPDO как субмодуль.

> ### Как использовать субмодули

> Для начала стоит почитать [документацию](https://git-scm.com/book/en/v2/Git-Tools-Submodules) по ним.

> Затем, если это новый проект, нужно субмодуль добавить:

> ```bash
> $ git submodule add https://github.com/username/reponame
> ```

> Эта команда склонирует и установит субмодуль в ваш проект. Затем вам нужно будет добавить папку с субмодулем в свой репозиторий командой `git add`. Она не будет добавлять всю папку с субмодулем, а добавит в git только ссылку на последний коммит из субмодуля.

> Чтобы другой разработчик мог склонировать проект со всеми зависимостями, нужно создать конфиг `.gitmodules` для субмодулей. В проекте Slackify он такой:

> ```
> [submodule "_build/xpdo"]
> 	path = _build/xpdo
> 	url = https://github.com/modxcms/xpdo.git
> 	branch = 2.x
> ```

> После этого при клонировании достаточно указать флаг `recursive` и git скачает все зависимые репозитории.

В итоге, у нас есть xPDO, xPDO можно использовать без подключения к БД, если в ней нет необходимости, xPDO можно подключить к коду компонента как внешнюю зависимость (git submodule). Теперь реализация build-скрипта.

## Давайте разбираться

Я опишу [build-скрипт](https://github.com/Alroniks/modx-slackify/blob/master/_build/build.transport.php) недавно выложенного мной дополнения [Slackify](https://github.com/Alroniks/modx-slackify). Этот компонент бесплатный и выложен в открытом доступе на GitHub, что облегчит самостоятельное изучение.

### Подключаем xPDO

Опустим задание констант с именем пакета и другие необходимые вызовы и подключим xPDO.

```php
require_once 'xpdo/xpdo/xpdo.class.php';
require_once 'xpdo/xpdo/transport/xpdotransport.class.php';
$xpdo = xPDO::getInstance('db', [
    xPDO::OPT_CACHE_PATH => __DIR__ . '/../cache/',
    xPDO::OPT_HYDRATE_FIELDS => true,
    xPDO::OPT_HYDRATE_RELATED_OBJECTS => true,
    xPDO::OPT_HYDRATE_ADHOC_FIELDS => true,
    xPDO::OPT_CONNECTIONS => [
        [
            'dsn' => 'mysql:host=localhost;dbname=xpdotest;charset=utf8',
            'username' => 'test',
            'password' => 'test',
            'options' => [xPDO::OPT_CONN_MUTABLE => true],
            'driverOptions' => [],
        ]
    ]
]);
```

Субмодуль xPDO я добавил в папку `_build`, которая нужна нам только на этапе разработки и сборки пакета и которая не попадет в основной архив компонента. Вторая копия xPDO на сайте с живым MODX нам не нужна.

В настройках подключения xPDO я задал в `dsn` имя БД, но оно не играет никакой роли. Важно, чтобы папка `cache` внутри xPDO была доступна для записи. На этом все, xPDO проинициализирован.

### Делаем хитрый хак с классами

Когда при создании пакета используется установленный MODX, все просто, мы берем и создаем объект нужного нам класса. MODX на самом деле находит нужный класс, находит для этого класса необходимую реализацию (класс c постфиксом `_mysql`), которая зависит от базы данных и после этого создает нужный объект (из-за этой особенности у вас при сборке пакета могут появится ошибки, что класс *_mysql не найден, это не страшно). Однако у нас нет ни базы, ни реализации. Нужно как-то подменить нужный класс, что мы и делаем.

```php
class modNamespace extends xPDOObject {}
class modSystemSetting extends xPDOObject {}
```

Мы создаем класс-пустышку (заглушку), который нужен для создания нужного объекта. Это не пришлось бы делать, если бы xPDO не проверял особым образом, какому классу принадлежит объект. Но он проверяет.

Но есть особые случаи, когда нужно сделать чуть больше, чем просто определить класс. Это случаи зависимостей между классами. Например нам в категорию нужно добавить плагин. В коде просто `$category->addOne($plugin);`, но в нашем случае это не сработает.

Если вы хоть раз смотрели в [схему БД MODX](https://github.com/modxcms/revolution/blob/2.x/core/model/schema/modx.mysql.schema.xml), то наверняка видели такие элементы как aggregate и composite. Про них написано в [документации](https://rtfm.modx.com/xpdo/2.x/getting-started/creating-a-model-with-xpdo/defining-a-schema/more-examples-of-xpdo-xml-schema-files), но если по простому, то они описывают взаимосвязи между классами.

В нашем случае в категории может быть несколько плагинов, за что в классе `modCategory` отвечает элемент aggregate. Следовательно, так как класс у нас без конкретной реализации, нам эту связь нужно указать руками. Проще это сделать переопределив метод `getFKDefinition`:

```php
class modCategory extends xPDOObject {
    public function getFKDefinition($alias)
    {
        $aggregates = [
            'Plugins' => [
                'class' => 'modPlugin',
                'local' => 'id',
                'foreign' => 'category',
                'cardinality' => 'many',
                'owner' => 'local',
            ]
        ];
        return isset($aggregates[$alias]) 
               ? $aggregates[$alias] 
               : [];
    }
}
```

У нас в компоненте используются только плагины, поэтому добавляем связи только для них. После этого метод `addMany` у класса `modCategory` сможет без особых проблем добавить нужные плагины в категорию, а затем и в пакет.
  
### Создаем пакет

```php
$package = new xPDOTransport($xpdo, $signature, $directory);
```

Как видим, все очень и очень просто. Вот тут нам понадобилось передать параметром `$xpdo`, который мы проинициализировали в самом начале. Если бы не этот момент, проблемы 2 и не было бы.
`$signature` — имя пакета, включая версию, `$directory` — место, куда будет заботливо положен пакет. Откуда берутся эти переменные посмотрите сами в исходниках.

### Создаем пространство имен и добавляем его в пакет

Пространство имен нам нужно для того, что бы к нему привязать лексиконы и системные настройки. В нашем случае только для этого, другие пока не рассматриваем.

```php
$namespace = new modNamespace($xpdo);
$namespace->fromArray([
    'id' => PKG_NAME_LOWER,
    'name' => PKG_NAME_LOWER,
    'path' => '{core_path}components/' . PKG_NAME_LOWER . '/',
]);
$package->put($namespace, [
    xPDOTransport::UNIQUE_KEY => 'name',
    xPDOTransport::PRESERVE_KEYS => true,
    xPDOTransport::UPDATE_OBJECT => true,
    xPDOTransport::RESOLVE_FILES => true,
    xPDOTransport::RESOLVE_PHP => true,
    xPDOTransport::NATIVE_KEY => PKG_NAME_LOWER,
    'namespace' => PKG_NAME_LOWER,
    'package' => 'modx',
    'resolve' => null,
    'validate' => null
]);
```

Первая часть понятна любому, кто хоть раз писал код для MODX. Вторая, с добавлением в пакет, чуть посложнее. Метод `put` принимает 2 параметра: сам объект и массив параметров, описывающих этот объект и его возможное поведение в момент установки пакета. Например `xPDOTransport::UNIQUE_KEY => 'name'` говорит о том, что для пространства имен в качестве уникального ключа в БД будет использоваться поле `name` с название самого пространства имен в качестве значения. Подробнее о параметрах можно почитать в [документации по xPDO](https://rtfm.modx.com/xpdo/2.x/), а лучше изучив исходный код.

Точно так же можно добавлять и другие объекты, например системные настройки.

```php
$package->put($setting, [
    xPDOTransport::UNIQUE_KEY => 'key',
    xPDOTransport::PRESERVE_KEYS => true,
    xPDOTransport::UPDATE_OBJECT => true,
    'class' => 'modSystemSetting',
    'resolve' => null,
    'validate' => null,
    'package' => 'modx',
]);
```

### Создаем категорию

С добавление категории у меня случился самый большой затык, когда я во всем это разбирался. Элементы, положенные в категорию, в модели xPDO должны как быть принадлежать этой категории, т.е. быть вложенными в нее, а уже потом сама категория должна быть вложена в пакет. И при этом нужно учитывать взаимосвязи между классами, которые я уже описывал выше. Чтобы это понять, осознать и правильно применить, ушло довольно много времени.

```php
$package->put($category, [
    xPDOTransport::UNIQUE_KEY => 'category',
    xPDOTransport::PRESERVE_KEYS => false,
    xPDOTransport::UPDATE_OBJECT => true,
    xPDOTransport::ABORT_INSTALL_ON_VEHICLE_FAIL => true,
    xPDOTransport::RELATED_OBJECTS => true,
    xPDOTransport::RELATED_OBJECT_ATTRIBUTES => [
        'Plugins' => [
            xPDOTransport::UNIQUE_KEY => 'name',
            xPDOTransport::PRESERVE_KEYS => false,
            xPDOTransport::UPDATE_OBJECT => false,
            xPDOTransport::RELATED_OBJECTS => true
        ],
        'PluginEvents' => [
            xPDOTransport::UNIQUE_KEY => ['pluginid', 'event'],
            xPDOTransport::PRESERVE_KEYS => true,
            xPDOTransport::UPDATE_OBJECT => false,
            xPDOTransport::RELATED_OBJECTS => true
        ]
    ],
    xPDOTransport::NATIVE_KEY => true,
    'package' => 'modx',
    'validate' => $validators,
    'resolve' => $resolvers
]);
```

Выглядит монструозно, но и не такое видали. Важный параметр `xPDOTransport::RELATED_OBJECTS => true`, который говорит о том, что в категории есть вложенные элементы, которые так же нужно упаковать и затем установить.

Так как большинство модулей содержит в себе различные элементы (чанки, сниппеты, плагины), то категория с элементами самый важный кусок транспортного пакета. Поэтому именно здесь заданы валидаторы и резолверы, которые выполняются в процессе установки пакета.

> Валидаторы выполняются перед установкой, резолверы — после.

Чуть не забыл, перед упаковкой категории в нее же нужно добавить наши элементы. Вот так:

```php
$plugins = include $sources['data'] . 'transport.plugins.php';
if (is_array($plugins)) {
    $category->addMany($plugins, 'Plugins');
}
```

### Добавляем другие данные в пакет

В пакет нужно добавить еще файл с лицензией, файл с логом изменений и файл с описанием компонента. Если нужно, то можно добавить еще специальный скрипт через атрибут «setup-options», который покажет окно перед установкой пакета. Это когда вместо «Установить» кнопка «Опции установки». И с версии MODX 2.4 появилась возможность указывать зависимости между пакетами с помощью атрибута «requires», причем в нем так же можно указать версию php и modx.

```php
$package->setAttribute('changelog', file_get_contents($sources['docs'] . 'changelog.txt'));
$package->setAttribute('license', file_get_contents($sources['docs'] . 'license.txt'));
$package->setAttribute('readme', file_get_contents($sources['docs'] . 'readme.txt'));
$package->setAttribute('requires', ['php' => '>=5.4']);
$package->setAttribute('setup-options', ['source' => $sources['build'] . 'setup.options.php']);
```

### Пакуем

```php
if ($package->pack()) {
    $xpdo->log(xPDO::LOG_LEVEL_INFO, "Package built");
}
```

Всё, забираем готовый пакет в `_packages`, ну или оттуда, куда вы настроили сборку.

## Что в итоге?

Результат превзошел мои ожидания, так как такой подход хоть и накладывает некоторые ограничения и местами добавляет некоторые неудобства, но выигрывает по возможностям применения.

Для сборки пакета достаточно выполнить 2 команды:

```bash
git clone --recursive git@github.com:Alroniks/modx-slackify.git
cd modx-slackify/_build && php build.transport.php
```

Первая — это клонирование репозитория и его субмодулей. Важный параметр `--recursive`, благодаря ему git скачает и установит помимо самого кода компонента все зависимости, описанные в виде субмодулей. 

Второе — сборка пакета непосредственно. После этого можно забирать готовый `package-1.0.0-pl.transport.zip` из папки `_packages` и загружать его, например в репозиторий.

Перспективы открываются широкие. Например, можно настроить хук в GitHub, который после коммита в ветку будет запускать на вашем сервере скрипт, который  соберет пакет и положит его во все сайты, которые у вас есть. Либо загрузит новую версию в какой-нибудь репозиторий, а вы в это время сделаете кофе себе, как я уже говорил в начале. Или можно придумать и написать тесты к модулю и запускать прогон тестов и сборку через Jenkins или Travis. Да кучу сценариев можно придумать. С таким подходом делать это теперь намного проще.

Задавайте вопросы, постараюсь ответить.

P.S. Не проходите мимо, [поставьте Slackify звезду на GitHub](https://github.com/Alroniks/modx-slackify), пожалуйста.
