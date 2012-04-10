Инструкция по установки <a href="http://gitlabhq.com/">GitLab 2.x</a>
================================

Данная инструкция описывает процесс установки <a href="http://gitlabhq.com/">GitLab</a> с использованием Gitolite на систему Centos 6 x86_64. Все нижеописанное может без проблем работать на RedHat 5-6, Centos 5, а также Fedora. Предполагается что все действия выполняются из под root, в противном случае используйте sudo перед каждой командой (нужен пользователь с правами).

1 Установка базовых пакетов
----------------------------
Для начала установим базовые пакеты, которые потребуются в дальнейшем.

	yum install make openssh-clients gcc libxml2 libxml2-devel libxslt libxslt-devel python-devel git

2 Установка Ruby
-----------------
### 2.1 RVM (рекомендуемый)
RVM — Ruby Version Manager. Программа для управления версиями Ruby. Подробнее: http://beginrescueend.com/rvm/

Установим RVM:

	curl -L get.rvm.io | bash -s stable

Обновим среду shell:

	source ~/.bash_profile

После этого откройте новое окно терминала.

Найти требования (выполните нижеследующею инструкцию):

	rvm requirements

Тут нас попросят установить список зависимостей. Copy/Past в помощь.

Установим Rubby 1.9.2:

	rvm install 1.9.2

Зададим установленную версию как версию по умолчанию:

	rvm use ruby 1.9.2 --default

Полезные команды:

	rvm list        # Посмотреть список установленных версий
	rvm list known  # Посмотреть список установленных версий

Работа с gemset (gem наборы):

	rvm gemset list                 # список наборов
	rvm use 1.9.2@gemset2 --create  # создать набор для Ruby 1.9.2
	rvm use 1.9.2@gemset2 --default # использовать gemset2 как набор по умолчанию для Ruby 1.9.2



### 2.2 RPM
Дело в том, что в репозиториях можно найти Ruby версии 1.8.7.35 (проверить можно при помощи команды yum info ruby) а для работы GitLab требуется версия 1.9.2. Можно собрать из исходников, я же предпочел скомпилировать RPM-пакет.

Установим нужные инструменты

	yum install -y readline-devel ncurses-devel gdbm-devel glibc-devel tcl-devel openssl-devel db4-devel byacc

Установим инструменты для сборки RPM

	yum install -y rpm-build rpmdevtools

Создадим дерево при помощи утилиты rpmdev-setuptree

	cd ~
	rpmdev-setuptree

Получим вот такую структуру

	|~
	   |-rpmbuild
	      |---BUILD
	      |---RPMS
	      |---SOURCES
	      |---SPECS
	      |---SRPMS

Переходим в директорию спецификаций и скачиваем

	cd ~/rpmbuild/SPECS
	curl https://raw.github.com/imeyer/ruby-1.9.2-rpm/master/ruby19.spec > ruby19.spec

Проверим версию Ruby запустив команду:

	cat ruby19.spec

Нам нужны две первые строки, старшая и младшая версия, будет что-то типа:

	%define rubyver         1.9.2
	%define rubyminorver    p290

Скачиваем нужный нам исходник:

	wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-p290.tar.gz > ~/rpmbuild/SOURCES

Приступаем к сборке RPM пакета. Запускаем:

	rpmbuild -bb ruby19.spec

и идем наливать чай, кофэ, курить и т.д.

Ну что ж теперь после долгого ожидания можно ставить руби.

	rpm -Uhv ~/rpmbuild/RPMS/x86_64/ruby-1.9.2p290-2.el6.x86_64.rpm

Это в моем случае, если у вас другая система то смотрите ~/rpmbuild/RPMS.

3 Установка дополнительных инструментов Ruby
---------------------------------------------
Обновим для начала gem

	gem update --system

Дабы не устанавливать документацию при установке гемов и тем самым ускорив в разы их установку, создадим файла:

	vi ~/.gemrc

и пропишем туда

	install: --no-rdoc --no-ri
	update:  --no-rdoc --no-ri

Далее ставим rails:

	gem install rails --include-dependencies

4 Подготовительная стадия
-------------------------
GitLab у меня крутиться под пользователем **gitlab**. Создадим данного пользователя и его директорию:

	adduser --shell /bin/bash --create-home --home-dir /home/gitlab gitlab

Переключимся на созданного пользователя и создадим пару ключей которая будет использована для администрирования gitolite

	su gitlab
	ssh-keygen -t rsa
	su

5 Устанавливаем Gitolite
-------------------------

	yum install gitolite

