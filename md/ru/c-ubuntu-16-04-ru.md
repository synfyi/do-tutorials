---
author: Mitchell Anicas
date: 2017-01-18
language: ru
license: cc by-nc-sa
source: https://www.digitalocean.com/community/tutorials/c-ubuntu-16-04-ru
---

# Начальная настройка сервера c Ubuntu 16.04

### Введение

После создания нового сервера необходимо предпринять несколько шагов по его базовой настройке. Это повысит безопасность и удобство использования вашего сервера и заложит прочный фундамент для последующих действий.

## Шаг 1 - Логин с рутовой учетной записью

Для того, чтобы осуществить вход на ваш сервер, вам необходимо знать публичный IP-адрес сервера. Вам также потребуется пароль учетной записи пользователя `root` или закрытый ключ, если вы установили SSH-ключ для аутентификации. Если вы ещё не зашли на сервер, возможно вы захотите ознакомиться с первым руководством данной серии, [Как подключиться к дроплету по SSH](how-to-connect-to-your-droplet-with-ssh), которое детально описывает этот процесс.

Если вы ещё не зашли на сервер, зайдите под учетной записью `root` при помощи следующей команды (замените выделенное красным на публичный IP-адрес вашего сервера):

    ssh root@IP_адрес_сервера

Завершите процесс входа, приняв предупреждение о подлинности хоста (host authenticity), если оно возникнет, а затем идентифицируя себя как `root` пользователя (с помощью пароля или секретного ключа). Если вы впервые заходите на сервер с использованием пароля, вам будет предложено изменить пароль учетной записи `root`.

### Об учетной записи root

Пользователь `root` является администратором в среде Linux и имеет очень широкий набор привилегий (прав). Из-за повышенных привилегий root-аккаунта _не рекомендуется_ пользоваться этой учетной записью на регулярной основе. Причиной этого является возможность случайно внести в систему деструктивные изменения.

Следующий шаг заключается в создании альтернативной пользовательской учетной записи с ограниченными привилегиями для повседневной работы. Мы продемонстрируем, как при необходимости получить расширенные полномочия во время использования этой учетной записи.

## Шаг 2 - Создание нового пользователя

Выполнив вход с помощью учетной записи `root`-пользователя вы можете создать новую учетную запись, которую можно будет использовать для входа на сервер в дальнейшем.

В этом примере мы создаём новую учетную запись пользователя с именем “sammy”. Вы можете использовать любое другое имя учетной записи, заменив текст, выделенный красным:

    adduser sammy

Вам зададут несколько вопросов, первым из которых будет пароль для новой учетной записи.

Задайте надёжный пароль и, по желанию, заполните дополнительную информацию. Вводить дополнительную информацию не обязательно, вы можете просто нажать `ENTER` в любом поле, которое хотите пропустить.

## Шаг 3 - Привилегии пользователя “root”

Теперь у нас есть новая учетная запись со стандартными привилегиями. Однако иногда нам может потребоваться выполнять задачи с привилегиями администратора.

Во избежание необходимости выхода из-под учетной записи обычного пользователя и входа с учетной записью `root`-пользователя, мы можем настроить возможность использования режима так называемого “супер-пользователя”, в котором наша обычная учетная запись временно получает привилегии `root`-пользователя. Это позволит нашему обычному пользователю выполнять команды с привилегиями администратора с помощью добавления слова `sudo` перед каждой командой.

Чтобы добавить эти привилегии нашей новой учетной записи, необходимо добавить ее в группу “sudo”. По умолчанию, в Ubuntu 16.04 пользователи, входящие в группу “sudo”, могут использовать команду `sudo`.

Из-под `root`-пользователя выполните следующую команду для добавления вашего нового пользователя в группу _sudo_ (замените выделенное красным на имя вашей новой учетной записи):

    usermod -aG sudo sammy

Теперь ваш пользователь сможет выполнять команды с привилегиями супер-пользователя! Вы можете найти более подробную информацию о том, как это работает в [статье о файле Sudoers](how-to-edit-the-sudoers-file-on-ubuntu-and-centos).

Для дальнейшего повышения безопасности вашего сервера, вы можете выполнить следующие шаги.

## Шаг 4 - Настройка авторизации по открытому ключу (Рекомендуется)

Следующий шаг в усилении безопасности вашего сервера - это настройка авторизации по открытому ключу для вашего нового пользователя. Данная настройка повысит безопасность вашего сервера, требуя закрытый SSH ключ для входа.

### Создание пары ключей

Если у вас ещё нет пары SSH-ключей, которая состоит из открытого и закрытого ключей, вам необходимо её создать. Если у вас уже есть ключ, который вы хотите использовать, перейдите к подразделу “Копирование открытого ключа”.

