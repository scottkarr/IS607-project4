# MongoDB-ETL
Scott Karr  
April 16, 2016  
This exercise walks through the process of extracting the R "flights"" database
that was preloaded into Postgres, tranforming the data in R and loading the data
into MongoDB.  The transformation process had performance limitions on a Mac
laptop running OSX with the flights table that contained 300K rows.  

The principal disadvantage encountered in using NoSQL was conversion to and
from the BSON object structure into a workable form in R.

The principal advantages of NoSQL include being able to handle:

    Large volumes of structured, semi-structured, and unstructured data.
    Agile sprints, quick iteration, and frequent code pushes.
    Object-oriented programming that is easy to use and flexible.
    Efficient, scale-out architecture instead of expensive, monolithic architecture.
https://www.mongodb.com/scale/advantages-of-nosql

#load packages


#connect to postgres flights db

```r
#assign connection parms and connect to flight db in Postgres
dbname <- "flights"
dbuser <- "postgres"
dbpass <- "postgres"
dbhost <- "localhost"
dbport <- 5432
drv <- dbDriver("PostgreSQL")
con <- dbConnect(drv, host=dbhost, port=dbport, dbname=dbname,user=dbuser, password=dbpass)
```

#etl flights-2-dataframes

```r
#collect flights database table results
#get airlines
query <- dbSendQuery(
            con, query <- "select * from public.airlines")
df_airlines <- fetch(query,n=-1)
query <- dbSendQuery(
            con, query <- "select * from public.airports")
df_airports <- fetch(query,n=-1)
df_airports <- data.frame(df_airports, StringAsFactor=F)
query <- dbSendQuery(
            con, query <- "select f.* 
                            from  public.flights f, public.airlines a
                            where 1=1
                            and   f.carrier = a.carrier
                            and   a.carrier in('US')
                            "
         )
df_flights <- fetch(query,n=-1)
query <- dbSendQuery(
            con, query <- "select * from public.planes")
df_planes <- fetch(query,n=-1)
query <- dbSendQuery(
            con, query <- "select * from public.weather")
df_weather <- fetch(query,n=-1)
#disconnect
dbDisconnect(con)
```

```
## [1] TRUE
```

#show airlines dataframe

```r
kable(df_airlines,align='l')
```



carrier   name                        
--------  ----------------------------
9E        Endeavor Air Inc.           
AA        American Airlines Inc.      
AS        Alaska Airlines Inc.        
B6        JetBlue Airways             
DL        Delta Air Lines Inc.        
EV        ExpressJet Airlines Inc.    
F9        Frontier Airlines Inc.      
FL        AirTran Airways Corporation 
HA        Hawaiian Airlines Inc.      
MQ        Envoy Air                   
OO        SkyWest Airlines Inc.       
UA        United Air Lines Inc.       
US        US Airways Inc.             
VX        Virgin America              
WN        Southwest Airlines Co.      
YV        Mesa Airlines Inc.          

#connect to mongodb nosql

```r
#connect to Mongo on localhost
m <- mongo.create(host = "localhost")
mongo.is.connected(m)
```

```
## [1] TRUE
```

```r
#version of MongoDB may be at issue with no data returned here!
db <- "flights"
mongo.get.database.collections(m, db = db)
```

```
## character(0)
```

#etl dataframes-2-mongodb

```r
#convert the dataframe to BSON
b_airlines <- mongo.bson.from.df(df_airlines) 
b_airports <- mongo.bson.from.df(df_airports) 
b_flights <- mongo.bson.from.df(df_flights) 
b_planes <- mongo.bson.from.df(df_planes) 
b_weather <- mongo.bson.from.df(df_weather)
#define namespace
#then load the mongo collections using
if (mongo.is.connected(m) == TRUE) {
    #airlines
    ns <- paste(db, "airlines", sep=".")
    lst <- split(df_airlines, rownames(df_airlines))
    bson_lst <- lapply(lst, mongo.bson.from.list)
    mongo.drop(m,ns)
    mongo.insert.batch(mongo = m, ns = "flights.airlines", lst = bson_lst)  
    #airports    
    ns <- paste(db, "airports", sep=".")
    lst <- split(df_airports, rownames(df_airports))
    bson_lst <- lapply(lst, mongo.bson.from.list)
    mongo.drop(m,ns)
    mongo.insert.batch(mongo = m, ns = "flights.airports", lst = bson_lst)  
    #flights
    ns <- paste(db, "flights", sep=".")
    lst <- split(df_flights, rownames(df_flights))
    bson_lst <- lapply(lst, mongo.bson.from.list)
    mongo.drop(m,ns)
    mongo.insert.batch(mongo = m, ns = "flights.airports", lst = bson_lst)  
    #planes
    ns <- paste(db, "planes", sep=".")
    lst <- split(df_planes, rownames(df_planes))
    bson_lst <- lapply(lst, mongo.bson.from.list)
    mongo.drop(m,ns)
    mongo.insert.batch(mongo = m, ns = "flights.planes", lst = bson_lst)  
    #weather
    ns <- paste(db, "weather", sep=".")
    lst <- split(df_weather, rownames(df_weather))
    bson_lst <- lapply(lst, mongo.bson.from.list)
    mongo.drop(m,ns)
    mongo.insert.batch(mongo = m, ns = "flights.weather", lst = bson_lst)  
}
```

```
## [1] TRUE
```

#re-retrieve dataframes from mongodb

```r
#get dataframes back out of mongodb
if(mongo.is.connected(m) == TRUE) {
  #list databases
  mongo.get.databases(m)
  #get collections into a data frame
  #airlines
  ns <- paste(db, "airlines", sep=".")
  c_airlines <- mongo.bson.to.list(mongo.bson.from.list(mongo.find.batch(m, ns)))
  #airports
  ns <- paste(db, "airports", sep=".")
  c_airports <- mongo.bson.from.list(mongo.find.batch(m, ns))
  #planes
  ns <- paste(db, "flights", sep=".")
  c_planes <- mongo.bson.from.list(mongo.find.batch(m, ns))
  #flights
  ns <- paste(db, "planes", sep=".")
  c_flights <- mongo.bson.from.list(mongo.find.batch(m, ns))
  #weather
  ns <- paste(db, "weather", sep=".")
  c_weather <- mongo.bson.from.list(mongo.find.batch(m, ns))
}
```

#show airlines dataframe

```r
# Creating many data frames embedding each of our results
l_airlines <- lapply(c_airlines, data.frame, stringsAsFactors = FALSE)
df_airlines <- data.frame(rbind_all(l_airlines))
kable(df_airlines[,c(2,3)],align='l')
```



carrier   name                        
--------  ----------------------------
9E        Endeavor Air Inc.           
MQ        Envoy Air                   
OO        SkyWest Airlines Inc.       
UA        United Air Lines Inc.       
US        US Airways Inc.             
VX        Virgin America              
WN        Southwest Airlines Co.      
YV        Mesa Airlines Inc.          
AA        American Airlines Inc.      
AS        Alaska Airlines Inc.        
B6        JetBlue Airways             
DL        Delta Air Lines Inc.        
EV        ExpressJet Airlines Inc.    
F9        Frontier Airlines Inc.      
FL        AirTran Airways Corporation 
HA        Hawaiian Airlines Inc.      
