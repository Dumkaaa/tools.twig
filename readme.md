# maximaster/tools.twig

Данная библиотека позволяет использовать twig шаблоны в компонентах 1С Битрикс. Обрабатываются файлы, имеющие расширение .twig. Если создать в директории шаблона компонента файл template.twig, то именно он будет использоваться при генерации шаблона.
К сожалению, ядро битрикса так устроено, что при наличии двух файлов .php и .twig, будет использовать только .php, поэтому для использования twig шаблона нужно позаботиться о том, чтобы php шаблон отсутствовал.

## Наследование шаблонов

Чтобы воспользоваться наследованием шаблонов, можно писать в extends абсолютный путь до шаблона. 
Для упрощения наследования в библиотеке присутсвует загрузчик, который позволяет по короткому имени шаблона получить к нему доступ. Синтаксис простой:

vendor:component_name[:template[:specific_template_file]]

Здесь
* **vendor** - это пространство имен разработчика, например bitrix или maximaster
* **component_name** - имя компонента, шаблон которого наследуется
* **template** - имя шаблона, который нужно унаследовать. Необязательный, по-умолчанию .default
* **specific_template_file** - конкретный файл шаблона (без расширения). Необязательный, по-умолчанию template

Например, вы хотите унаследовать шаблон new-year компонента maximaster:product. Для этого в шаблоне twig нужно написать 

```twig
{% extends "maximaster:product:new-year" %}
```

## Управление настройками

Библиотека конфигурируется с помощью файла /bitrix/.settings.php. Нужно завести в этом файле опцию maximaster, и оперировать значением tools->twig. Ниже описаны значения опций, которые заданы библиотекой по-умолчанию:

```php

//...
'maximaster' => array(
    'value' => array(
        'tools' => array(
            'twig' => array(
				// Режим отладки выключен
				'debug' => false,

				//Кодировка соответствует кодировке продукта
				'charset' => SITE_CHARSET,

				//кеш хранится в уникальной директории. Должен быть полный абсолютный путь
				'cache' => $_SERVER['DOCUMENT_ROOT'] . '/bitrix/cache/maximaster/tools.twig',

				//Автообновление включается только в момент очистки кеша
				'auto_reload' => isset( $_GET[ 'clear_cache' ] ) && strtoupper($_GET[ 'clear_cache' ]) == 'Y',

				//Автоэскейп отключен, т.к. битрикс по-умолчанию его сам делает
				'autoescape' => false,
            )
        )
    )
)
//...

```
При выборе значений для конфигов нужно опираться на [документацию twig по настройкам Twig_Environment](http://twig.sensiolabs.org/doc/api.html#environment-options). Поддерживаются все возможные согласно этой документации опции

## Расширение 

С версии 0.5 появилась возможность добавления собственных расширений. Реализуется это с помощью обработчика события **onAfterTwigTemplateEngineInited**. Событие не привязано ни к одному из модулей, поэтому при регистрации события в качестве идентификатора модуля нужно указать пустую строку.
В событие передается объект `\Twig_Environment`, с которым можно сделать определенные манипуляции.
Пример обработчика события, который зарегистрирует свое расширение:

```php

use Bitrix\Main\EventResult;

AddEventHandler('', 'onAfterTwigTemplateEngineInited', array('onAfterTwigTemplateEngineInited', 'addTwigExtension'));

class onAfterTwigTemplateEngineCreated
{
    public static function addTwigExtension(\Twig_Environment $engine)
    {
        $engine->addExtension(new MySuperDuperExtension());
        return new EventResult(EventResult::SUCCESS, array($engine));
    }
}
```

Здесь класс MySuperDuperExtension должен быть наследником класса `\Twig_Extension` или имплементацией интерфейса `\Twig_ExtensionInterface`.

## Дополнительные функции

С версии 0.5 в библиотеке появились дополнительные удобные функции для работы с шаблонами. На данный момент функция только одна, но их перечень постоянно будет пополняться.

###### russianPluralForm(int $num, array $words)
Функция помогает в выводе множественного числа для русских слов. К примеру, вам нужно вывести строку "21 билет". Для этого нужно воспользоваться функцией с такими параметрами в twig:

```twig
    {% set ticketsCount = 21 %}
    {{ russianPluralForm(ticketsCount, ['билетов', 'билет', 'билета']) }}
```

Порядок словоформ запомнить достаточно просто: 0 билетов, 1 билет, 2 билета. Для большинства слов такой порядок будет работать корректно.

## Работа с кешем

С версии 0.8 появилась возможность программно управлять очисткой кеша. Сделать это можно с помощью класса `\Maximaster\Tools\Twig\TwigCacheCleaner`. Данный класс можно использовать только в том случае, если кеш Twig хранится на диске. 
Класс имеет 2 метода очистки:
* `clearByName($name)` - очищает кеш шаблона по его имени 
* `clearAll()` - очищает весь кеш
Каждый их методов вернет количество удаленных файлов кеша