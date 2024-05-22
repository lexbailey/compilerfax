# CompilerFax

You fax it some C code, it compiles it, runs it, faxes back the result

A quick demo vid: https://youtu.be/pJ-25-pRhpY

## If you send me your code

Just a quick note for anyone sending faxes to any CompilerFax instance that I (Lex Bailey) am running (probably at EMF Camp). If you send me a fax then you are granting me a licence to share the contents of that fax for information/education/entertainment purposes. (I'd love to collect some of the best faxes, and best attempts to break out of the container, and share them as a fun look at how people used this service.) Thanks :)

## Tips for users

The OCR is far from perfect. You need to use a font that is good for OCR. I have had success using Calibri. FreeMono is not too bad, but tends to be a little less predictable.

Some of the worst problems you will encounter are some characters being misread:

* O -> 0
* i -> 1
* x -> X

There are others, of course. This is not a complete list. In my testing I have been avoiding using "i" and "x" as variable names. It seems to be hard for the ORC to read "x" in the correct case. It often wants to upper-case it for some reason. Similarly, avoid "i" if you can. Even with good programming fonts the OCR still struggles.

Put spaces around things...

`a+=1;` tends to be seen by the OCR program as a _word_ of sorts, rather than four seperate things. try using `a += 1 ;`

I also found that `i++` was often misread as `itt`. adding a space in there (`i ++`) sometimes helped, but the best thing to do was to use `i += 1`, and to avoid the variable name `i` entirely if possible, as mentioned above.

The OCR program is not trained specifically on code. It is mostly trained on prose, and so using full words will likely produce better results. `int total = 0;` is better than `int t = 0;`

Most importantly, DO NOT FORGET: you must have the `REPLY = <number> #` text in your program, or you won't get your results faxed back to you. (see the usage section below) The spaces are optional, but having the spaces helps the OCR.

If you're really having trouble with some text, then you could try getting creative with the program so that bad OCR has less of an effect.

For example, consider this program:

```
#include <stdio.h>

int main ( ) {
    printf ( "Hello World\n" ) ;
    int i = 1;
    while ( i < 11 ) {
        i += 1 ;
        printf ( "%d\n" , i - 1 ) ;
    }
}
```

It has quite a lot of instances of the letter i by itself, and the number 1 by itself or as part of the number 11. There's 9 instances of `i`, or `1`. This is 9 chances for the OCR to fail.

The obvious first thing to do is to rename `i` to something else, like `a`. This would get us down to only having to worry about the 5 instances of `1` which each be misread. That's 5 chances for the OCR to fail. But you can also remove all of the 1 digits...

The program below does exactly what the one above does, but has none of the likely-misread characters)

```
#include <stdio.h>
#define one ( 4 - 3 )
#define eleven ( 3 + 4 + 4 )

int main ( ) {
    printf ( "Hello World\n" ) ;
    int a = one ;
    while ( a < eleven ) {
        a += one ;
        printf ( "%d\n" , a - one ) ;
    }
}
```

the digits 3 and 4 are fairly distinct in most fonts, they are much less likely to be misread.

## Hardware

- Raspberry Pi running Raspbian Trixie (older versions have a bug in unshare that prevents the container from mapping users correctly)
- Fax modem (I used a startech USB modem)

You don't have to use a raspberry pi. You can use some other system, as long as it can run hylafax and talk to a modem.

## Setup

Sorry, these instructions are not very detailed right now, maybe I'll go into more detail when I'm in less of a rush, but...

1. install hylafax on the raspberry pi, and some other deps: `sudo apt install hylafax-server clang-format tesseract-ocr`
2. configure hylafax for your modem
3. (optional) swear at hylafax for all the trouble that it gave you in steps 1 and 2 (see `hylafax_is_broken` for details of bugs you might encounter)
4. clone this repo on to the ras pi (or your system of choice)
5. create a config file (a sourcme that sets OUTERUSER to `pi` (or whatever username you chose for the pi) and source it. it should also set FAXID to something like "CompilerFax@1234" where 1234 is your phone number
6. run `sudo ./prep_alpine_chroot`
7. set up the job queue program to run on boot (copy the systemd unit into the systemd config, enable it, start it) TODO add systemd unit file to repo

## Usage

1. Get a sheet of paper with some C code on it. Leave your reply phone number in the source code with the text "REPLY = 1234 #" where 1234 is replaced with your phone number.
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
