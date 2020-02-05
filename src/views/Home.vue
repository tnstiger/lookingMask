<template>
  <div>
    <div class="map" ref="map" v-if="siteOpen"></div>
    <div class="info" v-if="siteOpen">
      <template v-if="Object.keys(selectedStore).length == 0">
        <p class="title is-4">請先從上方地圖選擇超商</p>
        <p class="has-text-success">綠色: 三小時內有回報口罩充足</p>
        <p class="has-text-danger">紅色: 三小時內有回報口罩售完</p>
        <p>灰色: 三小時內無任何回報訊息</p>
      </template>
      <template v-else>
        <p class="title is-4">{{ selectedStore.name }}</p>
        <p class="subtitle is-5">回報目前口罩數量</p>
        <!-- <p class="label">成人口罩</p> -->
        <div class="buttons">
          <b-button type="is-danger" @click="submit('SHORTAGE')">售完</b-button>
          <b-button type="is-success" @click="submit('AVAILABLE')">充足</b-button>
        </div>
        <div class="record" v-for="record in records.slice(0, 10)">
          <article
            class="message"
            :class="{
              'is-danger': record.status == 'SHORTAGE',
              'is-success': record.status == 'AVAILABLE'
            }"
          >
            <div class="message-body">
              {{ record.created_at }} 回報口罩「{{
              status[record.status].text
              }}」
            </div>
          </article>
        </div>
      </template>
    </div>
    <b-modal :active.sync="isModalActive">
      <div class="information">
        <p class="title is-5">2020/02/04 更新</p>
        <p>
          政府徵用的口罩，在民生方面，原先超商通路自明(4)日起停止販售，請不要再前往超商購買口罩，本服務也將暫停使用。
          <br />
          <a target="_blank" href="https://www.mohw.gov.tw/cp-16-51286-1.html">公告連結</a>
        </p>

        <div class="modal-btn">
          <b-button type="is-dark" @click="iknow" expanded>我知道了</b-button>
        </div>
      </div>
    </b-modal>
    <div class="copyright">
      © 2020 | 好想工作室
      <a target="_blank" href="https://www.facebook.com/chanwei.wu">Howard</a> |
      <a target="_blank" href="https://www.facebook.com/weijen25">WJWang</a> |
      <a target="_blank" href="https://www.facebook.com/enshuo.hsu">enshuo@</a>
      |
      <a
        target="_blank"
        href="https://www.facebook.com/profile.php?id=100001664467434"
      >JohnLin</a>
      <div class="google">
        <a href="https://www.meetup.com/GDG-Tainan/" target="_blank">
          <img class="tainan" :src="require('@/assets/gdg-tainan.png')" alt />
        </a>
        <a href="https://cloud.google.com/" target="_blank">
          <img class="cloud" :src="require('@/assets/cloud-logo.png')" alt />
        </a>
      </div>
      <div>sponsored by Google Cloud</div>
    </div>
  </div>
</template>

<script>
import gmapsInit from "@/utils/gmaps";
import axios from "axios";
import dayjs from "dayjs";
import cookie from "js-cookie";
import { APIURL } from "@/config.js";

const CancelToken = axios.CancelToken;

let cancel,
  google,
  map,
  infoWindow,
  localPosition,
  markers = [],
  announcement = "0203";

let status = {
  UNKNOWN: {
    text: "無紀錄",
    color: "999999"
  },
  SHORTAGE: {
    text: "售完",
    color: "F14668"
  },
  AVAILABLE: {
    text: "充足",
    color: "48C774"
  }
};

