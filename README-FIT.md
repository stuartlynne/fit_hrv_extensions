# FIT File Developer Fields for HRV and multiple sensors
## Overview

This describe the developer fields used to add additional data to FIT data files:

- DFA Alpha1
- Respiration Rate
- Second Heart Rate Monitor
- ECG data
- Second Power Meter or FEC
- MO2 sensor
- VO2 sensor
- Core temperature sensor

Generally these are intended to facilitate:

- use of DFA A1, Respiration Rate and ECG data in post-workout analysis
- dual-power recording

Dual-power recording allows for post-workout comparison of multiple power sensors
including FE-C trainers.

The data is added to FIT files using *Developer Fields* to the *Record Message* and the *Hrv Message*.


## HRV

These are added to the Record message:

- Alpha1 - DFA A1 value for preceding period (typically 120s)
- Artifacts - the number of artifacts encountered in the preceding DFA A1 period
- Corrections - the number of corrections in the current sample (typically 1s)
- rmssd
- sdnn
- samples
- elapsed
- dropped\_percent

- readiness\_pa30
- readiness\_ra
- readiness\_mean\_pa30
- durability


*Corrections* are the actual number of corrected RR intervals in the current Record sample (e.g. preceding second).

*Artifacts* are the total number of corrected RR intervals in the current DFA A1 period (typically 120 seconds).


### Respiration Rate

This is the respiration rate for the current sample. Added to the Record Message.

- RespirationRate


## Heart Rate Monitor

### Second Heart Rate Monitor
There are three types of data being recorded:

- heart beat rate at 1 Hz
- RR interval data as received (between .5 and 5 Hz)

These fields are added to the Record message:

- heart\_rate\_2

These fields are added to the Hrv message:

- time\_2 - the RR interval data 


### ECG Data 

Some heart rate monitors (e.g. Polar H10) can provide ECG data. Saving this to the FIT
file with timestamps will allow post event comparison to other workout statistics, 
e.g. DFA A1, power, cadence, heart rate.

The Hrv message can be used to save this data. Typically one message will be sent with the
data from the heart rate monitor as received. Currently that appears to be 70 uvolt values
at a time. Periodically (typically when event, session or lap messages are sent) the timestamp
field will be added to help with synchronizing the ECG data to the data in the Record messages.

These fields are added to the Hrv message:

- uVolts (aka ECG) data 130 Hz, typically 70 values per message
- ts - timestamp, typically sent when event, lap or session messages sent


## Power
### Second Power meter or FEC Trainer

These add additional fields that duplicate the data fields already in the Record message
to carry data for an additional sensor, these are added to the Record Message:

- power\_2
- power\_fec
- cadence\_2
- cadence\_fec

The intent of these fields is to facilitate "dual-power" recording. It may be necessary
to add additional fields as appropriate as needed. 

## MO2

These are added to the Record Message.

- smO2
- thB

## VO2

These are added to the Record Message.

### Ventilatory
- respiratory\_frequency
- tidal\_volume
- ventilation

### Gas Exchange
- feO2
- vO2

### Environment
- ambient\_pressure
- device\_temperature
- device\_humidity
- ambient\_O2

### Ambient Gas Calibratioin
- raw\_O2
- O2\_ce
- calibration\_pressure
- calibration\_humidity
- sensor\_calibration\_temperature

## Body Temperature
- data\_quality
- skin\_temperature
- core\_temperature


## Hrv Message Considerations

The Hrv message as defined in the FIT SDK has a single field, 'time' that is allowed to have multiple values
per message. Adding additional developer fields to Hrv allows us to send multiple types of data that have
a similar requirement, specifically multiple values (aka an array) sent per message.

As defined the Hrv time field is the RR-interval time between heart beats, and has no additional time reference.
It is assumed that the RR-intervals start contemporaneously with the Record messages (but that is not
guaranteed.) The nature of the data is such that adding them together does provide a (reasonably accurate) time
reference with respect to the data itself by simply adding the values together you get the current offset from
the start of the first Hrv time field.

Fields added:

- time\_2 - rr interval data from a second heart rate monitor
- uvolts - ECG data from (for example) a Polar H10
- ts - a time stamp that can be used to synchronize ECG data to the record messages

The *time*, *time\2* and *uvolt* fields are sent as arrays of values. The suggested use is to periodically
send the accumulated data for *time* and *time2* together. 

ECG data sets are larger and will usually be
sent as received when received (typically for Polar H10, 70 samples every 70/130 seconds.)  
ECG data is at a fixed rate, like the time (for RR-intervals) field, a reasonably accurate time offset
from the first data point can be established by dividing by the sample rate, e.g. 130 Hz.

The *ts* time stamp should be sent when ECG data starts, and when major events happen such
as sending an Event, Session or Lap message, or if there is an interruption in any of the 
sensor data streams being used for time, time\_2 or uvolts. 
This ensures that we know where the other values in the time,
time\_w and uvolts streams can be place with respect to Record messages.


## Hrv Message Issues
There is a caveat that while Fit file parsers should have little problem parsing the data correctly,
some applications may incorrectly assume that all data in the Hrv message is 'time' without
looking at the field names.

For example, this code using the python fitparse library finds all of the Hrv data but 
does not correctly check for the field name:

'''
for record in fit\_file.get\_messages('hrv'):
    for record\_data in record:
        for RR\_interval in record\_data.value:
            if RR\_interval is not None:
                print('%.0f,' % (RR\_interval*1000.), file=sys.stdout)
'''

Adding a single line to check the field name is 'time' is all that is required to 
make this work correctly.

'''
for record in fit\_file.get\_messages('hrv'):
    for record\_data in record:
        if record\_data.name == 'time': # check that we have a 'time' field 
            for RR\_interval in record\_data.value:
                if RR\_interval is not None:
                    print('%.0f,' % (RR\_interval*1000.), file=sys.stdout)
'''

## Definition Messages

If multiple versions of the Record or Hrv message are being used, then each should use a different
local Id, e.g.:

- Hrv local id 10 - time, time2
- Hrv local id 11 - uvolts
- Hrv local id 12 - uvolts, ts

This will allow any mix of the messages to be sent without having the definition message redefined and resent
each time the format is changed.