Чтобы создать новую пару ключей, выполните следующую команду в терминале на вашей **локальной машине** (т.е. на вашем компьютере):

    ssh-keygen

Если ваш локальный пользователь называется “localuser”, вы увидите вывод следующего вида:

    ВыводGenerating public/private rsa key pair.
    Enter file in which to save the key (/Users/localuser/.ssh/id_rsa):

Нажмите “ENTER”, чтобы согласиться с адресом и именем файла (или введите другой адрес/имя файла).

Далее вам будет предложено ввести кодовую фразу для защиты ключа. Вы можете ввести кодовую фразу или оставить ее пустой.

**Обратите внимание:** Если вы оставите кодовую фразу пустой, вы сможете использовать закрытый ключ для авторизации без ввода кодовой фразы. Если вы зададите кодовую фразу, вам потребуется _и_ закрытый ключ _и_ кодовая фраза для входа. Добавление кодовой фразы к ключам является более безопасным, но оба метода имеют свои области применения и являются более безопасными, чем базовая авторизация паролем.

В результате этого, в поддиректории `.ssh` домашней директории пользователя _localuser_ будет создан закрытый ключ `id_rsa` и открытый ключ `id_rsa.pub`. Не передавайте закрытый ключ никому, кто не должен иметь доступ к вашим серверам!

### Копирование открытого ключа

После создания пары SSH-ключей, вам необходимо скопировать открытый ключ на ваш новый сервер. Мы опишем два простых способа сделать это.

#### Вариант 1. Использование ssh-copy-id

Если на вашей локальной машине установлен скрипт `ssh-copy-id`, вы можете установить ваш открытый ключ для любого пользователя, для которого вы знаете логин и пароль.

Запустите скрипт `ssh-copy-id`, указав имя пользователя и IP-адрес сервера, на который вы хотите установить ключ:

    ssh-copy-id sammy@IP_адрес_вашего_сервера

После того, как вы введёте пароль, ваш открытый ключ будет добавлен в файл `.ssh/authorized_keys` на вашем сервере. Соответствующий закрытый ключ теперь может быть использован для входа на сервер.

#### Вариант 2. Ручной перенос ключа

Если вы создали пару SSH-ключей, как описано в предыдущем пункте, выполните следующую команду в терминале на вашей **локальной машине** для печати открытого ключа (`id_rsa.pub`):

    cat ~/.ssh/id_rsa.pub

В результате выполнения данной команды на экран будет выведен ваш открытый SSH-ключ, выглядящий примерно так:

Содержимое id\_rsa.pub

    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDBGTO0tsVejssuaYR5R3Y/i73SppJAhme1dH7W2c47d4gOqB4izP0+fRLfvbz/tnXFz4iOP/H6eCV05hqUhF+KYRxt9Y8tVMrpDZR2l75o6+xSbUOMu6xN+uVF0T9XzKcxmzTmnV7Na5up3QM3DoSRYX/EP3utr2+zAqpJIfKPLdA74w7g56oYWI9blpnpzxkEd3edVJOivUkpZ4JoenWManvIaSdMTJXMy3MtlQhva+j9CgguyVbUkdzK9KKEuah+pFZvaugtebsU+bllPTB0nlXGIJk98Ie9ZtxuY3nCKneB+KjKiXrAvXUPCI9mWkYS/1rggpFmu3HbXBnWSUdf localuser@machine.local

Выделите открытый ключ и скопируйте его в буфер обмена.

Чтобы сделать возможным использование SSH-ключа для авторизации с учетной записью нового удалённого пользователя (remote user), вам необходимо добавить открытый ключ в специальный файл в домашней директории этого пользователя.

**На сервере** , осуществив вход с учетной записью `root`-пользователя, выполните следующие команды для переключения на нового пользователя (замените `demo` на ваше имя пользователя):

    su - sammy

Теперь вы находитесь в домашней директории нового пользователя.

Создайте новую директорию под названием `.ssh` и ограничьте права на доступ к ней при помощи следующих команд:

    mkdir ~/.ssh
    chmod 700 ~/.ssh

Теперь откройте файл в директории `.ssh` с названием `authorized_keys` в текстовом редакторе. Мы будем использовать `nano` для редактирования файла:

    nano ~/.ssh/authorized_keys

Далее добавьте ваш открытый ключ (который должен быть в буфере обмена) путем вставки в текстовый редактор.

Нажмите `CTRL-X` для закрытия файла, затем `y` для сохранения внесенных изменений, затем `ENTER` для подтверждения имени файла.

Теперь ограничьте права на доступ к файлу _authorized\_keys_ при помощи следующей команды:

    chmod 600 ~/.ssh/authorized_keys

Введите следующую команду _один_ раз для возврата к пользователю `root`.

    exit

Теперь вы можете заходить на сервер по SSH с учетной записью вашего нового пользователя, используя закрытый ключ для авторизации.

