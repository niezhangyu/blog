---
title: 简明 Python 教程第十章Windows下的解决方案
date: 2017/11/25
---


	import os
	import time

	source =r'C:\Users\PC-dev\Downloads'
	target_dir= 'E:\work'

	today = target_dir+os.sep+time.strftime('%Y%m%d')
	now =time.strftime('%H%M%S')

	if not os.path.exists(today):
    	os.mkdir(today)
    target=today + os.sep + now + '.rar'
	
	rar_command = 'WinRAR a -w %s %s -r' % (target,source)

	if os.system(rar_command) == 0:
    	print 'Successful backup to',target
	else:
    	print 'Bachup Failed'