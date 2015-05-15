enb-freeze [![Build Status](https://travis-ci.org/f-o-r/enb-freeze.svg?branch=master)](https://travis-ci.org/f-o-r/enb-freeze)
==============

Технологии для фриза через enb.

### Список технологий

  - freeze-base `./lib/base_tech.js` Базовая технология от которой нужно наследовать свои технологии
  - freeze-from-xslt `./techs/freeze-from-xslt.js` Технология для фриза xslt и статики внутри него

### Примеры использования

#### freeze-from-xslt
```javascript
nodeConfig.addTechs([
    [
        require('enb-freeze/techs/freeze-from-xslt'),  {
            source: '?.xsl',
            target: '_?.xsl',
            freezeRegex: /"\{fn:include-static\('([^']+\.(css|js|png|jp?g|gif))'\)\}"/,
            freezeDir: function(suffix) {
                return path.join(this.node.getRootDir(), 'freeze');
            }
        }
    ]
]);
```

### Параметры

Любуй технологию, наследованную от `freeze-base`(`./lib/base_tech.js`) можно параметризовать следующими ключами:

Обязательные:
  - `target`
  - `source`
  - `freezeDir` [Фризовая папка][freeze-dir-opt]

Имеющие значение по умолчанию:
  - `hash`(`sha1`) [Hash algo][hash-opt], поддерживаемый модулем `crypto`
  - `digest`(`hex`) [Digest algo][digest-opt], определенный в `./lib/digest.js`(hex, alphanum)
  - `freezePathPostprocess`(`null`) [Функция][freeze-path-postprocess-opt] постпроцессинга
  - `waitForTargets`(`[]`) [Список целей][wait-for-targets-opt] enb для ожидания
  - `waitForNodeTargets`(`{}`) Словарь, [сопсотовляющий цели узлам][wait-node-targets-opt], которые следует подождать
  - `grammarBlockComments`(`[]`) Пара(`['/*', '*/']`), содержащая символы начала и конца комментария соответственно

#### freezeDir
Служит для определения директории, в которую будут записаны фризовые файлы.

Сигнатура:
`𝑓(suffix :: String) -> absFreezeDirPath :: JustString`

Представляет из себя функцию, в которую приходит suffix обрабатываемой технологии. Должна всегда возвращать строку с абсолютным путем до фризовой папки.

#### hash
Сигнатура:
`hash :: String`

Будет проброшен в `createHash`, лучше обратиться к [официальной документации][create-hash-algo-link] про поддерживаемые алгоритмы.

#### digest

Может принимать значения `hex` и `alphanum`. Функции, отвечающие за каждый из типов подписей определены в `./lib/digest.js`.

`hex` digest просто вызывает `hash.digest('hex')` метод из модуля `crypto`.

`alphanum` это кастомная функция, использующая алфавит из 26 букв и 10 чисел для отображения подписи(обеспечивает короткую подпись). Она не тестировалась на наличие коллизий, так что на данный момент её стоит использовать на свой страх и риск.

#### freezePathPostprocess
Служит для постпроцессинга фризовых путей внутри файла.

Сигнатура:
`𝑓(parent :: String, carrier :: String, freezePath :: String) -> freezePath :: MaybeString`

Аргументы:

  - `parent` Абсолютный путь к родителю обрабатываемого файла
  - `carrier` Абсолютный путь к обрабатываемому файлу
  - `freezePath` Абсолютный путь к фризовому файлу

Если функция возвращает `null`, то применяется дефолтный постпроцессор(о нём ниже в оговорках).

Для примера, есть такой файл с псевдокодом, полученный после фриза всех include'ов внутри:
```
include /abs/path/to/freeze/dir/b.pseudo
```

Для простоты предположим, что `b.pseudo` это фризовое имя файла.
Проблема здесь в том, что мы имеем `include` по абсолютному пути, чаще всего мы хотим относительный путь.
После обработки дефолтным вариантом этой функции файл будет выглядеть так:
```
include b.pseudo
```

Есть несколько оговорок про эту функцию:
  1. В дефолтной реализации есть условие, от которого зависит выбор пути, относительно которого будет произведён `path.relative` для фризового файла. Если `parent === carrier`, то путь к фризовому файлу бедт рассчитан относительно папки в котрой лежит `carrier`, иначе относительно фризовой директории(см. `freezeDir`)
  2. В вашей реализации функции может быть логика, которая постпроцессит пути с разными суффиксами по разному

Пример. Допустим у нас есть некий CDN, куда мы выкладываем всё что фризится. Мы хотим заменить все пути в фризовых файла на URI до файла в CDN. Тогда функция может выглядеть так:
```javascript
var path = require('path');
function(parent, carrier, freezePath) {
    if(this.getSuffix(freezePath) === 'pseudo') {
        return '//cdn.example.com/my-project/' + path.basename(freezePath)
    }
    return null;
}
```

Обработаем с помощью этой функции файл:
```
include /abs/path/to/freeze/dir/b.pseudo
include /abs/path/to/freeze/dir/c.real
```

И получим на выходе:
```
include //cdn.example.com/my-project/b.pseudo
include c.real
```

Путь к `b.pseudo` был обработан нашей функцией, тогда как путь к `c.real` был обработан реализацией по умолчанию.

##### waitForTargets
Ждёт компиляции указанных целей.
Сигнатура:
`waitForTargets :: [String]`

Пример:
`waitForTargets: ['?.css', '?.js']`

##### waitForNodeTargets
Ждёт компиляции указанных целей в указанных узлах.
Сигнатура:
`waitForNodeTargets :: {String: [String]}`

Пример:
`waitForNodeTargets: {common: ['?.css', '?.js']}`


[freeze-dir-opt]: #freezeDir
[hash-opt]: #hash
[digest-opt]: #digest
[create-hash-algo-link]: https://nodejs.org/api/crypto.html#crypto_crypto_createhash_algorithm
[freeze-path-postprocess-opt]: #freezePathPostprocess
[wait-for-targets-opt]: #waitForTargets
[wait-for-node-targets-opt]: #waitForNodeTargets
