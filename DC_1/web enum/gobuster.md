
gobuster dir -u http://192.168.96.25/ -w /usr/share/wordlists/dirb/common.txt

	0                    (Status: 200) [Size: 7606]
	includes             (Status: 301) [Size: 317] [-->                                   http://192.168.96.25/includes/]
	index.php            (Status: 200) [Size: 7606]
	LICENSE              (Status: 200) [Size: 18092]
	misc                 (Status: 301) [Size: 313] [--> http://192.168.96.25/misc/]
	modules              (Status: 301) [Size: 316] [-->                                   http://192.168.96.25/modules/]
	node                 (Status: 200) [Size: 7606]
	profiles             (Status: 301) [Size: 317] [-->                                    http://192.168.96.25/profiles/]
	README               (Status: 200) [Size: 5376]
	robots.txt           (Status: 200) [Size: 1561]
	robots               (Status: 200) [Size: 1561]
	scripts              (Status: 301) [Size: 316] [-->                                    http://192.168.96.25/scripts/]
	sites                (Status: 301) [Size: 314] [-->                                    http://192.168.96.25/sites/]
	themes               (Status: 301) [Size: 315] [-->                                    http://192.168.96.25/themes/]
	user                 (Status: 200) [Size: 7459]
	web.config           (Status: 200) [Size: 2178]
	xmlrpc.php           (Status: 200) [Size: 42]

	

	
