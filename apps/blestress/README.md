BLE Stress Tests
================

blestress is a sample application for stress testing the NimBLE host and
controller with two boards. One board runs as the TX device (central,
scanner) and the other as the RX device (peripheral, advertiser). The RX
device runs 15 use cases one after another. For each use case it advertises
a 128-bit UUID that identifies the use case; the TX device scans for these
UUIDs and runs its part of the matching use case. Both devices print the
progress and a summary of results on the console.

Quick start
-----------

1. Create two targets, one per board. The role is selected with
   BLE_STRESS_TEST_ROLE (0 = TX device, 1 = RX device):

       newt target create blestress_tx
       newt target set blestress_tx app=@apache-mynewt-nimble/apps/blestress
       newt target set blestress_tx bsp=@apache-mynewt-core/hw/bsp/nordic_pca10056
       newt target set blestress_tx build_profile=optimized
       newt target set blestress_tx syscfg=BLE_STRESS_TEST_ROLE=0

       newt target create blestress_rx
       newt target set blestress_rx app=@apache-mynewt-nimble/apps/blestress
       newt target set blestress_rx bsp=@apache-mynewt-core/hw/bsp/nordic_pca10056
       newt target set blestress_rx build_profile=optimized
       newt target set blestress_rx syscfg=BLE_STRESS_TEST_ROLE=1

   Both boards do not have to use the same BSP, e.g. the TX device can be a
   nRF5340 DK (with BSP_NRF5340_NET_ENABLE=1 and NRF5340_EMBED_NET_CORE=1)
   while the RX device is a nRF52840 DK.

2. Optionally adjust the settings described below (BLE_STRESS_REPEAT,
   BLE_STRESS_UUID_BASE, BLE_MAX_CONNECTIONS, MSYS_1_BLOCK_COUNT).

3. Build, create images and load both boards. The application requires a
   bootloader (e.g. @mcuboot/boot/mynewt) to be loaded first:

       newt build blestress_tx
       newt create-image blestress_tx timestamp
       newt load blestress_tx

4. Open the console of each board (115200 8N1). The RX device has LED2
   turned on. Both devices print "Press ENTER to start:".

5. Press ENTER on the RX device first and then on the TX device. Only the
   first ENTER is taken into account. The RX device starts advertising the
   first use case; the TX device scans for it (for up to 20 seconds) and
   starts the tests.

6. Wait until both devices print "All tests completed" followed by the
   results. With BLE_STRESS_REPEAT=50 the whole run takes a few minutes.

Settings
--------

BLE_STRESS_TEST_ROLE  0 for the TX device, 1 for the RX device.

BLE_STRESS_REPEAT     Number of times the action of each use case is
                      repeated (e.g. number of connections, parameter
                      updates, received adverts); default 50.

BLE_STRESS_UUID_BASE  Last 13 bytes of the use case UUIDs. If more than one
                      pair of boards runs the tests within radio range,
                      give each pair a different base so that a TX device
                      only follows its own RX device.

BLE_MAX_CONNECTIONS   Upper limit for the multi connection use case (9).
                      The RX device needs the limit in the controller; for
                      a split (HCI) build the controller target has its own
                      limit, which then applies.

MSYS_1_BLOCK_COUNT    Size of the mbuf pool. In use case 10 the RX device
                      sends L2CAP SDUs of growing size until the next SDU
                      no longer fits into the pool (an SDU needs about
                      twice its size in mbufs), so a larger pool allows
                      larger SDUs. Note that the TX device must be able to
                      receive them as well.

LOG_LEVEL             1 (INFO) by default. Set to 0 to see all debug logs,
                      but keep in mind that plenty of adv packets will be
                      lost during the scan use case because of the console
                      load.

To run only a selected use case, change the loop in rx_stress_main_task_fn()
in rx_stress.c to call rx_stress_start(i) only for the wanted i. The TX
device follows whatever the RX device advertises.

Use cases
---------

No | Use case
---|-------------------------------------------------------------------------
 1 | Connect -> Connect Cancel
   | TX: initiates a connection to a random (non-existing) address and
   |     cancels it, first immediately, then after a random delay of up
   |     to 1 s. Each part is repeated BLE_STRESS_REPEAT times.
   | RX: nothing (only advertises the use case UUID).
 2 | Connect/Disconnect, legacy advertising
   | RX: legacy connectable advertising, restarted after each disconnect.
   | TX: connects and disconnects.
 3 | Connect/Disconnect, extended advertising
   | RX: extended connectable advertising (1M primary, 2M secondary PHY).
   | TX: connects and disconnects.
 4 | Connection parameter update initiated by the central
   | RX: advertises.
   | TX: connects and updates the connection parameters repeatedly.
 5 | Connection parameter update initiated by the peripheral
   | RX: advertises, then updates the connection parameters repeatedly.
   | TX: connects.
 6 | Scan of chained extended advertising data
   | RX: non-connectable extended advertising with ~1.6 kB of a known
   |     data pattern (AUX_CHAIN_IND).
   | TX: scans, reassembles the advert and checks it against the pattern.
   |     Counts adverts whose first packet was received and adverts
   |     received completely.
 7 | PHY update initiated by the central
   | RX: advertises.
   | TX: connects and cycles the PHY 1M -> 2M -> Coded -> 1M.
 8 | PHY update initiated by the peripheral
   | RX: advertises, then cycles the PHY.
   | TX: connects.
 9 | Multiple connections
   | RX: advertises with a new random address after each connection.
   | TX: connects to each instance (changing its own random address) until
   |     BLE_MAX_CONNECTIONS or BLE_STRESS_REPEAT is reached, then
   |     terminates all connections. Reports the max number of
   |     simultaneous connections.
