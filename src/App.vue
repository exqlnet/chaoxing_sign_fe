<template>
  <div id="app">
    <mt-navbar v-model="selected">
      <mt-tab-item id="1">选择签到人</mt-tab-item>
      <mt-tab-item id="2">未签到</mt-tab-item>
      <mt-tab-item id="3">已签到</mt-tab-item>
    </mt-navbar>
  <!-- tab-container -->
    <mt-tab-container v-model="selected">
      <mt-tab-container-item id="1">
        <mt-checklist
          title="选择需要签到的人"
          v-model="usersSelected"
          :options="users">
        </mt-checklist>
        <div class="confirm">
          <mt-button type="primary" @click.native="startSign">确认</mt-button>
        </div>
      </mt-tab-container-item>
      <mt-tab-container-item id="2">
        <p></p>
        <div class="scan-qrcode">
          <qrcode-stream @decode="onDecode" v-if="scanOpen" />
        </div>
        <div class="scan-open">
          <mt-switch v-model="scanOpen">开启扫码</mt-switch>
        </div>
        <div class="notOK">
          <mt-cell v-for="user, i in usersToSign" :title="user.name" v-bind:key="i" />
        </div>
 
      </mt-tab-container-item>
      <mt-tab-container-item id="3">
        <p></p>
        <div class="OK">
          <mt-cell v-for="user, i in usersSigned" :title="user.name" v-bind:key="i" />
        </div>
      </mt-tab-container-item>
    </mt-tab-container>
  </div>
</template>

<script>
import { QrcodeStream, QrcodeDropZone, QrcodeCapture } from 'vue-qrcode-reader'
import { Toast, Indicator } from 'mint-ui';
import Vue from "vue";

function getParameterByName(name, url = window.location.href) {
    name = name.replace(/[\[\]]/g, '\\$&');
    var regex = new RegExp('[?&]' + name + '(=([^&#]*)|&|#|$)'),
        results = regex.exec(url);
    if (!results) return null;
    if (!results[2]) return '';
    return decodeURIComponent(results[2].replace(/\+/g, ' '));
}

export default {
  name: 'App',
  data() {
    return {
      usersSelected: [],
      users: [],
      usersToSign: [],
      usersSigned: [],
      selected: "1",
      scanOpen: false,
      apiKey: "",
    }
  },
  components: {
    QrcodeStream,
  },
  mounted() {
    if (!this.setUpApiKey()) {
      return 
    }
    this.axios.get("/scan").then(res => {
      for (var i = 0; i < res.data.length; i++) {
        this.users.push(
          {
            "value": res.data[i].username,
            "label": res.data[i].name,
          }
        );
      }
    })
  },
  methods: {
    setUpApiKey() {
      this.apiKey = getParameterByName("api-key");
      if (this.apiKey == "" || this.apiKey == null) {
        Toast("403: 无权限访问")
        return false
      }
      console.log("apiKey: " + this.apiKey);
      Vue.axios.defaults.headers.common['api-key'] = this.apiKey;
      return true
    },
    startSign() {
      this.usersToSign = []
      for (var i = 0; i < this.users.length; i++) {
        if (this.usersSelected.indexOf(this.users[i].value) >= 0) {
          this.usersToSign.push({username: this.users[i].value, name: this.users[i].label})
        }
      }
      Toast("开始签到")
      this.selected = "2"
    },
    onDecode(content) {
      if (content === "") {
        return
      }
      let enc = content.split("&").filter(s => s.startsWith("enc="))
      if (enc.length === 0) {
        return
      }
      enc = enc[0].substr(4)
      
      this.scanOpen = false
      Indicator.open("正在签到...")
      let usersToSign = []
      for (let i = 0; i < this.usersToSign.length; i++) {
        usersToSign.push(this.usersToSign[i].username)
      }
      let data = {
        sign_users: usersToSign,
        enc: enc,
      }
      let self = this
      this.axios.post("/scan", data).then(res => {
        for (let i = 0; i < res.data.length; i++) {
          this.usersSigned.push(res.data[i])
        }
        let newUsersToSign = this.usersToSign.filter(u => {
          for (let i = 0; i < res.data.length; i++) {
            if (res.data[i].username === u.username) {
              return false
            }
          }
          return true
        })
        self.usersToSign = newUsersToSign
        Indicator.close()
        let names = res.data.map(u => u.name)
        Toast(`签到成功：${names.join(', ')}`)
      }).catch(() => {Indicator.close()})
    }
  }
}
</script>

<style>
div.confirm {
  margin-top: 5px;
}

div.OK {
  margin-top: 5px;
}

div.notOK {
  margin-top: 5px;
}
div.scan-qrcode {
  border: 1px solid;
  height: 200px;
  width: 60%;
  display: inline-block;
}
div.scan-open {
  display: inline-block;
}
</style>
