<template>
  <div id="app">
    <button class="normalButton" v-on:click="openLogWebSocket" :disabled="isSocketOn">Start</button>
    <button class="normalButton" v-on:click="closeLogWebSocket" :class="{hidden:!isSocketOn}">Stop</button>
    <input type="text" class="search" v-model="searchText" v-on:keypress="searchInputKeypressed" placeholder="search">
    <span style="color:grey" v-if="this.searchText.length>0" v-bind:style="searchSpanStyle">{{this.searchItems.length + ' logs contains "' + this.searchText + '"'}}</span>
    <div class="listWrap" ref="listWrap">
      <virtual-list ref="vsl" class="list" :start="startIndex" :variable="getVariableHeight" 
      :size="78" 
      :remain="7">
        <item
          v-for="(item, index) of filterItems" 
          :key="index"
          :index="index"
          :height="item.height"
          :content="item.content"
          :textStyle="item.textStyle"> 
        </item>
      </virtual-list>
    </div>
    <input type="text" class="filter" v-model="filterText" placeholder="Filter">
  </div>
</template>

<script>
import Vue from 'vue'
import VueNativeSock from 'vue-native-websocket'
import virtualList from 'vue-virtual-scroll-list';
import item from './Item.vue';

let url = getQueryString("host");
if (url != null) {
  Vue.use(VueNativeSock, getQueryString("host"), { 
          connectManually: true,
          reconnection: true, 
          reconnectionAttempts: 10,
          reconnectionDelay: 5000,
        });
}

export default {
  name: 'app',
  components: { item, 'virtual-list': virtualList },

  data () {
    return {
      items: new Array({height: 40, content: 'Coolog', textStyle:'color:#1E90FF;font-size:26px'}),
      searchItems: new Array(),

      scrolling: false,
      isSocketOn: false,
      isSocketConnecting: false,
      startIndex: 0,

      filterText: '',
      searchText: '',

      searchSpanStyle: {
        visibility: 'hidden',
      }
    }
  },

  watch: {
    searchText: function (val) {
      this.searchSpanStyle.visibility = 'hidden';
    }
  },

  computed: {
    filterItems: function () {
      var filterItems = this.items;
      return filterItems.filter(this.itemFilter);
    },
  },

  methods: {
    getVariableHeight (index) {
      let target = this.items[index];
      return target.height;
    },

    openLogWebSocket () {
      if (this.isSocketConnecting) {
        return;
      }
      this.isSocketConnecting = true;
      this.$connect();
      this.$options.sockets.onopen = function () {
        this.isSocketOn = true;
      };
      this.$options.sockets.onclose = function () {
        this.isSocketOn = false;
        this.isSocketConnecting = false;
        this.closeLogWebSocket();
        this.removeWebSocketListener();
      };
      this.$options.sockets.onerror = function (event) {
        this.isSocketOn = false;
        this.isSocketConnecting = false;
        this.closeLogWebSocket();
        this.removeWebSocketListener();
      };
      this.$options.sockets.onmessage =  function (evt) {
        let rowHeight = 15 * Math.ceil((evt.data.length / 134)+1);
        let type = evt.data.match(/\[##TYPE:(\S*)##\]/)[1];
        var textColor = '#000000';
        if (type === 'Debug') {
          textColor = '#696969';
        } else if (type === 'Info') {
          textColor = '#006400';
        } else if (type === 'Warning') {
          textColor = '#FF7F50';
        } else if (type === 'Error') {
          textColor = '#FF0000';
        }
        let itm = {
          height: rowHeight,
          content: evt.data,
          textStyle: 'color:'+textColor
        };
        this.items.push(itm);
        this.startIndex = this.items.length;
      };
    },

    closeLogWebSocket () {
      this.$disconnect();
    },

    removeWebSocketListener () {
      delete this.$options.sockets.onopen;
      delete this.$options.sockets.onclose;
      delete this.$options.sockets.onerror;
      delete this.$options.sockets.onmessage;
    },

    searchInputKeypressed (event) {
      if (event.keyCode != 13) {
        return;
      }
      this.searchSpanStyle.visibility = 'visible';
      this.searchItems = this.getSearchItems();
      if (this.searchItems.length > 0) {
        this.startIndex = this.searchItems[0].index;
        // var searchInnerHTML = this.$refs.listWrap.innerHTML;
        // let re = new RegExp(this.searchText+'(?![^<>]*>)', 'g');
        // this.$refs.listWrap.innerHTML = searchInnerHTML.replace(re, '<span style="color:#0f0;background-color:#ff0">'+this.searchText+'</span>');
      }
    },

    getSearchItems () {
      var searchItems = this.filterItems;
      searchItems = searchItems.filter(this.itemSearcher);
      return searchItems;
    },

    itemFilter (item) {
      return (item.content.indexOf(this.filterText) != -1);
    },

    itemSearcher (item) {
      return (item.content.indexOf(this.searchText) != -1);
    }
  }
}

function getQueryString(name) {  
  var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)", "i");  
  var r = window.location.search.substr(1).match(reg);  
  if (r != null) return unescape(r[2]); return null;  
}
</script>

<style>
    .normalButton {
      position: relative;
      font-family: Helvetica;
      margin: 5px 0px 5px 0px;
      font-size: 16pt;
    }

    .hidden {
      display: none;
    }

    .listWrap {
      position: relative;
      height: 560px;
    }

    .list {
      ackground: #fff;
      border-radius: 3px;
      border: 1px solid #ddd;
      -webkit-overflow-scrolling: touch;
      overflow-scrolling: touch;
    }

    .filter {
      position: relative;
      width: 150px;
      font-size: 16pt;
    }

    .search {
      position: relative;
      width: 250px;
      font-size: 14pt;
    }

</style>