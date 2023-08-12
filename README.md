<pre>
Урок 24 - Сбор и анализ логов.
Цель занятия -
	понять зачем нужны логи и как их собирать;
	освоить принципы логирование с помощью rsyslog;
	понять возможности logrotate для задач ротации логов;
	научиться работать с journald и auditd;
	узнать про adrtd и kdump.
Домашнее задание - Научится проектировать централизованный сбор логов. Рассмотреть особенности 
разных платформ для сбора логов.
Результат выполнения - полученные навыки по основам работы с rsyslog, logrotate, journald, auditd, abrtd и kdump.
Ага, приступаем к выполнению.
1. БОльшую часть работы по синхронизации времени мы, с вами, выполняем в Vagrantfile.
Объясняю - по умолчанию время на серверах устанавливается на UTC+0, а, нам надо, что бы было наше,
родное, Красноярское. Ручной метод, описанный в методичке я проверял - тоже работает.
2. Теперь, просто заходим по очереди на Веб сервер и Log сервер и проверяем время - одинаковое, Красноярское.
	vagrant ssh web
	systemctl status chronyd
	timedatectl
	exit
	vagrant ssh log
	systemctl status chronyd
	systemctl restart chronyd
	timedatectl
	exit
	Ну, вот, отлично!
3. Далее по методичке устанавливаем самую свежую версию nginx на виртуальной машине web -
	vagrant ssh web
	Устанавливаем пакеты, необходимые для подключения yum-репозитория:
	sudo yum install yum-utils -y
	Для подключения yum-репозитория создаем файл /etc/yum.repos.d/nginx.repo для скачивания самой свежей версии -
	cat > /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

	yum install epel-release -y		// А, этот epel-release не ставится, что-ли, по дефолту?
	yum install -y nginx
	Пока все идет хорошо и без ошибок!
4. Запускаем на web сервере наш nginx,буть он не ладен - 
	systemctl start nginx
	systemctl status nginx
	nginx -v
	ss -tln | grep 80
	exit
	exit
5. Пробуем в браузере выполнить вход на созданный сайт - http://192.168.56.10. Да, работает, но без картинок!!!
	Заново пересобираем ВМ и ставим nginx стандартной командой - 
	yum install epel-release -y
	yum install -y nginx
	Все, картинка появилась.
6. Переходим на сервер Log - 
	vagrant ssh log
	sudo -i
7. Проверяем - yum list rsyslog.
	Выхлоп команды systemctl status rsyslog мне больше понравился.
8. Открываем через mc файл - /etc/rsyslog.conf. Исправляем нужные строки. Хотя, можно было раскомментировать
	те, которые там были, но они немного отличаются.
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")
# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
	В конец файла добавляем -
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~ // Это что за символы? В методичке правильно написано?! Хотя nginx -t не выдал никаких ошибок. Но, и без них все проходит нормально!
9. Перезапускаемся - 
	systemctl restart rsyslog
	systemctl status rsyslog
10. Теперь проверяем открытые порты - 
	ss -tuln
	exit
	exit
11. Заходим, значит, в web сервер -
	vagrant ssh web
	sudo -i
12. Проверяем версию nginx: 
	rpm -qa | grep nginx
13. Добавляем в файл /etc/nginx/nginx.conf, после строки - error_log /var/log/nginx/error.log;
	error_log syslog:server=192.168.56.15:514,tag=nginx_error;
	access_log syslog:server=192.168.56.15:514,tag=nginx_access,severity=info combined;
	И, всё! Впадаем в состояние бессмысленого копипаста и переписывания директивы access_log в течении двух дней.
	После совета с преподавателем и Интернетом понимаю, что она (в смысле - директива) применяется в контексте hhtp.
	Нахожу его и прописываю там. Вот, теперь все отлично!
14. Проверяем - 
	nginx -t
	systemctl restart nginx
</pre>
15. В Firefox ([смотрим](page.jpg)) - Вау!!! Какая красатищща!!!
<pre>
16. Специально делаем ашибку для web-страницы - 
	rm /usr/share/nginx/html/img/header-background.png
	rm /usr/share/nginx/html/img/centos-logo.png
</pre>
	Очищаем историю браузера и ([видим](page_error.jpg)), что картинки как не бывало. Блин!!!  
17. Далее заходим на log-сервер и смотрим информацию об nginx:  
	([cat /var/log/rsyslog/web/nginx_access.log](nginx_access.log))  
	([cat /var/log/rsyslog/web/nginx_error.log](nginx_access.log))  
<pre>
	Да, натворили мы с вами дел!!! А копии не судьба было сделать? Как теперь восстанавливать?
