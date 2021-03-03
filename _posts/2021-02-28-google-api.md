---
layout: single
classes: wide
title:  "Building a Location Search API: Prototyping"
date:   2021-02-27 12:00:00 -0600
categories: go prototypes api
---
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

For one of my projects, I need to create a search API that lets users filter locations of interest by proximity to another landmark.

The premise is similar to Zillow.
Users mark property for sale, and other users search for marked property within a radius of a landmark.
Retailers will often use this technique to help customers find stores near them.
The need is everywhere, but I had trouble finding technical resources on how this is usually done.

Let's explore a solution with Go.

## Design

We're going to build an API with an endpoint `/within` which returns all marked locations within a radius of a landmark or set of coordinates.

```go
type LocationsController interface {
	Within(http.ResponseWriter, *http.Request)
}

type withinRequest struct {
	Address           string
	Radius, Lat, Long json.Number
}
```

We'll need some datastore to keep locations of interest.
For now, let's use a simple slice based store, and implement the `LocationsController` interface on that.

```go
type Location struct {
	Address             string
	Latitude, Longitude float64
}

type SimpleLocationsController []Location

func (s *SimpleLocationsController) Within(w http.ResponseWriter, req *http.Request) {
	// Todo!
}
```

In reality, location data should be stored in a relational or memcache database.
Something to look forward to!

The algorithm I'm envisioning is to get latitude and longitude data for all locations of interest, then use geometry to find which points are within a radius.

### Geocoding
Google's Maps API is unnervingly thorough, fast, and easy to use.
Also, they somehow already have my credit card.

Google provides the Geocoding API.
Given an address, or even vague landmark name, they will give us the coordinates we need.
We can then associate coordinates with all places of interest.

