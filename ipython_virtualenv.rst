======================================
ipython(0.13.1)安裝及與virtualenv(1.9.1)一起使用
======================================


1. create ipython profile setting file: 
-------------------------------------------------
::

   $ ipython profile create 

- 建立~/.config/ipython/profile_default/ipython_config.py

::

   $ ipython profile create <name>

- 建立~/.config/ipython/ **<name>** /ipython_config.py

2. 啟動ipython 
--------------------

- 可指定 profile, 或不指定則使用default profile

::

   $ ipython --profile=<name>


- 初次啟用時發生錯誤, **ImportError: No module named IPython.frontend.terminal.ipapp**, 需在/usr/bin/ipython檔案加入

::

    import sys
    if "/usr/lib/python2.7/dist-packages" not in sys.path:
        sys.path.append("/usr/lib/python2.7/dist-packages")

3. 讓ipython 與 virtualenv一併使用: 
----------------------------------------

- 需要在ipython_config.py中加入

::

    import site
    from os import environ
    from os.path import join
    import sys

    if 'VIRTUAL_ENV' in environ:
	virtual_env = join(environ.get('VIRTUAL_ENV'),
			   'lib',
			   'python%d.%d' % sys.version_info[:2],
			   'site-packages')

	# Remember original sys.path.
	prev_sys_path = list(sys.path)
	site.addsitedir(virtual_env)

	# Personal hack
	# Remove /usr/lib/python2.7/dist-packages and associates from the path
	# Counter the effects of this code, in /usr/bin/ipython
	# import sys
	    # if "/usr/lib/python2.7/dist-packages" not in sys.path:
	#     sys.path.append("/usr/lib/python2.7/dist-packages")
	for item in sys.path:
	    if '/usr/lib/python2.7/dist-packages' in item:
		sys.path.remove(item)


	# Reorder sys.path so new directories at the front.
	new_sys_path = []
	for item in list(sys.path):
	    if item not in prev_sys_path:
		new_sys_path.append(item)
		sys.path.remove(item)
	sys.path[1:1] = new_sys_path

	print 'VIRTUAL_ENV ->', virtual_env
	del virtual_env
    del sys, site, environ, join