18. Настройка аудита, контролирующего изменения конфигурации nginx.
19. Проверяем установленную версию пакета аудита -
	rpm -qa | grep audit
20. В конец файла /etc/audit/rules.d/audit.rules добавим следующие строки:
	-w /etc/nginx/nginx.conf -р wa -k nginx_conf
	-w /etc/nginx/default.d/ -p wa -k nginx_conf
21. Перезапускаем службу auditd:
	service auditd restart
22. Сотрем какую нибудь лишнюю строчку в файле /etc/nginx/nginx.conf.
23. Смотрим изменения - ausearch -f /etc/nginx/nginx.conf.
24. Далее настроим пересылку логов на удаленный сервер.
25. Установим пакет audispd-plugins:
	yum -y install audispd-plugins
26. Найдем и поменяем следующие строки в файле /etc/audit/auditd.conf:
	log_format = RAW
	name_format = HOSTNAME
27. В  файле /etc/audisp/plugins.d/au-remote.conf поменяем параметр active на yes.
28. В  файле /etc/audisp/audisp-remote.conf требуется указать адрес сервера и порт, 
	на который будут отправляться логи:
	[root@web ~]# cat /etc/audisp/audisp-remote.conf
#                                                                                                                                                                            
# This file controls the configuration of the audit remote                                                                                                                   
# logging subsystem, audisp-remote.                                                                                                                                          
#                                                                                                                                                                            
                                                                                                                                                                             
remote_server = 192.168.56.15
port = 60
##local_port =                                                                                                                                                               
transport = tcp
queue_file = /var/spool/audit/remote.log
29. Далее перезапускаем службу auditd:
	service auditd restart
30. Далее настроим Log-сервер.
	Откроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf:
	tcp_listen_port = 60
31. Перезапустим службу auditd:
	service auditd restart
32. Смотрим информацию об изменении атрибута:
	[root@web ~]# ls -l /etc/nginx/nginx.conf
	-rw-r--r—. 1 root root 1487 Dec 14 14:41 /etc/nginx/nginx.conf
	[root@web ~]# chmod +x /etc/nginx/nginx.conf
	[root@web ~]# ls -l /etc/nginx/nginx.conf
	-rwxr-xr-x. 1 root root 1487 Dec 14 14:41 /etc/nginx/nginx.conf - цвет названия поменялся на зеленый!!!
33. Выполняем на log-сервере -
	[root@log ~]# grep web /var/log/audit/audit.log
34. Ну, вот, все. Все шаги прошли без запинки. Теперь самое интересное. 
	Помимо базового задания рекомендуется сделать данное задание следующим образом:
	Все настройки сделать с помощью Ansible и добавить запуск Ansible playbook из Vagrantfile.
	Для запуска playbook по команде vagrant up достаточно добавить следующую конструкцию в раздел Boxes.
	Вроде на 15 занятии это делали, но там был один сервер и все операции выполнить одинаково
	для всех ВМ. А здесь два сервера и разные настройки для каждого!!!
</pre>
35. Создаем инвентори файл ([hosts](hosts)) и ([epel.yml](epel.yml))
<pre>
epel.yml:
---
- name: Обработка команд для всех виртуальных машин
  hosts: all
  become: true
  tasks:
    - name: Обновление пакетов
      yum:
        name: "*"
        state: latest
      register: yum_update
    - name: Настройка часового пояса
      timezone:
#        name: Europe/Moscow
        name: Asia/Krasnoyarsk
    - name: Установка пакета nginx
      yum:
        name: nginx
        state: present
      when: "'web' in inventory_hostname"
    - name: Выполнение двух действий
#      become: true
      shell:
        cmd: |
          yum install mc -y
          yum install tzdata -y
#      when: "'log' in inventory_hostname"
script --timing=time_loading_log loading.log
36. Выполняем по очереди команды -
	ansible-playbook epel.yml hosts --ask-become-pass
	ansible all -a "df -h" -u root
	ansible all -a "timedatectl" -u root
	ansible all -m yum -a "name=epel-release state=absent" -b
	ansible all -i hosts -m ping
	ansible all -m yum -a "name=nginx" -b
	ansible all -m yum -a "name=tzdata" -b
	ansible all -m setup -a 'filter=ansible_os_family'
	Что-то работает, что-то нет! Еще не во всем разобрался.
	Возможное решение: Прописать в файле YML процесс ожидания, пока другой процесс завершит свою работу с файлом блокировки,
	а затем повторить выполнение команды.
37. Идея Ansible понятна. Выполнить можно все, главное разобраться тщательнее!!!
38. Все.
</pre>
