# MDS-Data
Shared Mobility Data Methodology

## Data Retrieval 
MDS – or “Mobility Data Specification” – is comprised of a set of Application Programming Interfaces (APIs) that create standard communications between cities and private companies to improve their operations. 

To maintain compliance with shared mobility licensing permits, providers must provide and an [API  that projects trips and Status change data](https://github.com/openmobilityfoundation/governance/blob/main/technical/Understanding-MDS-APIs.md) according to [MDS standards](https://github.com/openmobilityfoundation/mobility-data-specification).(Currently using MDS 1.0-0.4.0)



## Data Aggregation for Privacy Concerns
### Personally identifiable information (PII) 
[PII](https://nacto.org/wp-content/uploads/2019/05/NACTO_IMLA_Managing-Mobility-Data.pdf) is commonly thought to be limited to direct unique personal identifiers such as name, address, social security number, or credit card number. However, all data canbecome PII depending on how easily and accurately it can be tied to an individual. The U.S. government defines PII as “information that can be used to distinguish or trace an individual’s identity, either alone or when combined with other personal or identifying information that is linked or linkable to a specificindividual.”

Geospatial data is, or can become, PII in two ways:

• **Recognizable Travel Patterns** – Even in anonymous datasets, people can be re-identified from their routine travel patterns – e.g. from home to work, school, stores, or religious institutions. The 2013
    Scientific Report article, “Unique in the Crowd: the privacy bounds of human mobility” found that, in a dataset of 1.5 million people over 6 months, and using location points triangulated from cellphone towers, “four spatio-temporal points are enough to uniquely identify 95% of the individuals.”
    
• **Combined With Other Data** – Geospatial mobility data can be combined with other data points to become PII (sometimes referred to as indirect or linked PII). For example, taken by itself, a single geospatial data point like a ride-hail drop-off location is not PII. But, when combined with a phonebook or reverse address look-up service, that data becomes easily linkable to an individual person. 

*For example, in 2014, a researcher requested anonymized taxi geo-location data from NYC Taxi and Limousine Commission under freedom of information laws, mapped them using MapQuest, and was able identify the home addresses of people hailing taxis in front of theHustler Club between midnight and 6am. Combining a home address with an address look-upwebsite, Facebook and other sources, the researcher was able to find the “property value, ethnicity,relationship status, court records and even a profile picture!” of an individual patron.*

The small number of data points necessary to identify an individual from their travel patterns, the
ubiquity of secondary data sets, and the ease with which they can be combined with geospatial trip data
to form PII, all mean that both the public and private sector should treat geopspatial trip data as PII for
collection, management, storage, and dissemination.
When it comes to mobility, privacy is related to the degree to which an individual trip is synonymous
with an individual person. For example, each dockless scooter trip is tied to an individual user and thus
broadcasts specific, unique information about an individual person’s behavior. Similarly, when passengers
are in the car, an app-based ride-hail or autonomous vehicle trip is linked to an individual. Such data
should be handled in accordance with city PII policies to ensure that individual privacy is protected while
simultaneously ensuring that cities have the necessary information to achieve public policy goals and
serve public needs. In contrast, ride-hail trips without a passenger, like “dead-heading” or circulating, or
shared trips with fixed stops, do not reveal personally identifiable patterns and can be easily shared. 

### Data Aggregation to Protect PII

The dataset intake process includes aggregations to prevent the disclosure of potentially identifying information that can occur in mobility data, while still allowing approximate, useful information to be released and published. Our disclosure control policy aggregates trip data into location and time bins as follows:

* First, the precision of start and end locations have been reduced by truncating latitude and longitude coordinates to two decimal points. This results in a grid of rectangular bins approximately 2,700 ft by 1,800 ft with an area of approximately 0.17 sq. miles. 

* Second, the timestamps are broken into five daypart time bins: AM Peak (6-9 AM), PM Peak (4-7PM) , Mid-Day, Night, and Weekend. These windows are based on typical travel patterns.

Finally, we aggregate trips by location and time bins by quarter. To prevent individual trips from being identified, we drop the location information in any group that consist of less than 3 trips. These sensitive trips are included in the data set as larger groups aggregated only by time bins. Within all groups, travel attributes such as distance traveled and trip duration are averaged.

Example location bins by coordinate truncation to 0.01 degrees Latitude and Longitude.

![Image of Grids](https://github.com/anthonyaanderson/MDS-Data/blob/main/SeattleGrid.png)

## Data Aggregations
Here is a list of the data aggregations used to analyse shared mobility data. 

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
To calculate the size of the bike share fleets in Seattle,  we checked all status changes for each provider back 7 days using using the [MDS Status Change Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/status_changes.json) 
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

The process to create an hourly snap shot(See code below): 
1.	Begin with an array of date-times for every hour of a day. This gives you the list of snapshot date-times to check. (*ex. 09/01/2020 1:00AM, 09/01/2020 2:00 AM, 09/01/2020 3:00 AM*)
2.	For each date-time, filter the status changes data set to gather all status changes for the seven days leading up to each date-time. (*For example, if the date-time is 09/01/2020 at 1:00 AM, gather all status changes between 08/25/2020 to 09/01/2020 at 1:00 AM.*)
3.	Create a list of all of the unique devices that appear within that filtered data. You will use that list to check the status of each device.
4.	For each device in that list, create a data set with only this device’s status changes.
5.	Save only the last status change that device recorded in this data set.
6.	Combine all of these devices status’s for each day-time  period into one data group.
7.	Group the data by day-time, provider, vehicle type, event type, and  event reason by counting the each datapoint.  

 
The snaphot data with have a group of event types per hour. Those event types are classified into if a device is inactive or active. 

| Event Type  | Event Type Reason | FleetCount |
| ------------- | ------------- | ------------- |
| available  | maintenance_drop_off  | Active  | 
| available  | rebalance_drop_off  | Active   | 
| available  | user_drop_off  | Active   | 
| available  | service_start  | Active   | 
| removed  | service_end  | No Longer Counted   | 
| removed  | maintenance_pick_up  | Inactive  | 
| removed  | rebalance_pick_up | Inactive  | 
| reserved  | user_pick_up | Active   | 
| unavailable  | low_battery | Active   | 
| unavailable  | maintenance  | Active   | 

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
