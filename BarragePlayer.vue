<template>
  <v-container>
    <div id="video" ref="videowindow">
      <div v-if="ifpoptext" id="poptextdiv" ref="poptextdiv"></div>
      <transition name="scale-transition">
        <div v-if="loading" id="loading">
          <v-row class="fill-height ma-0" align="center" justify="center">
            <v-progress-circular indeterminate color="primary" :size="70" :width="7"></v-progress-circular>
          </v-row>
        </div>
      </transition>
      <transition name="scale-transition">
        <div v-if="!ifpause&&!loading" id="ifpause" @click="play">
          <v-row class="fill-height ma-0" align="center" justify="center">
            <v-icon color="primary" size="80px">mdi-play-circle</v-icon>
          </v-row>
        </div>
      </transition>
      <video
        :src="this.initVideo.url"
        width="100%"
        height="100%"
        preload="auto"
        ref="video"
        @play="onPlayerPlay($event)"
        @pause="onPlayerPause($event)"
        @statechanged="playerStateChanged($event)"
        @canplay="onPlayerCanplay($event)"
        @error="onError($event)"
        @timeupdate="videoTimeUpdate"
        @click="play"
        @loadedmetadata="getVideoLength"
        @ended="onended($event)"
        @loading="onloadeddata($event)"
      >
        请更换现代浏览器
        <track
          v-id="ifcctext"
          kind="subtitles"
          src="/static/testcc.vtt"
          srclang="zh"
          label="简体中文"
          default
        />
      </video>
      <transition name="scroll-y-reverse-transition">
        <div id="control" ref="control" :style="controlshow">
          <v-slider
            class="timeline"
            dense
            :max="initVideo.videoLength"
            v-model="initVideo.currentTime"
            @change="changeVideoTime"
          ></v-slider>
          <v-row>
            <v-btn class="pa-1 ml-4 mt-1" icon @click="play">
              <v-icon v-if="playState === 'play'" color="white">mdi-play</v-icon>
              <v-icon v-else-if="playState === 'pause'" color="white">mdi-pause</v-icon>
              <v-icon v-else-if="playState === 'replay'" color="white">mdi-replay</v-icon>
            </v-btn>
            <div
              class="pa-1 ml-4 mt-1 white--text"
            >{{initVideo.currentTime | videoTime}} | {{initVideo.videoLength | videoTime}}</div>
            <v-text-field
              v-model="sendertext"
              :disabled="!this.$cookies.isKey('user')"
              solo-inverted
              rounded
              :label="$t('message.poptext')"
              color="white"
              dense
              dark
              full-width
              @focus="senderfocused"
            ></v-text-field>
            <v-btn class="pa-1 mr-1 ml-2 mt-1" icon @click="pushpoptext">
              <v-icon color="white">mdi-send</v-icon>
            </v-btn>
            <v-btn
              class="pa-1 mr-1 ml-2 mt-1"
              icon
              @click="showpoptextsetting=!showpoptextsetting,showvolume=false,showsendersetting=false"
            >
              <v-icon color="white">mdi-format-font</v-icon>
            </v-btn>
            <transition name="scroll-y-reverse-transition">
              <v-card
                dark
                v-if="showsendersetting"
                min-width="250px"
                id="sendersetting"
                class="pl-5 pr-5 pt-1 pb-1"
              >
                <v-row>
                  <v-color-picker v-model="colorpicker" mode="HEX"></v-color-picker>
                </v-row>
                <v-row>
                  <v-switch v-model="isrow" color="primary" label="滚动弹幕"></v-switch>
                </v-row>
                <v-row>预览：</v-row>
                <v-row>
                  <p id="textpre" class="title center">效果预览</p>
                  <br />
                  <br />
                </v-row>
              </v-card>
            </transition>
            <transition name="scroll-y-reverse-transition">
              <v-card
                dark
                v-if="showpoptextsetting"
                min-width="250px"
                id="poptextsetting"
                class="pl-5 pr-5 pt-4 pb-1"
              >
                <v-slider
                  color="primary"
                  label="弹幕屏占比:"
                  v-model="channelmax"
                  z-index="10"
                  thumb-label
                  dense
                  max="60"
                  min="1"
                ></v-slider>
                <v-slider
                  color="primary"
                  label="弹幕透明度:"
                  v-model="poptextop"
                  z-index="10"
                  thumb-label
                  dense
                  max="100"
                  min="0"
                ></v-slider>
                <v-switch v-model="allowoverwrite" color="primary" label="允许溢出"></v-switch>
              </v-card>
            </transition>
            <v-btn
              class="pa-1 mr-1 ml-2 mt-1"
              icon
              @click="ifpoptext=!ifpoptext,showvolume=false,showpoptextsetting=false,showsendersetting=false"
            >
              <v-icon v-if="ifpoptext" color="white">mdi-message-bulleted</v-icon>
              <v-icon v-else color="white">mdi-message-bulleted-off</v-icon>
            </v-btn>
            <v-btn
              class="pa-1 mr-2 ml-2 mt-1"
              icon
              @click="showvolume=!showvolume,showpoptextsetting=false,showsendersetting=false"
            >
              <v-icon v-if="volume>=70" color="white">mdi-volume-high</v-icon>
              <v-icon v-if="volume>=40&&volume<70" color="white">mdi-volume-medium</v-icon>
              <v-icon v-if="volume>0&&volume<40" color="white">mdi-volume-low</v-icon>
              <v-icon v-if="volume==0" color="white">mdi-volume-mute</v-icon>
            </v-btn>
            <transition name="scroll-y-reverse-transition">
              <v-card v-if="showvolume" id="volume">
                <v-row>
                  <v-col cols="12">
                    <v-slider
                      v-model="volume"
                      z-index="10"
                      thumb-label
                      @input="changevolume"
                      @change="closevolume"
                      dense
                      vertical
                      max="100"
                      min="0"
                    ></v-slider>
                  </v-col>
                </v-row>
              </v-card>
            </transition>

            <v-btn class="pa-1 mr-4 mt-1" icon @click="fullScreen">
              <v-icon v-if="!fullscreen" color="white">mdi-fullscreen</v-icon>
              <v-icon v-else color="white">mdi-fullscreen-exit</v-icon>
            </v-btn>
          </v-row>
        </div>
      </transition>
    </div>
  </v-container>
