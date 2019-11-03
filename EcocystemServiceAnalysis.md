---
title: "Ecosystem Services Analysis"
header-includes:
   - \usepackage{dplyr}
   - \usepackage{xtable}
   - \usepackage{stringr}
   - \usepackage{ggplot2}
output: 
  html_document:
    keep_md: true
---



## Introduction
The natural world provides human society with a huge variety of services, many of which often go unnonticed until the habitat is no longer there.  These services, known as ecosystem services, can relate to direct economic value (such as the provision of raw materials), food security, clean water provision, human health (mental and physical), or protection against natural disasters, including flood protection or carbon sequestration.  To help make the value of intact ecosystems clear, some studies have attempted to estimate the monetary value of certain ecosystems through the scientific study of individual locations or wider areas.  This value is often estimated either through the economic value of services provided by these ecosystems (as is the case with food provision or raw materials), or through the estimated costs that would be incurred if the ecosystem were to disappear (for example the cost of flood defences).

The Ecosystem Services Partnership provides the Ecosystem Service Valuation Database (https://www.es-partnership.org/services/data-knowledge-sharing/ecosystem-service-valuation-database/).  This database estimates the monetary values of ecosystem services provided by over 300 case studies, and can be used as a good indication of how valuable individual ecosystems are across the globe.  In turn, data extracted from this database can be used to illustrate the value of natural spaces, particularly in cases where natural spaces are being threatened with destruction for human use, for example farming or housing.

This script tidies and standardizes the data from the ESVD TEEB database, and provides plots that illustrate the most valuable biomes, or ecosystem type, in terms of the monetary value of the ecosystem services they provide, as well as the most valuable services provided by these ecosystems worldwide.

## Data Processing

```r
rm(list=ls())

library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
library(xtable)
library(stringr)
library(ggplot2)
setwd('C:/Users/jonny/Documents/DataIncubator')
```

Now we can read in the data, and make the data more amenable to processing (i.e. converting the monetary values into numeric values and changing the data to a tibble rather than a dataframe).


```r
headers = read.csv('ESVD-TEEB-database.csv',skip=4,header=FALSE,nrows=1,as.is=T) # Define the headers from the data
df = read.csv('ESVD-TEEB-database.csv',skip=6,header=FALSE) # Read in the data WITHOUT the headers
colnames(df)=headers # Set the headers of the dataframe to be equal to the 'headers' variable
df$Value = as.numeric(levels(df$Value))[df$Value] # Convert the monetary values to a numeric value
```

```
## Warning: NAs introduced by coercion
```

```r
df=as_tibble(df) # Convert to a tibble
```

The data contains information from a wide range of different currencies, so we need to convert these values to a standard currency.  In this case we'll use the US Dollar (USD).  To do this, we need to initialise a named vector, which assigns a conversion factor to each currency type.  We'll then apply the correct conversion factor to each value using the lapply() and mutate() functions.


```r
# Create a named vector to assign currency conversion factors to the data
ConvFactor<-
  c("Australian Dollar"=0.69,
    "Austrian Shilling"=0.08,
    "Botswana Pula"=0.092,
    "British Pound"=1.29,
    "Cambodian Riel"=0.0002,
    "Canadian Dollar"=0.76,
    "Chinese Yuan/Renminbi"=0.14,
    "Danish Krone"=0.15,
    "Djibouti Franc"=0.0056,
    "El Salvador Colon"=0.11,
    "Eritrean nakfa"=0.06667,
    "Euro"=1.11,
    "Indian Rupee"=0.014,
    "Italian Lira"=0.000001,
    "Japanese Yen"=0.0092,
    "Kazakhstan Tenge"=0.0026,
    "Mongolian Tugrik"=0.00037,
    "Nepalese Rupee"=0.0088,
    "New Zealand Dollar"=0.64,
    "Nigerian Naira"=0.0028,
    "Pakistan Rupee"=0.0064,
    "Peruvian Nuevo Sol"=0.3,
    "Philippine Peso"=0.02,
    "Samoan Tala"=0.374266,
    "South-Korean Won"=0.00086,
    "South African Rand"=0.068,
    "Sri Lanka Rupee"=0.0055,
    "Swedish Krona"=0.1,
    "Tanzanian Shilling"=0.00043,
    "Thai Baht"=0.033,
    "Uganda Shilling"=0.00027,
    "US Dollar"=1.0,
    "Vietnamese Dong"=0.000043,
    "West African CFA Franc (BCEAO)"=0.00169398)

# Apply the appropriate currency conversion factor to the data
ConvCurrFac=lapply(df$Currency, function(x) unname(ConvFactor[x]))
df$ConvFactor<-as.numeric(ConvCurrFac)

# Now incorporate this into the dataframe
df<-mutate(df,USDvalue=Value*ConvFactor)
```

Many of the data are now in the format of USD per hectare per year, but there are a small number of entries which are simply listing a flat annual value (i.e. not per hectare) or use other measures (such as per household, or per visitor).  We need to convert the factors that we can into per-hectare values, and remove any values which cannot be modified like this.


```r
# Remove any values which are not annual values
df_annual<-filter(df,ValueType=="Annual" |
                    ValueType=="Annualized NPV" |
                    ValueType=="Annual (Range)")

# Remove any values unsuitable for this analysis:
# i.e. those which do not use per hectare values, or cannot be converted
# due to a lack of service area recording
df_anPerHa<-filter(df_annual,str_detect(Unit,"person")==FALSE &
                     str_detect(Unit,"household")==FALSE)
df_anPerHa<-filter(df_anPerHa,str_detect(Unit,"ha")==TRUE |
                   (str_detect(Unit,"ha")==FALSE & is.na(ServiceArea)))

# Define a function which will divide values by their service area (in hectares)
hectarize <- function(unit,serviceArea,serviceValue){
  if(str_detect(unit,"ha")!=TRUE){
    hectarizedValue=serviceValue/serviceArea
    return(hectarizedValue)
  }
  else{
    return(serviceValue)
  }
}
  
## Convert to per hectare
USDPerHa <- df_anPerHa %>% rowwise %>% do(i=hectarize(.$Unit,.$ServiceArea,
                                               .$USDvalue))
df_anPerHa$USDPerHa=unname(unlist(USDPerHa))
rm(USDPerHa)

# Remove any values where the USDPerHa value is invalid
df_anPerHa=filter(df_anPerHa,!is.na(USDPerHa))
```

Now that we have the data in a standardised format, we can begin plotting the data.

## Results

We're interested in which biomes are the most beneficial to us, and what services these ecosystems provide.  Firstly, let's find the five most valuable biomes (by their USD per hectare per year value), and determine what services these ecosystems provide.  To make the plotting of the data more succint and coherent, we'll group the wide variety of services into a smaller subset.  Namely:
Human happiness - Services relating to spirituality, culture, recreational or aesthetic services.
Food and Water - Services relating to food and water security, including pollination, crop genetic diversity and water filtration.
Health - Services relating to human health, including medicines, air quality improvements or waste treatment.
Weather Protection - Services relating to the protection of human habitation from erosion or natural disasters.
Economic and Energy - Services which directly contribute to economic value, such as raw materials or energy generation.
Other - Services which fall outside these other subsets.


```r
## Which services are the most beneficial?
# Group by the Biome
byEco <- group_by(df_anPerHa,Biome)

# Find the sum of the values provided by these ecosystems
byEco_sum=summarise(byEco,Total=sum(USDPerHa))
# Order by the value provided and pull out the top 10
byEco_Ordered=byEco_sum[order(-byEco_sum$Total),]
byEco_top5=byEco_Ordered[1:5,]

## Subset the data into only the most valuable ecosystems
# Extract the names of the most valuable ecosystem
mostValuable=as.list(as.character(byEco_top5$Biome))

# Filter the data
byEcoTop<-filter(byEco,match(Biome,mostValuable)!=0)
byEcoTop$ESService=as.character(byEcoTop$ESService)
byEcoTop$ESService=as.factor(byEcoTop$ESService)

# Generalise the Service Provided to improve the coherency of the plot
ESgeneralise <- function(service){
  # Define the general groups
  happiness=c("Aesthetic","Ornamental","Spiritual","Inspiration","Cultural service [general]","Recreation","Cognitive")
  foodSecurity=c("Food","BioControl","Genepool","Genetic","Pollination","Soil fertility","Water","Water flows")
  health=c("Medical","Air quality","Waste")
  disasterProtection=c("Climate","Erosion","Extreme events","Nursery","Regulating service [general]")
  economicEnergy=c("TEV","Various","Provisioning service [general]","Energy","Raw materials")
  if(!is.na(match(service,happiness))){
    return("Human Happiness")
  }
  else if(!is.na(match(service,foodSecurity))){
    return("Food and Water")
  }
  else if(!is.na(match(service,health))){
    return("Health")
  }
  else if(!is.na(match(service,disasterProtection))){
    return("Weather Protection")
  }
  else if(!is.na(match(service,economicEnergy))){
    return("Economic and Energy")
  }
  else{
    return("Other")
  }
}

# Generalise the types of ecosystem service
byEcoTop$Service=mapply(ESgeneralise,byEcoTop$ESService)

# Plot the most valuable biomes, their value, and the services provided by them
ggplot(byEcoTop, aes(fill=Service, y=USDPerHa, x=Biome)) +
  geom_bar(position="stack", stat="identity") +
  theme(axis.text.x = element_text(angle=10)) +
  labs(y = "Annual USD value per hectare",
       title = "Most Valuable Biomes by Ecosystem Service")
```

![](EcocystemServiceAnalysis_files/figure-html/Most_Valuable_Biomes_Plotting-1.png)<!-- -->

We can see from the plot that grasslands are by far the most valuable biomes, by providing economic and energy services of nearly 8 million USD per hectare per year, globally.  After this, we have costal areas, which contribute their monetary value mainly through services relating to human happiness.

Another side to this story is determining which services human society are provided globally by ecosystems.  So let's see which services are the most valuable.


```r
## Which are the most valuable ecosystem services provided globally?
df_anPerHa$Service=mapply(ESgeneralise,df_anPerHa$ESService)
# Group by the ecosystem service
byService=group_by(df_anPerHa,Service)
# Summarise the services 
byService_sum=summarise(byService,Total=sum(USDPerHa))

# Plot the most valuable services
ggplot(byService, aes(fill=Biome, y=USDPerHa, x=Service)) +
  geom_bar(position="stack", stat="identity") +
  theme(axis.text.x = element_text(angle=10)) +
  labs(y = "Annual USD value per hectare",
       title = "Most Valuable Services by Biome")
```

![](EcocystemServiceAnalysis_files/figure-html/Service_plotting-1.png)<!-- -->

The barplot above shows that services directly providing an economic or energy value are the most valuable, with human happiness services coming in as the second most valuable.

## Conclusions
These data clearly illustrate how valuable ecosystems are to human society, providing billions of US dollars of value to the world by their presence alone.  Certain caveats to these data clearly exist.  For example, the value of services relating to the raw materials or food are likely to be more accurate as they can be directly measured, while the value of other services may have been under or overestimated.  

Despite these weaknesses, however, data such as these deserve further study, and can provide decision makers with invaluable information in making choices relating to cases where ecosystems could be restored or damaged by human activity.
