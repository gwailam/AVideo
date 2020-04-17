使用AVideo搭建全功能视频播放分享网站
BY 香菇肥牛 · PUBLISHED 04/16/2020 · UPDATED 04/16/2020

AVideo是一套免费开源的多功能视频播放分享网站套件，包含用户注册、视频上传、Youtube下载、转码播放、分享分类、评论社交、收费点播、广告投放、点赞订阅、频道收藏等等完备的视频网站功能，甚至可以用来做直播。AVideo的前身是YouPHPTube, 顾名思义，就是仿照Youtube开发的程序，而从功能及易用性的角度来看，使用这个套件搭建的网站完全可以媲美Youtube.  今天我们就来介绍怎样使用这款程序来搭建一个全功能的视频播放分享网站。本文作者为香菇肥牛，本文永久链接为https://qing.su/article/145.html, 如需转载请注明出处，谢谢！

1. 软硬件环境要求
由于是搭建视频分享网站，有许多的转码和视频储存需求，因此AVideo对于服务器性能的要求（尤其是CPU和硬盘）比较高。如果仅仅是个人的视频分享，建议独立双核CPU, 4 GB以上内存，100 GB以上的硬盘。如果需要开放给其他人使用，建议考虑更强劲的VPS系统或者独立服务器。

操作系统只要是普通的Linux 64 bit发行版一般都可以。该操作系统基于传统的服务器 + PHP + 数据库模型，服务器软件原生支持Apache和Litespeed (包括OLS), 没有Nginx的原生支持，但是可以更改配置文件实现基于Nginx的搭建。PHP需要7.1及以上的版本。数据库软件没有特别的要求，MySQL和MariaDB都可以。

在开始搭建之前，请将您的域名A记录解析到您用来搭建视频网站的服务器IP地址上。

本文将采用全新安装的Ubuntu 18.04 LTS 64 bit操作系统，使用Apache2, PHP 7.2, 以及MariaDB从零开始搭建这个全功能视频播放分享网站。本文的所有操作均在一台Online 3O独立服务器上。现在，我们开始教程，请用root账户登录您的服务器，或者使用sudo命令。

2. 搭建服务器环境
首先我们来搭建基础的LAMP环境。

2.1 搭建Apache2服务器
为了方便，我们直接从源安装Apache2.

1
2
apt-get update && apt-get upgrade
apt-get install apache2 wget git sudo tar
编辑网站配置文件/etc/apache2/sites-available/v.qing.su.conf, （这里请将v.qing.su换成您网站的域名，文中以后都用v.qing.su来指代域名，请您做相应替换）。在Apache配置文件中写入下列内容：

1
2
3
4
5
6
7
8
9
10
11
12
13
<VirtualHost *:80>
     ServerAdmin webmaster@example.com
     ServerName v.qing.su
     ServerAlias mv.qing.su
     DocumentRoot /srv/www/v.qing.su/public_html
     ErrorLog /srv/www/v.qing.su/logs/error.log
     CustomLog /srv/www/v.qing.su/logs/access.log combined
     <Directory /srv/www/v.qing.su/public_html/>
    Options +FollowSymLinks
    AllowOverride All
        Require all granted
     </Directory>
</VirtualHost>
 

然后启用该网站：

1
2
3
4
5
6
7
8
9
10
11
a2ensite v.qing.su
a2dissite 000-default.conf
a2enmod rewrite
mkdir -p /srv/www/v.qing.su/logs/
cd /srv/www/v.qing.su/
git clone https://github.com/WWBN/AVideo.git
mv AVideo public_html
cd public_html
git clone https://github.com/WWBN/AVideo-Encoder.git
mv AVideo-Encoder upload
service apache2 restart
这样，我们搭建好了Apache2服务器。注意到，这里我们已经从Github上获取了服务器的源程序和后台转码程序，并放在了网站目录中。

为了让访客更加信任我们的网站，我们需要安装SSL安全证书。这里我们将使用免费的Let’s Encrypt证书，其他证书可以通过类似方式安装。依次执行：

