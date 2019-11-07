<template>
  <div>
    <b-container class="no-print">
      <b-row>
        <b-col>
          <label class="title">DCRA PDF Text</label>
          <b-form-textarea
            v-model="dcraPdfText"
            placeholder="Paste PDF to text conversion here"
            rows="3"
          ></b-form-textarea>
        </b-col>
        <b-col>
          <label class="title">License Name Map</label>
          <b-form-textarea v-model="licenseMapText" placeholder="13CAP-00000034,bob" rows="3"></b-form-textarea>
        </b-col>
        <b-col>
          <div class="text-center">
            <b-button variant="primary" size="lg" @click="generate()">Generate</b-button>
          </div>
        </b-col>
      </b-row>
    </b-container>
    <b-container fluid class="mt-3" v-if="dcra.length > 0">
      <h1 class="date">{{ dcra[0].date }} SRV Location Lottery Results</h1>
      <b-tabs>
        <b-tab title="Standard" active>
          <b-row>
            <b-col class="table-header">License</b-col>
            <b-col class="table-header">Name</b-col>
            <b-col class="table-header">Sunday</b-col>
            <b-col class="table-header">Monday</b-col>
            <b-col class="table-header">Tuesday</b-col>
            <b-col class="table-header">Wednesday</b-col>
            <b-col class="table-header">Thursday</b-col>
            <b-col class="table-header">Friday</b-col>
            <b-col class="table-header">Saturday</b-col>
          </b-row>
          <b-row v-for="(r, i) in dcra" :key="i">
            <b-col class="table-item" :class="{ even: (i%2)==0 }">{{ r.license }}</b-col>
            <b-col class="table-item" :class="{ even: (i%2)==0 }">{{ r.name }}</b-col>
            <b-col
              class="table-item"
              :class="{ even: (i%2)==0 }"
              v-for="(s, j) in r.spots"
              :key="j"
            >{{ s }}</b-col>
          </b-row>
        </b-tab>
        <b-tab title="Day of Week">
          <b-row>
            <b-col class="table-header">Day Of Week</b-col>
            <b-col class="table-header">License</b-col>
            <b-col class="table-header">Name</b-col>
            <b-col class="table-header">Spot</b-col>
          </b-row>
          <b-row
            v-for="(r, i) in dcra2"
            :key="i"
            :class="{ 'page-break': (i<(dcra2.length-1) && dcra2[i].day!=dcra2[i+1].day) }"
          >
            <b-col class="table-item table-item-dayview" :class="{ even: (i%2)==0 }">{{ r.day }}</b-col>
            <b-col class="table-item table-item-dayview" :class="{ even: (i%2)==0 }">{{ r.license }}</b-col>
            <b-col class="table-item table-item-dayview" :class="{ even: (i%2)==0 }">{{ r.name }}</b-col>
            <b-col class="table-item table-item-dayview" :class="{ even: (i%2)==0 }">{{ r.spot }}</b-col>
          </b-row>
        </b-tab>
      </b-tabs>
    </b-container>
  </div>
</template>

<script>
import axios from "axios";

export default {
  name: "Home",
  async mounted() {
    this.dcraPdfText = (await axios.get("static/dcra.txt")).data;

    let data = this.$route.query.json;
    if (data) {
      try {
        let json = JSON.parse(atob(data));
        //this.dcraPdfText = json.dcra;
        this.licenseMapText = json.license;
        this.generate();
      } catch (ex) {
        // ...
      }
    }
  },
  methods: {
    generate() {
      let dcraText = this.dcraPdfText;

      //
      // sanitization
      //
      dcraText = dcraText.replace(/[^a-z0-9\s-]/gi, "");
      dcraText = dcraText
        .split("\n")
        .filter(l => l.length > 0)
        .join(" ");

      this.dcraPdfText = dcraText;

      //
      // save information to ls
      //
      if (this.licenseMapText) {
        this.$router.push({
          path: "/",
          query: {
            json: btoa(
              JSON.stringify({
                //dcra: dcraText,
                license: this.licenseMapText
              })
            )
          }
        });
      }

      //
      // we want dcraData to contain data looking like the following:
      // 13CAP-00000034 November 2019 OFF Food - Spot 34 OFF OFF Food - Spot 28 OFF OFF
      //
      let dcraData = [];
      let i = 0;
      while (i < dcraText.length) {
        let i2 = dcraText.indexOf("CAP-", i + 4) - 3;
        if (i2 < 0) i2 = dcraText.length;
        let str = dcraText.substring(i, i2);
        dcraData.push(str.trim());
        i = i2;
      }

      dcraData = dcraData.splice(1);

      let dcra = dcraData.map(row => {
        let data = row
          .split(" ")
          .map(s => s.trim())
          .filter(d => d.length > 1);
        let spots = [];

        for (let i = 3; i < data.length && spots.length < 7; i++) {
          if (data[i] == "OFF") {
            spots.push(data[i]);
          } else {
            // "Food - Spot 34" -> [ 'Food', Spot', '34' ]
            spots.push(`${data[i]} ${data[i + 2]}`);
            i += 2;
          }
        }

        return {
          license: data[0],
          name: "",
          date: `${data[1]} ${data[2]}`,
          spots: spots
        };
      });

      // update names from license map
      this.licenseMapText.split("\n").forEach(n => {
        if (!n) return;
        let licenseName = n.trim().split(",");

        let item = dcra.find(r => r.license == licenseName[0]);
        if (item) item.name = licenseName[1] || "";
      });

      this.dcra = dcra;

      let dcra2 = [];
      const DAY_OF_WEEK = [
        "Sunday",
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
        "Saturday"
      ];
      for (let i = 0; i < 7; i++) {
        let dayGroup = [];
        for (let j = 0; j < dcra.length; j++) {
          let spot = dcra[j].spots[i];
          if (spot == "OFF") continue;

          let spotNumber = parseInt(spot.match(/[0-9]+/)[0]); // extract spot number

          dayGroup.push({
            day: DAY_OF_WEEK[i],
            spot: spot,
            spotNumber: spotNumber,
            license: dcra[j].license,
            name: dcra[j].name
          });
        }

        dayGroup
          .sort((i, i2) => i.spotNumber - i2.spotNumber)
          .forEach(d => dcra2.push(d));
      }

      this.dcra2 = dcra2;
    }
  },
  data() {
    return {
      dcraPdfText: "",
      licenseMapText: "",
      dcra: [],
      dcra2: []
    };
  }
};
</script>

<style scoped>
@media print {
  .no-print,
  .no-print * {
    display: none !important;
  }
  .page-break {
    page-break-after: always;
  }
}

label.title {
  font-size: 18px;
  font-weight: bold;
}

.table-header {
  font-weight: bold;
}

.table-item {
  background: white;
}

.table-item.even {
  background-color: #ffe6ea;
}

.table-item-dayview.even {
  background-color: #ffddb3;
}
</style>
