<template>
  <div id="home">
    <b-container>
      <b-row>
        <b-col sm="12" md="6">
          <b-carousel
                  :interval="6000"
                  controls
                  indicators
          >
            <b-carousel-slide v-for="banner in banners"
                    :caption="banner.title"
                    :text="banner.text"
                    :img-src="banner.img"
                              :key="banner.title"
            ></b-carousel-slide>
          </b-carousel>
        </b-col>
        <b-col sm="12" md="6">
          <b-row>
            <b-col sm="6" md="6" lg="4" v-for="video in bannervideolist" :key="video.url">
              <b-card :img-src="video.cover">
                <b-card-text>
                  <a :href="video.url" target="_blank"><h4>{{ video.title }}</h4></a>
                </b-card-text>
              </b-card>
            </b-col>
          </b-row>
        </b-col>
      </b-row>
    </b-container>
    <b-container>
      <b-row>
        <h2>正在广播</h2>
      </b-row>
      <b-row>
        <b-col sm="6" md="4" lg="3" xl="2" v-for="video in broadcastlist" :key="video.url">
          <b-card :img-src="video.cover">
            <b-card-text>
              <a :href="video.url"><h4>{{ video.title }}</h4></a>
            </b-card-text>
          </b-card>
        </b-col>
      </b-row>
    </b-container>
    <b-container v-for="type in typelist" :key="type.name">
      <b-row>
        <h2>{{ type.title }}</h2>
      </b-row>
      <b-row>
        <b-col sm="6" md="4" lg="3" xl="2" v-for="video in vlist[type.name]" :key="video.url">
          <b-card :img-src="video.cover">
            <b-card-text>
              <a :href="video.url"><h4>{{ video.title }}</h4></a>
            </b-card-text>
          </b-card>
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>
<script>
  import Axios from 'axios'
  export default {
    name: 'home',
    data(){
      return {
        typelist:[],
        banners:[],
        bannervideolist:[],
        vlist:{},
        jsipfs:{},
        whitelist:[],
        broadcast:[],
        broadcastlist:[],
      }
    },
    methods:{
      async init(){
        document.title = 'VideoShare';
        await Axios.get('./type.json').then(async (res)=>{
          this.typelist = res.data.type;
          for (let i=0;i<this.typelist.length;i++){
            await Axios.get('./type_'+this.typelist[i].name+'.json').then(async (res)=>{
              this.vlist[this.typelist[i].name] = await this.find_video(res.data.list, 12);
            })
          }
        });
        await Axios.get('./banner.json').then(async (res)=>{
          this.banners = res.data.banners;
          this.bannervideolist = await this.find_video(res.data.bannervideolist, 3);
        });

        await Axios.get('./whitelist.json').then(async (res)=>{
          this.whitelist = res.data.whitelist;
        });
        const ipfs = await this.$ipfs;
        ipfs.swarm.connect("/ip4/127.0.0.1/tcp/9999/ws/ipfs/QmPKtUgdw97QS7zYoEVxY9EpuCavbtMmjSMp7usDXt1BGi");

        setInterval(async()=>{
          const peerInfos = await ipfs.swarm.peers();
            console.log(peerInfos.length+' nodes connect.');
            console.log(peerInfos);
        },10000);
        setInterval(()=>{
          this.broadcastlist = [];
          for (let i = 0; i < this.broadcast.length && this.broadcastlist.length < 6; i++) {
            this.broadcastlist.push(this.broadcast[this.broadcast.length - i - 1]);
          }
        },30000);

        const topic = 'VideoShare';
        const receiveMsg = async(msg) => {
          const from = msg.from;
          let distrust = true;
          for (let i = 0; i < this.whitelist.length; i++) {
            if(this.whitelist[i].id===from){
              distrust = false;
              break;
            }
          }
          if(distrust)return;
          let videohash = msg.data.toString();
          let video = await Axios.get('/ipfs/'+videohash+'/files.json').then(async (res)=>{
            return res.data;
          });
          this.broadcast.push({
            "title":video.title,
            "cover":video.cover,
            "url":'/ipfs/'+videohash,
          });
        };
        await ipfs.pubsub.subscribe(topic, receiveMsg)
      },
      async find_video(typelist, num){
        let res = [];
        for (let i  = 0; i<typelist.length;i++){
          let temp = await Axios.get('/ipns/'+typelist[i].id+'/'+typelist[i].usertype+'.json').then((res)=>{
            return res.data;
          }).catch((err)=>{
              return [];
          });
          for (let j = 0; j < temp.length; j++) {
            res.push(temp[j])
          }
        }
        let video = [];
        if(res.length<=num){
          video = res
        }else{
            for(let i=0;i<num;i++){
                let len = res.length-1;
                let number = Math.floor(Math.random()*len);
                video.push(res[number]);
                res.splice(number,1);
            }
        }
        return video;
      }
    },
    created() {
      this.init()
    },
  }
</script>
<style>
  .row >.col-sm-6, .row >.col-md-4, .row >.col-lg-3, .row >.col-xl-2 {
    padding-left: 1px;
    padding-right: 1px;
    padding-bottom: 15px;
  }
  .card > .card-body{
    padding: 0.1rem;
  }
  a > h4{
    font-size: 0.8rem;
    overflow: hidden;
    text-overflow:ellipsis;
    white-space: nowrap;
  }
</style>
