---
layout: post
tags: openHAB
title:  "Debouncing an openHAB rule"
---

I needed to debounce an openhab rule to cope with a seriously cheap 433MHz contact sensor that, on activation, would spam its ID to RFXCOM a few times. I couldn't find much on the web so I made a debouncer using locks.

I thought I'd share it here, as it could be useful to others. I left in the debug message so its easy to see how it works, and verify it in the logs if you want to.

```java
import java.util.concurrent.locks.Lock
import java.util.concurrent.locks.ReentrantLock

var Lock lock = new ReentrantLock()

rule "switch"
when
    Item Undecoded received update "107366"
then
    // tryLock will acquire the lock if it is available, and fail fast otherwise.
    if (lock.tryLock()) {
        logInfo("contact", "Contact sensor tripped")
        Thread::sleep(300)
        lock.unlock()
    } else {
        logDebug("doors", "Contact sensor tripped, but ignored due to lock")
    }
end
```
