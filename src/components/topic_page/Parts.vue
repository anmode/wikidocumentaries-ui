<template>
  <div v-if="results.length">
    <div class="gallery-component">
      <div class="toolbar">
        <h1 class="header-title">{{ $t('topic_page.Parts.headerTitle') }}</h1>
        <DisplayMenu @doDisplayChange="onDisplayChange"></DisplayMenu>
        <ToolbarMenu
          icon="wikiglyph-sort"
          :tooltip="$t('menus.sortMenu.tooltip')"
          :items="toolbarActionMenuItems"
          @doMenuItemAction="onDoMenuItemAction"
        >
          <div slot="menu-title">{{ $t('menus.sortMenu.title') }}</div>
        </ToolbarMenu>
      </div>
      <div class="intro">{{ $t('topic_page.Parts.intro') }}</div>
      <div v-if="viewMode === VIEW_MODES.GALLERY" class="gallery">
        <!--img :src="wikidocumentaries.galleryImageURL" class="gallery-image"/-->
        <router-link
          tag="div"
          v-for="item in results"
          :key="item.id"
          :to="getItemURL(item.item.value)"
          class="gallery-item"
        >
          <img :src="getImageLink(item.image)" class="gallery-image">
          <div class="thumb-image-info">
            <div class="gallery-title">{{ item.item.label }}</div>
            <div class="thumb-credit">{{ item.item.description }}</div>
          </div>
          <!--div class="thumb-image-header"-->
          <div>
            <div class="left-align">
              <!--ImagesActionMenu></ImagesActionMenu-->
            </div>
            <div class="right-align">
              <!--ImagesRemoveMenu></ImagesRemoveMenu-->
            </div>
          </div>
        </router-link>
      </div>
      <div v-else-if="viewMode === VIEW_MODES.LIST" class="list">
        <div v-for="item in results" :key="item.id" class="listrow">
          <a :href="getItemURL(item.item.value)">
            <b>{{ item.item.label }}</b>
            {{ item.item.description}}
          </a>
        </div>
      </div>
      <div v-else-if="viewMode === VIEW_MODES.MAP" id="PartsMapContainer" class="basemap"></div>
    </div>
  </div>
</template>

<script>
import ToolbarMenu from "@/components/menu/ToolbarMenu";
import { sortResults } from "@/common/utils";
import { MAPBOX_AT } from "@/common/tokens";
import axios from "axios";
import wdk from "wikidata-sdk";
import DisplayMenu from "@/components/menu/ComponentViewModeMenu";
import { VIEW_MODES } from "@/components/menu/ComponentViewModeMenu";
import mapboxgl from "mapbox-gl";
import bbox from '@turf/bbox';
import { geometryCollection, geometry } from '@turf/helpers'

const SORT_ACTIONS = {
  BY_LABEL: 0,
  BY_TIME: 1,
  SORT_REVERSE: 2,
  SORT_CLEAR: 3
};

/* const DISPLAY_ACTIONS = {
  GALLERY: 0,
  LIST: 1
}; */

const MAX_ITEMS_TO_VIEW = 1000;
const DEFAULT_SORT = ["item.label"];

let fullResults, currentSort, currentDisplay, myMap;

