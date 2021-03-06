---
author: Justin Ellingwood
date: 2017-03-21
language: ru
license: cc by-nc-sa
source: https://www.digitalocean.com/community/tutorials/nginx-ubuntu-16-04-ru
---

# Как установить Nginx в Ubuntu 16.04

### Введение

Nginx является одним из самых популярных веб-серверов в мире, его используют для хостинга самых больших и нагруженных сайтов в Интернете. Nginx в подавляющем большинстве случаев менее требователен к ресурсам, чем Apache; его можно использовать как в качестве веб-сервера, так и в качестве обратного прокси-сервера (reverse proxy).

В этой статье мы рассмотрим процесс установки Nginx на ваш сервер с Ubuntu 16.04.

## Перед установкой

Перед тем, как начать следовать описанным в этой статье шагам, убедитесь, что у вас есть обычный не-рутовый (non-root) пользователь с привилегиями `sudo`. Узнать, как настроить такого пользователя на вашем сервере, можно из [статьи о первичной настройке сервера на Ubuntu 16.04](initial-server-setup-with-ubuntu-16-04).

После того, как вы создали такого пользователя, зайдите на сервер используя его логин и пароль. Теперь вы готовы следовать шагам, описанным в этой статье.

## Шаг 1: Установка веб-сервера Nginx

Nginx доступен в стандартных репозиториях Ubuntu, поэтому его установка достаточно проста.

Поскольку мы собираемся использовать `apt` в первый раз в ходе этой сессии, начнём с обновления локального списка пакетов. Далее установим сервер:

    sudo apt-get update
    sudo apt-get install nginx

В результате выполнения этих команд `apt-get` установит Nginx и другие необходимые для его работы пакеты на ваш сервер.

## Шаг 2: Настройка файрвола

Перед тем, как начать проверять работу Nginx, нам необходимо настроить наш файрвол для разрешения доступа к сервису. При установки Nginx регистрируется в сервисе файрвола `ufw`. Поэтому настройка доступа осуществляется достаточно просто.

Для вывода настроек доступа для приложений, зарегистрированных в `ufw`, введём команду:

    sudo ufw app list

В результате выполнения этой команды будет выведен список профилей приложений:

    ВыводAvailable applications:
      Nginx Full
      Nginx HTTP
      Nginx HTTPS
      OpenSSH

Как видно из этого вывода, для Nginx настроено три профиля:

- **Nginx Full** : этот профиль открывает порты 80 (обычный, не шифрованный веб-трафик) и 443 (трафик шифруется с помощью TLS/SSL).
- **Nginx HTTP** : этот профиль открывает только порт 80 (обычный, не шифрованный веб-трафик).
- **Nginx HTTPS** : этот профиль открывает только порт 443 (трафик шифруется с помощью TLS/SSL).

Рекомендуется настраивать `ufw` таким образом, чтобы разрешать только тот трафик, который вы хотите разрешить в явном виде. Поскольку мы ещё не настроили SSL для нашего сервера, в этой статье мы разрешим трафик только для порта 80.

Сделать это можно следующей командой:

    sudo ufw allow 'Nginx HTTP'

Вы можете проверить изменения введя команду:

    sudo ufw status

В результате должен отобразиться вывод следующего вида:

    ВыводStatus: active
    
    To Action From
    -- ------ ----
    OpenSSH ALLOW Anywhere                  
    Nginx HTTP ALLOW Anywhere                  
    OpenSSH (v6) ALLOW Anywhere (v6)             
    Nginx HTTP (v6) ALLOW Anywhere (v6)

## Шаг 3: Проверка работы веб-сервера

После завершения процесса установки Ubuntu 16.04 запустит Nginx автоматически. Таким образом веб-сервер уже должен быть запущен.

Мы можем убедиться в этом выполнив следующую команду:

    systemctl status nginx

    Вывод● nginx.service - A high performance web server and a reverse proxy server
       Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
       Active: active (running) since Mon 2016-04-18 16:14:00 EDT; 4min 2s ago
     Main PID: 12857 (nginx)
       CGroup: /system.slice/nginx.service
               ├─12857 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
               └─12858 nginx: worker process

Как видно из вывода выше, сервис запущен и работает. Тем не менее, убедимся в его полной работоспособности путём запроса веб-страницы.

Для этого мы можем проверить, отображается ли веб-страница Nginx, доступная по умолчанию при вводе доменного имени или IP адреса сервера.

