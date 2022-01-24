Стенд собирается в автоматическом режиме - vagrant + script.

1. Создание своего RPM
- Установить пакеты: ```yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils openssl-devel zlib-devel pcre-devel gcc libtool perl-core openssl mc lynx```
- Скачать nginx-1.14.1-1.el7_4.ngx.src.rpm ```wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm```
- Распаковать ```rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm```
- Скачать openssl-1.1.1m.tar.gz  ```wget --no-check-certificate https://www.openssl.org/source/openssl-1.1.1m.tar.gz```
- Разархивировать  ```tar -xvf openssl-1.1.1m.tar.gz```
- Поставим все зависимости чтобы в процессе сборки не было ошибок ```yum-builddep rpmbuild/SPECS/nginx.spec```
- Перенос openssl-1.1.1m в каталог rpmbuild  ```mv /root/openssl-1.1.1m.tar.gz /root/rpmbuild/```

В папке SPECS лежит spec-файл, поправим его чтобы NGINX собирался с необходимыми нам опциями.
- Открываем файл ```nano SPECS/nginx.spec``` и добавляем в секцию %build необходимый нам модуль OpenSSL --with-openssl=/root/rpmbuild/openssl-1.1.1m:

```
%build
./configure %{BASE_CONFIGURE_ARGS} \
    --with-cc-opt="%{WITH_CC_OPT}" \
    --with-ld-opt="%{WITH_LD_OPT}" \
    --with-openssl=/root/rpmbuild/openssl-1.1.1m
```
- Устанавливаем зависимости  ```yum-builddep SPECS/nginx.spec```
- Собираем  ```rpmbuild -bb SPECS/nginx.spec```
- Видим два собранных пакета:  ```ll rpmbuild/RPMS/x86_64/```
[root@RPM]#  ll rpmbuild/RPMS/x86_64/
total 4392
-rw-r--r--. 1 root root 2006488 Jan 24 10:10 nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2489408 Jan 24 10:11 nginx-debuginfo-1.14.1-1.el7_4.ngx.x86_64.rpm
```
- Устанавливаем rpm пакет: ```yum localinstall -y RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm```
- Запускаем nginx  ```systemctl start nginx``` 
- Проверяем статус  ```systemctl status nginx```


2. Создаем свой репозиторий
- Создаем папку repo ```mkdir /usr/share/nginx/html/repo```
- Копируем наш скомпилированный пакет nginx в папку с будущим репозиторием  ```cp rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/```
- Скачиваем дополнительно пакет  ```wget https://downloads.percona.com/downloads/percona-release/percona-release-0.1-6/redhat/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm```
- Создаем репозиторий ```createrepo /usr/share/nginx/html/repo/``` и ```createrepo --update /usr/share/nginx/html/repo/```
- В location / в файле /etc/nginx/conf.d/default.conf необходимо добавить autoindex on.
```
location / {
root /usr/share/nginx/html;
index index.html index.htm;
autoindex on;
}
```
- Проверяем синтаксис ```nginx -t``` , ```nginx -s reload```
- Просмотрим наши пакеты через HTTP  ```lynx http://localhost/repo/``` , ```curl -a http://localhost/repo/```
- Для теста создаем файл ```touch otus.repo``` в ``` /etc/yum.repos.d```  и внесем в него:
```
cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```

- Можем посмотреть подключенный репозиторий ```yum list --showduplicates | grep otus```
```
[root@RPM]# yum list --showduplicates | grep otus
nginx.x86_64                                1:1.14.1-1.el7_4.ngx       otus
percona-release.noarch                      0.1-6                      otus
[root@RPM]# yum list | grep otus
percona-release.noarch                      0.1-6                      otus
```