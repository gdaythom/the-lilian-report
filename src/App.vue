<script setup>
import * as SW from 'stopword';

import { useRoute } from 'vue-router'
import { ref, computed } from 'vue'

import CsvFileReader from './components/CsvFileReader.vue'

const route = useRoute();

const fileData = ref([]);

const filtered = computed(() => {
  var filteredFileData = fileData.value;
  if(route.query.tier) {
    var key = route.query.tier;
    if(key == "NONE") {
      key = "";
    }
    filteredFileData = filteredFileData.filter(item => item["Embedded Data - tier"] === key);
  }
  if(route.query.platform) {
    if(route.query.platform == "iOS") {
      filteredFileData = filteredFileData.filter(item => item["Embedded Data - deviceOS"] === route.query.platform || "iPadOS");
    } else {
      filteredFileData = filteredFileData.filter(item => item["Embedded Data - deviceOS"] === route.query.platform);
    }
  }
  if(route.query.version) {
    var appVersion = route.query.version.split(".")
    filteredFileData = filteredFileData.filter(item => item["Embedded Data - appVersion"].includes(`1.${appVersion[1]}.`) || item["Embedded Data - appVersion"].includes(`2.${appVersion[1]}.`));
  }
  if(route.query.date) {
    filteredFileData = filteredFileData.filter(item => item["Embedded Data - dateCreated"].includes(route.query.date));
  }
  if(route.query.rating) {
    filteredFileData = filteredFileData.filter(item => item["App rating numeric"] === route.query.rating);
  }

  if(route.query.word) {
    filteredFileData = filteredFileData.filter(item => item["Q4 - Could you share more information about your experience with the Virgin Aust..."].indexOf(route.query.word) !== -1);
  }


  filteredFileData.sort((a,b) => a["App rating numeric"] - b["App rating numeric"]);
  return filteredFileData;
});

//const startDate = "2018-05-17";
const startDate = "2023-08-01";
const multiplier = 3;

const timeline = computed(() => {

  const appVersionsAsKeys = filtered.value.reduce((sums, entry) => {
    var key = entry["Embedded Data - appVersion"];
    if(key == "") {
      key = "NONE";
    } else {
      key = key.split(".")
      if(key[0] === '1') {
        key[0] = 2;
      }
      key = `${key[0]}.${key[1]}`;
    }
    sums[key] = {};
    return sums;
  },{});

  delete appVersionsAsKeys['NONE'];

  const appVersions = [];

  for (const [key, value] of Object.entries(appVersionsAsKeys)) {
    var appVersion = key.split(".")
    const filteredFileData = filtered.value.filter(item => item["Embedded Data - appVersion"].includes(`1.${appVersion[1]}.`) || item["Embedded Data - appVersion"].includes(`2.${appVersion[1]}.`));
    const results = filteredFileData.reduce((sums,entry) => {
      var key = entry["App rating numeric"];
      sums[key] = (sums[key] || 0) + 1;
      return sums;
    },{});

    const sortedRatings = Object.keys(results).sort().reduce(
      (obj, key) => { 
        obj[key] = results[key]; 
        return obj;
      }, 
      {}
    );

    const initialValue = 0;
    const sumWithInitial = filteredFileData.reduce(
      (accumulator, currentValue) => {
        return accumulator + parseInt(currentValue["App rating numeric"])
      },
      initialValue,
    );

    const unique = [...new Set(filteredFileData.map(item => {
      //var key = item["Embedded Data - dateCreated"];
      var key = item["Survey Metadata - Start Date (+00:00 GMT)"];

      //2022-12-23 22:28:28 (19)
      //2018-08-04T01:41:15-07:00 (25)
      if(key.includes("Z")) {
        key = key.slice(0, -10)
      }
      if(key.length == 19) {
        key = key.slice(0, -9)
      }
      if(key.length == 25) {
        key = key.slice(0, -15)
      }
      return key;
    }))];

    const sortedDates = unique.sort((a,b) => {
      return new Date(a) - new Date(b);
    });

    var dateRange1 = new Date(sortedDates[0]);
    var dateRange2 = new Date(sortedDates[sortedDates.length - 1]);
    var Difference_In_TimeRange = dateRange2.getTime() - dateRange1.getTime();

    var dateOffset1 = new Date(startDate);
    var dateOffset2 = new Date(sortedDates[0]);
    var Difference_In_TimeOffset = dateOffset2.getTime() - dateOffset1.getTime();

    appVersions.push({
      'version': key,
      'ratings': sortedRatings,
      'averageRating': Math.round((sumWithInitial / filteredFileData.length) * 10) / 10,
      'dates': sortedDates,
      'range': Math.round(Difference_In_TimeRange / (1000 * 3600 * 24)),
      'offset': Math.round(Difference_In_TimeOffset / (1000 * 3600 * 24))
    })
  }
  return appVersions.reverse();
});


