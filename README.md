# CompilerFax

You fax it some C code, it compiles it, runs it, faxes back the result

## Hardware

- Raspberry Pi running Raspbian Trixie (older versions have a bug in unshare that prevents the container from working correctly)
- Fax modem (I used a startech USB modem)

## Setup

Sorry, these instructions are not very detailed right now, maybe I'll go into more detail when I'm in less of a rush, but...

1. install hylafax on the raspberry pi
2. configure hylafax for your modem
3. (optional) swear at hylafax for all the trouble that it gave you in steps 1 and 2 (see `hylafax_is_broken` for details of bugs you might encounter)
4. clone this repo on to the ras pi
5. create a config file (a sourcme that sets OUTERUSER to `pi` (or whatever username you chose for the pi) and source it. it should also set FAXID to something like "CompilerFax@1234" where 1234 is your phone number
6. run `sudo ./prep_alpine_chroot`
7. set up the job queue program to run on boot (copy the systemd unit into the systemd config, enable it, start it)

## Usage

1. Get a sheet of paper with some C code on it. Leave your reply phone number in the source code with the text "REPLY=1234#" where 1234 is replaced with your phone number.
2. Fax the C code to CompilerFax
3. wait for a response from CompilerFax

## Troubleshooting

If you don't get a fax back, maybe your REPLY=<number># line was not detected correctly, or you used the wrong number

Alternatively it could by hylafax being bad

You can read the logs with journalctl

sudo journalctl -u compilerfax
sudo journalctl -u hylafax
sudo journalctl -u faxq
sudo journalctl -u hfaxd

you can inspect the logs from the compiler service (see the servicelog directory inside the maing working directory, which is the root of the repo)

you can also inspect the fax queue in /var/spool/hylafax/recvq, which will normally be empty, but might have jobs stuck in it if the compiler service is not working correctly

## Architecture

Hylafax does most of the heavy lifting. It handles all of the modem communication and the incoming and outgoing fax queues.

The programs that make the compiler work are:

`service_queue` - reads from the incoming fax queue, launches `build_and_run` for each incoming fax
`build_and_run` - runs once per fax. This process must complete for each incoming fax before the next fax starts to be processed. It does the OCR on the incoming .tif file, generates a report, and adds it to the outgoing fax queue with `sendfax`
`build_document` - is called by `build_and_run` to generate a postscript file, which can then be turned into a pdf to fax as a reply
`reset_container` - is called by `build_and_run` before each compilation and run job to reset the container to the base state (delete any files created by the last job)
`with_chroot` - this is the main wrapper around the alpine linux container. It does the `unshare` call to run a contained program in a bunch of empty namespaces. The user OUTERUSER is mapped to the `low` user in the container. root is also mapped, and the program runs as low. This prevents the contained program from modifying most of the container and stops the program for being able to escape the container (hopefully)
`line_numbers` - adds line numbers to the program listing (for building the report). Fairly boring program is this one.

Other programs, not used in normal operation of the service, but useful for maintenence:

`cleanup` - deletes the alpine linux root, and the copy of apk that installed it
`prep_alpine_chroot` - creates the alpine linux root using apk

normally you want to run `./cleanup` and then `./prep_alpine_chroot aarch64` if there's been a significant update to the container system. If in doubt then just run it, it can't do any harm
