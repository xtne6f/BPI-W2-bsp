SHELL=/bin/bash
TOP = $(PWD)/../..
include $(TOP)/make.include

obj-m := gpio_isr.o

.PHONY: all modules strip install clean

all: modules

modules:
	$(MAKE) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) -C $(LINUX_KERNEL_PATH) M=$(shell pwd) modules

strip:
	$(CROSS_COMPILE)strip gpio_isr.ko --strip-unneeded

install:
	cp -a gpio_isr.ko $(TOP)/bin/

clean:
	rm -fr Module.symvers ; rm -fr Module.markers ; rm -fr modules.order
	rm -fr *.mod.c *.mod *.o .*.cmd *.ko *~
	rm -fr .tmp_versions