Чтобы узнать больше о том, как работает авторизация по ключам, ознакомьтесь с этим руководством: [Как настроить авторизацию по SSH-ключам на сервере Linux](how-to-configure-ssh-key-based-authentication-on-a-linux-server).

## Шаг 5 - Отключение входа по паролю

Теперь, когда вы можете использовать SSH-ключи для входа на сервер, мы можем ещё больше обезопасить сервер путём отключения аутентификации по паролю. В результате этого осуществлять доступ к серверу по SSH можно будет только с использованием вашего открытого ключа. Иными словами, для входа на ваш сервер (за исключением использования консоли), вам необходимо будет иметь закрытый ключ, парный открытому ключу, установленному на сервере.

**Внимание:**   
Отключайте аутентификацию по паролю только в том случае, если вы установили открытый ключ так, как описано на шаге 4. В противном случае вы можете потерять доступ к вашему серверу.

Для отключения аутентификации по паролю выполните описанные ниже шаги.

Начните с открытия конфигурационного файла в текстовом редакторе под пользователем `root` или пользователем с правами `sudo`:

    sudo nano /etc/ssh/sshd_config

Найдите строку с `PasswordAuthentication`, раскомментируйте её, удалив `#` в начале строки, затем измените значение на `no`. Теперь строка должна выглядеть вот так:

sshd\_config - Отключение входа по паролю

    PasswordAuthentication no

В этом же файле две другие настройки, необходимые для отключения аутентификации по паролю, уже имеют корректные настройки по умолчанию, _не изменяйте_ эти настройки, если вы ранее не изменяли этот файл:

sshd\_config - Важные настройки по умолчанию

    PubkeyAuthentication yes
    ChallengeResponseAuthentication no

После внесения изменений в этот файл, сохраните и закройте его (`CTRL-X`, затем `Y`, далее `ENTER`)

Перезапустите демон SSH:

    sudo systemctl reload sshd

Теперь аутентификация по паролю отключена. Ваш сервер доступен для входа через SSH только с помощью ключа.

## Шаг 6 - Тестовый вход на сервер

Теперь, перед тем, как выйти с сервера, проверьте свои настройки. Не отключайтесь от сервера до тех пор, пока не убедитесь, что можете зайти на него через SSH.

В новом окне Терминала на вашей **локальной машине** войдите на сервер, используя созданный нами ранее аккаунт. Для этого введите команду (заменяя выделенное вашим именем пользователя и IP адресом):

    ssh sammy@IP_адрес_вашего_сервера

Если вы настроили аутентификацию с использованием открытого ключа для вашего пользователя, следуя инструкциям на шаге 4 и 5, для входа будет использован ваш закрытый ключ.

**Внимание:**   
Если вы создали пару ключей с использованием кодовой фразы, сервер запросит эту кодовую фразу в процессе входа. Если вы не задавали эту кодовую фразу, вы войдёте на сервер автоматически.

После предоставления аутентификационных данных, вы войдёте на сервер с помощью учётной записи вашего нового пользователя.

Если вам необходимо выполнить команду с привилегиями `root` наберите “sudo” перед командой:

    sudo ваша_команда

## Шаг 7 - Настройка базового файрвола

Серверы на Ubuntu 16.04 могут использовать файрвол UFW для разрешения соединений избранных сервисов. Мы легко можем настроить этот базовый файрвол.

Различные приложения могут создавать свои профили для UFW при установке. Эти профили позволяют UFW управлять этими приложениями по их именам. Сервис OpenSSH, который мы используем для соединения с сервером, также имеет свой профиль в UFW.

Убедиться в этом можно выполнив команду:

    sudo ufw app list

    ВыводAvailable applications:
      OpenSSH

Нам необходимо убедиться, что файрвол разрешает SSH-соединения, и мы сможем зайти на сервер в следующий раз. Мы можем разрешить SSH-соединения следующей командой:

    sudo ufw allow OpenSSH

Далее включим файрвол командой:

    sudo ufw enable

Наберите `y` и нажмите `ENTER` для продолжения. Вы можете убедиться, что SSH-соединения разрешены, следующей командой:

    sudo ufw status

    ВыводStatus: active
    
    To Action From
    -- ------ ----
    OpenSSH ALLOW Anywhere
    OpenSSH (v6) ALLOW Anywhere (v6)

Если вы устанавливаете и настраиваете дополнительные приложения и сервисы, вам будет необходимо настроить файрвол для разрешения входящего трафика для этих приложений. Вы можете ознакомиться с наиболее распространёнными операциями с UFW в [этой статье](ufw-essentials-common-firewall-rules-and-commands).

## Что дальше?

Теперь у вас есть хорошо настроенный сервер. Далее вы можете устанавливать на него любое необходимое программное обеспечение.