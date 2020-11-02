# MDS-Data
New Mobility Data Methodology


## Fleet Snapshot Methodology

1.	Take an array of date-times for a day at every hour. 
2.	For each day-time, I filter the status changes  data to before the date-time and back for 7 days.
3.	I then pull a list of all of the unique devices in that filtered data.
4.	For each device in that list I create a data set with only this device’s status changes.
5.	I save the last change.
6.	I then combine all of these last devices status’s for each day-time  period into one data group.
7.	After that I group the data by day-time, Provider, Vehicle type, event type, and  event reason by counting the each datapoint.  
Python Panda Function

### Python Function 

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
