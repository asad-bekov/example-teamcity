# Домашнее задание к занятию 11 «Teamcity»

# Задание к занятию 6 «Создание собственных модулей»

>Репозиторий: example-teamcity\
>Выполнил: Асадбеков Асадбек\
>Дата выполнения: июль 2025

Этот проект — учебный пример для развертывания собственного CI/CD на базе TeamCity, Nexus и облачных VM. Тут показаны примеры запусков, учет нюансов, нахождения артефактов и повтор `pipeline` с нуля.

---

## Запуск инфраструктуры

### 1. Поднятие виртуалки

В облаке (Yandex Cloud) создал 4 VM:

* `teamcity-server`
* `teamcity-agent`
* `nexus-01`
* `ansible-runner`

<details><summary>Скриншот VM</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/9.PNG">
</details>

---

### 2. Старт TeamCity (server + agent)

Используется Docker:

```bash
sudo docker run -d --name teamcity-server -p 8111:8111 jetbrains/teamcity-server
sudo docker run -d --name teamcity-agent --link teamcity-server:teamcity-server jetbrains/teamcity-agent
```

<details><summary>docker ps (server)</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/1.PNG">
</details>
<details><summary>docker ps (agent)</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/3.PNG">
</details>

---

### 3. Первый запуск TeamCity

В браузере:

```
http://<IP teamcity-server>:8111 
```

Шаги первого старта.

<details><summary>Первый старт</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/2.PNG">
</details>

---

### 4. Проверка агента

Вкладка **Agents**. Агент появляется автоматически.

<details><summary>Agents</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/4.PNG">
</details>

---

### 5. Разворачивание Nexus через Ansible

В папке `infrastructure`:

```bash
cd infrastructure
ansible-playbook -i inventory/prod.yml site.yml
```

<details><summary>Логи Ansible Nexus</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/5.PNG">
</details>

---

### 6. Проверка Nexus

Адрес:

```
http://<IP nexus>:8081
```

<details><summary>Nexus Welcome</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/6.PNG">
</details>

---

### 7. Импорт проекта в TeamCity

В TeamCity настраивал VCS Root на свой репозиторий:
[https://github.com/asad-bekov/example-teamcity](https://github.com/asad-bekov/example-teamcity)

---

## Где смотреть артефакты

После успешной сборки во вкладке **Artifacts** внутри сборки:

<details><summary>Artifacts TeamCity</summary>
<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/8.PNG">
</details>

Тут появляются .jar файлы, собранные Maven.

**Если артефактов нет:**
Проверить, что в `.teamcity/settings.kts` есть:

```kotlin
artifactRules = "target/*.jar"
```

---

## Как повторить pipeline

1. **Клонировать мой репозиторий:**

   ```bash
   git clone https://github.com/asad-bekov/example-teamcity.git
   ```
2. **Поднять виртуалки/Docker, как у меня (см. выше)**
3. **В файлах settings.xml и pom.xml прописать актуальные адреса Nexus/TeamCity** *(после каждой перезагрузки yandex cloud ip адреса меняются)*
4. **Импортировать Versioned Settings в TeamCity**:

   * Project → Versioned Settings → подключить свой git

<details><summary>Versioned Settings</summary>

<img src="https://github.com/asad-bekov/example-teamcity/blob/master/img/7.PNG">

</details>

5. **Проверка pipeline:** любой push/PR — автоматическая сборка, тесты, артефакты.

---

## Примеры конфигов

### Maven `settings.xml`

<details>
<summary>settings.xml</summary>

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 http://maven.apache.org/xsd/settings-1.2.0.xsd">
  <servers>
    <server>
      <id>nexus</id>
      <username>admin</username>
      <password>netolog1</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://130.193.46.216:8081/repository/maven-releases/</url>
    </mirror>
  </mirrors>
</settings>
```
</details>

### Maven `pom.xml` (фрагмент)

<details>
<summary>pom.xml</summary>

```xml
<project ...>
    <distributionManagement>
        <repository>
            <id>nexus</id>
            <url>http://130.193.46.216:8081/repository/maven-releases/</url>
        </repository>
    </distributionManagement>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>plaindoll.HelloPlayer</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
</details>

### Kotlin DSL TeamCity (`.teamcity/settings.kts`)

<details>
<summary>settings.kts</summary>

```kotlin
import jetbrains.buildServer.configs.kotlin.*
import jetbrains.buildServer.configs.kotlin.buildFeatures.perfmon
import jetbrains.buildServer.configs.kotlin.buildSteps.maven
import jetbrains.buildServer.configs.kotlin.triggers.vcs

version = "2025.03"

project {
    buildType(Build)
}

object Build : BuildType({
    name = "Build"
    vcs {
        root(DslContext.settingsRoot)
    }
    steps {
        maven {
            id = "Maven2"
            goals = "clean test"
            runnerArgs = "-Dmaven.test.failure.ignore=true"
            userSettingsSelection = "settings.xml"
        }
    }
    triggers {
        vcs { }
    }
    features {
        perfmon { }
    }
    artifactRules = "target/*.jar"
})
```
</details>
---

## Ссылки

* **Мой репозиторий:**
  [https://github.com/asad-bekov/example-teamcity](https://github.com/asad-bekov/example-teamcity)
* **TeamCity:**
  `http://<teamcity-server>:8111`
* **Nexus:**
  `http://<nexus-01>:8081`
* **CI конфиг:**
  `.teamcity/settings.kts`

---