1
2
3
4
5
6
apt-get install software-properties-common
add-apt-repository universe
add-apt-repository ppa:certbot/certbot
apt-get update
apt-get install certbot python-certbot-apache
certbot --apache
按照屏幕提示操作即可安装好SSL安全证书。到这里，我们成功搭建了Apache2服务器并配置了该视频分享网站的配置文件。

2.2 安装PHP
为了方便，我们直接从源安装PHP 7.2.

1
2
apt-get install php7.2-cli php7.2-common php7.2-json php7.2-opcache php7.2-readline php7.2-curl php7.2-mysql
service apache2 restart
这样，我们成功安装好了PHP.

2.3 安装并配置MariaDB
为了方便，我们直接从源安装MariaDB.执行下列命令：

1
2
3
apt-get install mariadb-server
service mariadb start
mysql_secure_installation
上述最后一行命令可以配置MariaDB的root用户密码，禁止远程root访问等安全设定。到这里，MariaDB已经安装好了，我们需要新建两个数据库供我们的视频播放站使用。首先进入MariaDB的命令行：

1
mysql -u root -p
然后我们解释一下为什么需要两个数据库。第一个数据库是给前端使用的，就是网站的播放（以及评论、分享、频道、订阅等等等）功能。我们把这个数据库命名为player. 第二个数据库是给后端使用的，主要实现视频的上传和转码功能。我们把这个数据库命名为encoder. 我们在MariaDB中执行：

1
2
3
4
5
create database player;
grant all on player.* to 'player' identified by 'v.qing.su';
create database encoder;
grant all on encoder.* to 'encoder' identified by 'v.qing.su';
quit;
其中，引号包含的部分为数据库用户的用户名和密码，请自行设定。这样，我们安装并配置好了MariaDB数据库。

2.4 安装服务器需要的其他组件
由于我们搭建的是一个大型视频播放转码网站，很多功能需要其他的组件来实现。我们依次执行下面的命令：

1
2
3
4
apt-get install ffmpeg
apt-get install libimage-exiftool-perl
apt-get install python3-pip
pip3 install youtube-dl
Youtube-dl的作用是可以直接帮我们把Youtube上面的视频同步到我们的网站上。Youtube-dl需要经常更新，否则就会失效。更新命令是：

1
pip3 install --upgrade youtube-dl
至此，我们成功安装了LAMP套件，并安装了视频分享服务器需要的其他组件。我们可以开始安装网站主程序了。

3. 安装AVideo
主程序的安装我们将分为两部分介绍，首先是前端部分的安装，然后是后台上传及转码部分的安装。

3.1 安装AVideo前端
因为之前在Apache2搭建的那部分，我们已经把AVideo的主程序和后台程序放在了网站目录中，所以我们只要直接打开我们的网站就可以开始安装了。访问你的视频网站的首页，我这里是https://v.qing.su 安装页面应该和下图类似。



可以看到，网页左侧提示，PHP的设定不满足服务器要求。我们需要更改php.ini文件(/etc/php/7.2/apache2/php.ini)，满足系统的设定。我这里将两个值均设为了1024M.

1
2
post_max_size = 1024M
upload_max_filesize = 1024M
更改完毕后，我们需要在网页右侧填入相应的信息，比如网站地址，网站标题，管理员邮箱等等。这里的数据库名，数据库用户名与密码应该与之前设定的前端数据库的相关信息一致。填完全部信息后，我们点击Install Now即可完成安装，安装好之后会有下图的提示。



如果您安装之后出现报错，请在本文下方留言，我将尽力解答。

3.2 安装AVideo后端
访问视频网站的后端首页，我这里是https://v.qing.su/upload/ 即可看到后台的安装页面，应该如下图类似。



和之前安装前端时类似，我们看到网页左侧出现了报错。这里我们需要再次修改php.ini文件，作如下更改.

1
2
max_execution_time = 7200
memory_limit = 512M
然后，我们在网页右侧填入之前设定的后端数据库信息。需要注意的是，我们需要在右下方填入前端网站信息，其中密码是之前设定的前端的后台管理密码。如果这个地方密码输错，回导致后台转码之后的视频文件被前端403拒绝，导致播放时找不到视频文件。全部输入完毕后，点急Install Now即可完成安装，安装好之后会有下图的提示。



