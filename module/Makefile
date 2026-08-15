KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

obj-m += snd-hda-scodec-cs35l41.o
obj-m += snd-hda-scodec-cs35l41-i2c.o
obj-m += snd-hda-scodec-cs35l41-spi.o

snd-hda-scodec-cs35l41-y := cs35l41_hda.o cs35l41_hda_property.o
snd-hda-scodec-cs35l41-i2c-y := cs35l41_hda_i2c.o
snd-hda-scodec-cs35l41-spi-y := cs35l41_hda_spi.o

ccflags-y += -I$(src)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
