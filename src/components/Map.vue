<template>
  <div class="px-2 lg:px-8 transition-all duration-500" :style="state === 1 && `filter: saturate(0%)`">
    <MapTemplate
      :selectedYear="selectedYear"
      :selected="selected"
      :mapHeightCalculated="mapHeightCalculated"
      :mapIsMoving="mapIsMoving"
      :state="state"
    />
    <Legend
      :state="state"
      :zoomLevel="zoomLevel"
      :minZoom="minZoom"
      :maxZoom="maxZoom"
      :initZoomLevel="initZoomLevel"
      :selectedYear="selectedYear"
      :colorScale2009="colorScale2009"
      :colorScale2022="colorScale2022"
      :minRent="minRent"
      :maxRent="maxRent"
      :meanRent="meanRent"
      :selectedButton="selectedButton"
      :unselectedButton="unselectedButton"
      :legendScale="legendScale"
      @zoomIn="onZoom('in')"
      @zoomOut="onZoom('out')"
      @zoomInitial="onZoom('initial')"
      @toggleYear2022="toggleYear(2022)"
      @toggleYear2009="toggleYear(2009)"
      @toggleColorScales2022="toggleColorScales(2022)"
      @toggleColorScales2009="toggleColorScales(2009)"
    />
  </div>
</template>
<script setup lang="ts">
import Legend from "./Legend.vue";
import MapTemplate from "./MapTemplate.vue";
//@ts-ignore
import * as d3 from "d3";
import { computed, onMounted, ref, watch } from "vue";
import "leaflet/dist/leaflet.css";
//@ts-ignore
import * as topojson from "topojson";
//@ts-ignore
import L from "leaflet";

// Import the Leaflet MapTiler Plugin
import "@maptiler/leaflet-maptilersdk";

// types
type Properties = {
  OTEIL?: string;
  PROGNOSERA: string;
  ANGEBOTSMIETEN: number;
};

type Feature = {
  type: string;
  properties: {
    OTEIL: string;
    PROGNOSERA: string;
    ANGEBOTSMIETEN: number;
  };
  geometry: {
    type: string;
    coordinates: number[];
  };
};

type MapInfo = {
  type: string;
  features: Feature[];
  objects: {
    progs: {
      type: string;
      features: Feature[];
    };
  };
};

type Data = {
  Prognoseraum: string;
  Angebotsmieten: number;
};

// props
const props = defineProps({
  state: Number,
});

// variables
const selected = ref<Properties>();
let colorsBothYears = ref(false);
let mapInfo: MapInfo;
let selectedYear = ref(2022);
let colorScale2022 = ref(true);
let colorScale2009 = ref(false);

let minRent = ref();
let maxRent = ref();
let meanRent = ref();
let minRentC = ref();
let maxRentC = ref();
let meanRentC = ref();
let cScale = d3.scaleLinear();

let data2009: Data[];
let data2022: Data[];
let completeDataSet: Data[];
let LMainMap: any;
let LBorderMap: any;
let borderSVGs: any;
let fillSVGs: any;
let pathG: any;

let zoomLevel = ref();
const maxZoom = 14.0;
let minZoom = ref<number>();
let initZoomLevel: number;

let mapIsMoving = ref(false);

const width = window.innerWidth;
const height = window.innerHeight;
const legendHeight = width < 1024 ? 280 : 199;

const mapHeightCalculated = `height:${height - legendHeight}px`;

// state handlers
// watch state for changes
watch(
  () => props.state,
  (value) => {
    if (value === 2.5) {
      selected.value = {
        PROGNOSERA: "Zentrum",
        ANGEBOTSMIETEN: 16.17,
      };
      d3.select("#Zentrum").attr("opacity", 1);
      showTooltip(selected.value, [width / 2, height / 2]);
    }
    if (value !== 2.5) {
      d3.select("#Zentrum").attr("opacity", 0);
    }
    if (value === 2.6) {
      selected.value = {
        PROGNOSERA: "Marzahn",
        ANGEBOTSMIETEN: 7.21,
      };
      d3.select("#Marzahn").attr("opacity", 1);
      showTooltip(selected.value, [width / 2 + 60, height / 2 - 400]);
    }
    if (value !== 2.6) {
      d3.select("#Marzahn").attr("opacity", 0);
    }
    if (value !== 2.5 && value !== 2.6) {
      hideTooltip();
    }
    if (value && value < 3) {
      selectedYear.value = 2022;
      colorScale2022.value = true;
      colorScale2009.value = false;
      showData(data2022);
    }
    if (value === 3) {
      selectedYear.value = 2022;
      colorScale2022.value = true;
      colorScale2009.value = true;
      showData(data2022);
    }
    if (value === 4) {
      selectedYear.value = 2009;
      colorScale2009.value = true;
      colorScale2022.value = true;
      showData(data2009);
    }
    if (value === 5) {
      selectedYear.value = 2009;
      colorScale2009.value = true;
      colorScale2022.value = false;
      showData(data2009);
    }
  }
);