</template>

<script>
export default {
  data: () => ({
    colorpicker: "#FFFFFF", //默认弹幕颜色
    isrow: true, //默认发送滚动弹幕
    sendertext: "", //默认弹幕文字
    showsendersetting: false, //默认是否打开发送设置
    poptextop: 0,
    showpoptextsetting: false, //默认是否打开弹幕设置
    allowoverwrite: true, //允许弹幕溢出覆盖
    channelmax: 40, //默认最大弹道数
    channelrow: 40, //目前滚动弹道数
    channeltop: 40, //目前竖直固定弹道数
    timenow: "", //现在的时间
    ifpoptext: true, //默认是否显示弹幕
    ifcctext: true, //默认是否显示字幕
    ifpause: false,
    controlshow: "opacity:1",
    showvolume: false,
    loading: true,
    volume: 100, //默认音量
    src: "",
    playState: "play",
    haserror: "",
    fullscreen: null,
    initVideo: {
      play: false, //播放还是暂停 true播放中
      videoLength: 0, //时长
      url: "", //视频Url
      currentTime: 0, //当前播放时间
      lastTime: null, //标记时间戳
      name: ""
    },

    params: "",
    poptexttest: ""
  }),
  components: {},
  methods: {
    play() {
      this.showsendersetting = false;
      this.showvolume = false;
      this.showpoptextsetting = false;
      //播放视频
      if (this.initVideo.play) {
        this.playState = "play";
        this.ifpause = false;
        this.$refs.video.pause();
      } else {
        Math.abs(this.initVideo.currentTime - this.$refs.video.currentTime) > 2
          ? (this.$refs.video.currentTime = this.initVideo.currentTime)
          : "";
        this.ifpause = true;
        this.$refs.video.play();
        this.playState = "pause";
      }
    },
    onError(e) {
      this.haserror = false;
      if (this.src != "") {
        this.playState = "replay";
        this.haserror = true;
      }
    },
    onPlayerCanplay(event) {
      this.haserror = false;
      this.loading = false;
    },
    onPlayerPlay(event) {
      this.playState = "pause";
      this.initVideo.play = true;
      this.controlshow = " ";
    },
    onPlayerPause(event) {
      this.playState = "play";
      this.initVideo.play = false;
      this.controlshow = "opacity:1;";
    },
    fullScreen() {
      //全屏
      this.showsendersetting = false;
      this.showvolume = false;
      this.showpoptextsetting = false;
      if (this.isFullscreen()) {
        document.webkitExitFullscreen();
      } else {
        this.$refs.videowindow.webkitRequestFullScreen();
        this.$refs.control.webkitRequestFullScreen();
        this.$refs.volume.webkitRequestFullScreen();
      }
    },
    isFullscreen() {
      return (
        document.fullscreenElement ||
        document.msFullscreenElement ||
        document.mozFullScreenElement ||
        document.webkitFullscreenElement ||
        false
      );
    },
    videoTimeUpdate() {
      //更新视频时间。节流，每秒触发一次
      let nowTime = Date.now();
      let gapTime = 1000;
      this.timenow = this.videoTime1(this.$refs.video.currentTime);

      if (
        !this.initVideo.lastTime ||
        nowTime - this.initVideo.lastTime > gapTime
      ) {
        if (this.$refs.video) {
          let time = this.$refs.video.currentTime;
          this.initVideo.currentTime = time;
        }
        this.initVideo.lastTime = nowTime;
      }
    },
    changeVideoTime(val) {
      //改变视频时间
      if (this.$refs.video) {
        this.$refs.video.currentTime = val;
      }
    },
    getVideoLength() {
      //获取视频时长
      this.initVideo.videoLength = this.$refs.video.duration;
    },
    handPlay(status) {
      //视频播放暂停 status 1播放 2暂停

      status == 1
        ? (this.initVideo.play = true)
        : (this.initVideo.play = false);
    },
    ended(event) {
      this.play();
    },
    loading(event) {
      this.loading = true;
    },
    changevolume() {
      this.$refs.video.volume = this.volume / 100; //改变音量
    },
    closevolume() {
      this.showvolume = false;
    },

    videoTime1(value) {
      var s = Number(value); //时间转换

      s = Math.floor(s);
      //console.log(s);
      var m = s / 60;
      m = Math.floor(m);
      //console.log(m);
      s = s % 60;
      if (s < 10) {
        s = String("0" + String(s));
      } else if (10 <= s && s < 60) {
        s = String(s);
      } else {
        s = String(s);
      }

      return m + ":" + s;
    },
    poptextCreater(type, channel, color, text) {
      //弹幕生成器
      if (type == "row") {
        console.log(text);
        var poptextspan = document.createElement("span");
        var content = document.createTextNode(text);
        poptextspan.style.fontSize = "20px";
        poptextspan.style.color = color;
        poptextspan.style.textAlign = "left";
        poptextspan.style.position = "absolute";
        poptextspan.style.top = String((this.channelmax - channel) * 22) + "px";
        poptextspan.style.textShadow = "-2px 2px #000";
        poptextspan.style.whiteSpace = "nowrap";
        poptextspan.classList.add("start");
        poptextspan.style.animation =
          "mov " + String(40 - text.length * 0.2) + "s";
        poptextspan.style.animationPlayState = "paused";
        poptextspan.appendChild(content);
        document.getElementById("poptextdiv").appendChild(poptextspan);
        this.channelrow--;
        var t1 = setInterval(() => {
          if (this.ifpause) {
            if (poptextspan.style.animationPlayState == "paused") {
              console.log("running");
              poptextspan.style.animationPlayState = "running";
            }
          } else {
            if (poptextspan.style.animationPlayState == "running") {
              poptextspan.style.animationPlayState = "paused";
            }
          }
        }, 100);
        poptextspan.addEventListener("animationend", () => {
          clearInterval(t1);
          document.getElementById("poptextdiv").removeChild(poptextspan);
        });
      } else {
        var poptextspan = document.createElement("span");
        var content = document.createTextNode(text);
        poptextspan.style.color = color;
        poptextspan.style.position = "absolute";
        poptextspan.style.fontSize = "20px";
        poptextspan.style.textAlign = "center";
        poptextspan.style.width = "100%";
        poptextspan.style.textShadow = "-2px 2px #000";
        poptextspan.style.top = String((this.channelmax - channel) * 22) + "px";
        poptextspan.appendChild(content);
        document.getElementById("poptextdiv").appendChild(poptextspan);
        this.channeltop--;
        var nowtime = this.$refs.video.currentTime + 5;
        //console.log(nowtime);
        var t1 = window.setInterval(() => {
          if (this.$refs.video.currentTime > nowtime) {
            document.getElementById("poptextdiv").removeChild(poptextspan);
            clearInterval(t1);
          }
        }, 100);
        //计时器，设定5s后删除此时间点的弹幕
      }
    },
    senderfocused() {
      if (this.initVideo.play) {
        this.play();
      } else {
        this.showvolume = false;
        this.showpoptextsetting = false;
      }
      this.showsendersetting = true;
    },
    pushpoptext() {
      //发送弹幕
      if (this.isrow) {
        var type = "row";
        var channel = this.channelrow;
      } else {
        var type = "top";
        var channel = this.channeltop;
      }
      var color = this.colorpicker;
      var text = this.sendertext;
      if (text != "") {
        this.sendertext = "";
        this.showsendersetting = false;
        var params = new URLSearchParams();
        var time = this.timenow;
        params.append("gid", this.$route.query.gid);
        params.append("index", this.$route.query.index);
        params.append("uid", this.$cookies.get("user")); //获得发送者id
        params.append("color", color);
        params.append("text", text);
        params.append("type", type);
        params.append("time", time); //你要传给后台的参数值 key/value
        this.$axios({
          method: "post",
          url: "", //要发送的地址
          data: params
        })
          .then(response => {
            if (type == "row") {
              console.log(text);
              var poptextspan1 = document.createElement("span");
              var content = document.createTextNode(text);
              poptextspan1.style.border = "2px solid #ff0000";
              poptextspan1.style.fontSize = "20px";
              poptextspan1.style.color = color;
              poptextspan1.style.textAlign = "left";
              poptextspan1.style.position = "absolute";
              poptextspan1.style.top =
                String((this.channelmax - channel) * 22) + "px";
              poptextspan1.style.textShadow = "-2px 2px #000";
              poptextspan1.style.whiteSpace = "nowrap";
              poptextspan1.classList.add("start");
              poptextspan1.style.animation =
                "mov " + String(40 - text.length * 0.2) + "s";
              poptextspan1.style.animationPlayState = "paused";
              poptextspan1.appendChild(content);
              document.getElementById("poptextdiv").appendChild(poptextspan1);
              this.channelrow--;
              var t2 = setInterval(() => {
                if (this.ifpause) {
                  if (poptextspan1.style.animationPlayState == "paused") {
                    //console.log("running");
                    poptextspan1.style.animationPlayState = "running";
                  }
                } else {
                  if (poptextspan1.style.animationPlayState == "running") {
                    poptextspan1.style.animationPlayState = "paused";
                  }
                }
              }, 100);
              poptextspan1.addEventListener("animationend", () => {
                clearInterval(t2);
                document.getElementById("poptextdiv").removeChild(poptextspan1);
              });
            } else {
              var poptextspan1 = document.createElement("span");
              var content = document.createTextNode(text);
              poptextspan1.style.color = color;
              poptextspan1.style.position = "absolute";
              poptextspan1.style.fontSize = "20px";
              poptextspan1.style.textAlign = "center";
              poptextspan1.style.width = "100%";
              poptextspan1.style.textShadow = "-2px 2px #000";
              poptextspan1.style.top =
                String((this.channelmax - channel) * 22) + "px";
              poptextspan1.appendChild(content);
              document.getElementById("poptextdiv").appendChild(poptextspan1);
              this.channelrtop--;
              var nowtime = this.$refs.video.currentTime + 5;
              //console.log(nowtime);
              var t2 = window.setInterval(() => {
                if (this.$refs.video.currentTime > nowtime) {
                  document
                    .getElementById("poptextdiv")
                    .removeChild(poptextspan1);
                  clearInterval(t2);
                }
              }, 100);
              //计时器，设定5s后删除此时间点的弹幕
            }
          })
          .catch(function(error) {
            console.log(error);
          });
      } else {
      }
    }
  },
  mounted() {
    this.fullscreen = setInterval(() => {
      //修改全屏按钮
      this.fullscreen =
        document.fullscreenElement ||
        document.msFullscreenElement ||
        document.mozFullScreenElement ||
        document.webkitFullscreenElement ||
        false;
    }, 500);
  },

  created() {},
  beforeCreate() {},
  watch: {
    timenow: function(time) {
      //弹幕池刷新
      this.channelrow = this.channelmax;
      this.channeltop = this.channelmax;
      for (var i = 0, l = this.poptexttest.length; i < l; i++) {
        if (this.poptexttest[i].time == time) {
          if (this.poptexttest[i].type == "top") {
            //判断弹幕类型
            if (this.channeltop <= 0) {
              //固定弹幕溢出判断
              if (this.allowoverwrite) {
                //判断是否允许溢出

                this.channeltop = this.channelmax;
                this.poptextCreater(
                  "top",
                  this.channeltop,
                  this.poptexttest[i].color,
                  this.poptexttest[i].text
                );
              }
            } else {
              //console.log(this.poptexttest[i].text);
              this.poptextCreater(
                "top",
                this.channeltop,
                this.poptexttest[i].color,
                this.poptexttest[i].text
              );
            }
          } else {
            if (this.channelrow <= 0) {
              if (this.allowoverwrite) {
                this.poptextCreater(
                  "row",
                  this.channelrow,
                  this.poptexttest[i].color,
                  this.poptexttest[i].text
                );
                this.channelrow = this.channelmax;
              }
            } else {
              this.poptextCreater(
                "row",
                this.channelrow,
                this.poptexttest[i].color,
                this.poptexttest[i].text
              );
            }
          }
        }
      }
    },
    poptextop: function() {
      document.getElementById("poptextdiv").style.opacity = //自动隐藏控制条
        parseFloat(100 - this.poptextop) * 0.01;
    },
    colorpicker: function() {
      document.getElementById("textpre").style.color = this.colorpicker; //修改弹幕颜色
    }
  },
  filters: {
    videoTime: function(value) {
      var s = Number(value); //时间转换

      s = Math.floor(s);
      //console.log(s);
      var m = s / 60;
      m = Math.floor(m);
      //console.log(m);
      s = s % 60;
      if (s < 10) {
        s = String("0" + String(s));
      } else if (10 <= s && s < 60) {
        s = String(s);
      } else {
        s = String(s);
      }

      return m + ":" + s;
    }
  }
};
</script>