Инсталляция создаст юзера **gitolite** и с домашним каталогом */var/lib/gitolite*. Но мы его не будем использовать.
Вместо этого создадим системного пользователя **git** с домашним каталогом в */home/git*

	adduser --system --shell /bin/sh --comment 'gitolite' --create-home --home-dir /home/git git

Скопируем публичный ключ созданный в предыдущем шаге:

	cp /home/gitlab/.ssh/id_rsa.pub /home/git/gitlab.pub

Переключимся на юзера **git** и инициализируем репозиторий с данным ключом:

	su git
	gl-setup ~/gitlab.pub

После нажатия на "enter", о котором нас попросит процесс установки, откроется файл настроек для редактирования. Единственное что нужно тут поменять это $REPO_UMASK на 0007

	$REPO_UMASK = 0007;

Жмем *i* меняем **ESC** и *:wq*
На этом все, репозиторий создан. Можно переключаться обратно на root.

	su

Присвоим пользователю **gitlab** группу **git** чтобы он имел доступ к репозиториям.

	usermod -a -G git gitlab

[RVM Ruby install] Присвоим пользователю **gitlab** группу **rvm** чтобы on мог запускать Ruby.

	usermod -a -G rvm gitlab

Назначим полные права группе для репозиториев

	chmod -R g+rwX /home/git/repositories/