Setup is quite simple, consisting of [enabling the API](https://developers.google.com/maps/documentation/geocoding/cloud-setup), and [grabbing an API key](https://developers.google.com/maps/documentation/geocoding/get-api-key).

The Go client for Google Maps is [google-maps-services-go](https://github.com/googlemaps/google-maps-services-go), with the docs on [pkg.go.dev](https://pkg.go.dev/googlemaps.github.io/maps). The incantation `go get googlemaps.github.io/maps` will get us started.

Using the API is oddly simple.
```go
func main() {
	addresses := []string{
		"Chicago, IL",
		"Sears Tower",
		"University of Illinois at Chicago",
		"North Central College",
		"Aurora, IL",
	}

	c, err := maps.NewClient(maps.WithAPIKey(os.Getenv("GCP_API_KEY")))
	if err != nil {
		log.Fatal(err)
	}

	var ctrl SimpleLocationsController
	for _, address := range addresses {
		results, err := c.Geocode(context.Background(), &maps.GeocodingRequest{
			Address: address,
		})
		if err != nil {
			log.Fatal(err)
		}
		if len(results) == 0 {
			log.Printf("no results found: %s", address)
			continue
		}

		ctrl = append(ctrl, Location{
			Latitude:  results[0].Geometry.Location.Lat,
			Longitude: results[0].Geometry.Location.Lng,
			Address:   results[0].FormattedAddress,
		})
	}
}
```
You'll notice I'm disregarding other possible results in favor of the first one.
In reality, the UI consuming this API should use Google's front-end Maps API to help the user narrow down their selection.

## Dreaded Math
With this coordinate data, we can calculate distances.
Note that Earth coordinates are spherical coordinates.
The arcane formula for calculating the [distance of two points on the surface of a sphere](https://en.wikipedia.org/wiki/Great-circle_distance) is the spherical law of cosines.

$$
d = R\arccos(\sin(lat_1)\sin(lat_2) + \cos(lat_1)\cos(lat_2)\cos(long_1 - long_2))
$$

You'll have to take Wikipedia's word for it.

If we go-ify this, while being conscious of converting degrees to radians, we get this:
```go
// EarthRadius is in kilometers.
const EarthRadius = 6371.0

// DistanceFrom calculates the great-circle distance between the two coordinates,
// in kilometers.
func (l Location) DistanceFrom(lat, long float64) float64 {
	lat1 := lat * math.Pi / 180
	lat2 := l.Latitude * math.Pi / 180
	long1 := long * math.Pi / 180
	long2 := l.Longitude * math.Pi / 180
	return EarthRadius * math.Acos(
		(math.Sin(lat1)*math.Sin(lat2))+
			(math.Cos(lat1)*math.Cos(lat2)*math.Cos(long2-long1)))
}
```

## Implementing `/within`

Let's tie this together!

```go
type SimpleLocationsController struct {
	Locations  []Location
	
	// Throw the maps client in here, in case we're asked to geocode an address.
	MapsClient *maps.Client
}

func (s *SimpleLocationsController) Within(w http.ResponseWriter, req *http.Request) {
	var body withinRequest
	// ... parse out the request body.
	// ... get the request coordinates, optionally geocoding a given address.

	var locations []Location
	for _, loc := range s.Locations {
		dist := math.Round(loc.DistanceFrom(lat, long))
		if dist <= rad {
			locations = append(locations, loc)
		}
	}

	if err := json.NewEncoder(w).Encode(locations); err != nil {
		log.Print(err)
	}
}
```

Of course we'll need to build an HTTP server:
```go
func main() {
	// ...
	
	http.HandleFunc("/within", ctrl.Within)
	log.Printf("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```


## Uh, okay.
Alright, so it's not that impressive.
Especially when I cut out the boilerplate.
I did say this was a prototype, didn't I?
As it so happens, getting this far isn't hard.

You may be looking with disapproval at these couple of lines:
```go
	for _, loc := range s.Locations {
		dist := math.Round(loc.DistanceFrom(lat, long))
```

Yes, I'm a neanderthal.
Surely that doesn't scale.
Maybe.

We can set up an exercise easily.
I converted this prototype to ingest from a CSV full of randomly generated locations.
I ran the server with various sized data sets, and threw a bunch of HTTP requests at it with this test client:
```go
func main() {
	tr := &http.Transport{
		MaxIdleConns:        100,
		MaxIdleConnsPerHost: 100,
	}
	client := &http.Client{Transport: tr}

	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for i := 0; i < 10; i++ {
				func(start time.Time) {
					defer func() {
						fmt.Println(time.Since(start))
					}()

					body, _ := json.Marshal(map[string]interface{}{
						"lat": strconv.FormatFloat(rand.Float64()*90*math.Pow(-1,
							float64(rand.Int()%2)), 'f', 4, 64),
						"long": strconv.FormatFloat(rand.Float64()*180*math.Pow(-1,
							float64(rand.Int()%2)), 'f', 4, 64),
						"radius": rand.Int() % 100,
					})
					b := bytes.NewBuffer(body)
					resp, err := client.Post("http://localhost:8080/within",
						"application/json", b)
					if err != nil {
						log.Print(err)
						return
					}
					resp.Body.Close()
				}(time.Now())
			}
		}()
	}
	wg.Wait()
}
```
Yeah I know [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) exists. Let me have my goroutine fun.

At larger datasets, contexts will need to come into play.
Otherwise you'll be very sad when you CTRL+C the benchmark but your CPU is still melting.
```go
	for _, loc := range s.Locations {
		select {
		case <-req.Context().Done():
			log.Println("cancelled")
			w.WriteHeader(http.StatusBadRequest)
			return

		default:
			dist := math.Round(loc.DistanceFrom(lat, long))
			if dist <= rad {
				locations = append(locations, ret{Location: loc,
					Distance: dist})
			}
		}
	}
```

Here's some rough averages of time per request, on my laptop with decent specs:

| Size of dataset | Average Time per Request |
|------|------|
| 10^5 | .04s |
| 10^6 | .3s  |
| 10^7 | 3s   |
| 10^8 | 45s  |

So we're drifting into unacceptable territory around more than a million locations.
But Zillow supposedly has [110 million](https://ipropertymanagement.com/research/zillow-statistics) properties posted.
That will definitely blow away the CPU credits on your `t3.micro` instance.

![](https://inahga-public.s3-us-west-2.amazonaws.com/mem.png)
*Yes, 10^8 locations took a lot of memory.*

## Summary 
Unfortunately, my project will probably not see more than a hundred locations, so trying to scale harder than this is not necessary.

That doesn't mean I'm not going to do it though.

Things to ponder on for next time:
- Perhaps a smarter [algorithm](https://en.wikipedia.org/wiki/Range_searching) or [data structure](https://en.wikipedia.org/wiki/K-d_tree).
- Need to run it against a real database. This won't scale horizontally otherwise.

Feel free to shoot me any tips. You can check out the full source of the work done thus far [here](https://github.com/inahga/inahgohno/blob/81751f9ac7fab8ca63499e6f4f8200ea43500042/cmd/gmaps/main.go).
