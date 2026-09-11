# Преобразование XQuery в SQL

Раздел показывает, какой SQL формирует WebSoft для одного и того же XQuery в
разных сборках и СУБД.

## Статьи

- [WebSoft HCM 434 и MSSQL](./434-mssql/index.md)
- [WebSoft HCM 434 и PostgreSQL](./434-postgresql/index.md)
- [WebSoft HCM 906 и MSSQL](./906-mssql/index.md)
- WebSoft HCM 906 и PostgreSQL — не проверено.

Во всех статьях используются одинаковые XQuery. Это позволяет сравнивать
именно работу сборки и SQL provider, а не разные исходные запросы.

## Матрица запросов

[Скачать набор из 61 XQuery в JSON](./files/cases.json). Этот файл используется
без изменений при проверке каждой сборки и СУБД.

## Источник

[Документация `tools.xquery()`](https://docs.websoft.ru/_wt/6774676213731516150/parent_id/6809298112856471128)
рекомендует эту функцию для длинных и иерархических запросов.
