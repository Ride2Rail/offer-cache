# Offer Cache

**Version:** 1.0 

**Date:** 22.06.2021

Reference for the _Offer Cache_ schema adopted by the _OfferEnricherAndRanker_. The schema defines how information about offers associated to a specific request are stored in a [Redis](https://redis.io/) instance.

## Offer Cache Schema

Redis is a simple key-value store where values can have a specific data type. Available data types are documented at this [link](https://redis.io/topics/data-types-intro).
The _Offer Cache_ schema defines the keys and the the data type for each value stored by the different components of the _OfferEnricherAndRanker_.

Assuming that each request received by the _OfferEnricherAndRanker_ is associated with a specific *request_id*, the schema is designed so that it is possible to retrieve from the  _Offer Cache_ all the information associated to it and to the related travel offers. 

Information is stored at different levels (Request, Offer, OfferItem. Leg) using composite keys. The schema is presented in this document specifying the component computing and writing the described data to the _Offer Cache_.

### Offer Parser

For each level a prefix to compose the key is specified. Mandatory information are marked with an asterisk.

#### Request-level information (Key prefix `<request_id>:` )
- *user_id*: Identifier of the user performing the Mobility Request. String.
- *traveller_id*: Identifier of the traveller profile used by the user to perform the Mobility Request. String.
- *start_time*: Desired starting time specified by the user in the Mobility Request. String formatted as xsd:datetime.
- *end_time*: Desired ending time specified by the user in the Mobility Request. String formatted as xsd:datetime.
- *start_point*: Desired origin coordinates specified by the user in the Mobility Request. String formatted as a GeoJson Point.
- *end_point*: Desired destination coordinates specified by the user in the Mobility Request. String formatted as GeoJson Point.
- *cycling_dist_to_stop*: The maximum cycling distance user is willing to cycle to the requested stop or between two stops. String formatted as xsd:NonNegativeDecimal (xs:decimal)
- *walking_dist_to_stop*: The maximum walking distance user is willing to walk to the requested stop or between two stops. String formatted as xsd:NonNegativeDecimal (xs:decimal).
- *walking_speed*: The maximum walking speed of the user. String formatted as xsd:IndividualTransportSpeed.
- *cycling_speed*: The maximum cycling speed of the user. String formatted as xsd:IndividualTransportSpeed.
- *driving_speed*: The maximum driving speed of the user. String formatted as xsd:IndividualTransportSpeed.
- *max_transfers*: The maximum number of transfers the user wants to perform during the whole trip xs:nonNegativeInteger.
- *expected_duration*: The minumum transfer time that the user expects when changing the transportation modes. String formatted as xs:nonNegativeInteger.
- *via_locations*: List of coordinates through which the user would like to travel. List of strings tring formatted as a GeoJson Point.
- *offers**: List of `<offer_id>` computed for the Mobility Request. List of strings representing the `<offer_id>` in keys.

#### Offer-level information (Prefix `<request_id>`:`<offer_id>`: )
- *duration**: Duration of the Trip associated to the offer. String formatted as xsd:duration.
- *start_time**: Starting time of the Trip associated to the offer. String formatted as xsd:datetime.
- *end_time**: Ending time of the Trip associated to the offer. String formatted as xsd:datetime.
- *num_interchanges**: Number of interchanges of the Trip associated to the offer. Integer.
- *length*: Length of the trip in meters. Integer representing meters.
- *bookable_total*: Total of all offer items that are actually bookable. HashMap containing two keys:
  - *value** : Integer (last two digits are decimal, e.g., 6215 EUR represents 62,15 euro).
  - *currency* : String formatted as ISO4217 currency code, e.g. EUR for Euro.
- *complete_total**: Total of all offer items. HashMap containing two keys as for *bookable_total*.
- *legs**: Ordered list of `<leg_id>` composing the Trip associated to the offer. List of strings representing the `<leg_id>` in keys.
- *offer_items*: List of `<item_id>` composing the offer. An offer can have no offer item if the associated legs can be performed without any entitlement. List of strings representing `<item_id>` keys.
	
#### Offer-Item-level information  (Prefix `<request_id>`:`<offer_id>`:`<item_id>`: )
- *price**: Price for the offer item. HashMap containing two keys as for *bookable_total*.
- *name*: Printable ticket name. String.
- *fares_authority_ref*: Reference to a Fares Authority defining the price for the offer item. String.
- *fares_authority_text*: Textual description or name of Fares Authority defining the price for the offer item. String.
- *legs**: List of `<leg_id>` covered by this offer item. List of strings representing the `<leg_id>` in keys.
	
#### Leg-level information  (Prefix `<request_id>`:`<offer_id>`:`<leg_id>`: )
- *leg_type**: Type of leg. String. Admissible values are: _timed_ (timetabled service with fixed start and end time), _continuous_ (can be performed in the time window between start and end time), _ridesharing_ (continuous leg associated to a ridesharing service).
- *start_time**: Starting time of the Leg associated to the offer.String formatted as xsd:datetime.
- *end_time**: Ending time of the Leg associated to the offer. String formatted as xsd:datetime.
- *duration**: Duration of the Leg associated to the offer. String formatted as xsd:duration. If _continuous_ can be different from (end_time - start_time) because a time window to perform the leg can be defined.
- *length*: Length of the leg in meters. Not always available. Integer representing meters
- *transportation_mode**: Transportation Mode for the Leg. String. Admissible values are: 
  - for _timed_ legs: all, unknown, air, bus, trolleyBus, tram, coach, rail, intercityRail, urbanRail, metro, water, cableway, funicular, taxi
  - for _continuous_ legs: walk, cycle, taxi, 'self-drive-car, motorcycle, truck
  - for _ridesharing_ legs: others-drive-car
- *leg_stops*: Coordinates of the stops performed during the leg. String formatted as GeoJson LineString. At least coordinates of starting and ending point.
- *leg_track*: Coordinates of the track followed during the leg. String formatted as GeoJson LineString.
- *travel_expert*: Identifier of the Travel Expert associated to the leg. String.
- *line* [only for _timed_ legs]: Identifier of the Line for the service of a _timed_ leg. String.
- *journey* [only for _timed_ legs]:Identifier of the Service Journey for the service of a _timed_ leg. String.
- *driver* [only for _ridesharing_ legs]: HashMap containing two keys:
  - *id*: Identifier of the Driver for the service of a _ridesharing_ leg.
	- *text*: Text description associated to the driver offering the leg.
- *vehicle* [only for _ridesharing_ legs]: HashMap containing two keys:
  - *id*: Identifier of the Vehicle for the service of a _ridesharing_ leg
  - *text*: Text description associated to the vehicle used to perform the leg.
- *passenger_ref* [only for ridesharing legs]: Identifier of the Passenger for the service of a _ridesharing_ leg. String.

#### Additional Leg Information (Prefix `<request_id>`:`<offer_id>`:`<leg_id>`:)

The following optional information are stored at a Leg-level. For each key the expected range for the value is reported [value type, lower bound, upper bound].

- *likelihood_of_delays* [float, 0, 1]: Number of times that previous users reported delays over the total number of times that users used the offer 
- *last_minute_changes* [float, 0, 1]: Measurement of likelihood of last-minute changes from the TSP. This can be expressed as a probability from 0 to 1. These probability does not measure the delays deriving from the changes, which is measured by "likelihood of delays" above.
- *frequency_of_service* [int, 1, 20]: Number of scheduled trips per hour (depending on time and day of the trip). 
- *user_feedback* [float, 1, 5]: Average value reported by users about experienced comfort on five stars scale.
- *cleanliness* [float, 1, 5]: Average value reported by users about experienced cleanlinnes (of stops, stations and vehicles) used by a given TSP on five stars scale.
- *seating_quality* [float, 1, 5]: Average value reported by users about experienced seats comfort of vehicles used by a given TSP on five stars scale.
- *space_available* [float, 1, 5]: Average value reported by users about experienced available space in vehicles used by a given TSP on five stars scale.
- *silence_area_presence* [int, 0, 1]: 1- if there is a silence area in a given train,  0- otherwise.
- *privacy_level* [float, 1, 5]: Average value reported by users about experienced privacy level in vehicles used by a given TSP on five stars scale.
- *bike_on_board* [int, 0, 1]: 1- if a bike can be loaded to the PuT vehicle, 0- otherwise.
- *business_area_presence* [int, 0, 1]: Average value reported by users about experiences of business area is vehicles used by a given TSP on five stars scale.
- *internet_availability*  [int, 0, 1]: Average value reported by users about experienced wifi quality offered by a given TSP on five stars scale.
- *plugs_or_charging_points* [int, 0, 1]: 1- if plugs/charging points are available; 0- otherwise.
- *safety_features* [float, 1, 5]: Average value reported by users about the safety of the trip perceived as passangers on five stars scale.
- *passenger_feedback*  [float, 1, 5]: Average value reported by users on the overall travel experience value reported on five stars scale.
- *ride_smoothness* [float, 1, 5]: Average value reported by users about smoothness (e.g. presence and intensity of sudden changes in the speed and direction of the vehicle) as perceived as passengers on the five stars scale.
  
Keys below are only for _ridesharing_ legs:
- *number_of_persons_sharing_trip*  [int, 1, 20]: Number of persons in the vehicle.
- *shared_with_other_passengers* [int, 0, 1]: 1- if the trip is shared with other passengers (besides the driver); 0- otherwise.
- *can_share_cost* [int, 0, 1]: 1- if for the ride-sharing offer is possible to share the costs among several passengers, 0- otherwise.
- *vehicle_age* [int, 0, 25]: Number of years that the vehicle has been operative.
- *certified_driver* [int, 0, 1]: 1- if the driver performed X trips and did not receive complaints from previous users; 0- otherwise.
- *driver_license_issue_date* [xsd:datetime]: Date of issue of the driver license.
- *repeated_trip* [int, 0, 1]: 1- if the driver has done the trip more times than a certain a number (X times); 0- otherwise.

### Feature Collector

Feature collectors compute normalised determinant factors for the offer storing them at the Offer-level.

#### price-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *ticket_coverage*: (minmax/z-) score calculated by considering the fraction of the trip duration covered by ticket(s). Calculated as sum of leg durations comprehensed by ticket(s) divided by sum of all leg durations.
- *total_price*: (minmax/z-) score calculated by considering aggregated prices in EUR.

#### tsp-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *can_share_cost*: (minmax/z-) score calculated by considering aggregated can_share_cost attributes.
- *cleanliness*: (minmax/z-) score calculated by considering aggregated cleanliness attribute values.
- *space_available*: (minmax/z-) score calculated by considering aggregated space_available attribute values.
- *ride_smoothness*: (minmax/z-) score calculated by considering aggregated space_available attribute values.
- *seating_quality*: (minmax/z-) score calculated by considering aggregated ride_smoothness attribute values.
- *internet_availability*: (minmax/z-) score calculated by considering aggregated internet_availability attribute values.
- *plugs_or_charging_points*: (minmax/z-) score calculated by considering aggregated plugs_or_charging_points attribute values.
- *silence_area_presence*: (minmax/z-) score calculated by considering aggregated silence_area_presence attribute valuesv
- *privacy_level*: (minmax/z-) score calculated by considering aggregated privacy_level attribute values.
- *user_feedback*: (minmax/z-) score calculated by considering aggregated user_feedback attribute values.
- *bike_on_board*: (minmax/z-) score calculated by considering aggregated bike_on_board attribute values.
- *likelihood_of_delays*: (minmax/z-) score calculated by considering aggregated likelihood_of_delays attribute values.
- *last_minute_changes*: (minmax/z-) score calculated by considering aggregated last_minute_changes attribute values.
- *frequency_of_service*: (minmax/z-) score calculated by considering aggregated frequency_of_service attribute values.
- *business_area_presence*: (minmax/z-) score calculated by considering aggregated business_area_presence attribute values.

#### traffic-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *traffic*: (minmax/z-) :  ratio : Duration with current Flow / Duration with Free Flow. The higher the ratio the worst the traffic condition.
 
#### environmental-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *co2_per_km_offer*: (minmax/z-) : The total gCO2 this offer cause weighted by the distance of each leg.
- *total_co2_offer*: (minmax/z-) :    The total gCO2 this offer cause.

#### position-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *road_dist_norm*: (minmax/z-) : The total distance performed by transport modes using a road (car, bus, motorbike).
- *total_stops_norm*: (minmax/z-) : The total number of stops this offer have.
- *total_legs_norm*: (minmax/z-) : The total number of legs this offer have.

#### active-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *leg_fraction*: (minmax/z-) : Calculate the number of legs performed by walk of bike divided by the total number of legs.
- *total_walk_distance*: (minmax/z-) : The distance traveled by walk.
- *bike_walk_distance*: (minmax/z-) : The distance traveled with bike or walk.
- *bike_walk_legs*: (minmax/z-) : The number of legs by walk or bike.

#### weather-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *weather*: (minmax/z-) : score giving a probability of delay. Calculated from the weather scenarios.

#### panoramic-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *panoramic*: (minmax/z-) : total number of relevant spots (historical sites, monuments and landscapes) that can be found within an area of 100 m radius around the starting and ending coordinates of each leg.

#### time-fc (Prefix `<request_id>`:`<offer_id>`:`df`:)
- *duration*: (minmax/z-) overall trip duration (in minutes), including all connections. It is computed as scheduled arrival time – scheduled departure time (obtained from TRIAS).
- *time_to_departure*: (minmax/z-) waiting time (in minutes) between the moment the user looks for trips and the corresponding departure time of each offer. Calculated as scheduled departure time – current time
- *waiting_time*: (minmax/z-) waiting time between trip legs (in minutes). Calculated as the sum of time intervals between consecutive legs.
- *rush_overlap*: (minmax/z-) percentage of the *duration* that overlaps with a rush-hour interval (defined by deafult as 8am-10am and 17pm-20pm).

#### geolocation-fc (Prefix `<request_id>`:`<offer_id>`:`city_coordinates`:)
 - *city_coordinates*: (string) city located at the given coordinates. The lat:lon coordinates are used as the key and the city as the value. The city is obtained by reverse geolocation using the Nominatim API.

### OC Core 
Final scores computed as a result of the offer categorization process for each offer and offer category (Prefix `<request_id>`:`<offer_id>`:).

- *categories**: Final scores computed as a result of the offer categorization process for each offer category. HashMap containing the scores for each offer category. Keys are: quick, reliable, cheap, comfortable, door_to_door, environmentally_friendly, short, multitasking, social, panoramic, healthy.
  - `<category-key>`: Float representing the computed membership score of the offer for the offer category specified by the key.

## Deployment
  
A simple docker-compose file is provided to deploy a Redis instance for the `offer-cache`. Persistance is enabled mounting the `data` folder as a volume in the container, port `6379` is exposed also on `localhost`. The following command can be executed to launch the `offer-cache`.

```bash
$ docker-compose up
```

While the container is running, it is possible to access a `redis-cli` bash executing the following command.

```bash
$ docker exec -it cache redis-cli
```