function hideTooltip() {
  const tooltip = d3.select("#tooltip");
  tooltip.classed("opacity-0", true);
  tooltip.classed("opacity-90", false);
}

function onZoom(direction?: "in" | "out" | "initial") {
  if (direction) {
    if (direction === "initial") {
      LBorderMap.setView([52.5, 13.4], initZoomLevel);
      LMainMap.setView([52.5, 13.4], initZoomLevel);
    } else if (direction === "in") {
      LMainMap.zoomIn();
      LBorderMap.zoomIn();
    } else {
      LMainMap.zoomOut();
      LBorderMap.zoomOut();
    }
  }

  setTimeout(() => {
    zoomLevel.value = LMainMap.getZoom();
  }, 250);
}

const selectedButton = computed(() => {
  return "bg-gray-400 text-white border-none rounded pointer-events-none ";
});

const unselectedButton = computed(() => {
  return "bg-gray-50 text-gray-800 border-gray-50";
});

function toggleYear(year: number) {
  if (year === 2009) {
    selectedYear.value = 2009;
    colorScale2009.value = true;
    showData(data2009);
  } else if (year === 2022) {
    selectedYear.value = 2022;
    colorScale2022.value = true;
    showData(data2022);
  }
}

function toggleColorScales(year: number) {
  year === 2009 ? (colorScale2009.value = !colorScale2009.value) : (colorScale2022.value = !colorScale2022.value);
  colorsBothYears.value = !colorsBothYears.value;
  showData(selectedYear.value === 2022 ? data2022 : data2009);
}

function showTooltip(area: Properties, coords: number[]) {
  let positionX;
  let positionY;

  const tooltip = d3.select("#tooltip");
  d3.select("#rent-indicator").classed("hidden", false);
  tooltip.classed("opacity-0", false);
  tooltip.classed("opacity-90", true);

  // position tooltip based on pointer
  if (width > 668) {
    if (coords[0] > width - 250) {
      positionX = coords[0] - 200;
    } else {
      positionX = coords[0] + 50;
    }

    if (coords[1] < 250) {
      positionY = coords[1] + 100;
    } else {
      positionY = coords[1] - 150;
    }

    tooltip.style("left", positionX + "px").style("top", positionY + "px");
  }

  selected.value = area;
}

const legendScale = computed(() => {
  return `background: linear-gradient(90deg,${cScale(minRent.value)}, ${cScale(meanRent.value)}, ${cScale(maxRent.value)} )`;
});

function currentPositionOnLegendScale() {
  const currentRent = selected.value?.ANGEBOTSMIETEN;
  const position = currentRent && ((currentRent - minRent.value) / (maxRent.value - minRent.value)) * 99;
  const rentIndicator = d3.select("#rent-indicator");
  rentIndicator.style("left", `${position?.toFixed(2)}%`);
}

function valuesAllData() {
  completeDataSet = data2009.concat(data2022);
  completeDataSet.forEach((d) => {
    d.Angebotsmieten = +d.Angebotsmieten;
  });
  minRentC.value = d3.min(completeDataSet, (d: Data) => d.Angebotsmieten);
  maxRentC.value = d3.max(completeDataSet, (d: Data) => d.Angebotsmieten);
  meanRentC.value = Number(d3.mean(completeDataSet, (d: Data) => d.Angebotsmieten).toFixed(2));
}

