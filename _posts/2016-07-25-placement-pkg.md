---
layout:  post
title: "Placement: An R package to Access the Google Maps API"
comments:  true
published: true
author: "Derek Darves"
date: 2016-07-24 08:00:00
categories: [R, 'Google Maps']
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---
	
	


A few months ago I set out to write an R package for accessing the Maps API with 
my employer's (paid) [Google for Work/Premium account](https://developers.google.com/maps/documentation/directions/get-api-key#premium-auth). At the time, I was unable to find an R package that could generate the encrypted signature, send the URL to Google and process the JSON returns in one fell swoop. Following Google's [directions for Python](https://github.com/googlemaps/google-maps-services-python), however, I was able to create an R function that creates valid signatures for a URL request using the   [`digest`](https://cran.r-project.org/web/packages/digest/index.html) package's implementation of the *sha-1* algorithm. Along the way I added a few additional features that are useful in our workgroup, including (1) a function to retrieve Google Map's distance and travel time estimates (via public transit, driving, cycling, or walking) between two places  ([`drive_time`](https://github.com/DerekYves/placement/blob/master/R/drive_time.R)), (2) a general purpose function for stripping address vectors of nasty characters that may break a geocode request ([`address_cleaner`](https://github.com/DerekYves/placement/blob/master/R/address_cleaner.R)), and (3) methods for accessing the Google API with a (free) standard account (see also the excellent [`ggmaps`](https://cran.r-project.org/web/packages/ggmap/ggmap.pdf) package, which provides a similar facility for geocoding with Google's standard API). 

In daily use I've seen few issues thus far, and I've used earlier versions of this package to geocode about a quarter million physical locations in North America. The placement package, which includes examples, can be [viewed on Github](https://github.com/DerekYves/placement) and installed in the usual way:


{% highlight r %}
library(devtools)
install_github("DerekYves/placement")
library(placement)
{% endhighlight %}

Here's a few examples using the standard (free) API (see [here](https://developers.google.com/maps/documentation/directions/get-api-key) to get a free API key from Google, which has higher quota limits than supplying an empty string):


{% highlight r %}
# Get coordinates for the Empire State Building and Google
address <- c("350 5th Ave, New York, NY 10118, USA",
			 "1600 Amphitheatre Pkwy,
			 Mountain View, CA 94043, USA")

coordset <- geocode_url(address, auth="standard_api", privkey="",
            clean=TRUE, add_date='today', verbose=TRUE)
{% endhighlight %}



{% highlight text %}
## Sending address vector (n=2) to Google...
## Finished. 2 of 2 records successfully geocoded.
{% endhighlight %}



{% highlight r %}
# View the returns
print(coordset[ , 1:5])
{% endhighlight %}



{% highlight text %}
##        lat        lng location_type
## 1 40.74844  -73.98566       ROOFTOP
## 2 37.42234 -122.08437       ROOFTOP
##                                             formatted_address status
## 1 Empire State Building, 350 5th Ave, New York, NY 10118, USA     OK
## 2        1600 Amphitheatre Pkwy, Mountain View, CA 94043, USA     OK
{% endhighlight %}


Distance calculations (note that some transit options are not accessible with the standard API):



{% highlight r %}
# Bike from the NYC to Google!
address <- c("350 5th Ave, New York, NY 10118, USA",
			 "1600 Amphitheatre Pkwy, 
			 Mountain View, CA 94043, USA")

# Google allows you to supply geo coordinates *or* a physical address 
# for the distance API. In this example, we will supply coordinates
# from our previous call. Google requires a string format of: 
#   "lat,lng" (with no spaces) for coordinates.

start <- paste(coordset$lat[1],coordset$lng[1], sep=",")
end   <- paste(coordset$lat[2],coordset$lng[2], sep=",")

# Get the travel time by bike (a mere 264 hours!) and distance in miles:
howfar_miles <- drive_time(address=start, dest=end, auth="standard_api",
						   privkey="", clean=FALSE, add_date='today',
						   verbose=FALSE, travel_mode="bicycling",
						   units="imperial")

# Get the distance in kilometers using physical addresses instead of lat/lng:
howfar_kms <- drive_time(
     address="350 5th Ave, New York, NY 10118",
		dest="1600 Amphitheatre Pkwy, Mountain View, CA",
		auth="standard_api", privkey="", clean=FALSE,
		add_date='today', verbose=FALSE, travel_mode="bicycling",
		units="metric"
		)

with(howfar_kms, 
	 cat("Cycling from NYC to ", destination,
	 	":\n", dist_txt, " over ", 
	 	time_txt, sep=""), sep="")
{% endhighlight %}



{% highlight text %}
## Cycling from NYC to 1600 Amphitheatre Pkwy, Mountain View, CA 94043, USA:
## 5,345 km over 11 days 17 hours
{% endhighlight %}

Address cleaning function:


{% highlight r %}
# Clean a "messy" or otherwise incompatible address vector:
address <- c(" 350 5th Ave. ½, New York, NY 10118, USA ",
			 "  ª1600  Amphitheatre Pkwy, 
			 Mountain View, CA 94043, USA")

# View the return:
address_cleaner(address)
{% endhighlight %}



{% highlight text %}
## 	* Replacing non-breaking spaces
## 	* Removing control characters
## 	* Removing leading/trailing spaces, and runs of spaces
## 	* Transliterating latin1 characters
## 	* Converting special address markers
## 	* Removing all remaining non-ASCII characters
## 	* Remove single/double quotes and asterisks
## 	* Removing leading, trailing, and repeated commas
## 	* Removing various c/o string patterns
{% endhighlight %}



{% highlight text %}
## [1] "350 5th Ave.  1/2, New York, NY 10118, USA"           
## [2] "a1600 Amphitheatre Pkwy, Mountain View, CA 94043, USA"
{% endhighlight %}

If you would like to apply this function to multiple address fields stored in separate columns (e.g., only "street 1" and "city"), you might try something like:


{% highlight r %}
address_df[] <- sapply(address_df, placement::address_cleaner)
{% endhighlight %}



{% highlight text %}
## Error in lapply(X = X, FUN = FUN, ...): object 'address_df' not found
{% endhighlight %}

Using your Google for Work account obviously requires a client ID and API key, but the methods to do so are well documented in the package help files. Feel free to shoot me an email if you run into any issues!  