А также права на чтение для каталога */home/git*. Иначе **gitlab** не сможет читать из */home/git/repositories/*.

	chmod g+r /home/git

Подключимся из под **gitlab** к git чтобы проверить доступ и сохранить настройки:

	su gitlab
	ssh git@localhost

После ответа на вопросы, получим вывод:

	PTY allocation request failed on channel 0
	hello gitlab, this is gitolite v2.2-13-gf0d712e running on git 1.7.1
	the gitolite config gives you the following access:
	     R   W 	gitolite-admin
	    @R_ @W_	testing
	Connection to localhost closed.

Переключимся обратно на root

	su

Все ок, теперь можно приступать к установке GitLab.


6 Дополнительные инструменты
-----------------------------

Ставим python-pip. В репах стара версия поэтому собираем вручную как сказано в доках

	curl http://python-distribute.org/distribute_setup.py | python
	easy_install pip

Далее нам надо sqlite и sqlite-devel. Первый скорее всего установлен.

	yum install sqlite sqlite-devel -y

Ставим Redis. Доступен в репозиториях [CentAlt](http://centos.alt.ru/)

	yum install redis -y
	chkconfig --levels 235 redis on
	service redis start

Еще нам потребуется gcc-c++ компилятор.

	yum install gcc-c++

Устанавливаем python pygments

	pip install pygments

Установим bundler, который в дальнейшем выполнит всю установку gitlab

	gem install bundler


7 Установка GitLab
-------------------
Переключимся юзера gitlab и перейдем в его каталог

	su gitlab
	cd ~

Клонируем gitlabhq в текущий каталог

	git clone git://github.com/gitlabhq/gitlabhq.git

перейдем в каталог

	cd gitlabhq

**[RPM Ruby install]** Тут походу требуются права root, не знаю может и сработало бы и из под root, я не стал заморачиваться и просто добавил пользователя gitlab в sudoers
Из под root	

	su
	vi /etc/sudoers
	# добавить строчку
	gitlab    ALL=(ALL)       ALL

Отказываемся от установки документации как было сказано выше:

	su gitlab
	vi ~/.gemrc
	install: --no-rdoc --no-ri 
	update:  --no-rdoc --no-ri

Запускаем установку GitLab

	su gitlab
	bundle install

**[Error install: charlock_holmes]** Если вываливается ошибка о невозможности установки CharlockHolmes, то:

  su
  yum install icu libicu libicu-devel
  su gitlab
  bundle install

**[Error install: mysql2]** Если вываливается ошибка о невозможности установки mysql2, то:

	su
	yum install mysql-devel
	su gitlab
	bundle install

**[Error install: sqlite3]** Это потому что старая версия sqlite. gem sqlite3 нуждается в версии 3.6.16+. Характерно для старых ОС Centos 5, RHEL 5. Проверьте установленный sqlite:

	yum info sqlite

Sqlite 3.6.20 доступен в тестовых репозиториях ATrpms. Пример установки для Centos 5/RHEL5 x86_64

	su
	rpm --import http://packages.atrpms.net/RPM-GPG-KEY.atrpms
	rpm -Uvh http://dl.atrpms.net/all/atrpms-repo-5-5.el5.x86_64.rpm
	yum --enablerepo=atrpms-testing update sqlite sqlite-devel

После завершения установки получим:

	Your bundle is complete! Use `bundle show [gemname]` to see where a bundled gem is installed.

8 Настройка
------------
Скопируем конфигурацию

	cp config/gitlab.yml.example config/gitlab.yml

Смотрим файл *config/gitlab.yml*

	vi config/gitlab.yml

Тут нам надо настроить доступ к gitolite. По умолчанию там такие настройки:

	git_host:
	  system: gitolite
	  admin_uri: git@localhost:gitolite-admin
	  base_path: /home/git/repositories/
	  host: localhost
	  git_user: git
	  # port: 22

В принципе если делали как было указано выше то ничего менять не надо.

###Настройка базы данных:
####Sqite

	cp config/database.yml.sqlite config/database.yml

####MySQL

	sudo -u gitlab cp config/database.yml.example config/database.yml

Отредактируйте настройки:

	vi config/database.yml

Создаем базы данных

	RAILS_ENV=production rake db:setup
	RAILS_ENV=production rake db:seed_fu

9 Запуск
---------

	thin start -p 3000 -e production

Дождемся запуска, об этом даст знать строчка:

	Listening on 0.0.0.0:3000, CTRL+C to stop

Теперь можно открыть браузер и попробовать зайти на http://your_host:3000
Если не получается зайти, то проверяем iptables.

10 Nginx + thin
---------------
Знаю что есть passenger и passenger-install-nginx-module но насколько я понял там собирается свой nginx еще и древней версии что меня не очень устраивает.
Запустить данную схему мне пока не удалось, точнее я ее настроил но запускается через раз.
Вот инструкция (подразумевается что nginx уже установлен):
из под root

	thin install
	cp /etc/rc.d/thin /etc/init.d

создаем файл настроек бэкенда /etc/thin/gitlab.yml

	vi /etc/thin/gitlab.yml

и пишем такой вот конфиг

	user: gitlab
	group: gitlab
	chdir: /home/gitlab/gitlabhq/
	log: log/gitlabhq.thin.log
	socket: /tmp/gitlab.sock
	pid: tmp/pids/gitlab.pid

	environment: production
	timeout: 30
	max_conns: 1024

	max_persistent_conns: 512
	no-epoll: true
	servers: 1
	daemonize: 1

**[RVM Ruby install]**

	rvm wrapper 1.9.2 bootup thin
	# если меняли gemset, то:
	rvm wrapper 1.9.2@<gemset> bootup thin

Правим скрипт запуска:

	vi /etc/init.d/thin

	- DAEMON=/usr/local/rvm/gems/ruby-1.9.2-p318/bin/thin
	+ DAEMON=/usr/local/bin/bootup_thin

Пробуем запустить:

	service thin start
	
Ошибки и прочее вылавливаем в ~/gitlabhq/log/gitlabhq.thin.log

В общем у меня тут проблемы в том что сокет создается через раз. Может это связано с последним коммитом автора, так как на рабочем сервере работает такая связка и никаких проблем.

**[UPD]** В последний раз когда ставил все заработало нормально. Вот только thin не может удалить pid файл при остановке. Найти решение данной проблемы не удалось. Думаю это баг в программе.
Пришлось заюзать костыль в /etc/init.d/thin, после остановки сервера добавил

  rm -f /home/gitlab/gitlabhq/tmp/pids/gitlab.*

Настройка nginx. Создаем файл /etc/nginx/conf.d/gitlab.conf

	upstream thin_cluster {
	    server unix:/tmp/gitlab.0.sock;
	    #server unix:/tmp/gitlab.1.sock
	}
	
	server {
	    listen       80;
	    server_name  gitlab.your_host; #!!!
	
	    access_log  /var/log/nginx/gitlab-proxy-access;
	    error_log   /var/log/nginx/gitlab-proxy-error;
	
		proxy_set_header   Host $http_host;                                                                                                                     
		proxy_set_header   X-Real-IP $remote_addr;                                                                                                                   
		proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header   X-Forwarded-Proto $scheme;
		
		client_max_body_size       10m;
		client_body_buffer_size    128k;
		
		proxy_connect_timeout      90;
		proxy_send_timeout         90;
		proxy_read_timeout         90;
		
		proxy_buffer_size          4k;
		proxy_buffers              4 32k;
		proxy_busy_buffers_size    64k;
		proxy_temp_file_write_size 64k;
	    
	    
	    
	    root /home/gitlab/gitlabhq/public;
	    proxy_redirect off;
	
	    location / {
	        try_files $uri/index.html $uri.html $uri @cluster;
	    }
	
	    location @cluster {
	        proxy_pass http://thin_cluster;
	    }
	}

Перезапускаем nginx.

	service ngix reload

Если все ok, то можно добавить в автозагрузку:

	chkconfig --levels 235 thin on
