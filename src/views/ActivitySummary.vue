<template lang="pug">
div
  h3 Summary for {{ periodReadable }}
  | {{ periodStartShort }} - {{ periodEndShort }}
  br
  | Active time: {{ readableDuration }}

  b-alert(variant="danger" :show="errormsg.length > 0")
    | {{ errormsg }}

  aw-periodusage(:periodusage_arr="period_activity", :link_prefix="link_prefix" :dateformat="dateformat")

  ul.nav.nav-tabs.my-3
    li.nav-item.aw-nav-item
      a.nav-link.aw-nav-link(@click="periodLength = 'week'" :class="{ active: periodLength == 'week' }")
        h5 Weekly
    li.nav-item.aw-nav-item
      a.nav-link.aw-nav-link(@click="periodLength = 'month'" :class="{ active: periodLength == 'month' }")
        h5 Monthly
    li.nav-item.aw-nav-item
      a.nav-link.aw-nav-link(@click="periodLength = 'year'" :class="{ active: periodLength == 'year' }")
        h5 Yearly

  aw-summaryview(:period="period")

</template>

<style scoped lang="scss">
.aw-nav-link {
  background-color: #eee;
  border: 2px solid #eee !important;
  border-bottom: none !important;
  margin-left: 0.1em;
  margin-right: 0.1em;
  border-top-left-radius: 0.5rem !important;
  border-top-right-radius: 0.5rem !important;
}

.aw-nav-link:hover {
  background-color: #fff;
}

.aw-nav-item:hover {
  background-color: #fff !important;
}
</style>

<script>
import moment from 'moment';
import time from "../util/time.js";
import query from '../queries.js';

export default {
  name: "ActivitySummary",

  data: () => {
    return {
      errormsg: "",
      duration: 0,
      periodLength: "week",
      period_activity: [],
    }
  },

  computed: {
    dateformat: function() {
        if (this.periodLength == "week") { return "YYYY[ W]WW"; }
        else if (this.periodLength == "month") { return "YYYY-MM"; }
        else if (this.periodLength == "year") { return "YYYY"; }
    },
    periodReadable: function() { return moment(this.periodStart).format(this.dateformat); },
    readableDuration: function() { return time.seconds_to_duration(this.duration) },
    host: function() { return this.$route.params.host },
    date: function() {
      var dateParam = this.$route.params.date;
      var dateMoment = dateParam ? moment(dateParam) : moment();
      return dateMoment.format();
    },
    periodStart: function() { return moment(this.date).startOf(this.periodLength).format(); },
    periodEnd: function() { return moment(this.periodStart).add(1, this.periodLength).format(); },
    period: function() { return this.periodStart + "/" + this.periodEnd },
    periodStartShort: function () { return moment(this.periodStart).format("YYYY-MM-DD") },
    periodEndShort: function () { return moment(this.periodEnd).format("YYYY-MM-DD") },
    windowBucketId: function() { return "aw-watcher-window_" + this.host },
    afkBucketId:    function() { return "aw-watcher-afk_"    + this.host },
    link_prefix:    function() { return "/activity/summary/"   + this.host },
  },

  watch: {
    '$route': function() {
      console.log("Route changed");
      this.refresh();
    },
    'periodLength': function() {
      this.refresh();
    },
  },

  mounted() {
    this.refresh();
  },

  methods: {
    setDate: function(date) { this.$router.push(this.link_prefix+'/'+date); },

    errorHandler: function(error) {
      this.errormsg = "" + error + ". See dev console (F12) and/or server logs for more info.";
      throw error;
    },

    refresh: function() {
      this.queryPeriodActivity();
    },

    queryPeriodActivity: async function() {
      var timeperiods = [];
      for (var i=-8; i<=8; i++) {
        var start = moment(this.periodStart).add(i, this.periodLength).format();
        var end = moment(start).add(1, this.periodLength).format();
        timeperiods.push(start + '/' + end);
      }
      this.period_activity = await this.$aw.query(timeperiods, query.dailyActivityQuery(this.afkBucketId)).catch(this.errorHandler);
      if (this.period_activity[8].length > 0) {
        this.duration = this.period_activity[8][0].duration;
      } else {
        this.duration = 0;
      }
    },
  }
}
</script>