const wordCloud = computed(() => {
  var wordCounts = { };
  if(filtered.value.length > 0) {
    filtered.value.forEach((feedback) => {
      var alphaOnlyReview = feedback['Q4 - Could you share more information about your experience with the Virgin Aust...'].replace(/[^a-zA-Z\s]+/g, '');
      var filteredReview = alphaOnlyReview.split(/\b/);
      var words = SW.removeStopwords(filteredReview);
      words = words.filter(function(str) {
        return /\S/.test(str);
      });
      for(var i = 0; i < words.length; i++)
        words[i].toLowerCase() !== " " ? wordCounts["_" + words[i].toLowerCase()] = (wordCounts["_" + words[i].toLowerCase()] || 0) + 1 : "";
    });
  }

  // wordCounts = Object.fromEntries(
  //   Object.entries(wordCounts).filter(item => item > 5)
  // )
  // Object.filter = (obj, predicate) => 
  //   Object.keys(obj)
  //         .filter( key => predicate(obj[key]) )
  //         .reduce( (res, key) => (res[key] = obj[key], res), {} );

  // wordCounts = Object.filter(wordCounts, item => item > 20); 

  const sortable = Object.fromEntries(
    Object.entries(wordCounts).sort(([,a],[,b]) => b-a)
  );

  return sortable;
});

/*
  Get a count of all the tiers.
*/
const tiers = computed(() => {
  const results = filtered.value.reduce((sums,entry) => {
    var key = entry["Embedded Data - tier"];
    if(key == "") {
      key = "NONE";
    }
    sums[key] = (sums[key] || 0) + 1;
    return sums;
  },{});
  const sortedResults = Object.fromEntries(
    Object.entries(results).sort(([,a],[,b]) => b-a)
  );
  return sortedResults;
});

/*
  Get a count of all the platform.
*/
const platforms = computed(() => {
  const results = filtered.value.reduce((sums,entry) => {
    var key = entry["Embedded Data - deviceOS"];
    if(key == "") {
      key = "NONE";
    }
    if(key == "iPadOS") {
      key = "iOS";
    }
    sums[key] = (sums[key] || 0) + 1;
    return sums;
  },{});
  const sortedResults = Object.fromEntries(
    Object.entries(results).sort(([,a],[,b]) => b-a)
  );
  delete sortedResults['NONE'];
  return sortedResults;
});

/*
  Get a count of all the app versions.
*/
const versions = computed(() => {
  const results = filtered.value.reduce((sums,entry) => {
    var key = entry["Embedded Data - appVersion"];
    // if(key.includes("(")) {
    //   key = key.slice(0, -7)
    // }
    if(key == "") {
      key = "NONE";
    } else {
      key = key.split(".")
      if(key[0] === '1') {
        key[0] = 2;
      }
      key = `${key[0]}.${key[1]}`;
    }
    sums[key] = (sums[key] || 0) + 1;
    return sums;
  },{});
  const sortedResults = Object.fromEntries(
    Object.entries(results).sort(([,a],[,b]) => b-a)
  );
  delete sortedResults['NONE'];
  return sortedResults;
});

/*
  Get a count of all the dates.
*/
const dates = computed(() => {
  const results = filtered.value.reduce((sums,entry) => {
    //var key = entry["Embedded Data - dateCreated"];
    var key = entry["Survey Metadata - Start Date (+00:00 GMT)"];
    if(key.includes("Z")) {
      key = key.slice(0, -10)
    }
    if(key.includes("/")) {
      var dateArr = key.split("/",3);
      var yearArr = dateArr[2].split(" ");
      key = `${yearArr[0]}-${dateArr[1]}`;
    }
    if(key.length == 19) {
      key = key.slice(0, -12)
    }
    if(key.length == 25) {
      key = key.slice(0, -18)
    }
    if(key == "") {
      key = "NONE";
    }
    sums[key] = (sums[key] || 0) + 1;
    return sums;
  },{});
  const sortedResults = Object.fromEntries(
    Object.entries(results).sort(([,a],[,b]) => b-a)
  );
  delete sortedResults['NONE'];
  return sortedResults;
});

/*
  Get a count of all the rating.
*/
const ratings = computed(() => {
  const results = filtered.value.reduce((sums,entry) => {
    var key = entry["App rating numeric"];
    sums[key] = (sums[key] || 0) + 1;
    return sums;
  },{});
  const sortedResults = Object.fromEntries(
    Object.entries(results).sort(([,a],[,b]) => b-a)
  );
  return sortedResults;
});


