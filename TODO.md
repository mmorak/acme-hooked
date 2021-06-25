# TODO

This file lists several improvement ideas. Pull requests are welcome.

## Improvements

- wildcard domain certificates - maybe they just work(TM)?
- don't get new nonce every time, it's always supplied by each request (except the first)
- have a loop submitting the challenges (send-signed-req), then another verifying them (poll-until-not)
- poll-until-not can be optimized (request first, wait/assert later)
- retry requests (-do-request) several times (how often?) + timeout (how long?) + wait (how long?)
- log account ID(?)
- testing against pebble
- continuous integration
- windows/mac support
- turn hook argument in python into a python function
- convert bash scripts to POSIX shell

## Interesting Additions (Possibly With Trade-offs)
- switch from openssl + subprocess to some crypto library (check how common this dependency is)