export default {
  name: "home",
  data() {
    return {
      siteOpen: true,
      selectedStore: {},
      selectedMarker: null,
      records: [],
      status: status,
      isModalActive: true
    };
  },
  async mounted() {
    if (this.siteOpen) {
      this.initMap();
    }
    this.isModalActive =
      cookie.get("announcement") == announcement ? false : true;
  },
  methods: {
    async initMap() {
      google = await gmapsInit();
      infoWindow = new google.maps.InfoWindow();
      navigator.geolocation.getCurrentPosition(
        position => {
          this.launchMap(position.coords.latitude, position.coords.longitude);
        },
        // error case, use third part geo IP if position not found
        async () => {
          let geoInfo = (await axios.get("https://ipapi.co/json/")).data;
          alert(
            "因為無法從瀏覽器上取得地理位置，地圖起始位置會有所偏差，FB、Line 裡面的瀏覽器不允許取得地理位置，建議用原生瀏覽器開啟。"
          );
          this.launchMap(geoInfo.latitude, geoInfo.longitude);
        },
        { enableHighAccuracy: true, timeout: 10000, maximumAge: 60000 }
      );
    },
    launchMap(latitude, longitude) {
      // set default map
      localPosition = new google.maps.LatLng(latitude, longitude);
      map = new google.maps.Map(this.$refs.map, {
        center: localPosition,
        zoom: 15
      });

      // new store import flow
      google.maps.event.addListener(map, "click", e => {
        if (e.placeId) {
          setTimeout(async () => {
            if (confirm("把這個地點加入回報地標？")) {
              infoWindow.close();
              infoWindow.open(map);
              let name = prompt("請輸入此地標名稱");
              if (name && name != undefined && !/[0-9a-zA-Z\."!]+/.test(name)) {
                let store = { name, lat: e.latLng.lat(), lng: e.latLng.lng() };
                await axios.post(`${APIURL}/store`, store);
                alert("已將加入此地標");
                this.searchStore();
              }
            }
          }, 500);
        }
      });

      this.searchStore();

      // reload store when map dragend
      map.addListener("dragend", () => {
        localPosition = map.getCenter();
        if (cancel) {
          try {
            cancel();
          } catch (e) {}
        }
        this.searchStore();
      });
    },
    async searchStore() {
      let params = {
        radius: 1000,
        lat: localPosition.lat(),
        lng: localPosition.lng()
      };

      let stores = (
        await axios({
          method: "GET",
          url: `${APIURL}/store`,
          params: params,
          cancelToken: new CancelToken(function executor(c) {
            cancel = c;
          })
        })
      ).data;

      let records = (
        await axios({
          method: "GET",
          url: `${APIURL}`,
          params: { stores: stores.map(place => place.id).join(",") },
          cancelToken: new CancelToken(function executor(c) {
            cancel = c;
          })
        })
      ).data;

      // clear marker
      while (markers.length) {
        markers.pop().setMap(null);
      }

      // add marker and marker click event
      stores.forEach(store => {
        let lastRecord = records.find(record => {
          return (
            record.store == store.name &&
            dayjs().isBefore(dayjs(record.created_at).add(3, "hour"))
          );
        });
        let markerColor = lastRecord
          ? status[lastRecord.status].color
          : status["UNKNOWN"].color;
        let marker = new google.maps.Marker({
          map,
          position: { lat: store.lat, lng: store.lng },
          title: store.name,
          label: {
            text: store.name,
            fontSize: "12px",
            color: "#333"
          },
          icon: {
            url: `https://cdn.mapmarker.io/api/v1/pin?icon=fa-star&size=50&background=${markerColor}`
          }
        });

        marker.addListener("click", () => {
          this.selectedStore = store;
          this.selectedMarker = marker;
          this.records = records.filter(record => {
            return record.store == store.name;
          });
        });

        markers.push(marker);
      });
    },
    async submit(selectedStatus) {
      let payload = {
        store_id: this.selectedStore.id,
        store: this.selectedStore.name,
        status: selectedStatus
      };
      if (confirm(`確認回報口罩狀態「${status[selectedStatus].text}」?`)) {
        await axios.post(`${APIURL}`, {
          ...payload
        });

        this.selectedMarker.setIcon({
          url: `https://cdn.mapmarker.io/api/v1/pin?icon=fa-star&size=50&background=${status[selectedStatus].color}`
        });
        this.records.unshift({
          ...payload,
          created_at: dayjs().format("YYYY-MM-DD HH:mm:ss")
        });
        alert("已送出， 感謝回報");
      }
    },
    iknow() {
      this.isModalActive = false;
      cookie.set("announcement", announcement);
    }
  }
};
</script>
<style lang="scss" scoped>
.map {
  height: 50vh;
}
.info {
  margin-top: 30px;
  text-align: center;
  padding-bottom: 50px;
}
.buttons {
  justify-content: center;
}
.message {
  margin: 10px 30px;
}
.copyright {
  background-color: #e4e4e4eb;
  position: fixed;
  bottom: 0;
  width: 100%;
  padding: 10px;
  font-size: 12px;
  font-weight: 600;
  text-align: center;
  color: #636363;
  a {
    color: #636363;
    text-decoration: underline;
  }
}
.information {
  background-color: #fff;
  // color: red;
  padding: 30px;
  .modal-btn {
    margin-top: 30px;
  }
}
.google {
  height: 30px;
  img {
    height: 40px;
    &.cloud {
      height: 17px;
      margin: 12px 0;
    }
  }
}
/deep/ .gm-style-mtc,
/deep/ .gm-fullscreen-control,
/deep/ .gm-svpc,
/deep/ .gmnoprint,
/deep/ .gm-style-cc {
  display: none;
}
</style>
