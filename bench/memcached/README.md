A concurrency bug in memcached-1.4.4.
============================

This is an atomicity violation (data race free).


Details
--------------------

If you want more details, please refer to issue#127 in
memcached issue track system.

http://code.google.com/p/memcached/issues/detail?id=127

The bug happens when two clients concurrently increment or
decrement certain cached data. The inplace incr/decr is not
atomic, causing some updates being lost.

When a client uses 'memcached_increment' to increase or
decrease the value of an item, the server will call
function 'process_arithmetic_command'. (assuming text mode)

memcached.c (line 2759)

        static void process_arithmetic_command(...) {
          ...
          it = item_get(key, nkey)
          ...
          add_delta(c, it, incr, delta, temp);
          ...
        }

The function will first get a reference to the item using
'item_get', and then increment/decrement the value via
function 'add_delta'.

memcached.c (line 401)

        add_delta(conn *c, item *item, int incr,
                  const int64_t delta, char *buf) {
            enum delta_result_type ret;
        
            pthread_mutex_lock(&cache_lock);
            ret = do_add_delta(c, item, incr, delta, buf);
            pthread_mutex_unlock(&cache_lock);
            return ret;
        }

The function 'add_delta' will grab 'cache_lock' and call
function 'do_add_delta' to perform the real operation.

memcached.c (line 2822)

        do_add_delta(conn *c, item *it, const bool incr,
                     const int64_t delta, char *buf) {
          ...
          snprintf(buf, INCR_MAX_STORAGE_LEN, "%llu",
                   (unsigned long long)value);
          res = strlen(buf);
          if (res + 2 > it->nbytes) { /* need to realloc */
            item *new_it;
            new_it = do_item_alloc(ITEM_key(it), it->nkey,
                                   atoi(ITEM_suffix(it) + 1),
                                   it->exptime, res + 2 );
            if (new_it == 0) {
              return EOM;
            }
            memcpy(ITEM_data(new_it), buf, res);
            memcpy(ITEM_data(new_it) + res, "\r\n", 2);
            item_replace(it, new_it);
            do_item_remove(new_it); /* release our reference */
          } else { /* replace in-place */
            /* When changing the value without replacing the
               item, we need to update the CAS on the existing
               item. */
            ITEM_set_cas(it, (settings.use_cas) ? get_cas_id() : 0);
        
            memcpy(ITEM_data(it), buf, res);
            memset(ITEM_data(it) + res, ' ', it->nbytes - res - 2);
          }

          return OK;
        }

The function 'do_add_delta' will calculate the length
of the new value. If the new length is bigger than the
original length, reallocation is needed. The system will
create a new item, copy the content from the old item,
and unlink the old item from the hash table. However, it
is possible that another thread is still working on the
old item (e.g. increment/decrement), causing update to the
old item being lost.

The buggy interleaving is shown below:

        Thread 1                              Thread 2

        process_arithmetic_command(...)       process_arithmetic_command(...)
        {                                     {
          ...                                   ... 
          it = item_get(key, nkey)
        
                                                it = item_get(key, nkey)
                                                ...
                                                add_delta(...)
                                                {
                                                  ...
                                                  // operate on new item
                                                  item_replace(it, new_it);
                                                  ...
                                                }
          ...
          add_delta(...)
          {
            ...
            // operate on old item
            memcpy(..., it, res);
            ...
          }
            
        }                                     }


