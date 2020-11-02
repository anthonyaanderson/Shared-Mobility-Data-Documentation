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
To calculate the number of shared mobility trips daily we use an aggreagrated count of the  [MDS Trips Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/trips.json)
Process:
1) Filter the trips to in seattle, a duration of longer than 30 seconds and a distance of longer than 1 meter. 
2) Pull the travel date by the trips end time. 
3) Aggregate the dataframe into a daily count of trips by provider and vehilce type.

Python Code:
```python
def get_trip_count(df_trips):
    
    # filter criteria
    df_trips = df_trips[df_trips['TripDuration'] > 30]
    df_trips = df_trips[df_trips['TripDistance'] > 0]
    
    # extract the travel date from the trisp end time.
    df_trips['travel_date'] = df_trips['EndTimeLocal'].apply(lambda x: x.strftime("%Y-%m-%d"))
    df_trips['trip_count'] = 1
    
    # Aggregate dataframe by travel date, provider name, and vehicle type
    df_tripcount = df_trips.groupby(['travel_date','ProviderName','VehicleType'], as_index=False).agg({'trip_count':'sum'})
    
    return df_tripcount
```
### Fleet Count(Pre June 2020)
To calculate the size of the bike share fleets in Seattle,  we checked all status changes for each provider back 7 days. 
Status Changes with event reason's that are not "service ends" or "maintaince picks up" are calssified as  "In Service."
Status Changes with the event reason of "maintanience pick up"  are classified as "In Maintanince."

```python
def get_fleet_size(df_status):

    df_status['event_time_utc'] = pd.to_datetime(df_status['EventTimeLocal'])
    df_status.drop_duplicates(keep="last",inplace=True) 
    
    # limit records to locations in Seattle
    df_status['location'] = df_status[['Latitude','Longitude']].apply(lambda x: inSeattle(*x), axis=1)
    df_status = df_status[df_status['location'] == 'In Seattle']
    
    df_providers = df_status.groupby(['ProviderName'], as_index=False).agg({'location':'first'})
    
    #print (df_providers)
    providers = df_providers['ProviderName'].to_list()
    print (providers)
    
    Snapshot_date = []
    Count_in_service = []
    Count_in_maintenance = []
    Provider_name = []
    Vehicle_type = []
    
    
    # Iterate through each provider
    for provider in providers:

        df_status_provider = df_status[df_status['ProviderName'] == provider]
        
        # Minimum and maximum start dates
        min_date = df_status_provider['event_time_utc'].min().replace(hour=5, minute=0, second=0, microsecond=0)
        max_date = df_status_provider['event_time_utc'].max().replace(hour=5, minute=0, second=0, microsecond=0)
        report_start_date = min_date + timedelta(days=7)
        report_duration = (max_date - report_start_date).days + 2
        print (min_date, max_date, report_start_date, report_duration)
        
        print ('min_date:',min_date,'max_date:',max_date, 'report_start_date:',report_start_date)
        
        # Iterate through each day in the dataset and calculate the fleet size for that time
        for i in range(report_duration):
        
            snapshot_end_date = report_start_date + timedelta(days=i)
            snapshot_start_date = snapshot_end_date + timedelta(days=-7)  
            travel_date = snapshot_end_date.strftime("%Y-%m-%d")
            print ("travel_date", travel_date,"snapshot_end_date",snapshot_end_date,"snapshot_start_date",snapshot_start_date)

            df = df_status_provider.copy()

            # fleet size calculation critieria:
            # event is within one week prior to snapshot
            df = df[(df['event_time_utc'] > snapshot_start_date) & (df['event_time_utc'] <= snapshot_end_date)]

            # sort chronologically, then take the first event in the time period
            df = df.sort_values(by=['DeviceId','event_time_utc'], ascending=[False, False])
            df = df.drop_duplicates(subset=['DeviceId'], keep='first')
            
    
            for vehicle_type in df.VehicleType.unique():
                Snapshot_date.append(travel_date)
                Provider_name.append(provider)
                Vehicle_type.append(vehicle_type)
                Count_in_service.append(df["EventTypeReason"][(df["EventTypeReason"] != 'maintenance_pick_up') & (df["EventTypeReason"] != 'service_end') & (df["VehicleType"] == vehicle_type)].count())
                Count_in_maintenance.append(df["EventTypeReason"][(df["EventTypeReason"] == 'maintenance_pick_up') & (df["VehicleType"] == vehicle_type)].count())

    df_fleetsize = pd.DataFrame()
    df_fleetsize['travel_date'] = Snapshot_date
    df_fleetsize['vehicle_type'] = Vehicle_type
    df_fleetsize['provider_name'] = Provider_name
    df_fleetsize['count_in_service'] = Count_in_service
    df_fleetsize['count_in_maintenance'] = Count_in_maintenance
    
    return df_fleetsize
```

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

Python Code:
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