如果您安装之后出现报错，请在本文下方留言，我将尽力解答。安装完毕之后，我们可以删除两个install 文件夹。

1
2
rm -r /srv/www/v.qing.su/public_html/install/
rm -r /srv/www/v.qing.su/public_html/upload/install/
这样，我们完成了网站前后端的安装，可以开始配置和使用网站了。

4. 配置AVideo
接着，我们需要来配置整个网站。需要注意的是，AVideo是一个非常强大、功能丰富的系统，里面各种配置非常多样，因此在这里我没有办法介绍所有的配置，仅会介绍一些基本的后台设定。如果您需要实现某种功能，可以查看他们的Github以及官方网站，上面有详细的说明。

4.1 连接前端与后端
第一次登录AVideo系统，需要在前端里设定后台上传与转码地址。我们打开https://v.qing.su, 在右上角使用admin用户登录（密码是在安装的时候输入的）。登录之后我们可以看到下面的后台页面。



点击左侧Admin Panel, 然后依次找到Settings –> Site Settings –>Advanced Configuration, 如下图所示。我们在Encoder URL输入框中输入我们的后端网址，我这里是https://v.qing.su/upload/, 然后点击Save保存即可。



4.2 配置SMTP发信
大型的视频分享网站需要实现用户注册、验证、通知等功能，因此我们需要配置SMTP发信。在刚才这个界面的右边，我们输入SMTP服务器的地址、验证协议、端口、用户名和密码等信息，然后点击Test Email. 如果成功，会看到下图的提示。



如果发送失败，常见的原因是SMTP服务器的地址、端口、用户名密码等信息有错误。如果报错信息是Fail to call sendmail之类的，说明服务器上没有安装sendmail, 这时候需要执行下面的命令安装sendmail.

apt-get install sendmail

配置好SMTP发信之后，保存即可生效。

4.3 设置LOGO和Favicon
依次点击左侧Site Configurations –> Regular Configuration, 即可在网页右侧上传我们的网站图标。这里我们随手弄了一个香菇视频的LOGO, 上传之后刷新网页即可生效。



其他的设置，比如视频分类、模板主题、用户控制、付费下载、广告联盟等等，您可以自行研究，这里不再详述。

5. 导入视频及播放
网站全部搭建及配置完毕之后，我们就可以导入视频并播放啦。导入或者上传视频需要登录后台，我这里是https://v.qing.su/upload/, 或者也可以从前端点击“上传”链接进入后台。



后台支持三种视频导入方式，浏览器上传、视频网站导入、本地视频批量导入。每种导入方式类似，我们这里示范直接从youtube导入视频。点击Import Video, 输入Youtube视频链接，然后点击Share, 即可开始导入转码。下面的分辨率设置默认会全部勾选，可根据个人喜好调整，我自己一般只选择HD. Advanced部分不建议勾选。

视频转码完毕之后，我们就可以在我们的视频站播放了，如下图。



6. 常见报错及解决方案
6.1 后台解码完毕之后，前端找不到视频
出现这个问题大概率是因为后台登录的时候使用了错误的前端网站用户名密码信息。可以查看前端网站的日志/srv/www/v.qing.su/public_html/videos/avideo.log找到报错信息。通常，重新登录后台即可解决问题。

6.2 后台解码完毕之后，前端能找到视频，但是一播放就Server Error 500
出现这个问题可能有很多种原因，首先需要查看上面的日志并找到报错信息。如果您无法解决，可以将日志中的报错信息回复在这里，我可以尝试帮您解决。

6.3 后台Compatibility Check报错Video Directory not Writable
如果安装的时候没有出现问题但是安装好之后出现index.phpvideos文件夹不可写的报错，通常是因为.htaccess文件出错了，如下图。



此时仅需下载一份原版的程序并用其中的.htaccess文件替换掉当前的.htaccess文件即可。

6.4 网站丢失字体, CSS等文件，无法正常显示
可能是由于Apache配置文件的网站目录之后多打了一个斜杠’/’. 删掉这个斜杠并且重启Apache2应该就可以解决了。

如果您还遇到过其他问题，欢迎留言与我讨论哈。

至此，我们成功使用AVideo搭建了全功能的视频分享网站，并成功配置了前后台的各项功能。
