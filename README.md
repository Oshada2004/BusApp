# Bus Route Finder

## Google Cloud Setup

**Official Doc:** <https://developers.google.com/maps/get-started>

### Steps

1. Create a GCP project: <https://console.cloud.google.com/>
2. Enable APIs:
   - Maps SDK for Android: <https://developers.google.com/maps/documentation/android-sdk/cloud-setup>
   - Maps SDK for iOS: <https://developers.google.com/maps/documentation/ios-sdk/cloud-setup>
   - Places API (New): <https://developers.google.com/maps/documentation/places/web-service/cloud-setup>
   - Routes API: <https://developers.google.com/maps/documentation/routes/cloud-setup>
3. Create API keys:
   - Android Maps key (for RN app map rendering)
   - iOS Maps key (for RN app map rendering)
   - Server key (for FastAPI backend)
4. Restrict keys:
   - Android: Add package name + SHA-1 certificate fingerprint
   - iOS: Add bundle identifier
   - Server: Restrict by API (Places, Routes) and optionally by IP

**API Key Best Practices:** <https://developers.google.com/maps/api-key-best-practices>

---

## Feature 1: Text Search Places with Suggestions (Origin/Destination)

**What it does:** User types in a text field; backend calls Google Places Autocomplete API and returns place suggestions with `placeId`.

**Official Doc:** <https://developers.google.com/maps/documentation/places/web-service/autocomplete>

### Backend Code (FastAPI)

**Endpoint:** `/places`

```python
from fastapi import FastAPI, HTTPException
import httpx, os

app = FastAPI()
API_KEY = os.getenv("GOOGLE_MAPS_API_KEY")
client = httpx.AsyncClient(timeout=10)

@app.get("/places")
async def places(q: str, sessionToken: str | None = None):
    """
    Autocomplete place suggestions.
    Doc: https://developers.google.com/maps/documentation/places/web-service/autocomplete
    """
    url = "https://maps.googleapis.com/maps/api/place/autocomplete/json"
    params = {"input": q, "key": API_KEY}
    if sessionToken:
        params["sessiontoken"] = sessionToken
    
    r = await client.get(url, params=params)
    data = r.json()
    
    if data.get("status") not in ("OK", "ZERO_RESULTS"):
        raise HTTPException(502, f"Places API error: {data.get('status')}")
    
    return [
        {"description": p["description"], "placeId": p["place_id"]}
        for p in data.get("predictions", [])
    ]
```

**Response Example:**

```json
[
  { "description": "Colombo Fort, Sri Lanka", "placeId": "ChIJ..." },
  { "description": "Kandy, Sri Lanka", "placeId": "ChIJ..." }
]
```

### React Native Code

**Call the backend:**

```javascript
// api.js
export async function fetchPlaces(query) {
  const res = await fetch(`http://YOUR_SERVER/places?q=${encodeURIComponent(query)}`);
  return res.json();
}
```

**UI Flow:**

- User types in `TextInput` → debounce (300ms) → call `fetchPlaces()`
- Display results in a list
- On selection, store `{ description, placeId }`

---

## Feature 2: Get All Possible Bus Routes (with Transit Info)

**What it does:** Takes origin and destination `placeId`s, calls Google Routes API (Directions v2) with `travelMode=TRANSIT` and `transitPreferences` for buses, returns all alternative routes with bus numbers, durations, stops, and transit details.

**Official Doc:** <https://developers.google.com/maps/documentation/routes/compute_route_directions>  
**Transit Routes Doc:** <https://developers.google.com/maps/documentation/routes/transit>

### Backend Code (FastAPI)

**Endpoint:** `/routes` (POST)

```python
@app.post("/routes")
async def routes(originPlaceId: str, destinationPlaceId: str, departureTime: str = "now"):
    """
    Get transit (bus) routes.
    Doc: https://developers.google.com/maps/documentation/routes/compute_route_directions
    Transit: https://developers.google.com/maps/documentation/routes/transit
    """
    url = "https://routes.googleapis.com/directions/v2:computeRoutes"
    headers = {
        "Content-Type": "application/json",
        "X-Goog-Api-Key": API_KEY,
        "X-Goog-FieldMask": "routes.legs.steps.transitDetails,routes.legs.steps.travelMode,routes.polyline,routes.duration,routes.distanceMeters"
    }
    body = {
        "origin": {"placeId": originPlaceId},
        "destination": {"placeId": destinationPlaceId},
        "travelMode": "TRANSIT",
        "computeAlternativeRoutes": True,
        "transitPreferences": {
            "allowedTravelModes": ["BUS"],
            "routingPreference": "FEWER_TRANSFERS"
        }
    }
    if departureTime != "now":
        body["departureTime"] = departureTime
    
    r = await client.post(url, json=body, headers=headers)
    data = r.json()
    
    if "routes" not in data:
        raise HTTPException(502, "No routes found")
    
    routes_out = []
    for idx, route in enumerate(data["routes"]):
        polyline_encoded = route.get("polyline", {}).get("encodedPolyline", "")
        duration_sec = int(route.get("duration", "0s").rstrip("s"))
        distance_m = route.get("distanceMeters", 0)
        
        legs_out = []
        for leg in route.get("legs", []):
            for step in leg.get("steps", []):
                if step.get("travelMode") == "TRANSIT":
                    td = step.get("transitDetails", {})
                    stop_details = td.get("stopDetails", {})
                    transit_line = td.get("transitLine", {})
                    
                    legs_out.append({
                        "mode": "TRANSIT",
                        "busNumber": transit_line.get("nameShort", ""),
                        "busName": transit_line.get("name", ""),
                        "headsign": td.get("headsign", ""),
                        "departureStop": stop_details.get("departureStop", {}).get("name", ""),
                        "arrivalStop": stop_details.get("arrivalStop", {}).get("name", ""),
                        "departureTime": stop_details.get("departureTime", ""),
                        "arrivalTime": stop_details.get("arrivalTime", ""),
                        "stopCount": td.get("stopCount", 0),
                    })
        
        routes_out.append({
            "id": f"r{idx+1}",
            "polyline": polyline_encoded,
            "durationMin": round(duration_sec / 60),
            "distanceKm": round(distance_m / 1000, 2),
            "legs": legs_out
        })
    
    return routes_out