const fileProcessed = (element) => {
  
  //Remove all \n within double quotes
  element = element.replace(/"[^"]*(?:""[^"]*)*"/g, function(m) { return m.replace(/\n/g, ''); })
  element = element.replace(/"[^"]*(?:""[^"]*)*"/g, function(m) { return m.replace(/\r/g, ''); })
  //Replace empty cell with whitespace
  element = element.replace(/,(?=[^\s])/g, ', ');

  //Generate the keys
  let csv = element.trim().split("\n");
  const keys = csv[0].split(',').map(s => s.trim());
  
  //Generate the values
  const values = csv.splice(1, csv.length - 1).map((item) => item.match(/(".*?"|[^",]+)(?=\s*,|\s*$)/g));
  //Generate all the objects 
  fileData.value = values.map((item) => {
    item = item.map(s => s.trim());
    const object = {};
    keys.forEach((key, index) => (object[key] = item.at(index)));
    return object;
  });
}

const themes = [
  {
    key: 'Design',
    options: ['unintuitive', 'unclear', 'navigate']
  },
  {
    key: 'Bugs',
    options: ['slow', 'bug', 'bugs', 'error', 'errors']
  },
  {
    key: 'Stability',
    options: ['crash', 'crashes', 'freeze', 'freezes', 'slow', 'bug', 'bugs', 'closing']
  },
  {
    key: 'Auth',
    options: ['log', 'logged', 'login', 'logout']
  },
  {
    key: 'Boarding',
    options: ['boarding', 'pass', 'boardingpass']
  },
  {
    key: 'Seating',
    options: ['seat', 'seats', 'seating', 'economy', 'economyx']
  },
  {
    key: 'Velocity',
    options: ['status', 'credits', 'points', 'frequent', 'flyer', 'member']
  },
  {
    key: 'Flight',
    options: ['cancelled', 'cancel', 'delayed', 'delay']
  },
  {
    key: 'Check-in',
    options: ['check', 'checkin', 'checked', 'checkedin']
  }
];

const reviewThemes = computed(() => {
  var results = [];
  filtered.value.forEach((item) => {
    assignThemes(item["Q4 - Could you share more information about your experience with the Virgin Aust..."]).split(', ').forEach((item) => {
      results.push(item)
    });
  });
  var filteredResults = results.filter(elm => elm);
  const themeCounts = {};
  for (const num of filteredResults) {
    themeCounts[num] = themeCounts[num] ? themeCounts[num] + 1 : 1;
  }
  var themeResults = [];
  for (const [key, value] of Object.entries(themeCounts)) {
    themeResults.push(`${key} (${value})`);
  }
  return themeResults.join(', ');
});

const assignThemes = (element) => {
  var elementThemes = [];
  
  themes.forEach((item) => {
    item.options.forEach((option) => {
    if(element.toLowerCase().indexOf(option) !== -1) {
      elementThemes.push(item.key);
    }
    });
  });
  return [...new Set(elementThemes)].join(', ');
}

</script>

<template>
  <div class="p-8">
    <h1 class="mb-8 underline font-bold text-xl">The Lilian Report</h1>

    <CsvFileReader @fileLoaded="fileProcessed" class="my-8" />
    
    <div class="border p-2 my-8 overflow-x-scroll">
      <div v-for="(release, key) in timeline" :key="key" class="my-2">
        <div
          :style="`margin-left: ${release.offset * multiplier}px; width: ${release.range * multiplier}px`"
          class="text-center bg-red-100"
          :class="{
            'bg-red-100': release.averageRating < 1,
            'bg-orange-100': release.averageRating > 1 && release.averageRating < 2,
            'bg-yellow-100': release.averageRating > 2 && release.averageRating < 3,
            'bg-green-100': release.averageRating > 4
          }
        ">
          {{ release.version }}
        </div>
      </div>
    </div>

    <div class="grid grid-cols-4 gap-4">
      <!-- <div>
        <p>Tiers</p>
        <p v-if="route.query.tier">
          Selected: {{ route.query.tier }} 
          <router-link :to="{ name: 'home', query: {...route.query, ...{ tier: null }} }">Clear</router-link>
        </p>
        <ul>
          <li v-for="(value, key) in tiers" :key="key">
            <router-link :to="{ name: 'home', query: {...route.query, ...{ tier: key }} }">{{ key }} ({{ value }})</router-link>
          </li>
        </ul>
      </div> -->
      <div>
        <p>OS</p>
        <ul>
          <li v-for="(value, key) in platforms" :key="key">
            <router-link :to="{ name: 'home', query: {...route.query, ...{platform: key }} }">{{ key }} ({{ value }})</router-link>
          </li>
        </ul>
      </div>
      <div>
        <p>App Version</p>
        <ul>
          <li v-for="(value, key) in versions" :key="key">
            <router-link :to="{ name: 'home', query: {...route.query, ...{ version: key }} }">{{ key }} ({{ value }})</router-link>
          </li>
        </ul>
      </div>
      <div>
        <p>Dates</p>
        <ul>
          <li v-for="(value, key) in dates" :key="key">
            <router-link :to="{ name: 'home', query: {...route.query, ...{ date: key }} }">{{ key }} ({{ value }})</router-link>
          </li>
        </ul>
      </div>

      <div>
        <p>Ratings</p>
        <ul>
          <li v-for="(value, key) in ratings" :key="key">
            <router-link :to="{ name: 'home', query: {...route.query, ...{ rating: key }} }">{{ key }} ({{ value }})</router-link>
          </li>
        </ul>
      </div>
    </div>

    
    <div class="my-8">
      <p>Results: {{ filtered.length }}</p>
      <ul class="mb-4">
        <!-- <li v-if="route.query.tier">Tier: {{ route.query.tier }}</li> -->
        <li v-if="route.query.platform">OS: {{ route.query.platform }} <router-link :to="{ name: 'home', query: {...route.query, ...{ platform: null }} }">Clear</router-link></li>
        <li v-if="route.query.version">App Version: {{ route.query.version }} <router-link :to="{ name: 'home', query: {...route.query, ...{ version: null }} }">Clear</router-link></li>
        <li v-if="route.query.date">Date: {{ route.query.date }} <router-link :to="{ name: 'home', query: {...route.query, ...{ date: null }} }">Clear</router-link></li>
        <li v-if="route.query.rating">Rating: {{ route.query.rating }} <router-link :to="{ name: 'home', query: {...route.query, ...{ rating: null }} }">Clear</router-link></li>
        <li v-if="route.query.word">Word: {{ route.query.word }} <router-link :to="{ name: 'home', query: {...route.query, ...{ word: null }} }">Clear</router-link></li>
      </ul>

      <div>
        <div v-for="(value, key) in ratings" :key="key">
          <div
            :style="`width: ${value}px`"
            :class="{
              'bg-red-100': key == 1,
              'bg-orange-100': key == 2,
              'bg-yellow-100': key == 3,
              'bg-blue-100': key == 4,
              'bg-green-100': key == 5
            }"
          >
            <p class="block">{{ key }}</p>
            <small class="block">({{ Math.round(100*(value/filtered.length)) }}%)</small>
          </div>
        </div>
      </div>

    </div>
    <div class="mb-8" v-if='!route.query.rating || route.query.rating && route.query.rating < 3'>
      <p>Themes</p>
      {{ reviewThemes }}
    </div>

    <div class="mb-8">
      <p>Words</p>
      <span v-for="(value, key) in wordCloud" :key="key">

        <router-link class="border p-2 inline-block" v-if="value > 4" :to="{ name: 'home', query: {...route.query, ...{ word: key.slice(1) }} }">
          {{ key.slice(1) }} ({{ value }})
        </router-link>

      </span>
    </div>

    <div>
      <div v-for="(value, key) in filtered" :key="key">
        <div
          class="flex justify-between p-4 mb-8"
          :class="{
            'bg-red-100': value['App rating numeric'] == 1,
            'bg-orange-100': value['App rating numeric'] == 2,
            'bg-yellow-100': value['App rating numeric'] == 3,
            'bg-blue-100': value['App rating numeric'] == 4,
            'bg-green-100': value['App rating numeric'] == 5
          }"
        >
          <p class="w-2/3">
            {{ value["Q4 - Could you share more information about your experience with the Virgin Aust..."] }}
          </p>
          <p>
            <span v-if='value["App rating numeric"] < 6'>
              {{ assignThemes(value["Q4 - Could you share more information about your experience with the Virgin Aust..."]) }}
            </span>
          </p>
          <p>
            <!-- {{ value["Embedded Data - tier"] }} —  -->
            {{ value["Embedded Data - deviceOS"] }} — 
            {{ value["Embedded Data - appVersion"] }} - 
            <!-- {{ value["Embedded Data - dateCreated"] }} -  -->
            {{ value["App rating numeric"] }}
          </p>
        </div>
      </div>
    </div>
  </div>

  <RouterView />
    <!-- <pre>
https://dev.to/ramonak/javascript-how-to-merge-multiple-objects-with-sum-of-values-43fd
https://www.digitalocean.com/community/tutorials/vuejs-file-reader-component
https://hasnode.byrayray.dev/convert-a-csv-to-a-javascript-array-of-objects-the-practical-guide
https://stackoverflow.com/questions/29957390/counting-occurrences-of-object-values
https://stackoverflow.com/questions/11456850/split-a-string-by-commas-but-ignore-commas-within-double-quotes-using-javascript
    </pre> -->



</template>

<style scoped>

</style>
