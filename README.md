# rest.controller
Добавление своих методов REST API
# [Добавление своих методов REST API](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=99&LESSON_ID=7985&LESSON_PATH=8771.5380.7985)
 При создании собственных приложений для коробочных редакций возникает потребность в добавлении новых методов для них в REST API.

 Порядок действий:

1. [Регистрируется](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&LESSON_ID=3493) обработчик события **OnRestServiceBuildDescription** модуля **rest**. Обработчик возвращает массив вот такой структуры::

   ```
   return array(
      'имя_scope' => array(
         'имя_метода' => array(
            'callback' => обработчик_метода,
            'options' => array(),
         ),
         'имя_метода' => array(
            'callback' => обработчик_метода,
            'options' => array(),
         ),
      )
   );
   ```

    Имя **scope** - произвольное. Если регистрируется собственный scope, то он будет доступен в списке разрешений для локальных приложений. Для добавления методов в существующий scope, укажите имя метода. Если добавляются методы, доступные всем приложениям, то вместо имени указывается константа `\CRestUtil::GLOBAL_SCOPE`.

    Имя метода также произвольное, допускается использование как верхнего, так и нижнего регистра. Но рекомендуется придерживаться нижнего регистра. Традиционное именование - **имя\_scope.имя\_сущности.действие**.

    Обработчик метода - стандартный PHP псевдо-тип [callback](http://www.php.net/types.callable). Анонимные функции пока не поддерживаются.

2. Функция-обработчик получит на вход 3 параметра:
   * ассоциативный массив данных вызова без авторизационных параметров;
   * параметр для возврата постранички;
   * экземпляр класса *\\CRestServer*, из которого получаются некоторые полезные данные.

3. Функция-обработчик может:
   * либо вернуть ответ (массив или скалярное значение), которые будут приведены к json- или xml-виду,
   * либо бросить исключение, которое будет перехвачено и отображено в виде REST-ошибки.

    Если требуется указать HTTP-статус ошибки, то воспользуйтесь классом `\Bitrix\Rest\RestException`. Если при этом ранее ядром было сгенерировано старое исключение (**$APPLICATION->ThrowException()**), то оно перезапишет ошибку в ответе.

4. К моменту вызова функции-обработчика проверки прав доступа уже выполнены, а объект **$USER** уже проинициализирован с авторизацией привязанного к токену пользователя.

    Пример:

   ```
   class RestTest
   {
      public static function OnRestServiceBuildDescription()
      {
         return array(
            'sigurdtest' => array(
               'sigurdtest.test' => array(
                  'callback' => array(__CLASS__, 'test'),
                  'options' => array(),
               ),
            )
         );
      }

      public static function test($query, $n, \CRestServer $server)
      {
         if($query['error'])
         {
            throw new \Bitrix\Rest\RestException(
               'Message',
               'ERROR_CODE',
               \CRestServer::STATUS_PAYMENT_REQUIRED
            );
         }

         return array('yourquery' => $query, 'myresponse' => 'My own response');
      }
   }

   AddEventHandler('rest', 'OnRestServiceBuildDescription', array('\RestTest', 'OnRestServiceBuildDescription'));
   ```

   [![Нажмите на рисунок, чтобы увеличить](https://dev.1c-bitrix.ru/upload/medialibrary/ab1/f9xwilx2dc2kt38y49cluyskb573x9o4/tiraj14_sm.jpg)](javascript:ShowImg('/upload/medialibrary/aad/sqm31gsiyjhrl61u0qccicssy1nyoxzq/tiraj14.jpg',1272,890,'%D0%94%D0%BE%D0%B1%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5 %D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F'))

   ```
   http://cp.sigurd.bx/rest/sigurdtest.test?auth=qcdfzzjtp8hvcbjl42eg93wuw5n0mvsb&test=222

   HTTP/1.1 200 OK

   Content-Type: application/json; charset=utf-8

   {
       "result": {
          "myresponse": "My own response",  
          "yourquery": {
              "test": "222"
          }
      }
   }

   http://cp.sigurd.bx/rest/sigurdtest.test.xml?auth=qcdfzzjtp8hvcbjl42eg93wuw5n0mvsb&error=1

   HTTP/1.1 402 Payment Required

   Content-Type: text/xml; charset=utf-8

   <?xml version="1.0" ?>
   <response>
      <error>ERROR_CODE</error>
      <error_description>Message</error_description>
   </response>
   ```

#### Как определить обработчик отсутствующего в описании REST-метода "на лету" ####

 Для этого используйте событие **onFindMethodDescription**, вызываемое перед отдачей ошибки `METHOD_NOT_FOUND`. Оно позволяет подставить описание метода "на лету".

#### Как работать с навигацией в своих методах реста ####

```
\Bitrix\Main\Loader::includeModule('rest');
class ApiTest extends \IRestService
{
    public static function OnRestServiceBuildDescription()
    {
        return array(
            'apitest' => array(
                'api.test.list' => array(
                    'callback' => array(__CLASS__, 'getList'),
                    'options' => array(),
                ),
            )
        );
    }

    public static function getList($query, $nav, \CRestServer $server)
    {
        $navData = static::getNavData($nav, true);

        $res = \Bitrix\Main\UserTable::getList(
            [
                'filter' => $query['filter']?:[],
                'select' => $query['select']?:['*'],
                'order' => $query['order']?:['ID' => 'ASC'],
                'limit' => $navData['limit'],
                'offset' => $navData['offset'],
                'count_total' => true,
            ]
        );

        $result = array();
        while($user = $res->fetch())
        {
            $result[] = $user;
        }

        return static::setNavData(
            $result,
            array(
                "count" => $res->getCount(),
                "offset" => $navData['offset']
            )
        );
    }
}

AddEventHandler('rest', 'OnRestServiceBuildDescription', array('\ApiTest', 'OnRestServiceBuildDescription'));
```

 На примере ORM класс работы с пользователями показывается как зарегистрировать свой метод для работы с таблицами пользователей. Если вам необходимо сделать постраничную навигацию для методов не ORM-классов, например, для *getNavData()*, то вторым параметром передайте: `false ($navData = static::getNavData($nav, false);)`, тогда вернётся навигация соответствующая методам старого ядра.

 Пример запроса:

```
/rest/api.test.list?order[ID]=ASC&filter[<ID]=1000&select[]=ID&select[]=NAME&start=200
```
