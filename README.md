# MDS Data Collection: Shared Mobility Data Methodology

The City of Seattle, through its Department of Transportation (SDOT), works to deliver a transportation system that provides safe and affordable access to places and opportunities. This is reflected in the work SDOT does to manage the public right-of-way, ensure safe movement of people and goods, and improve infrastructure and mobility choices throughout the city. As SDOT works to advance our core values of equity, safety, mobility, sustainability, livability, and excellence, we collect data necessary for performing our work while protecting against the misuse of personal mobility data.

## Data Retrieval 
Shared micromobility vendors permitted to operate in Seattle are required to share data about their operations with SDOT using the [Mobility Data Specification](https://github.com/openmobilityfoundation/mobility-data-specification) (MDS).

MDS comprises a set of application programming interfaces (APIs) that standardize communications between cities and private mobility companies. The [provider](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider) API includes the following data:

* [Trips](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/README.md#trips) - start and end time, start and end location, distance (meters), and duration (seconds)

* [Status changes](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/README.md#status-changes) - status of each vehicle, including available, reserved, removed, and non-operational

* [Real-time data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/README.md#realtime-data) - vendors must also expose a public General Bikeshare Feed Specification (GBFS) URL to facilitate customer-facing applications such as trip planners

## Data Aggregation to Protect User Privacy
Though the vehicle and trip data SDOT collects from vendors does not contain personal information associated with an individual (e.g., name, contact information, payment information), SDOT takes proactives steps to further safeguard bike and scooter share users' privacy. 

Here's why:

### A changing understanding of personally identifiable information (PII) 
[PII](https://nacto.org/wp-content/uploads/2019/05/NACTO_IMLA_Managing-Mobility-Data.pdf) is commonly thought to be limited to direct unique personal identifiers such as name, address, social security number, or credit card number. However, all data can become PII depending on how easily and accurately it can be tied to an individual. The U.S. government defines PII as “information that can be used to distinguish or trace an individual’s identity, either alone or when combined with other personal or identifying information that is linked or linkable to a specific individual.”

Geospatial data is, or can become, PII in two ways:

* **Recognizable travel patterns** – Even in anonymous datasets, people can be re-identified from their routine travel patterns – e.g. from home to work, school, stores, or religious institutions. The 2013 Scientific Report article, [“Unique in the Crowd: the privacy bounds of human mobility,”](https://www.nature.com/articles/srep01376) found that, in a dataset of 1.5 million people over 6 months, and using location points triangulated from cellphone towers, “four spatio-temporal points are enough to uniquely identify 95% of the individuals.”
    
* **Combined with other data** – Geospatial mobility data can be combined with other data points to become PII (sometimes referred to as indirect or linked PII). For example, taken by itself, a single geospatial data point like a ride-hail drop-off location is not PII. But, when combined with a phonebook or reverse address look-up service, that data becomes easily linkable to an individual person. 

*For example, in 2014, a researcher requested anonymized taxi geo-location data from NYC Taxi and Limousine Commission under freedom of information laws, mapped them using MapQuest, and was able identify the home addresses of people hailing taxis in front of the Hustler Club between midnight and 6am. Combining a home address with an address look-upwebsite, Facebook and other sources, the researcher was able to find the “property value, ethnicity, relationship status, court records and even a profile picture!” of an individual patron.*

The small number of data points necessary to identify an individual from their travel patterns, the ubiquity of secondary data sets, and the ease with which they can be combined with geospatial trip data to form PII, all mean that both the public and private sector should treat geopspatial trip data as PII for collection, management, storage, and dissemination.

When it comes to mobility, privacy is related to the degree to which an individual trip is synonymous with an individual person. For example, each dockless scooter trip is tied to an individual user and thus broadcasts specific, unique information about an individual person’s behavior. Similarly, when passengers are in a car on an app-based ride-hail or autonomous vehicle trip, that trip is linked to an individual. In contrast, data about ride-hail trips without a passenger (“dead-heading”) and shared trips with fixed stops do not reveal personally identifiable patterns and can be easily shared.

To protect individual privacy while collecting operational data necessary for actively managing the shared micromobility program, SDOT applies the [Mobility Data Privacy and Handling Guidelines](http://www.seattle.gov/Documents/Departments/Tech/Privacy/SDOT_Mobility_Data_Guidelines.pdf) and the below data aggregation steps.

### Data Aggregation Steps
In partnership with Seattle IT, SDOT aggregates data about individual trips to prevent the disclosure of potentially identifying information that can occur in mobility data, while still allowing approximate, useful information to be released and published.

Our data policy aggregates trip data into location and time bins as follows:

* First, the precision of start and end locations have been reduced by truncating latitude and longitude coordinates to two decimal points. This results in a grid of rectangular bins approximately 2,700 ft by 1,800 ft with an area of approximately 0.17 sq. miles. 

* Second, the timestamps are broken into five daypart time bins: AM peak (6-9 AM), PM peak (4-7 PM) , midday, night, and weekend. These windows are based on typical travel patterns.

* Finally, we aggregate trips by location and time bins by quarter. To prevent individual trips from being identified, we drop the location information in any group that consist of less than 3 trips. These sensitive trips are included in the data set as larger groups aggregated only by time bins. Within all groups, travel attributes such as distance traveled and trip duration are averaged.

Example location bins by coordinate truncation to 0.01 degrees latitude and longitude:

![Image of Grids](https://github.com/anthonyaanderson/MDS-Data/blob/main/SeattleGrid.png)

## Other Data Aggregations

### Trip Count & Details
To calculate the daily number of shared mobility trips, we use an aggregated count of the  [MDS Trips Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/trips.json).

Process:
1) Filter the trips to those that happen in Seattle city limits, have a duration longer than 30 seconds, and a distance greater than zero meters. 
2) Pull the travel date by the trip end time. 
3) Aggregate the dataframe into a daily count of trips, sum of trip distance and trip durantion by provider and vehicle type.

<details>
  <summary>Python Code</summary>
  
```python
def get_trip_count(df_trips):
    
    # filter criteria
    df_trips = df_trips[df_trips['TripDuration'] > 30]
    df_trips = df_trips[df_trips['TripDistance'] > 0]
    
    # extract the travel date from the trisp end time.
    df_trips['travel_date'] = df_trips['EndTimeLocal'].apply(lambda x: x.strftime("%Y-%m-%d"))
    df_trips['trip_count'] = 1
    
    # Aggregate dataframe by travel date, provider name, and vehicle type
    df_tripcount = df_trips.groupby(['travel_date','ProviderName','VehicleType'], as_index=False).agg({'trip_count':'sum', 'trip_duration':'sum', 'trip_distance':'sum'})
    
    return df_tripcount
```

</details>

### Fleet Snapshot Methodology (since June 2020)
In order to count the number of shared mobility devices on city streets at any given time, we take a snapshot count every hour on the hour using the [MDS Status Change Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/status_changes.json).

The process to create an hourly snapshot (See code below under "Details"): 
1.	Begin with an array of date-times for every hour of a day. This gives the list of snapshot date-times to check. (*ex. 09/01/2020 1:00AM, 09/01/2020 2:00 AM, 09/01/2020 3:00 AM*)
2.	For each date-time, filter the status changes data set to gather all status changes for the seven days leading up to each date-time. (*For example, if the date-time is 09/01/2020 at 1:00 AM, gather all status changes between 08/25/2020 to 09/01/2020 at 1:00 AM.*)
3.	Create a list of all of the unique devices that appear within that filtered data (for use in checking the status of each device).
4.	For each device in that list, create a data set with only this device’s status changes.
5.	Save only the last status change that device recorded in this data set.
6.	Combine all device statuses for each day-time period into one data group.
7.	Group the data by day-time, provider, vehicle type, event type, and event reason by counting each datapoint.  
 
The snaphot data will have a group of event types per hour. Those event types include or exclude devices from the fleet count as follows:

| Vehicle State  | Event Type | FleetCount |
| ------------- | ------------- | ------------- |
| available  | maintenance_drop_off  | included  | 
| available  | rebalance_drop_off  | included   | 
| available  | user_drop_off  | included  | 
| available  | service_start  | included   | 
| removed  | service_end  | no longer counted   | 
| removed  | maintenance_pick_up  | out for maintenance  | 
| removed  | rebalance_pick_up | out for maintenance  | 
| reserved  | user_pick_up | included   | 
| unavailable  | low_battery | included   | 
| unavailable  | maintenance  | included   | 

<details>
  <summary>Python Code</summary>
  
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
</details>

### Equity Data

To calculate the trips and fleet in our designated equity areas, we use the same process as our [trip count data](https://github.com/anthonyaanderson/Shared-Mobility-Data-Documentation/blob/main/README.md#trip-count--details)  and [fleet size data.](https://github.com/anthonyaanderson/Shared-Mobility-Data-Documentation/blob/main/README.md#fleet-snapshot-methodology-since-june-2020)  

Process:

filter the trips and the status change data to the equity zones.
Then run the trip count and fleet snap shot analysis above on the filtered data. 

<details>
  <summary>Python Code</summary>
  
```python
# Get trips and status changes from datamarts into geodataframes
TripsGDF = gpd.GeoDataFrame(tripsDM, geometry=gpd.points_from_xy(tripsDM.LongitudeStart, tripsDM.LatitudeStart))
sc_gdf = gpd.GeoDataFrame(statuschangesDM, geometry=gpd.points_from_xy(sc.Longitude, sc.Latitude))

# Use a spatial join to join Equity areas to the locatiosn of the trip endpoints and status changes. 
Trip_EA = gpd.sjoin(TripsGDF,EquityAreas, how="left", op="within")
SC_EA = gpd.sjoin(sc_gdf,EquityAreas, how="left", op="within")

# Trips Agg Function
def get_equity_count(df):
    
    df_trips = df
    df_trips['end_time_utc'] = pd.to_datetime(df_trips['EndTimeLocal'])

    df_trips = df_trips[df_trips['TripDuration'] > 30]
    df_trips = df_trips[df_trips['TripDistance'] > 0]
    df_trips['travel_date'] = df_trips['end_time_utc'].apply(lambda x: x.strftime("%Y-%m-%d"))
    
    df_trips['trip_count'] = 1
    df_trips.loc[df_trips['EArea'] == 'Northern', 'Northern'] = 1 
    df_trips.loc[df_trips['EArea'] == 'Central', 'Central'] = 1
    df_trips.loc[df_trips['EArea'] == 'Southern', 'Southern'] = 1
    df_trips.loc[df_trips['EArea'] == 'Southern', 'EquityTotal'] = 1
    df_trips.loc[df_trips['EArea'] == 'Central', 'EquityTotal'] = 1
    df_trips.loc[df_trips['EArea'] == 'Northern', 'EquityTotal'] = 1
    df_equitycount = df_trips.groupby(['travel_date','ProviderName','VehicleType'], as_index=False).agg({'trip_count':'sum','Northern':'sum','Central':'sum','Southern':'sum','EquityTotal':'sum'})
    
    return df_equitycount
    
# Status Change Agg Function
def get_hourlysnapshot(SC, rundate = '10-24-2020'):
  
  dev =[]
  EventTimeLocal =[]
  EventType =[]
  EventTypeReason =[]
  ProviderName =[]
  VehicleType =[]
  EArea = []
  Snaptime = []
  Timestamp = pd.to_datetime('now')
  
  rng = pd.date_range(rundate, periods=24, freq='1H')
  
  for d in rng:
  
    filterSC =  SC[SC['EventTimeLocal'] < d]
    days=7    
    cutoff_date = d - pd.Timedelta(days=days)
    filterSC2 = filterSC[filterSC['EventTimeLocal'] > cutoff_date] 
    Devices = filterSC2.DeviceId.unique()

    for i in Devices:

      IDevice = filterSC2[filterSC2['DeviceId']== i]
      LastStatus = IDevice.iloc[-1:]
      dev.append(i)
      EventTimeLocal.append(LastStatus.iloc[0]["EventTimeLocal"])
      EventType.append(LastStatus.iloc[0]["EventType"])
      EventTypeReason.append(LastStatus.iloc[0]["EventTypeReason"])
      ProviderName.append(LastStatus.iloc[0]["ProviderName"])
      VehicleType.append(LastStatus.iloc[0]["VehicleType"])
      EArea.append(LastStatus.iloc[0]["EArea"])
      Snaptime.append(d)

  df = pd.DataFrame()
  df["Device"] = dev
  df['EventTimeLocal'] = EventTimeLocal
  df['EventType'] = EventType
  df['EventTypeReason'] = EventTypeReason
  df['ProviderName'] = ProviderName
  df['VehicleType'] = VehicleType
  df['EArea'] = EArea
  df['Time'] = Snaptime
  df['count']= 1
  
  Snapshot = df.groupby(['Time','ProviderName','VehicleType', 'EArea', 'EventType', 'EventTypeReason',], as_index=False).agg({'count':'sum'})
 
  return Snapshot
  
# Run joined dataframes through agg functions. 

TripsEAreas = get_equity_count(Trips_EA)

df = get_hourlysnapshot(SC_EA, check)
df.loc[df['EArea'] == 'Northern', 'Northern'] = 1 
df.loc[df['EArea'] == 'Central', 'Central'] = 1
df.loc[df['EArea'] == 'Southern', 'Southern'] = 1
df.loc[df['EArea'] == 'Southern', 'EquityTotal'] = 1
df.loc[df['EArea'] == 'Central', 'EquityTotal'] = 1
df.loc[df['EArea'] == 'Northern', 'EquityTotal'] = 1

FleetEAreas = df.groupby(['Time','ProviderName','VehicleType','EventType', 'EventTypeReason'], as_index=False).agg({'count':'sum','Northern':'sum','Central':'sum','Southern':'sum','EquityTotal':'sum'})

```

</details>

### Ridership

Every month each permmited provider provided the city with ridership data. We use that data to see how many unique riders are using shared devices in the city. 
This data is combined from each months reporting. 

## Archived Data Aggregations
<details>
  <summary>Archived Data Aggregations</summary>


### Fleet Count (Prior to June 2020)
To calculate the size of the bike share fleets in Seattle, we check all status changes for each provider looking back 7 days using the [MDS Status Change Endpoint Data](https://github.com/openmobilityfoundation/mobility-data-specification/blob/main/provider/status_changes.json) 

Status changes with event types that are not "service ends" or "maintenance_pick_up" are classified as "In Service."
Status Changes with the event reason of "maintenance_pick_up"  are classified as "In Maintenance."
<details>
  <summary>Python Code</summary>
  
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

</details>

### Transit Trips
 We are no longer using these calculations.  

</details>
