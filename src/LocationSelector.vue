<template>
  <div class="map-container">
  </div>
</template>

<script lang="ts">
import L, { LeafletMouseEvent, Map, TileLayerOptions } from "leaflet";
import "leaflet/dist/leaflet.css";
import { notify } from "@kyvg/vue3-notification";
import { defineComponent, PropType } from "vue";

export interface LocationDeg {
  longitudeDeg: number;
  latitudeDeg: number;
}

interface MapOptions extends TileLayerOptions {
  templateUrl: string;
  initialLocation?: LocationDeg;
  initialZoom?: number;
}

interface GeoJSONProp {
  url?: string;
  geojson?: GeoJSON.FeatureCollection | GeoJSON.Feature | GeoJSON.GeometryCollection;
  style: Record<string,any>;  // eslint-disable-line @typescript-eslint/no-explicit-any
}

interface Place extends LocationDeg { 
  color?: string;
  fillColor?: string;
  fillOpacity?: number;
  radius?: number;
  name?: string;
}

const defaultMapOptions: MapOptions = {
  templateUrl: 'https://{s}.google.com/vt/lyrs=p&x={x}&y={y}&z={z}',
  minZoom: 1,
  maxZoom: 20,
  subdomains:['mt0','mt1','mt2','mt3'],
  attribution: `&copy <a href="https://www.google.com/maps">Google Maps</a>`,
  className: 'map-tiles'
};

interface CloudData {
  lat: number;
  lon: number;
  cloudCover: number;
}

