# Websoft.Zip — сборка 434

Руководство относится к WebSoft HCM Server `2022.1.3.434` и
Websoft.Zip `1.22.6.8`. Примеры выполняются в серверном агенте.

!!! danger "Архив должен быть доверенным"
    В этой DLL нет проверки имён записей при распаковке. На стенде успешно
    извлечены `../one.txt`, `../../two.txt`, `sub/../../mixed.txt` и абсолютный
    путь. Все вызовы вернули `1`, ошибки не возникли. Злоумышленник может
    записать файл за пределами каталога назначения.

## Подготовка файлов

Методы DLL принимают файловые пути, а не URL вида `x-local://`. Для теста
создайте отдельный каталог и преобразуйте URL через `UrlToFilePath()`:

```js
var baseUrl = 'x-local://trash/websoft-zip-example'
var sourceUrl = baseUrl + '/source'
var nestedUrl = sourceUrl + '/nested'
var archiveUrl = baseUrl + '/archive.zip'

ObtainDirectory(UrlToFilePath(sourceUrl), true)
ObtainDirectory(UrlToFilePath(nestedUrl), true)
PutFileData(
    UrlToFilePath(sourceUrl + '/alpha.txt'),
    Base64Decode('YWxwaGE=')
)
PutFileData(
    UrlToFilePath(nestedUrl + '/beta.txt'),
    Base64Decode('YmV0YQ==')
)
PutFileData(
    UrlToFilePath(sourceUrl + '/кириллица.txt'),
    Base64Decode('dGV4dA==')
)
```

`PutFileData()` принимает бинарные данные, поэтому содержимое файлов передаётся
через `Base64Decode()`.

## Создание архива

Получите объект, проверьте версию, создайте архив, добавьте каталог и закройте
архив:

```js
var zip = tools.get_object_assembly('Zip')
var version = zip.GetVersion()

var createResult = zip.CreateArchive(UrlToFilePath(archiveUrl))
var addResult = zip.AddDirectory(UrlToFilePath(sourceUrl))

zip.Close()
```

`CreateArchive()` удаляет существующий файл по этому пути. Проверяйте
`createResult` и `addResult`: ожидаемое успешное значение — `1`.

`AddDirectory()` рекурсивно сохраняет файлы и относительные пути. Пустые файлы
не добавляются, пустые каталоги не сохраняются.

Для одного файла или маски используйте `AddFile()` либо `AddFiles()`:

```js
var zip = tools.get_object_assembly('Zip')

zip.CreateArchive(UrlToFilePath(baseUrl + '/text-files.zip'))
zip.AddFiles(UrlToFilePath(sourceUrl + '/*.txt'))
zip.Close()
```

`AddFile()` и `AddFiles()` принимают как точное имя, так и маску.

## Чтение списка

Откройте уже закрытый архив и вызовите `ListFiles()`:

```js
var zip = tools.get_object_assembly('Zip')
var openResult = zip.OpenArchive(UrlToFilePath(archiveUrl))
var files = zip.ListFiles()

alert(tools.object_to_text(files, 'json'))
zip.Close()
```

Список содержит `alpha.txt`, `nested/beta.txt` и `кириллица.txt`.

Не вызывайте `ListFiles()` сразу после добавления в создаваемый архив. До
`Close()` центральный каталог ZIP ещё отсутствует: метод возвращает
`undefined`, а `GetError()` содержит `Central directory currently does not
exist`.

## Сжатие

Начальное значение `CompressionLevel` равно `1`.

```js
var zip = tools.get_object_assembly('Zip')
var previousLevel = zip.CompressionLevel

zip.CompressionLevel = 0
zip.CreateArchive(UrlToFilePath(baseUrl + '/stored.zip'))
zip.AddFile(UrlToFilePath(sourceUrl + '/alpha.txt'))
zip.Close()

zip.CompressionLevel = previousLevel
```

`CompressionLevel` поддерживает два режима:

- `0` — Store, без сжатия;
- любое ненулевое число — Deflate.

Значения `1` и `7` дали одинаковый результат: ненулевое значение выбирает
Deflate, но не задаёт его уровень.

Метод `SetCompressionLevel(value)` меняет то же состояние и возвращает `1` при
успехе.

## Кодировка имён

Начальное значение `CharSet` — `Unicode (UTF-8)`. Поддерживаемое .NET имя
кодировки передаётся строкой:

