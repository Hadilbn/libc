BUF_SIZE=24

all: retlib prtenv
	touch badfile

retlib: retlib.c
	gcc -m32 -DBUF_SIZE=$(BUF_SIZE) -fno-stack-protector -z noexecstack -o retlib retlib.c

prtenv: prtenv.c
	gcc -m32 -o prtenv prtenv.c

setuid:
	sudo chown root:root retlib
	sudo chmod 4755 retlib

clean:
	rm -f retlib prtenv badfile