function drawBorders() {
  const borderPaths = borderSVGs
    .selectAll("path")
    .data(topojson.feature(mapInfo, mapInfo.objects.progs).features)
    .enter()
    .append("path")
    .attr("d", (d: Feature) => pathG(d))
    // set css attribute transition for opacity
    .style("transition", "opacity 0.15s")
    .attr("stroke", "rgb(0,0,0,0.7)")
    .attr("stroke-width", 3)
    .attr("opacity", 0)
    .attr("fill", "transparent")
    .attr("id", (d: Feature) => d.properties.PROGNOSERA)

    // interaction
    .on("mouseenter", (e: MouseEvent, d: Feature) => {
      !mapIsMoving.value && d3.select(e.currentTarget).attr("opacity", 1);
      const [x, y] = d3.pointer(e, d3.select("#main-map").node());
      showTooltip(d.properties, [x, y]);
      currentPositionOnLegendScale();
    })
    .on("mouseleave", (e: MouseEvent) => {
      d3.select(e.currentTarget).attr("opacity", 0);
      hideTooltip();
      d3.select("#rent-indicator").classed("hidden", true);
    });

  // place svg based on zoom
  const onZoom = () => {
    borderPaths.attr("d", pathG);
  };

  // initialize positioning
  onZoom();
  // reset whenever map is moved
  LBorderMap.on("moveend zoomend", onZoom);
}

// show data
function showData(dataSource: Data[]) {
  // create dataIndex to avoid nested loops
  let dataIndex: { [key: string]: number } = {};
  dataSource.forEach((d) => {
    dataIndex[d.Prognoseraum] = +d.Angebotsmieten; // '+' converts string to number
  });

  // extend topojson properties with addiotional data from dataset
  mapInfo.features = topojson.feature(mapInfo, mapInfo.objects.progs).features.map((d: any) => {
    // mapInfo.features = mapInfo.features.map(d => {
    d.properties.ANGEBOTSMIETEN = dataIndex[d.properties.PROGNOSERA];
    return d;
  });

  // color scale
  if (colorScale2009.value && colorScale2022.value) {
    maxRent.value = maxRentC.value;
    minRent.value = minRentC.value;
    meanRent.value = meanRentC.value;
  } else {
    maxRent.value = d3.max(mapInfo.features, (d: Feature) => d.properties.ANGEBOTSMIETEN);
    minRent.value = d3.min(mapInfo.features, (d: Feature) => d.properties.ANGEBOTSMIETEN);
    meanRent.value = Number(d3.mean(mapInfo.features, (d: Feature) => d.properties.ANGEBOTSMIETEN).toFixed(2));
  }
  cScale = d3
    .scaleLinear()
    .domain([minRent.value, meanRent.value, maxRent.value])
    .range(["#1E6E6B", "#EDE9D6", "#B24424"])
    .interpolate(d3.interpolateRgb.gamma(-0.1));

  // d3 data
  const areaPaths = fillSVGs
    .selectAll("path")
    .data(mapInfo.features)
    .enter()
    .append("path")
    .attr("d", (d: Feature) => pathG(d))
    .attr("stroke", "white")
    .attr("stroke-width", 0.2)
    .attr("fill", "transparent");

  // fill svgs
  fillSVGs
    .selectAll("path")
    .transition()
    .attr("fill", (d: Feature) =>
      Number.isNaN(d.properties.ANGEBOTSMIETEN) ? "white" : cScale(d.properties.ANGEBOTSMIETEN)
    );

  // place svg based on zoom
  const onZoom = () => {
    areaPaths.attr("d", pathG);
    // sync maps
    const newMapCenter = LMainMap.getCenter();
    // Move Border Overlay Map to new center and zoomlevel
    LBorderMap.setView(new L.LatLng(newMapCenter.lat, newMapCenter.lng), LMainMap.getZoom(), { animate: true });
  };
  // initialize positioning
  onZoom();
  // reset whenever map is moved
  LMainMap.on("moveend zoomend", onZoom);
}

