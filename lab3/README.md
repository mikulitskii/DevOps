# Лабораторная работа №3. Loki + Zabbix + Grafana

## Цели работы:
Подключить к тестовому сервису Nextcloud мониторинг + логирование. Осуществить визуализацию через Grafana.

## Выполение работы:
### 1. Логирование
Создаём файл [docker-compose.yml](docker-compose.yml), который содержит в себе тестовые сервисы Nextcloud, Loki, Promtail, Grafana, Zabbix и Postgres для него.
</br>Так же создаём конфигурационный файл [promtail_config.yml](promtail_config.yml) для Promtail.
</br>Стартуем docker-compose.yml командой `docker-compose up -d`, в результате должны увидеть, что все контейнеры с сервисами успешно создались и запустились (проверить можно командой 	`docker ps`):

![screenshot](img/1.png)

![screenshot](img/2.png)

Далее открываем в браузере Nextcloud по адресу localhost:8080, и регистрируем аккаунт:

![screenshot](img/3.png)

![screenshot](img/4.png)

Проверяем, что логи записываются в лог фаел, для этого выполним команды:

```
docker exec -it <ID контейнера с nextcloud> bash
cat data/nextcloud.log
```

![screenshot](img/5.png)

Так же проверяем в логах promtail, что он "подцепил" нужный нам log-файл: должны быть строчки, содержащие **msg=Seeked/opt/nc_data/nextcloud.log**.
Для этого выполним команду `docker logs <ID контейнера с promtail>`

![screenshot](img/6.png)

Видим желанные строки, всё хорошо.

### 2. Мониторинг
Переходим к настройке Zabbix. Подключаемся к веб-интерфейсу localhost:8082, и логинимся, используя логин **Admin** и пароль **zabbix**.

![screenshot](img/7.png)

В разделе **Data collection → Templates** делаем **Import** кастомного шаблона (темплейта) для мониторинга nextcloud. Для импорта нужно предварительно
создать файл [template.yml](template.yml).

![screenshot](img/8.png)

![screenshot](img/9.png)

Чтобы Zabbix и Nextcloud могли общаться по своим коротким именам внутри докеровской сети, в некстклауде необходимо “разрешить” это имя. Для этого
нужно зайти на контейнер некстклауда под юзером **www-data** и выполнить команду `php occ config:system:set trusted_domains 1 --value="nextcloud"`:

![screenshot](img/10.png)

Переходим в **Data collection -> Hosts**, жмём **Create host**. Указываем имя контейнера nextcloud, видимое имя - любое, хост группа - Applications.
В поле Templates выбираем добавленный ранее шаблон **Templates/Applications -> Test ping template**

![screenshot](img/11.png)

![screenshot](img/12.png)

Настройка хоста закончена. Переходим в раздел **Monitoring -> Latest data**. Через какое-то время там должны появиться первые данные, в
нашем случае значение healthy.

![screenshot](img/13.png)

На этом мониторинг можно считать успешно настроенным. Для проверки включим в некстклауде **maintenance mode** командой `php occ maintenance:mode --on` в контейнере,
проверяем, что сработал триггер (в разделе **Monitoring → Problems**). Потом выключаем режим обратно `php occ maintenance:mode --off` и убеждаемся, что
проблема помечена как "решенная".

![screenshot](img/14.png)

![screenshot](img/15.png)


### 3. Визуализация
В терминале выполняем команду `docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"` , затем `docker restart grafana`

![screenshot](img/16.png)

Заходим в графану localhost:3000, раздел **Administration -> Plugins**. Найти там **Zabbix** и активируем, нажав **Enable**.

![screenshot](img/17.png)

Далее подключаем Loki к Grafana, раздел **Connections -> Data sources -> Loki**. В настройках подключения указываем имя loki и адрес http://loki:3100 , все
остальное можно оставить по дефолту:

![screenshot](img/18.png)

Сохраняем подключение, нажав **Save & Test**. Если нет ошибок и сервис предлагает перейти к визуализации и/или просмотру данных, значит всё
настроено правильно

![screenshot](img/19.png)

Далее можно перейти в **Explore**, выбрать в качестве селектора **job** либо **filename** — если все было правильно настроено, то нужные значения будут в выпадающем списке. Затем нажать **Run query** и увидеть свои логи.

![screenshot](img/20.png)

![screenshot](img/21.png)

Аналогично создаём подключение для Zabbix. В версии указанной изначально в Лабораторной работе подключить не удастся, поэтому сменил grafana на версию 12.0.1 и установил плагин Zabbix вручную (через интерфейс не создаётся). Также url данный в лабораторной работе некорректен, потому что в docker-complose.yml изначально указано другой значение. Правильный URL

```
http://zabbix-front:8080/api_jsonrpc.php
```

![screenshot](img/22.png)

![screenshot](img/23.png)

![screenshot](img/24.png)

![screenshot](img/25.png)

### 4. Задание: создать дашборд
Пример дашборда с данными из логов nexcloud и loki в качестве источника данных. Были применены трансформации Extract fileds и Organize fields.

![screenshot](img/26.png)

Пример дашборда с данными от zabbix

![screenshot](img/27.png)

![screenshot](img/28.png)


## Ответы на вопросы
Вопрос: В чем разница между мониторингом и observability?

**Мониторинг** — это сбор конкретных метрик о состоянии системы, доступности, ошибках.  
Например: работает ли сервис, сколько пользователей активно, время отклика.

---

**Observability (наблюдаемость)** — более широкое понятие, которое включает мониторинг, но выходит дальше.  
Это способность по собранной информации (метрики + логи + трейсы) понять внутреннее состояние системы, даже если что-то пошло не так.  

Другими словами, observability — это не только сбор метрик, но и детализированных логов и трассировок, чтобы быстро обнаружить и объяснить причину проблем.
