ZABBIX_SOURCE = ../../..

zabbix_module_docker.so: zabbix_module_docker.c
	gcc -fPIC -shared -o zabbix_module_docker.so zabbix_module_docker.c \
	    -I$(ZABBIX_SOURCE)/include \
	    -I$(ZABBIX_SOURCE)/include/common \
	    $(shell pkg-config --cflags --libs jansson 2>/dev/null || echo "-ljansson") \
	    -Wall -Wextra -Wno-unused-parameter

clean:
	rm -f zabbix_module_docker.so

.PHONY: clean
