# Logan's Project 3: Spark Mania

### Base problems data description and insights

Our NAM weather data set is five years of highly comprehensive weather data spread across North America and its surrounding waters; notably, Hawaii and Alaska are not included. There's up to four records per day, at midnight, 6am, 12pm, and/or 6pm. The data schema is fairly consistent besides columns 4, 12, and 17, i.e., 'precipitable_water_entire_atmosphere_single_layer', 'total_cloud_cover_entire_atmosphere_single_layer', 'wind_speed_gust_surface', which are null strings in 2015 and 2016 data, strings of float data in 2017, and actual float data in 2018 and 2019.

(( Note: orion cluster seemed to be in very poor health during my runtime tests ))

Following is a summary of each problem and their runtimes.


### Strangely Snowy

Runtime: 14 min

Problem: Find locations that are snowy while their surroundings are not.

Summary:
- Select location and snow data from the dataset
- Key records by geohashes calculated from their respective latitude longitude data
- Compute the average of snow level for all records respective to their geohashes
- Find the closest three geohashes in the data set:
    - Filter out geohashes that don't start with the same character (same 1/32nd of the globe)
    - Calculate the distance to all data set geohashes in the 1/32nd
    - Take the lowest three
- Report the location and snowy index of data points with the most snow,
  and the average snowy index of their closest three neighbors

Insights: Using the RDD API is the way to go. Finding 'closest geohash to the left' is ridiculously difficult, even with package functions that spit out the geohash string of the immediate adjacent geohash; the lat longs of data collection stations in this data set are effectively random with respect to geohash boxes. I digressed to closest distance without directionality, which still took a ton of comp time.


### Climate Chart

Runtime: 8 min

Problem: Given a geohash, print a chart that includes monthly high, low, average temperatures and monthly average rainfall.

Summary:
- Select time, location, temperature, and precipitation data from the dataset
- Get user input of a valid geohash in the dataset
- Find all records that fall within the given geohash
- Sort records by month
- Aggregate monthly temperature data into minimums, maximums, and averages
- Aggregate monthly precipitation data into averages
- Make a pretty plot by plagiarizing teach

Insights: How to do aggregation in distributed RDD's, how in the world is weather measured and represented around the world


### Travel Startup

Runtime: 8 min

Problem: Pick five regions, compute a custom 'comfort index' for each month of each region, find the best month to visit each region, and include some media to convince a person to visit these regions.

Summary:
- Pick five places and google their coordinates
  (Santa Cruz, Cedar City UT, Freeport Bahamas, Niagara Falls NY, Hill City / Mount Rushmore SD)
- Set temperature, humidity, and precipitation preferences and relative importances
  (variables temp_pref, hmdy_delta, etc.; I set them to cool, medium
   humidity, and low precipitation with more sensitivity on humidity)
- Select time, location, temperature, humidity, and precipitation data from the dataset
- Find the closest geohashes to the chosen five places in the dataset and filter out all others
- Compute monthly averages for temperature, humidity, and precipitation respective to geohashes
- Calculate 'comfort indices' per geohash per month by taking the root-mean-square
  of ((value - pref) / delta) for all features (temp, humidity, precip)
- Report the month with the lowest root-mean-square value for each place
- Display some images of the sights to see at each place

Insights: Took me a while to realize the geohashes of Santa Cruz and whatnot weren't in the dataset. I think I made a bomb comfort index by including relative sensitivity as a sort of personal standard deviation measure.


### Escaping the Fog

Runtime: 4 min

Problem: Find the least foggy locations and show them on a map.

Summary:
- Select location and visibility data from the dataset
- Compute geohashes and respective global average visibility
- Find the lowest visibility geohashes and filter out the rest
- Use folium to load and display a world map
- Place pins on the world map for each geohash found

Insights: Was confused for a while why cold peninsulas and marshes have no fog, then realized I needed to invert my sort direction. Most of the run time seems to come from populating the map and pins.


### SolarWind, inc.

Runtime: 6 min

Problem: Find the best locations for solar farms, wind farms, and combo solar-wind farms.

Summary:
- Select location, visibility, and wind pressure data from the dataset
  (wind speed data is missing for 2015 and 2016 so I used wind pressure instead)
- Get global mean and standard deviation for visibility and wind pressure
- Find the locations that have the highest positive deviation from visibility or wind pressure
  (list of highest visibility locations and separate list of highest wind locations)
- Find the locations that have the highest summary positive deviations
- Plot some color-coded pins on a map from the highest data points in each of the three lists

Insights: Nothing too wild. Solar farms tend towards the equator, however wind farms are fairly spread out, looks mostly coastal, flatlands, or ocean. Combo farms tend to match wind farms more than solar farms.


### Climate Change

Runtime: 4 min

Problem: Calculate correlation coefficients of various weather measures vs temperature over time per geohash in the dataset. Discern outliers and analyze them.

Summary:
- Select time, location, wind pressure, humidity, temperature,
  precipitation, vegetation, and visibility data from the dataset
- Key all records to (geohash, year-month) and average all features per geohash / year-month
- Aggregate record features into (geohash, features_lists), where features_lists
  is lists for each feature per geohash in chronological order
- Calculate features_lists into correlation coefficients
- Find the highest / lowest correlation coefficients globally and feature-wise

Insights: Pretty hard to do a complex calculation like covariance and correlation coefficient in an RDD. I used numpy arrays in RDD's to get distributed multi-dimensional data, but I'm sure something like a couple aggregateByKey() calls would perform better because Spark can likely distribute such problems better. It seems vegetation is almost globally highly positively correlated with rise in temperature, omitting over-water regions and a few negative correlated points. Precipitation is also globally highly positively correlated with temperature. Surprisingly, humidity is slightly negatively correlated.




## Self-Guided Analysis: Books

### Base problems data description and insights

Dataset Description: Books. Title and Author provided as the first two lines in all entries; otherwise relatively unordered data, mostly just bags of words.

Following is a summary of each problem and their runtimes.


### Collocations

Runtime: 20 s

Problem: Find words that more often that not come in distinct pairs, or collocations. 

Summary:
- Split corpus into 10 dataframes
  (Putting them all in one I get 'sending too much data' jupyter interrupts)
- Filter and flatmap all words in each dataframe
- Get wordcounts for each dataframe, accumulate them into one dataframe, and collect()
- Filter, flatmap, and pair all words in each dataframe
  (Basically a 2-word windows slid across all text of all books)
- Get counts for each word pair, accumulate them into one dataframe, and collect()
- Do some numpy indexing magic and output the top contenders

Insights: Wow, strings are ridiculous in RDD's. I'm very surprised the runtime is so quick; I'm using all 1000 of the non-huge books in /bigdata/datasets/books.dataset and it's a big chunk of data. Also surprisingly, most of the results make sense, though there are a lot of first-last names.


### Lexical Prowess

Runtime: 6 s

Problem: Discern a measure of lexical sophistry for books in the dataset.

Summary:
- Split corpus into 10 dataframes
- Calculate book length, average word length, and number of distinct words respective to each book
- Do some numpy magic to calculate means and standard deviations for features across all books
- Assign books a sophistication score based on their features - mean / std dev
- Output the books with the highest scores
