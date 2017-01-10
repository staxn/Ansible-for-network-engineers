# Основы playbooks

Playbook (файл сценариев) — это файл в котором описываются действия, которые нужно выполнить на какой-то группе хостов.

Внутри playbook:
* play - это набор задач, которые нужно выполнить для группы хостов
* task - это конкретная задача. В задаче есть, как минимум:
 * описание (название задачи можно не писать, но очень рекомендуется)
 * модуль и команда (действие в модуле)


## Синтаксис playbook

Playbook описываются в формате YAML.

Поэтому, если вы забыли синтаксис YAML, его можно повторить в [разделе YAML](book/chapter10/3_yaml.md) или в [документации Ansible](http://docs.ansible.com/ansible/YAMLSyntax.html).


### Пример синтаксиса playbook

Посмотрим на простой пример plabook (файл 1_show_commands_with_raw.yml):
```
---

- name: Run show commands on routers
  hosts: cisco-routers
  gather_facts: false

  tasks:

    - name: run sh ip int br        
      raw: sh ip int br | ex unass

    - name: run sh ip route
      raw: sh ip route


- name: Run show commands on switches
  hosts: cisco-switches
  gather_facts: false

  tasks:

    - name: run sh int status
      raw: sh int status

    - name: run sh vlan
      raw: show vlan
```


В этом playbook у нас два сценария (play). Разберемся с первым:
* ```name: Run show commands on routers``` - это имя сценария (play). Этот параметр обязательно должен быть в любом сценарии
* ```hosts: cisco-routers``` - сценарий будет применяться к устройствам в группе cisco-routers
 * тут может быть указано и несколько групп, например, таким образом: ```hosts: cisco-routers:cisco-switches```. Подробнее, в [документации](http://docs.ansible.com/ansible/intro_patterns.html)
* обычно в play надо указывать параметр __remote_user__. Но, так как мы указали его в конфигурационном файле Ansible, мы можем пропустить его в play.
* ```gather_facts: false``` - в данном случае, нам нужно отключить сбор фактов (информации) об устройстве, так как для сетевого оборудования надо использовать отдельные модули для сбора фактов.
 * в разделе [конфигурационный файл](book/chapter15/2_configuration.md) мы рассматривали как отключить сбор фактов по умолчанию. Если вы его отключили в конфигурационном файле, то параметр gather_facts в play не нужно указывать.
* ```tasks:``` - дальше идет перечень задач
 * в каждой задаче настроено имя (опционально) и действие. Действие может быть только одно.
 * в действии мы указываем какой модуль использовать и параметры модуля.

И тот же playbook с отображением элементов:

![Ansible playbook](https://raw.githubusercontent.com/natenka/Ansible-for-network-engineers/master/images/playbook.png)

Теперь попробуем запустить playbook:
```
$ ansible-playbook 1_show_commands_with_raw.yml
```

![Ansible playbook](https://raw.githubusercontent.com/natenka/Ansible-for-network-engineers/master/images/playbook_execution.png)

> **Note** Обратите внимание, что для запуска playbook используется другая команда. Для ad-hoc команды, мы использовали команду ansible. А для playbook - ansible-playbook.

Для того, чтобы убедится, что команды, которые мы указывали в задачах, выполнились на устройствах, запустите playbook с опцией -v (вывод сокращен):
```
$ ansible-playbook 1_show_commands_with_raw.yml -v
```

[Verbose playbook](https://raw.githubusercontent.com/natenka/Ansible-for-network-engineers/master/images/playbook-verbose.png)

В следующих разделах мы научимся отображать эти данные в нормальном формате и посмотрим, что с ними можно делать.

### Порядок выполнения задач и сценариев

Сценарии (play) и задачи (task) выполняются последовательно, в том порядке, в котором они описаны в playbook.

Если в сценарии, например, две задачи, то сначала первая задача должна быть выполнена для всех устройств, которые указаны в параметре hosts. Только после того, как первая задача была выполнена для всех хостов, начинается выполнение второй задачи.

Если в ходе выполнения playbook, возникла ошибка в задаче на каком-то устройстве, это устройство исключается, и другие задачи на нем выполняться не будут.

Например, заменим пароль пользователя cisco на cisco123 (правильный cisco) на маршрутизаторе 192.168.100.1, и запустим playbook заново:
```
$ ansible-playbook 1_show_commands_with_raw.yml
```

![Ansible playbook](https://raw.githubusercontent.com/natenka/Ansible-for-network-engineers/master/images/playbook_failed_execution.png)

Обратите внимание на ошибку в выполнении первой задачи для маршрутизатора 192.168.100.1.

Во второй задаче 'TASK [run sh ip route]', Ansible уже исключил маршрутизатор и выполняет задачу только для маршрутизаторов 192.168.100.2 и 192.168.100.3.


Еще один важный аспект - Ansible выдал нам сообщение:
```
to retry, use: --limit @/home/nata/pyneng_course/chapter15/1_show_commands_with_raw.retry
```

Если, при выполнении playbook, на каком-то устройстве возникла ошибка, Ansible создает специальный файл, который называется точно так же как playbook, но расширение меняется на retry.
(Если вы выполняете задания паралельно, то этот файл должен появится у вас.)

В этом файле хранится имя или адрес устройства на котором возникла ошибка.
Так выглядит файл 1_show_commands_with_raw.retry сейчас:
```
192.168.100.1
```

Создается этот файл для того, чтобы мы могли перезапустить playbook заново только для проблемного устройства (устройств).
То есть, мы исправляем проблему с устройством, и заново запускаем playbook.

В нашем случае, нам нужно опять настроить правильный пароль на маршрутизаторе 192.168.100.1, а затем перезапустить playbook таким образом:
```
$ ansible-playbook 1_show_commands_with_raw.yml --limit @/home/nata/pyneng_course/chapter15/1_show_commands_with_raw.retry
```

![Ansible playbook](https://raw.githubusercontent.com/natenka/Ansible-for-network-engineers/master/images/playbook-retry.png)

Ansible взял список устройств, которые перечислены в файле retry и выполнил playbook только для них.

Можно было запустить playbook и так (то есть, писать не полный путь к файлу retry):
```
$ ansible-playbook 1_show_commands_with_raw.yml --limit @1_show_commands_with_raw.retry
```

Параметр --limit очень полезная вещь. Он позволяет ограничивать, для каких хостов или групп будет выполняться playbook, при этом, не меняя сам playbook.

Мы могли бы запустить playbook для только маршрутизатора 192.168.100.1, таким образом:
```
$ ansible-playbook 1_show_commands_with_raw.yml --limit 192.168.100.1
```

Мы поговорим о параметре limit и других полезных параметрах Ansible в отдельном разделе.

### Идемпотентность

Модули Ansible идемпотентны. Это означает, что мы можем выполнять модуль (задачу) сколько угодно раз, но при этом модуль будет выполнять изменения, только если система не находится в желаемом состоянии.

Это дает возможность запускать playbook сколько угодно раз.

Но, есть исключения из такого поведения.
Например, модуль raw всегда вносит изменения.
Поэтому, когда мы смотрели на выполнение playbook выше, мы всегда видели состояние changed.

Но, если, например, в задаче указано, что на сервер Linux надо установить пакет httpd, то он будет установлен только в том случае, если его нет.
То есть, действие не будет повторяться снова и снова, при каждом запуске.
А лишь тогда, когда пакета нет.

Также, не всё так просто с сетевыми оборудованием. Например, когда мы отправляем команды show, идемпотентность не актуальна, так как эта команда не меняет поведение системы.
