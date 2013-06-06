=============================================
利用nginx 及 gunicorn來佈署Django application
=============================================
    
Django的設定
------------

- 正式deploy時, 要將static content(css, js, images, 與user-uploaded files)與程式分開

  - STATIC_ROOT setting::

        STATIC_ROOT = '/var/www/html/djangophp/static/'

  - MEDIA_ROOT setting::
  
        MEDIA_ROOT = '/var/www/html/djangophp/media/’

  - STATIC_ROOT設定值:

    - 在正式deployment環境下才會使用到的設定

    - 可以利用 **$python manage.py collectstatic** 來將STATICFILES_DIRS設定的static file directories的內容copy到STATIC_ROOT下


  - MEDIA_ROOT設定:

    - 需在development環境與正式deployment環境下使用不同的設定值,例如:

      - development環境::
            
 	    MEDIA_ROOT = os.path.join(_PATH, 'media')

      - deployment setting::

            MEDIA_ROOT = '/var/www/html/djangophp/media/'

    - 因此如果需要(例如測試主機與正式主機的user-upload files需要同步)則必須手動將檔案複製到MEDIA_ROOT路徑下

      notes: 檔案上傳將會寫入到MEDIA_ROOT路徑中, 因此這裡必須讓執行gunicorn的process所有權者(migulu)有write的權限. 可是這樣會需要/var/www/html/djangophp/media/ 該路徑以上的所有parent directories都要開放給migulu user write的權限. 這樣不是好的方法.

      因此還是將MEDIA_ROOT設定在development環境下的MEDIA_ROOT 亦即 /home/migulu/djangophp/djangophp/media/ 下, 然後建立一個目錄連結(mount 方式) 由 '/var/www/html/djangophp/media/' 指向development環境下的MEDIA_ROOT設定, 如此可以讓gunicorn process可以寫入檔案, nginx process可以讀取. ::
      
          mount --bind /home/migulu/djangophp/djangophp/media/ /var/www/html/djangophp/media

  - urls.py中的root urlpattern設定: 
    - 為了讓django與php能夠在同一個server下執行, nginx中的location設定須將兩者的url區分開, 因此django的root url改到django/下 ::

          urlpatterns += patterns('',
   	      url(r'^django/$', 'orion.views.news_list', name='news_list'),
	  )

gunicorn指令
------------
- 使用gunicorn(另一個指令為gunicorn_django,但在django1.4後, 建議使用gunicorn)

- ‘djangophp’為 django project name

- ‘application’ is the wsgi variable name in wsgi.py module

- the binding ip/port is localhost:8001

  - since we are now using nginx as the reverse proxy, here should use the localhost, so the django project can only be accessed through nginx, not accessed directly. ::

        $/home/migulu/Dropbox/2.7.3/bin/gunicorn djangophp.wsgi:application -u migulu -g migulu --settings=djangophp.settings_production -b 127.0.0.1:8001

- settings_production.py中僅僅是將MEDIA_ROOT改寫


nginx設定
---------

- 設定檔路徑: /etc/nginx/conf.d/virtual.conf ::
      
    server {
       listen       8080;
       server_name  200.100.100.124;
       root /var/www/html/host; #for php static files
       access_log /var/log/nginx/access.log;
       error_log /var/log/nginx/error.log;
       location /django/ {
	   proxy_pass_header Server;
	   proxy_set_header Host $http_host;
	   proxy_redirect off;
	   proxy_set_header X-Real-IP $remote_addr;
	   proxy_set_header X-Scheme $scheme;
	   proxy_connect_timeout 10;
	   proxy_read_timeout 10;
	   proxy_pass http://localhost:8001/django/; #注意'django’的url路徑
       }
       location /djangophp/media/ { # matching url pattern for  media files
	   root /var/www/html;
       }
       location /djangophp/static/ { # matching url pattern for  static files
	   root /var/www/html;
       }
       location / {
	   index   index.php;
       }
       location ~ \.php$ {
	   fastcgi_pass    127.0.0.1:9000;
	   fastcgi_index   index.php;
	   fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
	   include         fastcgi_params;
       }
    }

- reload nginx server::

      $/etc/init.d/nginx --reload

- 注意STATIC_ROOT及MEDIA_ROOT路徑下的directory/file permissions.

  - 必須要讓執行nginx的process的所有權者(nginx), 能夠有這些路徑的read權限

  - 若是要能夠寫入檔案(user uploading files), 則需讓執行gunicorn的process所有權者(migulu)有MEDIA_ROOT的write 權限(或者利用前面所提及的mount方式之目錄連結).



