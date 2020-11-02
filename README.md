# MDS-Data
New Mobility Data Methodology

## Data Retrieval 
MDS – or “Mobility Data Specification” – is comprised of a set of Application Programming Interfaces (APIs) that create standard communications between cities and private companies to improve their operations. 

To maintain compliance with shared mobility licensing permits, providers must provide and an API  that projects trips and vehicle fleet data according to [MDS standards](https://github.com/openmobilityfoundation/mobility-data-specification).(Currently using MDS 1.0-0.4.0)

We then create aggregate data groupings for reporting into this dashboard from that database.

## Data Aggregation for Privacy Concerns

TBD
## Data Aggregations
Here is a list of the data aggregations we use to analyse our data. 

### Trip Count

### Fleet Count(Pre June 2020)

### Fleet Snapshot Methodology
In order the count the number of shared mobility devices that are avaialble on city streets we are taking a snapshot count on the hour using the [MDS Status Change Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/status_changes.json)

The process: 
1.	Take an array of date-times for every hour of a day. This gives you the list of snapshot datetimes to check.
2.	For each day-time, filter the status changes data set to before the date-time you are check and back for 7 days. This filter will give you all statuc changes up to your day-time and checks 7 days back to see the status that have updated a week before this time. 
3.	Create  a list of all of the unique devices within that filtered data. you will use that list to check the status of each device.
4.	For each device in that list, Create a data set with only this device’s status changes.
5.	Save only the last status change that device recorded in this data set.
6.	Combine all of these devices status’s for each day-time  period into one data group.
7.	Group the data by day-time, Provider, Vehicle type, event type, and  event reason by counting the each datapoint.  

Python Panda Function
```python
#SC is the internal Status Changes Database
#rundate is the date you want to run the snapshot

def get_hourlysnapshot(SC, rundate):
  dev =[]
  EventTimeLocal =[]
  EventType =[]
  EventTypeReason =[]
  ProviderName =[]
  VehicleType =[]
  Snaptime = []
  
  #Step 1: rng will get a date time for every hour for this date.
  rng = pd.date_range(rundate, periods=24, freq='1H')
  
  #Step 2: using a for loop to filter the status change data for each day-time
  for d in rng:
    filterSC =  SC[SC['EventTimeLocal'] < d]
    days=7    
    cutoff_date = d - pd.Timedelta(days=days)
    filterSC2 = filterSC[filterSC['EventTimeLocal'] > cutoff_date] 
    #Step 3: create a list of all devices that are listed in this filtered data set
    Devices = filterSC2.DeviceId.unique()
    
    #Step 4:This loop will get filter the status change data to just the data for a specific device. 
    for i in Devices:
      IDevice = filterSC2[filterSC2['DeviceId']== i]
      #Step 5: Use only the last status change for each device
      LastStatus = IDevice.iloc[-1:]
      dev.append(i)
      EventTimeLocal.append(LastStatus.iloc[0]["EventTimeLocal"])
      EventType.append(LastStatus.iloc[0]["EventType"])
      EventTypeReason.append(LastStatus.iloc[0]["EventTypeReason"])
      ProviderName.append(LastStatus.iloc[0]["ProviderName"])
      VehicleType.append(LastStatus.iloc[0]["VehicleType"])
      Snaptime.append(d)
   
  #Step 6: Then add all the last status of each device for each time into a data frame
  df = pd.DataFrame()
  df["Device"] = dev
  df['EventTimeLocal'] = EventTimeLocal
  df['EventType'] = EventType
  df['EventTypeReason'] = EventTypeReason
  df['ProviderName'] = ProviderName
  df['VehicleType'] = VehicleType
  df['Time'] = Snaptime
  df['count']= 1
  
  #Step 7: Aggragate the data by the hour, provider, vehicle type, and status chaneg events.
  Snapshot = df.groupby(['Time','ProviderName','VehicleType', 'EventType', 'EventTypeReason',], as_index=False).agg({'count':'sum'})
  
  #Return the aggregated data.
  return Snapshot
 ```
### Max Hourly Fleet Snap
### Min Hourly Fleet Snap
### Equity Trips
### Equity Fleet
### Transit Trips

### Ridership
### Trip Location  Maps
### Trips Durations(Hours, Minute Seconds)
### Trip Distance(Miles, Feet, Kilometers, Meters)
