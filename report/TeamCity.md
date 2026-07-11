

## Домашнее задание к занятию 11 «Teamcity»
**Подготовка к выполнению**

- В Yandex Cloud создайте новый инстанс (4CPU4RAM) на основе образа jetbrains/teamcity-server.
- Дождитесь запуска teamcity, выполните первоначальную настройку.
- Создайте ещё один инстанс (2CPU4RAM) на основе образа jetbrains/teamcity-agent. Пропишите к нему переменную окружения SERVER_URL: "http://<teamcity_url>:8111".
- Авторизуйте агент.
- Сделайте fork [репозитория](https://github.com/aragastmatb/example-teamcity).
- Создайте VM (2CPU4RAM) и запустите [playbook](https://github.com/netology-code/mnt-homeworks/blob/MNT-video/09-ci-05-teamcity/infrastructure).

**Основная часть**

- Создайте новый проект в teamcity на основе fork.
- Сделайте autodetect конфигурации.
- Сохраните необходимые шаги, запустите первую сборку master.
- Поменяйте условия сборки: если сборка по ветке master, то должен происходит mvn clean deploy, иначе mvn clean test.
- Для deploy будет необходимо загрузить settings.xml в набор конфигураций maven у teamcity, предварительно записав туда креды для подключения к nexus.
- В pom.xml необходимо поменять ссылки на репозиторий и nexus.
- Запустите сборку по master, убедитесь, что всё прошло успешно и артефакт появился в nexus.
- Мигрируйте build configuration в репозиторий.
- Создайте отдельную ветку feature/add_reply в репозитории.
- Напишите новый метод для класса Welcomer: метод должен возвращать произвольную реплику, содержащую слово hunter.
- Дополните тест для нового метода на поиск слова hunter в новой реплике.
- Сделайте push всех изменений в новую ветку репозитория.
- Убедитесь, что сборка самостоятельно запустилась, тесты прошли успешно.
- Внесите изменения из произвольной ветки feature/add_reply в master через Merge.
- Убедитесь, что нет собранного артефакта в сборке по ветке master.
- Настройте конфигурацию так, чтобы она собирала .jar в артефакты сборки.
- Проведите повторную сборку мастера, убедитесь, что сбора прошла успешно и артефакты собраны.
- Проверьте, что конфигурация в репозитории содержит все настройки конфигурации из teamcity.
В ответе пришлите ссылку на репозиторий.


# Разбор решения задания.
## Подготовительная часть
Созданы 2 ВМ в Яндекс облаке:
![](<Pasted image 20260710134605.png>)
Авторизовал агента
![](<Pasted image 20260710140151.png>)
Сделал форк учебного репозитория в своем гитхаб
![](<Pasted image 20260710140516.png>)
Подключил TeamCity к форку репозитория
![](<Pasted image 20260710143406.png>)
Создал третью машину на базе Rocky Linux 9.6. Склонировал папку с плейбуком на локальную машину и запустил его. С костылями в виде отключения Selinux playbook отработал, Nexus запустился
![](<Pasted image 20260710161135.png>)
![](<Pasted image 20260710161216.png>)

## **Основная часть**
- В TeamCity создан новый проект на основе fork-репозитория GitHub.
- После подключения VCS Root выполнен `autodetect` конфигурации сборки, в результате чего TeamCity определил Maven-проект и создал `build step` с `goals` `clean test`.
- Первая сборка ветки `master` была успешно выполнена на подключённом `build agent`.
- В ходе сборки выполнен `checkout` исходного кода, запуск `Maven`-сборки и прохождение 5 unit-тестов `WelcomerTest`
![](<Pasted image 20260710185417.png>)


![](<Pasted image 20260710185717.png>)

![](<Pasted image 20260710190147.png>)

![](<Pasted image 20260710190324.png>)

Для мониторинга веток кроме master заполнил поле `Branch specification` в `Edit VCS Root` 
```text
+:refs/heads/*
```
Эта запись говорит TeamCity отслеживать все ветки из Git, а **default branch** `refs/heads/master` останется основной веткой проекта. TeamCity использует `branch specification` именно для мониторинга дополнительных веток помимо **default branch**. Если оставить поле пустым, то ветка `feature/add_reply` может не появиться в TeamCity как полноценная `build branch`. А согласно заданию необходимо, чтобы сборка по feature-ветке запускалась автоматически и работала иначе, чем  `master`.

Переделал текущую конфигурацию из одного `Maven step` в два шага с разными условиями выполнения. TeamCity поддерживает `execution conditions` для `build steps`, в том числе запуск шага только в `default branch` через параметр `teamcity.build.branch.is_default`. (https://www.jetbrains.com/help/teamcity/build-step-execution-conditions.html)
- для **master** выполнять `clean deploy`;
- для **всех остальных веток** выполнять `clean test`.
Для этого создаk **два Maven build steps** со своими условиями выполнения для каждого:
1. `Maven Test`
2. `Maven Deploy`

![](<Pasted image 20260710201926.png>)
Параметр `teamcity.build.branch.is_default` :
- если `true`  то build идёт по `default branch`, то есть по `master`; (https://teamcity.jetbrains.com/app/dsl-documentation/root/build-step-conditions/index.html)
- если не `true` то  это не `master`, значит будет запущен запускать `clean test`.
Таким образом первый шаг `Maven Test` будет работать  **не на master**.
![](<Pasted image 20260710202521.png>)
Сохранено. 
Следующим шагом добавил step `Maven`
![](<Pasted image 20260710203159.png>)
с условиями:
![](<Pasted image 20260710203558.png>)
В итоге два `step`: 
![](<Pasted image 20260710203648.png>)
шаг `clean test` выполняется **только не** на `default branch`, а `clean deploy` - только на `default branch` `master`
Создал файл `nexus-settings.xml` (https://www.jetbrains.com/help/teamcity/maven.html)
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>admin123</password>
    </server>
  </servers>
</settings>
```
и сохранил его во вкладке `Maven Settings`
![](<Pasted image 20260710211458.png>)
В настройках `step` `Maven Deploy` привязал файл `nexus-settings`
![](<Pasted image 20260710211939.png>)
Закоммитил и запушил изменения в удаленный репозиторий (форк учебного) в который смотрит Teamcity
Запустил кнопкой `Run`
![](<Pasted image 20260710213343.png>)
Отработал `clean deploy`
![](<Pasted image 20260710213602.png>)
В nexus:
![](<Pasted image 20260710213906.png>)
![](<Pasted image 20260710214243.png>)

Для синхронизации с моим репозиторием (форком учебного) выбрал такие настройки:
![](<Pasted image 20260710215038.png>)
Проект успешно  попал в гитхаб, причем вместе  с файлом, где прописан пароль от nexus (галочка о хранении секретов отдельно проставлялась) 
![](<Pasted image 20260710223317.png>)
![](<Pasted image 20260710223959.png>)
Создал новую ветку в репозитории `feature/add_reply`
```bash
git checkout -b feature/add_reply
```
Добавил в конец файла `Welcomer.java`метод, возвращающий отдельную реплику, содержащую слово `hunter`
```java
public String sayReply() {
        return "A hunter must hunt.";
}
```
В конец файла `WelcomerTest.java` добавил новый тест, который ищет слово `hunter` в реплике
```java
@Test
public void welcomerSaysReply() {
        assertThat(welcomer.sayReply(), containsString("hunter"));
}
```
Cохранил изменения и запушил в новую ветку:
```bash
git add .
git commit -m "Add reply for hunter"
git push -u origin feature/add_reply
```
![](<Pasted image 20260710232808.png>)
Автоматический запуск сборки для feature-ветки не произошёл, так как в `build configuration` отсутствуют `triggers`.
запустил вручную с явным указанием новой ветки
![](<Pasted image 20260710233702.png>)
успешно 
![](<Pasted image 20260710233812.png>)
Слил (merge) обе ветки: feature/add_reply:
```bash
git checkout master
git pull origin master
git merge feature/add_reply
git push origin master
```
Запустил build (триггеры не настраивались по условиям задачи, поэтому автоматический запуск не возможен).
В сборке по ветке `master` нет собранного артефакта .
![](<Pasted image 20260711000456.png>)
Судя по логу, тесты прошли успешно, но сборка закончилась `failed` поскольку Maven во время `deploy` начал загружать `.pom` и `.jar` (plaindoll-0.0.2.jar) в Nexus. 
Nexus вернул HTTP 400, потому что release-репозиторий запрещает обновление уже опубликованных  артефактов, а файл был загружен туда с таким же релизом ранее (https://latchkey.dev/learn/service-status/nexus-400-repository-does-not-allow-updating-assets-in-ci)

Для изменения правил сборки артефактов заполнил соответствующее поле:
![](<Pasted image 20260711002231.png>)
значением: `target/*.jar`
![](<Pasted image 20260711002544.png>)
Это указывает TeamCity публиковать все `.jar` из каталога `target` как `build artifacts` (https://www.jetbrains.com/teamcity/tutorials/general/working-with-artifacts/)
Для корректной сборки и чтобы избежать ошибки из-за повторной публикации артефактов в nexus подправил `pom` файл где повысил версию сборки до 0.0.3. Запушил все в удаленный репозиторий и запустил ручную сборку.
![](<Pasted image 20260711003603.png>)
Все закончилось успешно. Все тесты прошли корректно. Артефакты собраны.