// leaflet setup
function initLeaflet() {
  // initial zoom level
  if (width < 450) initZoomLevel = 9.25;
  if (width >= 450 && width < 1280) initZoomLevel = 10;
  if (width >= 1280) initZoomLevel = 11.25;

  minZoom.value = initZoomLevel;

  const mapOptions = {
    minZoom: minZoom.value,
    maxZoom: maxZoom,
    scrollWheelZoom: false,
    zoomControl: false,
    zoomSnap: 0.75,
    zoomDelta: 0.75,
    dragging: width < 1280 ? false : true,
    tap: width < 1280 ? false : true,
    maxBoundsViscosity: 0.8,
    touchZoom: width < 1280 ? true : false,
  };

  // leaflet map setup
  LMainMap = L.map("main-map", mapOptions).setView([52.5, 13.4], initZoomLevel);
  LBorderMap = L.map("border-map", mapOptions).setView([52.5, 13.4], initZoomLevel);

  zoomLevel.value = LMainMap.getZoom();

  const maxBounds = L.latLngBounds([52.71, 14.13], [52.3, 12.7]);
  LMainMap.setMaxBounds(maxBounds);
  LBorderMap.setMaxBounds(maxBounds);

  // Create a MapTiler Layer inside Leaflet
  new L.MaptilerLayer({
    apiKey: "hsl5L4grOcWPCDYacsGF",
    style: "https://api.maptiler.com/maps/f9fd35b5-3026-423b-ac7e-4dc7537efabb/style.json?key=hsl5L4grOcWPCDYacsGF",
    attributionControl: false,
  }).addTo(LMainMap);

  // scale
  L.control.scale({ imperial: false, maxWidth: 150 }).addTo(LMainMap);

  //initialize svg to add to map
  L.svg().addTo(LMainMap);
  L.svg().addTo(LBorderMap);

  //LMainMap Create selection using D3
  const overlay = d3.select(LMainMap.getPanes().overlayPane);
  const tiles = d3.select(LMainMap.getPanes().tilePane);
  const svg = overlay.select("svg");
  fillSVGs = svg.append("g");

  // adjust z-levels for correct layering
  tiles.style("z-index", 5);
  tiles.style("pointer-events", "none");
  LMainMap.getPanes().overlayPane.style.zIndex = 4;
  LMainMap.getPanes().mapPane.style.zIndex = 1;

  // LBorderMap for borders
  const overlay2 = d3.select(LBorderMap.getPanes().overlayPane);
  const svg2 = overlay2.select("svg").attr("pointer-events", "auto");
  borderSVGs = svg2.append("g");

  // Use Leaflets projection API for drawing svg path (creates a stream of projected points)
  const projectPoint = function (this: any, x: number, y: number) {
    const point = LMainMap.latLngToLayerPoint(new L.LatLng(y, x));
    this.stream.point(point.x, point.y);
  };

  // Use d3's custom geo transform method to implement the above
  const projection = d3.geoTransform({ point: projectPoint });
  // creates geopath from projected points (SVG)
  pathG = d3.geoPath().projection(projection);

  // Use capturing phase to allow clicks to pass through (before it reaches the target element)
  document.getElementById("border-map")?.addEventListener(
    "mousedown",
    function (event) {
      // Prevent click event from triggering on the top element
      event.stopImmediatePropagation();
      // Manually trigger the click event on the bottom element
      document.getElementById("main-map")?.dispatchEvent(new MouseEvent("mousedown", event));
    },
    true
  );

  // detect zooming and moving phases to hide border map
  LBorderMap.on("zoomstart", () => {
    LMainMap.dragging.disable();
    mapIsMoving.value = true;
  });

  LMainMap.on("zoomstart", () => {
    LMainMap.dragging.disable();
    mapIsMoving.value = true;
  });

  LBorderMap.on("zoomend", () => {
    LMainMap.dragging.enable();
    mapIsMoving.value = false;
    zoomLevel.value = LMainMap.getZoom();
  });

  LMainMap.on("zoom", () => {
    mapIsMoving.value = true;
    LMainMap.dragging.disable();
  });

  LMainMap.on("move", () => {
    mapIsMoving.value = true;
    LMainMap.doubleClickZoom.disable();
  });

  LMainMap.on("movestart", () => {
    mapIsMoving.value = true;
    LMainMap.doubleClickZoom.disable();
  });

  LBorderMap.on("movestart", () => {
    mapIsMoving.value = true;
    LMainMap.doubleClickZoom.disable();
  });

  LBorderMap.on("moveend", () => {
    mapIsMoving.value = false;
    LMainMap.doubleClickZoom.enable();
    LMainMap.dragging.enable();
  });

  showData(data2022);
  drawBorders();
}

onMounted(() => {
  // load data
  Promise.all([
    d3.csv("mieten_berlin_2009.csv"),
    d3.csv("mieten_berlin_2022.csv"),
    d3.json("simplified.topo.json"),
  ]).then((data) => {
    data2009 = data[0];
    data2022 = data[1];
    mapInfo = data[2];
    valuesAllData();
    initLeaflet();
  });
});
</script>