```js
var zip = tools.get_object_assembly('Zip')
var previousCharSet = zip.CharSet

zip.CharSet = 'windows-1251'
zip.CreateArchive(UrlToFilePath(baseUrl + '/windows-1251.zip'))
zip.AddFile(UrlToFilePath(sourceUrl + '/кириллица.txt'))
zip.Close()

zip.CharSet = previousCharSet
```

DLL успешно прочитала созданные ею имена `кириллица.txt` в кодировках
`windows-1251` и `utf-8`. Совместимость со сторонними архиваторами не
проверялась.

Изменения `CharSet` и `CompressionLevel` влияют на последующие обращения к
общему объекту библиотеки. После работы возвращайте прежние значения.

## Распаковка

Полная распаковка доверенного архива:

```js
var outputUrl = baseUrl + '/output'
ObtainDirectory(UrlToFilePath(outputUrl), true)

var zip = tools.get_object_assembly('Zip')
var openResult = zip.OpenArchive(UrlToFilePath(archiveUrl))
var extractResult = zip.Extract(UrlToFilePath(outputUrl))

zip.Close()
```

Извлечение одной записи по точному имени:

```js
var zip = tools.get_object_assembly('Zip')

zip.OpenArchive(UrlToFilePath(archiveUrl))
var result = zip.ExtractFiles(
    '',
    'nested/beta.txt',
    UrlToFilePath(outputUrl)
)
zip.Close()
```

Первый параметр `ExtractFiles()` текущая реализация не использует. Второй
параметр сравнивается с полным именем записи без учёта регистра; маски не
поддерживаются.

`ExtractFiles()` возвращает `1`, даже если запись не найдена. Подтверждайте
результат через `FileExists()` или проверку содержимого.

## Ошибки и коды возврата

```js
var zip = tools.get_object_assembly('Zip')
var result = zip.OpenArchive(
    UrlToFilePath('x-local://trash/missing.zip')
)

if (result != 1) {
    alert(zip.GetError())
}
```

Ошибки DLL этой сборки записываются в штатный журнал `xhttp_`.

`GetError()` хранит последнюю ошибку и не очищается после успешной операции.
Поэтому анализируйте вместе код возврата, ожидаемый файл и текст ошибки.

Код `1` не всегда означает выполненное действие:

- после `Close()` методы `AddFile()`, `Extract()` и `ExtractFiles()` могут
  вернуть `1`, ничего не сделав;
- `ExtractFiles()` возвращает `1`, если указанной записи нет;
- `Save()` возвращает `1` только при наличии открытого объекта архива и не
  записывает данные отдельно от `Close()`.

## `OpenArchive()` и параметр `access`

Второй параметр имеет тип .NET `System.IO.FileAccess`. Если специальный режим
не нужен, не передавайте его:

```js
zip.OpenArchive(UrlToFilePath(archiveUrl))
```

## Публичный API

| Элемент | Назначение |
| --- | --- |
| `GetVersion()` | Возвращает версию DLL |
| `GetError()` | Возвращает сохранённый текст последней ошибки |
| `CharSet` | Получает или задаёт кодировку имён |
| `CompressionLevel` | Получает или задаёт режим сжатия |
| `SetCompressionLevel(value)` | Задаёт тот же режим методом |
| `CreateArchive(path)` | Удаляет существующий файл и создаёт новый архив |
| `OpenArchive(path[, access])` | Открывает архив; по умолчанию только для чтения |
| `OpenOrCreate(path)` | Открывает существующий архив для чтения и записи либо создаёт новый |
| `AddFile(path)` | Добавляет точное имя или маску в корень архива |
| `AddFiles(path)` | Эквивалентен `AddFile()` |
| `AddFilesToPath(path, pathInArchive)` | Добавляет файлы в заданный путь архива |
| `AddDirectory(path)` | Рекурсивно добавляет файлы каталога |
| `AddDirectoryToPath(path, pathInArchive)` | Рекурсивно добавляет файлы в заданный путь архива |
| `ListFiles()` | Возвращает массив имён из центрального каталога |
| `Extract(outputPath)` | Извлекает весь архив |
| `ExtractFiles(filesPath, entryName, outputPath)` | Извлекает запись с точным именем |
| `Save()` | Возвращает признак открытого объекта архива |
| `Close()` | Завершает работу и освобождает архив |

Примеры создания, добавления, чтения и распаковки проверены на стенде.
`OpenOrCreate()`, `AddFilesToPath()` и `AddDirectoryToPath()` отдельно не
проверялись.

[Вернуться к сравнению сборок](../index.md).
