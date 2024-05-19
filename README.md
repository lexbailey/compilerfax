# CompilerFax

You fax it some C code, it compiles it, runs it, faxes back the result

## Hardware

- Raspberry Pi
- Fax modem (I used a startech USB modem)

## Setup

Sorry, these instructions are not very detailed right now, maybe I'll go into more detail when I'm in less of a rush, but...

1. install hylafax on the raspberry pi
2. configure hylafax for your modem
3. (optional) swear at hylafax for all the trouble that it gave you in steps 1 and 2 (see `hylafax_is_broken` for details of bugs you might encounter)
4. clone this repo on to the ras pi
5. create a config file (a sourcme that sets OUTERUSER to `pi` (or whatever username you chose for the pi) and source it
6. run `sudo ./prep_alpine_chroot`
7. configure hylafax to add incoming faxes to the job queue using the FaxDispatch file (copy it to `/var/spool/hylafax/etc/FaxDispatch`
8. set up the job queue program to run on boot (copy the systemd unit into the systemd config, enable it, start it)

## Usage

1. Get a sheet of paper with some C code on it
2. Fax the C code to CompilerFax
3. wait for a response from CompilerFax
