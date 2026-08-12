gobuster dir -u http://192.168.96.18/ -w /usr/share/wordlists/dirb/common.txt

	.htpasswd            (Status: 403) [Size: 297]
	.hta                 (Status: 403) [Size: 292]
	.htaccess            (Status: 403) [Size: 297]
	index.html           (Status: 200) [Size: 328]
	manual               (Status: 301) [Size: 315] [-->
	http://192.168.96.18/manual/]
	server-status        (Status: 403) [Size: 301]

gobuster dir -u http://symfonos.local/h3l105/ -w /usr/share/wordlists/dirb/common.txt

	.hta                 (Status: 403) [Size: 300]
	.htpasswd            (Status: 403) [Size: 305]
	.htaccess            (Status: 403) [Size: 305]
	index.php            (Status: 301) [Size: 0] [--> 
	http://symfonos.local/h3l105/]
	wp-admin             (Status: 301) [Size: 326] [--> 
	http://symfonos.local/h3l105/wp-admin/]
	wp-includes          (Status: 301) [Size: 329] [--> 
	http://symfonos.local/h3l105/wp-includes/]
	wp-content           (Status: 301) [Size: 328] [--> 
	http://symfonos.local/h3l105/wp-content/]
	xmlrpc.php           (Status: 405) [Size: 42]


gobuster dir -u http://symfonos.local/h3l105/ -w /usr/share/seclists/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt

	wp-content/plugins/akismet/ (Status: 200) [Size: 0]
	wp-content/plugins/hello.php/ (Status: 500) [Size: 0]
	wp-content/plugins/hello.php (Status: 500) [Size: 0]
	wp-content/plugins/mail-masta/ (Status: 200) [Size: 1826]
	wp-content/plugins/site-editor/ (Status: 200) [Size: 0]