export default defineComponent({

  emits: ["place", "update:modelValue", "error"],

  props: {
    
    
    activatorColor: {
      type: String,
      default: "#ffffff"
    },
    showCloudCover: {
      type: Boolean,
      default: false
    },
    detectLocation: {
      type: Boolean,
      default: true
    },
    modelValue: {
      type: Object as PropType<LocationDeg>,
      default() {
        return {
          latitudeDeg: 42.3814,
          longitudeDeg: -71.1281
        };
      }
    },
    mapOptions: {
      type: Object as PropType<MapOptions>,
      default() {
        return defaultMapOptions;
      }
    },
    initialPlace: {
      type: Object as PropType<Place>,
      default: null
    },
    places: {
      type: Array as PropType<Place[]>,
      default() {
        return [];
      }
    },
    placeCircleOptions: {
      type: Object as PropType<Record<string, any>>,  // eslint-disable-line @typescript-eslint/no-explicit-any
      default() {
        return {
          color: "#0000FF",
          fillColor: "#3333FF",
          fillOpacity: 0.5,
          radius: 150 
        };
      }
    },
    placeSelectable: {
      type: Boolean,
      default: true
    },
    selectable: {
      type: Boolean,
      default: true
    },
    selectedCircleOptions: {
      type: Object as PropType<Record<string,any>>,  // eslint-disable-line @typescript-eslint/no-explicit-any
      default() {
        return {
          color: "#FF0000",
          fillColor: "#FF0033",
          fillOpacity: 0.5,
          radius: 200
        };
      }
    },
    selectionEvent: {
      type: String as PropType<'click' | 'dblclick'>,
      default: 'click'
    },
    worldRadii: {
      type: Boolean,
      default: false
    },
    
    geoJsonFiles: {
      type: Array as PropType<GeoJSONProp[]>,
      default: () => []
    },
    
    selectedCloudCover: {
      type:  Array as PropType<CloudData[]>,
      default: null
    }
  },

  mounted() {
    if (this.initialPlace) {
      this.selectedPlace = this.initialPlace;
    }
    if (this.detectLocation) {
      this.getLocation(true);
    }
    this.setup(true);
  },

  data() {
    return {
      eclipsePath: null as L.GeoJSON | null,
      placeCircles: [] as L.CircleMarker[],
      hoveredPlace: null as Place | null,
      selectedCircle: null as L.CircleMarker | null,
      selectedPlace: null as Place | null,
      selectedPlaceCircle: null as L.CircleMarker | null,
      cloudCoverRectangles: L.layerGroup(),
      map: null as Map | null,
      basemap: null as L.TileLayer | null,
    };
  },

  methods: {

    async loadCloudCover(): Promise<void> {
      return fetch('https://raw.githubusercontent.com/johnarban/solar-eclipse-2024-ds/use-median-cloud-cover/src/assets/one_deg_median_cc.csv')
        .then(response => response.text())
        .then(csvData => {
          this.parseData(csvData);
        })
        .catch(error => {
          console.error('Error fetching data:', error);
        });
    },

    parseData(csvData: string) {
      Papa.parse(csvData, {
        header: true,
        dynamicTyping: true,
        complete: (result) => {
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          result.data.forEach((row: any) => {
            const lat = parseFloat(row.lat);
            const lon = parseFloat(row.lon);
            const cloudCover = parseFloat(row.cloud_cover);
            // check for nan
            if (isNaN(lat) || isNaN(lon) || isNaN(cloudCover)) {
              return;
            }

        const rect = this.createRectangle(lat, lon, cloudCover);
        if (rect) {
          this.cloudCoverRectangles.addLayer(rect);
        }
      });
      // perhaps we should check if it is already added to the map. if it is, why remove it? 
      this.cloudCoverRectangles.addTo(this.map as Map); // Not sure why, but TS is cranky w/o the Map cast
      console.log('added to map', this.cloudCoverRectangles);
    },

    
    createRectangle(lat: number, lon: number, cloudCover: number): L.Rectangle {
      const color = this.getColor(cloudCover);
      
      return L.rectangle([
        [lat + 0.25, lon - 0.25],
        [lat - 0.25, lon + 0.25],
      ], {
        stroke: true,
        color: color,
        weight: .01,
        opacity: cloudCover,
        fillColor: color,
        fillOpacity: this.sigmoid(cloudCover)
      });
    },
    
    sigmoid(val: number | null): number {
      if (val === null) {
        return 0;
      }
      // return sigmoid
      const y = (val - 0.5) / .12;
      const z = Math.exp(y);
      return z / (1 + z);
    },

    getColor(_cloudCover:number) {
      // Calculate HSL color based on a gradient
      const hue = 0;
      const saturation = '0%';
      const lightness = '100%'; // 50% to 100%

      return `hsla(${hue}, ${saturation}, ${lightness},.9)`;
    },    
    
    getLocation(startup=false) {
      const options = { timeout: 10000, enableHighAccuracy: true };

      navigator.geolocation.getCurrentPosition(
        (position) => {
          this.updateValue({
            longitudeDeg: position.coords.longitude,
            latitudeDeg: position.coords.latitude
          });

          if (this.map) {
            this.map.setView([position.coords.latitude, position.coords.longitude], this.map.getZoom());
          }
        },
        (_error) => {
          const msg = "Unable to autodetect location. Location will default to Cambridge, MA, USA, or you can\nuse the location selector to manually input a location.";
          if (startup) {
            notify({
              group: "startup-location",
              type: "error",
              text: msg,
              duration: 4500
            });
          } else {
            this.$emit("error", msg);
          }
        },
        options
      );
    },

    circleForLocation(location: LocationDeg, circleOptions: Record<string,any>): L.CircleMarker {  // eslint-disable-line @typescript-eslint/no-explicit-any
      return this.circleMaker([location.latitudeDeg, location.longitudeDeg], circleOptions);
    },

    circleForSelection() : L.CircleMarker | null {
      if (this.selectedPlace) {
        return null;
      }
      const circle = this.circleForLocation(this.modelValue, { ...this.selectedCircleOptions, interactive: false });
      circle.bringToFront();
      return circle;
    },

    circleForPlace(place: Place): L.CircleMarker {
      const options = (place === this.selectedPlace) ? this.selectedCircleOptions : this.placeCircleOptions;
      const circle = this.circleForLocation(place, options);
      if (place.name) {
        circle.bindTooltip(place.name);
      }
      return circle;
    },

    onPlaceSelect(place: Place) {
      this.updateValue({
        longitudeDeg: place.longitudeDeg,
        latitudeDeg: place.latitudeDeg
      });
      this.$emit('place', place);
      this.selectedPlace = place;
    },

    onMapSelect(event: LeafletMouseEvent) {
      let longitudeDeg = event.latlng.lng + 180;
      longitudeDeg = ((longitudeDeg % 360) + 360) % 360;  // We want modulo, but JS % operator is remainder
      longitudeDeg -= 180;
      this.selectedPlace = null;
      this.updateValue({
        latitudeDeg: event.latlng.lat,
        longitudeDeg
      });
    },

    setup(initial=false) {
      const mapContainer = this.$el as HTMLDivElement;
      const location: L.LatLngExpression = initial && this.mapOptions.initialLocation ?
        this.locationToLatLng(this.mapOptions.initialLocation) :
        this.latLng;

      const initialZoom = this.mapOptions.initialZoom ?? 4;
      const zoom = initial ? initialZoom : (this.map?.getZoom() ?? initialZoom);
      const map = L.map(mapContainer).setView(location, zoom);
      
      const options = { ...defaultMapOptions, ...this.mapOptions };
      this.basemap = L.tileLayer(options.templateUrl, options);
      this.basemap.addTo(map);

      this.updateCloudCover(this.showCloudCover);
      this.bringLocationAndPathToFront();

      this.placeCircles = this.places.map(place => this.circleForPlace(place));
      this.placeCircles.forEach((circle, index) => {
        circle.on('mouseover', () => {
          const place = this.places[index];
          this.hoveredPlace = place;
          circle.openTooltip([place.latitudeDeg, place.longitudeDeg]);
        });

        if (this.placeSelectable) {
          circle.on('click', () => {
            this.onPlaceSelect(this.places[index]);
          });
        }

        circle.on('mouseout', () => {
          this.hoveredPlace = null;
        });

        circle.addTo(map);
      });

      this.selectedCircle = this.circleForSelection();
      this.selectedCircle?.addTo(map);

      map.doubleClickZoom.disable();
      if (this.selectable) {
        map.on(this.selectionEvent, this.onMapSelect);
      }

      map.attributionControl.setPrefix('<a href="https://leafletjs.com" title="A JavaScript library for interactive maps" target="_blank" rel="noopener noreferrer" >Leaflet</a>');
      
      // show the geojson files
      this.geoJsonFiles.forEach((geojsonrecord) => {
        const url = geojsonrecord.url;
        const geo = geojsonrecord.geojson;
        const style = geojsonrecord.style;
        if (url) {
          fetch(url)
            .then((response) => response.json())
            .then((data) => {
              const geoJSON = L.geoJSON(data, { style }).addTo(map);
              if (url.includes("center")) {
                geoJSON.bringToFront();
                this.eclipsePath = geoJSON;
              }
            })
            .catch((error) => {
              console.error('Error:', error);
            }); 
        } else if (geo) {
          L.geoJSON(geo, {
            style: style,
            pointToLayer: function (feature, latlng) {
              if (feature.properties.absoluteRadius) {
                style.radius = feature.properties.absoluteRadius;
                return L.circle(latlng, style);
              } else {
                return L.circleMarker(latlng, style);
              }
              
            },
            onEachFeature: function (feature, layer) {
              if (feature.properties && feature.properties.popupContent) {
                layer.bindPopup(feature.properties.popupContent);
              }
            }
          }).addTo(map);
        }
      });

      this.eclipsePath?.bringToFront();
      this.selectedCircle?.bringToFront();
      
      this.map = map;
    },

    updateValue(value: LocationDeg) {
      this.$emit('update:modelValue', value);
    },

    updateCircle() {
      if (this.map) {
        this.selectedCircle?.remove();
        this.selectedCircle = this.circleForSelection();
        if (this.selectedCircle) {
          this.selectedCircle.addTo(this.map as Map); // Not sure why, but TS is cranky w/o the Map cast
          this.bringLocationAndPathToFront();
        }
      }
    },

    bringLocationAndPathToFront() {
      this.eclipsePath?.bringToFront();
      this.selectedCircle?.bringToFront();
    },

    locationToLatLng(location: LocationDeg): L.LatLngExpression {
      return [location.latitudeDeg, location.longitudeDeg];
    },

    updateCloudCover(value: boolean) {
      if (value && this.selectedCloudCover != null) {
        this.cloudCoverRectangles.remove(); // do we need to remove it from the map? but how do we make sure it's not already on the map?
        this.cloudCoverRectangles.clearLayers(); // clear the rectangles is what we want
        this.parseResult(this.selectedCloudCover);
      }
    }

  },

  computed: {
    circleMaker(): (latlng: L.LatLngExpression, options: L.CircleMarkerOptions) => L.CircleMarker {
      return this.worldRadii ? L.circle : L.circleMarker;
    },
    latLng(): L.LatLngExpression {
      return this.locationToLatLng(this.modelValue);
    }
  },

  watch: {

    selectedCloudCover(val: CloudData[] | null) {
      if (val !== null) {
        this.updateCloudCover(this.showCloudCover);
        this.bringLocationAndPathToFront();
      }
    },
    
    modelValue() {
      this.updateCircle();
      if (this.map && !this.map.getBounds().contains(this.latLng)) {
        this.map.setView(this.latLng);
      }
    },
    
    mapOptions(newOptions: MapOptions, oldOptions: MapOptions) {
      if (oldOptions === null || newOptions === null) {
        return;
      }
      if (newOptions.templateUrl !== oldOptions.templateUrl) {
        this.basemap?.setUrl(newOptions.templateUrl ?? defaultMapOptions.templateUrl);
      }
    },
    
    
    showCloudCover(value: boolean) {
      this.updateCloudCover(value);
      this.bringLocationAndPathToFront();
    },
    places() {
      this.map?.remove();
      this.setup();
    },
    selectedPlace(newPlace) {
      const index = this.places.indexOf(newPlace);
      const oldSelectedCircle = this.selectedPlaceCircle;
      this.selectedPlaceCircle = this.placeCircles[index];

      oldSelectedCircle?.setStyle(this.placeCircleOptions);
      this.selectedPlaceCircle?.setStyle(this.selectedCircleOptions);
    }
  }
  
});
</script>

<style lang="less">
.map-container {
  height: 100%;
  width: 100%;
  margin: auto;
  padding: 5px 0px;
  border-radius: 5px;
  
  .leaflet-bottom.leaflet-right::before {
    content: " Credit: © Leaflet.js";
    top: 100%;
    left: 100%;
    transform: translate(-100%, -100%);
    pointer-events: auto;
  }

  .leaflet-bottom.leaflet-right::before {
    /* match formatting for actual attribution */
    color: #0078a8;
    background-color: rgba(255,255,255,0.8);
    font-size: 0.75em;
    padding-inline: 0.5em;
    padding-block: 0.3em;
  }

  .leaflet-bottom.leaflet-right:hover::before {
    content: "";
    background-color: transparent;
  }

  .leaflet-bottom.leaflet-right:hover > .leaflet-control-attribution {
    display: block;
  }


  .leaflet-control-attribution {
    display: none;
  }

  path.leaflet-interactive:focus {
    outline: none;
  }

  path.leaflet-interactive:focus-visible {
    outline: auto 5px black;
  }

  
}
</style>