export default {
  name: "Parts",
  created() {
    this.VIEW_MODES = VIEW_MODES; // Export for use in template
  },
  components: {
    ToolbarMenu,
    DisplayMenu
  },
  data() {
    return {
      results: [],
      viewMode: VIEW_MODES.GALLERY,
      toolbarActionMenuItems: [
        {
          id: SORT_ACTIONS.BY_LABEL,
          text: "menus.sortMenu.optionAlpha"
        },
        {
          id: SORT_ACTIONS.BY_TIME,
          text: "menus.sortMenu.optionTime"
        },
        {
          id: SORT_ACTIONS.SORT_REVERSE,
          text: "menus.sortMenu.optionRev"
        },
        {
          id: SORT_ACTIONS.SORT_CLEAR,
          text: "menus.sortMenu.optionClear"
        }
      ]
    };
  },
  mounted() {
    currentSort = DEFAULT_SORT.slice();
    currentDisplay = VIEW_MODES.GALLERY;
    var title = this.$store.state.wikidocumentaries.title;
    const statements = this.$store.state.wikidocumentaries.wikidata.statements;
    mapboxgl.accessToken = MAPBOX_AT;
    let sparql;
    sparql = `
SELECT ?item ?itemLabel ?itemDescription (SAMPLE(?image) AS ?image) (SAMPLE(?coordinates) AS ?coordinates) (MIN(?_time) AS ?time) 
WHERE {
  { wd:Q407542 wdt:P527 ?item. }
  UNION
  { ?item wdt:P361 wd:Q407542 . }
  UNION
  { ?item wdt:P131 wd:Q407542 .
  ?item wdt:P31/wdt:P279* wd:Q56061. }
  UNION
  { wd:Q407542 wdt:P150 ?item . }
  OPTIONAL { ?item wdt:P625 ?coordinates . }
  OPTIONAL { ?item wdt:P18 ?image . }
  MINUS { ?item wdt:P31 wd:Q5 .}
  OPTIONAL { ?item wdt:P580 ?startdate. }
  OPTIONAL { ?item wdt:P582 ?enddate .}
  BIND(STR(COALESCE(?startdate, ?enddate)) AS ?_time)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],fi,sv,en,de,fr,it,es,no,nb,et,nl,pl,ca,se,sms,is,da,ru,et". }
}
GROUP BY ?item ?itemLabel ?itemDescription ?image
LIMIT 1000
        `.replace(/Q407542/g, this.$store.state.wikidocumentaries.wikidataId)
        .replace(/fi/g, this.$i18n.locale);;
    const [url, body] = wdk.sparqlQuery(sparql).split("?");
    axios
      .post(url, body)
      .then(response => {
        fullResults = wdk.simplify.sparqlResults(response.data);
        this.results = selectResults(this.$i18n.locale);
        this.viewMode = currentDisplay;
      })
      .catch(error => console.log(error));
  },
  computed: {
    wikidocumentaries() {
      return this.$store.state.wikidocumentaries;
    }
  },
  watch: {},
  methods: {
    onDoMenuItemAction(menuItem) {
      switch (menuItem.id) {
        case SORT_ACTIONS.BY_LABEL:
          currentSort = ["item.label"];
          break;
        case SORT_ACTIONS.BY_TIME:
          currentSort = ["time", "item.label"];
          break;
        case SORT_ACTIONS.SORT_REVERSE:
          if (currentSort[0].charAt(0) == "-")
            currentSort[0] = currentSort[0].substr(1);
          else currentSort[0] = "-" + currentSort[0];
          break;
        case SORT_ACTIONS.SORT_CLEAR:
          currentSort = DEFAULT_SORT.slice();
          break;
      }
      this.results = selectResults(this.$i18n.locale);
    },
    onDisplayChange(menuItem) {
      currentDisplay = menuItem.id;
      if (currentDisplay == VIEW_MODES.MAP) {
        this.viewMode = currentDisplay;
        const lat = this.$store.state.wikidocumentaries.wikidata.geo.lat;
        const lon = this.$store.state.wikidocumentaries.wikidata.geo.lon;

        this.$nextTick(function () {
          myMap = new mapboxgl.Map({
            container: "PartsMapContainer",
            style: "mapbox://styles/mapbox/streets-v11",
            center: [lon, lat],
            zoom: 12,
          });

          let points = new Array();

          this.results.forEach(function (item) {
            let popupHtml = "";
            let koord;
            let wikidataPoint;
            if (item.coordinates) {
              koord = item.coordinates.split("(")[1].split(")")[0].split(" ");
              wikidataPoint = geometry('Point', koord);
              points.push(wikidataPoint);
              popupHtml =
              '<a href="' + 
              item.item.value +
              '"><div class="popup-body">' +
                (item.image
                  ? '<img src="' + item.image + '" class="popup-image">'
                  : '') +
                '<div class="popup-txt">' +
                '<div class="gallery-title">' +
                item.item.label +
                "</div>" +
                '<div class="thumb-credit">' +
                (item.item.description ? item.item.description : "") +
                "</div></div></div></a>";
              console.log(item);
              var marker = new mapboxgl.Marker()
                .setLngLat(koord)
                .setPopup(
                  new mapboxgl.Popup() // add popups
                    .setMaxWidth("300px")
                    .setHTML(popupHtml)
                )
                .addTo(myMap);
            }
          });
          const bounds = bbox(geometryCollection(points))
          myMap.fitBounds(bounds, {
            padding: 50,
            maxZoom: 19
          });
        });
      } else {
        if (myMap) {
          myMap.remove();
          myMap = null;
        }
        this.results = selectResults(this.$i18n.locale);
        this.viewMode = currentDisplay;
      }
      // console.log("ViewMode changed to: ", this.viewMode);
      // console.log("VIEW_MODES: ", VIEW_MODES);
    },
    fitTitle(title) {
      var newTitle = title;
      return newTitle;
    },
    navigate(target) {
      this.$router.push({ target });
    },
    getItemURL(value) {
      return "/" + value + "?language=" + this.$i18n.locale;
    },
    getImageLink(value) {
      return value.replace(/\s/g, _) + "?width=500";
    }
  }
};

const selectResults = lcl => {
  let filteredResults = fullResults;
  if (currentSort[0].includes("time"))
    filteredResults = filteredResults.filter(x => x.time);
  if (currentDisplay === VIEW_MODES.GALLERY) {
    if (filteredResults.find(x => x.image)) {
      // If GALLERY and at least one image
      filteredResults = filteredResults.filter(x => x.image); // select only results with an image
    } else {
      currentDisplay = VIEW_MODES.LIST;  // GALLERY with no images => change to LIST
    }
  }
  return filteredResults
    .sort(sortResults(currentSort, lcl))
    .slice(0, MAX_ITEMS_TO_VIEW);
};
</script>

<style scoped>
.basemap {
  width: 100%;
  height: 40vh;
}
</style>