10 | L2CAP connection oriented channel
   | RX: L2CAP server (PSM 0x80, MTU 64000). On every request from TX it
   |     sends an SDU, each one 500 bytes larger than the previous one
   |     (starting at 1000 bytes), until the next SDU does not fit into
   |     the mbuf pool.
   | TX: connects the channel, requests SDUs and measures the average
   |     throughput and the largest SDU received. Ends the test after 7 s
   |     without data.
11 | Advertise/Connect/Continue advertising/Disconnect
   | RX: extended advertising; disconnects immediately after each
   |     connection and continues with the same advertising instance.
   | TX: scans for the advert and connects, repeatedly.
12 | GATT indications
   | RX: sends indications, the next one after the previous one was
   |     confirmed. Measures the average indication round trip time.
   | TX: receives indications.
13 | GATT notifications
   | RX: sends notifications back to back. Measures the average time per
   |     notification.
   | TX: counts received notifications.
14 | GATT Subscribe/Notify/Unsubscribe
   | RX: notifies on subscription.
   | TX: discovers the CCCD, then repeatedly subscribes, waits for the
   |     notification and unsubscribes. Measures the average time from
   |     subscription to notification.
15 | Connect/Discover/Write/Disconnect
   | RX: advertises, restarted after each disconnect.
   | TX: connects, discovers the write characteristic, writes 20 bytes
   |     and disconnects, repeatedly.

The RX device exposes a GATT service with UUID 0xC0DE and characteristics
0xC1DE (read), 0xC2DE (write), 0xC3DE (indicate) and 0xC4DE (notify) used
by use cases 12 to 15. The device name is "STRESS"; do not change it,
use case 6 relies on the length of the advertising fields.

How it works
------------

The RX device advertises service data containing a UUID128 of the form
C0 DE <use case> <BLE_STRESS_UUID_BASE>. The TX device scans for known
UUIDs; when it finds one that was not performed yet, it remembers the
advertiser address, adapts to the advertised use case and runs the test.
When the TX device finds no new use case within 20 seconds, it assumes all
tests are done and prints the results.

Each device has a main task that starts the use cases one by one. The
tests themselves are driven by GAP/L2CAP/GATT events. To synchronize the
main task with the events a semaphore is used: after starting a use case
the main task waits on the semaphore, and the event handler of the use case
releases a token when the use case is completed. The RX device completes a
use case when it reaches the required count and the TX device disconnects,
the TX device when it reaches the required count or gives up (e.g. after
several failed connection attempts).

Use case 6 has no connection that could signal its end to the RX device, so
the RX device also runs a legacy "switcher" advertising instance (instance
0); when the TX device has finished the use case it connects to the
switcher and the RX device moves to the next use case on disconnect.

Expected logs
-------------

TX device (LOG_LEVEL 1):

    Start scan for test
    Start test num 1 - Stress Connect -> Connect Cancel
    First part of test completed
    Start second part: Connect->random delay->cancel
    Stress test 1 completed
    Start scan for test
    Start test num 2 - Stress Connect/Disconnect legacy
    Stress test 2 completed
    ...
    Start test num 15 - Stress Connect/Send/Disconnect
    Stress test 15 completed
    Start scan for test
    All tests completed
    Tests results:
    Use case 1 - Stress Connect -> Connect Cancel:
    Con attempts = 100
    Con success (expected 0) = 0
    Use case 2 - Stress Connect/Disconnect legacy:
    Con attempts = 50
    Con success = 50
    ...

RX device:

    Start test num 1 - Nothing to do
    Start test num 2 - Stress Connect/Disconnect legacy
    Stress test 2 completed
    ...
    Start test num 15 - Stress Connect/Send/Disconnect
    Stress test 15 completed
    All tests completed
    Tests results:
    ...

Results that are measured on one device only (e.g. scan results on TX,
indication timing on RX) are printed on that device; the other device
prints which device measured them.

Known limitations
-----------------

- There is no pass/fail verdict; compare the counters with
  BLE_STRESS_REPEAT and look for ERROR log lines.
- The RX device waits for the TX device to finish each use case. If the TX
  device gives up a use case (e.g. the RX device is out of range), the RX
  device may wait forever and both boards need to be reset.
- State is not reset after a run; reset both boards to run the tests again.
