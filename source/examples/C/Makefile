all: hwserver hwclient version
version:
	gcc ./version.c -o ./version -lzmq
hwserver:
	gcc ./hwserver.c -o ./hwserver -lzmq
#end_hwserver
hwclient:
	gcc ./hwclient.c -o ./hwclient -lzmq
#end_hwclient
task:
	gcc ./taskvent.c -o ./taskvent -lzmq
	gcc ./taskwork.c -o ./taskwork -lzmq
	gcc ./taskwork2.c -o ./taskwork2 -lzmq
	gcc ./tasksink.c -o ./tasksink -lzmq
	gcc ./tasksink2.c -o ./tasksink2 -lzmq
		
#end_task
clean:
	rm hwserver hwclient  version 
