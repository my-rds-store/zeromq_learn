all: hwserver hwclient version
version:
	gcc ./version.c -o ./version -lzmq
hwserver:
	gcc ./hwserver.c -o ./hwserver -lzmq
#end_hwserver
hwclient:
	gcc ./hwclient.c -o ./hwclient -lzmq
#end_hwclient
clean:
	rm hwserver hwclient  version 
