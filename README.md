# Оптимизация запуска тестов
Описание по оптимизации запуска тестов в CI

## Предусловия
1. Скрипт и плагин подходят для проекта, с монорепозиторием. 
2. Правила оформления мердж реквестов на проекте подразумевают, что все изменения подаются одним коммитом.
3. Для объединения изменений используется стратегия rebase. 

## Скрипт

### Пример проекта

Рассмотрим проект, в котором можно оптимизировать запуск IT.

`MonoRepo/coffe-service/src/main/java/com/sabeshkin/system/<без_версий>/...`

`MonoRepo/study-service/src/main/java/com/sabeshkin/system/<без_версий>/...`

`MonoRepo/landing/src/main/java/com/sabeshkin/system/v1/...`

`MonoRepo/landing/src/main/java/com/sabeshkin/system/v2/...`

Все IT содержат в названии `...v1IntegrationTest` или `...v2IntegrationTest`

Нужен скрипт, который проанализирует какие файлы были изменены в крайнем коммите и в зависимости от результата выполнит только интеграционные тесты, которые содержат (contains) в себе "v1" или только те, что содержат (contains) в себе "v2", либо выполнит все тесты.
  
Если поменялся любой файл по _пути_, который содержит  "v1" - выполнить только интеграционные тесты, которые содержат (contains) в себе "v1"  
Если поменялся любой файл по _пути_, который содержит  "v2" - выполнить только интеграционные тесты, которые содержат (contains) в себе "v2"  
Если поменялся любой файл по _пути_, который содержит  "v2" И поменялся любой файл по пути, который содержит  "v1" - выполнить все тесты  
Если поменялся любой файл по _пути_, который НЕ содержит  "v2" ИЛИ "v1" - выполнить все тесты  

### Сам скрипт
```bash
#!/bin/bash

# Получаем список измененных файлов в последнем коммите
changed_files=$(git diff --name-only HEAD^ HEAD)

# Флаги для определения изменений
has_v1=false
has_v2=false
has_non_versioned=false

# Анализируем измененные файлы
for file in $changed_files; do
  if [[ $file == *"v1"* ]]; then
    has_v1=true
  elif [[ $file == *"v2"* ]]; then
    has_v2=true
  else
    # Проверяем, что это файл из system, но без версии
    if [[ $file == *"com/sabeshkin/system/"* ]]; then
      has_non_versioned=true
    fi
  fi
done

# Определяем, какие тесты запускать
if $has_v1 && $has_v2 || $has_non_versioned; then
  echo "Запускаем ВСЕ интеграционные тесты (изменены и v1, и v2 ИЛИ изменены файлы без версии)"
  # Запускаем все IT тесты, пропускаем unit-тесты
  mvn verify -Dskip.surefire.test=true
elif $has_v1; then
  echo "Запускаем только интеграционные тесты для v1"
  # Запускаем только v1 IT тесты, пропускаем unit-тесты
  mvn verify -Dskip.surefire.test=true -Dit.test="*v1IntegrationTest"
elif $has_v2; then
  echo "Запускаем только интеграционные тесты для v2"
  # Запускаем только v2 IT тесты, пропускаем unit-тесты
  mvn verify -Dskip.surefire.test=true -Dit.test="*v2IntegrationTest"
else
  echo "Запускаем ВСЕ интеграционные тесты (изменены файлы вне system или другие условия)"
  # Запускаем все IT тесты, пропускаем unit-тесты
  mvn verify -Dskip.surefire.test=true
fi
```

## Maven плагин для получения списка измененных модулей

### Предусловие для плагина

* Сборка происходит на раннерах использующих общий кэш для артефактов.
* Используется только в CI, т.к. меняет pom.xml на ветке.

### Назначение
Запуск тестов только на те модули, которые были задеты изменениями прямо или косвенно. 

### Преимущество
Учитываются зависимости модулей на семантическом уровне, а не только синтаксис путей, что позволяет более тонкий выбор, относительно скрипта.

### Результат работы
Результатом работы плагина должны стать два файла changes.list и dependencies.list,
в первом список модулей файлы которых поменялись в последнем commit,
во втором в дополнение к измененным модулям, добавляются зависимые от них.

### Запуск

Для запуска выполните:
```sh
mvn su.rastov.maven:version-based-on-commit-maven-plugin:0.2.1:change-version
```

Инкрементальная сборка:

```yaml
incrementInstall:
  script:
    - mvn su.rastov.maven:version-based-on-commit-maven-plugin:change-version
    - mvn --projects=$(cat changes.list)  install

```

Запуск юнит тестов только для измененных и зависимых от них модулей:

```yaml
incrementTest:
  script:
    - mvn su.rastov.maven:version-based-on-commit-maven-plugin:change-version
    - mvn --projects=$(cat dependencies.list)  test
```

### Пример

Предположим, имеем проект следующей структуры модулей
(в скобках указанны зависимости):

```
root
|-dao
| |-dao-api
| |-dao-impl (dao-api)
|-parent
|-service
| |-service-api
| |-service-impl (dao-api, service-api)
|-web (service-api)
|-app (dao-impl, service-impl, web)
|-jacoco (dao-api, dao-impl, service-api, service-impl, web, app)
```

В истории git имеем следующие commit:

1. commit 191c85f3 Tue Jul 20 2024
2. commit 96f43e01 Tue Jul 21 2024
3. commit 73542265 Tue Jul 22 2024
4. commit a832ebf4 Tue Jul 23 2024
5. commit f00494df Tue Jul 23 2024

Какие модули менялись в каких commit и какую версию будут иметь модули после запуска plugin:

| Модули       | 191c85f3 | 96f43e01 | 73542265 | a832ebf4 | f00494df | Версия                |
|:-------------|:--------:|:--------:|:--------:|:--------:|:--------:|-----------------------|
| root         |    X     |          |          |          |          | не изменится          |
| dao          |    X     |          |          |          |          | не изменится          |
| dao-api      |    X     |    X     |          |          |          | `2024_07_21_96f43e01` |
| dao-impl     |    X     |    X     |          |          |    Х     | `2024_07_23_f00494df` |
| parent       |    X     |          |          |          |          | не изменится          |
| service      |    X     |          |    X     |          |          | не изменится          |
| service-api  |    X     |          |    X     |          |          | `2024_07_22_73542265` |
| service-impl |    X     |          |    X     |          |          | `2024_07_22_73542265` |
| web          |    X     |          |          |    Х     |          | `2024_07_23_a832ebf4` |
| app          |    X     |          |          |          |    Х     | `2024_07_23_f00494df` |
| jacoco       |    X     |          |          |          |          | `2024_07_20_191c85f3` |

Содержимое файлов:

changes.list

```text
dao/dao-impl, app
```

dependencies.list

```text
dao/dao-impl, app, jacoco
```

Если запуск происходил бы на commit 73542265 то, содержимое файлов было бы следующим:

changes.list

```text
service/service-api, service/service-impl
```

dependencies.list

```text
service/service-api, service/service-impl, web, app, jacoco
```



### Сайд эффекты

* В gitlab последний коммит в МР это не тот, что делали вы, а как ни странно
  commit c merge request :), и если для gitlab реальный коммит находится скорей всего правильно,
  в других системах, поведение может отличатся
* При сборке на раннер может стягиваться не вся история коммитов, и как это отразится
  на плагине не известно.
* Не известно как поведет себя плагин если использовать его при стратегии merge, изначально
  создавался для проекта, где стратегия rebase.
* Изменения в модулях с `<package>pom</package>` игнорируются, но они могут влиять на сборку,
  тесты и так далее.

