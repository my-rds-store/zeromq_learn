all: hwserver hwclient version
hwserver:
	gcc ./hwserver.c -o ./hwserver -lzmq
hwclient:
	gcc ./hwclient.c -o ./hwclient -lzmq
version:
	gcc ./version.c -o ./version -lzmq
clean:
	rm hwserver hwclient  version 
