---
layout: post
tags: openHAB
title:  "Using Undecoded items from the RFXCom binding in openhab"
---

In the upcoming 1.9.0 release of openhab, the RFXCom binding will support Undecoded items.

You can add the 1.9.0 binding to a 1.8 openhab by copying the jar to the addons folder, get a pre-release (beta 3) jar from here:
https://bintray.com/openhab/mvn/openhab/1.9.0.b3/view#files/org/openhab/binding/org.openhab.binding.rfxcom/1.9.0.b3

You'll have to use `setMode` in `openhab.cfg` to turn on undecoded message. The RFXCom device won't save the setting for undecoded as its supposed to be a debug switch for them to use when adding new devices to their list. Using `setMode` will turn it on every time. This is my `setMode` line, which basically turns on undecoded and homeeasy. You might need to turn on more, post in the comments if you need a hand.

/etc/openhab/configurations/openhab.cfg:
```
rfxcom:setMode=0d00001103531c80000f00000000
```

Once you have the new binding running, you'll need an undecoded item to pick up the events.

/etc/openhab/configurations/items/undecoded.items:
```
String Undecoded {rfxcom="<UNDECODED:RawData"}
```

You should then start seeing items appear in your events.log.

/var/log/openhab/events.log:
```
2016-07-05 08:56:58 - Undecoded state updated to 070301271356ECC0
```

The status is made up of 4 bytes of header, then the data. From this example, the bytes are:
(07) Length, 7 bytes.
(03) Message type - 03 is Undecoded, so they should all have 03 here.
(01) Sub-type - 01 is ARC.
(27) Sequence number. This should increment for every update.
(1356ECC0) The payload, in this case, 4 bytes of data.

This is all documented in [RFXComUndecodedRFMessage.java](https://github.com/openhab/openhab/blob/master/bundles/binding/org.openhab.binding.rfxcom/src/main/java/org/openhab/binding/rfxcom/internal/messages/RFXComUndecodedRFMessage.java) on github.

Because all the undecoded items comes through on just one item, its worth splitting them out into separate items using unbound items and a rule to process the undecoded updates. Start by making some items. My undecoded devices are contact sensors that I've put on doors, so the example will focus on that.

/etc/openhab/configurations/items/doors.items:
```
Switch FrontDoor "Front door [%s]"
Switch CatFlap "Cat flap [%s]"
```

For the sake of simplicity, now I'd like a rule to notify me when my front door is opened. You can obviously do anything. My sensor only triggers when the contact is broken, so we only have an Open trigger. To cope with that, I've told it to automatically record as closed 5 seconds later.

This rule uses a custom value selector of Data instead of RawData in the items file, to strip off the RFXCom headers which change every time. You'll need to roll your own rfxcom binding to use it, or get in touch in the comments and I'll provide one until the [pull request](https://github.com/openhab/openhab/pull/4520) makes it.

The ID of the sensor was detected by looking at the events.log as above.

/etc/openhab/configurations/rules/frontdoor.rules:
```
rule "Front door sensor"
when
    Item Undecoded received update "1356ECC0"
then
    sendCommand(FrontDoor, ON)
end

rule "Front door opened"
when
    Item FrontDoor changed to ON
then
    sendBroadcastNotification("Front door opened")
    Thread::sleep(5000)
    sendCommand(FrontDoor, OFF)
end
```

Assuming you're logged into your openhab with the app (either direct or through myopenhab), you should now be notified when your door is opened.
