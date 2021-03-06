An order violation concurrency bug in transmission-1.42.
===========================

Transmission is a multi-thread BitTorrent download client. 

Bug Detail
-------------

In transmission-1.42, a shared variable "h->bandwidth" is 
initialized by "h->bandwidth = tr_bandwidthNew(h, NULL)"
in session.c, line 281. Read access to this variable is
supposed to occur after initialization. Unfortunately, 
without proper synchronization, "assert(h->bandwidth)"
in bandwidth is likely to be executed before the
initialization and read an uninitialized variable, causing
an assertion failure.

(P.S. The function "allocateBandwidth" is called from an
event callback function which is supposed to be called
500 milliseconds after "t->peerMgr" is created. And this
is the reason why the programmer assumes that the variable
"h->bandwidth" is always initialized first.)


The buggy interleaving is like the following:

        Thread 1                                  Thread 2
        
        tr_handle *                               void
        tr_sessionInitFull(...) {                 allocateBandwidth(...) {
          ...                                       ...
          h->peerMgr = tr_peerMgrNew( h );
        
                                                    assert(tr_isBandwidth(b));
        
          h->bandwidth = 
             tr_bandwidthNew(h, NULL);
        }                                         }


