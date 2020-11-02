# MDS-Data
New Mobility Data Methodology

## Data Retrieval 
MDS – or “Mobility Data Specification” – is comprised of a set of Application Programming Interfaces (APIs) that create standard communications between cities and private companies to improve their operations. 

To maintain compliance with shared mobility licensing permits, providers must provide and an API  that projects trips and vehicle fleet data according to [MDS standards](https://github.com/openmobilityfoundation/mobility-data-specification).(Currently using MDS 1.0-0.4.0)

Data is retrieved from  each provider’s API  and placed into a database. The database is projected into and open data portal at (OPEN DATA)

We then create aggregate data groupings for reporting into this dashboard from that database.

## Data Aggregation for Privacy Concerns

TBD
## Data Aggregations
Here is a list of the data aggregations we use to analyse our data. 

### Trip Count

### Fleet Count(Pre June 2020)

### Fleet Snapshot Methodology
In order the count the number of shared mobility devices that are avaialble on city street we are using a count on the hour using the [MDS Status Change Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/status_changes.json)

1.	Take an array of date-times for a day at every hour. 
2.	For each day-time, I filter the status changes  data to before the date-time and back for 7 days.
3.	I then pull a list of all of the unique devices in that filtered data.
4.	For each device in that list I create a data set with only this device’s status changes.
5.	I save the last change.
6.	I then combine all of these last devices status’s for each day-time  period into one data group.
7.	After that I group the data by day-time, Provider, Vehicle type, event type, and  event reason by counting the each datapoint.  
Python Panda Function

```python
#SC is the Statsus Changes Database
#rundate is the date you want to run the snapshot
#
def get_hourlysnapshot(SC, rundate):
  dev =[]
  EventTimeLocal =[]
  EventType =[]
  EventTypeReason =[]
  ProviderName =[]
  VehicleType =[]
  Snaptime = []
  
  #rng will get a date tiem for every hour for this date.
  rng = pd.date_range(rundate, periods=24, freq='1H')
  
  #this loop will run through all the hours in the day
  for d in rng:
    filterSC =  SC[SC['EventTimeLocal'] < d]
    days=7    
    cutoff_date = d - pd.Timedelta(days=days)
    filterSC2 = filterSC[filterSC['EventTimeLocal'] > cutoff_date] 
    Devices = filterSC2.DeviceId.unique()
    
    #This loop will get the last status change for each device in this hour
    for i in Devices:
      IDevice = filterSC2[filterSC2['DeviceId']== i]
      LastStatus = IDevice.iloc[-1:]
      dev.append(i)
      EventTimeLocal.append(LastStatus.iloc[0]["EventTimeLocal"])
      EventType.append(LastStatus.iloc[0]["EventType"])
      EventTypeReason.append(LastStatus.iloc[0]["EventTypeReason"])
      ProviderName.append(LastStatus.iloc[0]["ProviderName"])
      VehicleType.append(LastStatus.iloc[0]["VehicleType"])
      Snaptime.append(d)
   
  #This area puts the data into a panda dataframe
  df = pd.DataFrame()
  df["Device"] = dev
  df['EventTimeLocal'] = EventTimeLocal
  df['EventType'] = EventType
  df['EventTypeReason'] = EventTypeReason
  df['ProviderName'] = ProviderName
  df['VehicleType'] = VehicleType
  df['Time'] = Snaptime
  df['count']= 1
  
  #This are aggragates the data by the hour, provider, vehicle type, and status chaneg events.
  Snapshot = df.groupby(['Time','ProviderName','VehicleType', 'EventType', 'EventTypeReason',], as_index=False).agg({'count':'sum'})
  
  #This returns the aggregated data.
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