```

**Response Example:**

```json
[
  {
    "id": "r1",
    "polyline": "u{~vFz|...",
    "durationMin": 92,
    "distanceKm": 115.3,
    "legs": [
      {
        "mode": "TRANSIT",
        "busNumber": "101",
        "busName": "CTB Colombo-Kadawatha",
        "headsign": "Kadawatha",
        "departureStop": "Colombo Fort",
        "arrivalStop": "Kadawatha",
        "departureTime": "2025-10-17T10:32:10Z",
        "arrivalTime": "2025-10-17T10:49:42Z",
        "stopCount": 12
      }
    ]
  }
]
```

### React Native Code

**Call the backend:**

```javascript
// api.js
export async function fetchRoutes(originPlaceId, destinationPlaceId) {
  const res = await fetch(`http://YOUR_SERVER/routes`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ originPlaceId, destinationPlaceId, departureTime: "now" })
  });
  return res.json();
}
```

---

## Feature 3: Draw All Bus Routes as Polylines with Start/End Markers

**What it does:** Decode the polyline from each route and render it on a map using `react-native-maps`. Add markers for the start and end points.

**Official Doc (Polyline Encoding):** <https://developers.google.com/maps/documentation/utilities/polylinealgorithm>  
**react-native-maps Doc:** <https://github.com/react-native-maps/react-native-maps>

### Install Dependencies

```bash
npm install react-native-maps @mapbox/polyline
```

**iOS:** `cd ios && pod install`

### Android Setup

**Doc:** <https://developers.google.com/maps/documentation/android-sdk/config>

**AndroidManifest.xml:**

```xml
<application>
  <meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="YOUR_ANDROID_MAPS_API_KEY" />
</application>
```

### iOS Setup

**Doc:** <https://developers.google.com/maps/documentation/ios-sdk/config>

**AppDelegate (Objective-C):**

```objc
@import GoogleMaps;
// in didFinishLaunchingWithOptions:
[GMSServices provideAPIKey:@"YOUR_IOS_MAPS_API_KEY"];
```

### React Native Code

**Decode and Render:**

```javascript
// MapScreen.jsx
import React from 'react';
import MapView, { Polyline, Marker } from 'react-native-maps';
import polyline from '@mapbox/polyline';

// Decode polyline (Google encoding)
// Doc: https://developers.google.com/maps/documentation/utilities/polylinealgorithm
const decode = (encoded) => 
  polyline.decode(encoded).map(([lat, lng]) => ({ latitude: lat, longitude: lng }));

export default function MapScreen({ routes, selectedIndex = 0 }) {
  if (!routes?.length) return null;

  const firstRoute = decode(routes[0].polyline);
  const startCoord = firstRoute[0];
  const endCoord = firstRoute[firstRoute.length - 1];

  return (
    <MapView
      style={{ flex: 1 }}
      provider="google"
      initialRegion={{
        latitude: startCoord.latitude,
        longitude: startCoord.longitude,
        latitudeDelta: 0.2,
        longitudeDelta: 0.2,
      }}
    >
      {/* Draw all routes as polylines */}
      {routes.map((r, idx) => {
        const coords = decode(r.polyline);
        return (
          <Polyline
            key={r.id}
            coordinates={coords}
            strokeColor={idx === selectedIndex ? '#1976D2' : '#9E9E9E'}
            strokeWidth={idx === selectedIndex ? 5 : 3}
          />
        );
      })}

      {/* Start and End Markers */}
      <Marker coordinate={startCoord} title="Start" pinColor="green" />
      <Marker coordinate={endCoord} title="End" pinColor="red" />
    </MapView>
  );
}
```