<style>
#video {
  position: relative;
}
#control {
  opacity: 0;
  position: absolute;
  z-index: 6;
  width: 100%;
  height: 50px;
  background: linear-gradient(rgba(0, 0, 0, 0), #000);
  bottom: 0;
  transition: opacity 1s;
  -moz-transition: opacity 1s; /* Firefox 4 */
  -webkit-transition: opacity 1s; /* Safari 和 Chrome */
  -o-transition: opacity 1s; /* Opera */
}
.timeline {
  position: absolute;
  width: 100%;
  top: -20px;
}
#volume {
  z-index: 6;
  position: absolute;
  right: 65px;
  bottom: 50px;
  background: rgba(0, 0, 0, 0.5);
}
#loading {
  z-index: 6;
  position: absolute;
  width: 100%;
  height: 100%;
}
#ifpause {
  z-index: 6;
  position: absolute;
  width: 100%;
  height: 100%;
}
#poptextdiv {
  position: absolute;
  overflow: hidden;
  width: 100%;
  height: 100%;
}
#control:hover {
  opacity: 1;
}
#poptextsetting {
  z-index: 6;
  position: absolute;
  right: 50px;
  bottom: 50px;
  background: rgba(0, 0, 0, 0.5);
}
#sendersetting {
  z-index: 6;
  position: absolute;
  right: 150px;
  bottom: 50px;
  background: rgba(0, 0, 0, 0.5);
}
#textpre {
  width: 100%;
  margin: auto;
  text-align: center;
}
.start {
  animation-timing-function: linear;
  animation: mov 20s;
  -moz-animation: mov 20s; /* Firefox 4 */
  -webkit-animation: mov 20s; /* Safari 和 Chrome */
  -o-animation: mov 20s; /* Opera */
}
@keyframes mov {
  from {
    right: -200px;
  }
  to {
    right: 3000px;
  }
}

@-moz-keyframes mov /* Firefox */ {
  from {
    right: -200px;
  }
  to {
    right: 3000px;
  }
}

@-webkit-keyframes mov /* Safari 和 Chrome */ {
  from {
    right: -200px;
  }
  to {
    right: 3000px;
  }
}
@-o-keyframes mov /* Opera */ {
  from {
    right: -200px;
  }
  to {
    right: 3000px;
  }
}
</style>