Если у вас ещё нет настроенного доменного имени для вашего сервера, вы можете узнать, [как настроить домен в Digital Ocean из этой статьи](https://digitalocean.com/community/articles/how-to-set-up-a-host-name-with-digitalocean).

Если вы не хотите настраивать доменное имя для вашего сервера, вы можете использовать публичный IP адрес вашего сервера. Если вы не знаете публичного IP адреса сервера, вы можете найти этот IP адрес следующей командой:

    ip addr show eth0 | grep inet | awk '{ print $2; }' | sed 's/\/.*$//'

В результате будет выведено несколько IP адресов. Попробуйте вставить каждый из них в браузер.

Другим способом определить свой IP адрес будет проверка, как ваш сервер виден из Интернета:

    sudo apt-get install curl
    curl -4 icanhazip.com

Наберите полученный IP адрес или доменное имя в вашем веб-браузере. Вы должны увидеть страницу Nginx по умолчанию.

    http://доменное_имя_или_IP_адрес

![Страница Nginx по умолчанию](https://raw.githubusercontent.com/opendocs-md/do-tutorials-images/master/img/nginx_1604/default_page.png)

Если вы видите подобную страницу в своём браузере, вы успешно установили Nginx.

## Шаг 4: Управление процессом Nginx

Теперь, когда Nginx установлен и мы убедились в его работоспособности, ознакомимся с некоторыми базовыми командам для управления нашим веб-сервером.

Для остановки веб-сервера используйте команду:

    sudo systemctl stop nginx

Для запуска остановленного веб-сервера наберите:

    sudo systemctl start nginx

Для перезапуска веб-сервера можно использовать следующую команду:

    sudo systemctl restart nginx

Если вы вносите изменения в конфигурацию Nginx, часто можно перезапустить его без закрытия соединений. Для этого можно использовать следующую команду:

    sudo systemctl reload nginx

По умолчанию Nginx настроен на автоматический запуск при запуске сервера. Если такое поведение веб-сервера вам не нужно, вы можете отключить его следующей командой:

    sudo systemctl disable nginx

Для повторного включения запуска Nginx при старте сервера введите:

    sudo systemctl enable nginx

## Шаг 5: Важные файлы и директории Nginx

Теперь, когда мы знаем основные команды для управления веб-сервером, ознакомимся с основными директориями и файлами.

### Контент

- `/var/www/html`: веб-контент, который по умолчанию состоит только из тестовой страницы Nginx, которую мы видели ранее, находится в директории `/var/www/html`. Путь к этой директории можно настроить в файлах конфигурации Nginx.

### Конфигурация сервера

- `/etc/nginx`: директория конфигурации Nginx. Все файлы конфигурации Nginx находятся в этой директории.
- `/etc/nginx/nginx.conf`: основной файл конфигурации Nginx. Этот файл используется для внесения изменений в глобальную конфигурацию Nginx.
- `/etc/nginx/sites-available`: директория, в которой хранятся “серверные блоки” для каждого сайта (серверные блоки являются приблизительным аналогом виртуальных хостов в Apache). Nginx не будет использовать конфигурационные файлы в этой директории, если они не имеют соответствующих ссылок в директории `sites-enabled` (см. ниже). Обычно все настройки серверного блока осуществляются в этой директории, а затем сайт активируется путём создания ссылки в другой директории.
- `/etc/nginx/sites-enabled/`: в этой директории хранятся серверные блоки для активированных сайтов. Обычно это достигается путём создания ссылок на конфигурационные профили сайтов, расположенные в директории `sites-available`.
- `/etc/nginx/snippets`: в этой директории хранятся фрагменты конфигурации, которые можно использовать при конфигурации любых сайтов. Фрагменты конфигурации, которые потенциально могут быть использованы в нескольких файлах конфигурации, являются прекрасными кандидатами для создания этих сниппетов.

### Логи сервера

- `/var/log/nginx/access.log`: каждый запрос к вашему веб-серверу записывается в этот файл лога, если иное не задано настройками Nginx.
- `/var/log/nginx/error.log`: любые ошибки Nginx будут записываться в этот файл.

## Заключение

Теперь, когда у вас есть установленный и настроенный веб-сервер, вы можете выбирать, какой контент отдавать пользователям, и какие другие технологии вы можете использовать в дополнение к веб-серверу.

Об [использовании серверных блоков Nginx можно узнать подробнее в этой статье](how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04). Если вы хотите использовать более полный стек приложений, рекомендуем ознакомиться со [статьёй о настройке стека LEMP на сервере с Ubuntu 16.04](how-to-install-linux-nginx-mysql-php-lemp-stack-in-ubuntu-16-04).
