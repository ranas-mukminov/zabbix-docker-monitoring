ZABBIX_SOURCE = ../../..

CFLAGS += -fPIC -shared -Wall -Wextra -Wno-unused-parameter -Werror -Wformat-security -Wstack-protector
INCLUDES = -I$(ZABBIX_SOURCE)/include -I$(ZABBIX_SOURCE)/include/common
LIBS = $(shell pkg-config --cflags --libs jansson 2>/dev/null || echo "-ljansson")

zabbix_module_docker.so: zabbix_module_docker.c
	gcc $(CFLAGS) -o zabbix_module_docker.so zabbix_module_docker.c $(INCLUDES) $(LIBS)

check:
	cppcheck --enable=all --suppress=missingIncludeSystem --inconclusive --force zabbix_module_docker.c

clean:
	rm -f zabbix_module_docker.so

.PHONY: clean check
