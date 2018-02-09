<template>
    <div class="music">
        <div class="music-content">
            <div class="music-left">
                <div class="music-btn">
                    <router-link to="/music/playlist" tag="span">正在播放</router-link>
                    <router-link to="/music/toplist" tag="span">排行榜</router-link>
                    <!--<router-link to="/music/search" tag="span">搜索</router-link>-->
                    <!--<router-link to="/music/sheetlist" tag="span">歌单</router-link>-->
                    <router-link to="/music/historyList" tag="span">我听过的</router-link>
                </div>
                <!--<transition name="">-->
                <keep-alive>
                    <router-view v-if="$route.meta.keepAlive" class="music-list"></router-view>
                </keep-alive>
                <router-view v-if="!$route.meta.keepAlive" class="music-list"></router-view>
                <!--</transition>-->
            </div>
            <lyric class="music-right" :lyric="lyric" :lyricIndex="lyricIndex"></lyric>
        </div>
        
        <div class="music-bar" :class="{disable:!musicReady||!currentMusic.id}">
            <a class="music-bar-btn btn-prev" @click="prev"></a>
            <a class="music-bar-btn btn-play" :class="{'btn-play-pause':playing}" @click="play"></a>
            <a class="music-bar-btn btn-next" @click="next"></a>
            <div class="music-music">
                <template v-if="percentMusic">
                    <div class="music-bar-info">{{currentMusic.name}}<span> - {{currentMusic.ar[0].name}}</span></div>
                    <div class="music-bar-time">{{currentTime | format}}/{{duration | format}}</div>
                    <mm-progress class="music-progress" :percent="percentMusic"
                                 @percentChange="progressMusic"></mm-progress>
                </template>
                <template v-else>
                    <mm-progress class="music-progress" :percent="0"></mm-progress>
                </template>
            </div>
            <div class="music-bar-volume">
                <a class="music-bar-btn btn-volume" :class="{'btn-volume-no':isMute}" @click="switchMute"></a>
                <mm-progress @percentChange="volumeChange"
                             :percent="volume"></mm-progress>
            </div>
        </div>
        
        <!--遮罩-->
        <div class="mmPlayer-bg" :style="picUrl"></div>
        <div class="mmPlayer-mask"></div>
    </div>
</template>

<script>
    import {getLyric} from 'api/music'
    import mmPlayerMusic from './mmPlayer'
    import {mapGetters, mapMutations, mapActions} from 'vuex'
    import MmProgress from 'base/mm-progress/mm-progress'
    import Lyric from 'components/lyric/lyric'
    
    export default {
        name: "music",
        components: {
            MmProgress,
            Lyric
        },
        data() {
            return {
                musicReady: false,
                currentTime: 0,
                duration: 0,
                lyric: null,
                lyricIndex: 0,
                isMute: false,
                volume: .5,
            }
        },
        mounted() {
            this.$nextTick(() => {
                mmPlayerMusic.initAudio(this)
            })
        },
        computed: {
            picUrl() {
                return this.currentMusic.id ? 'background-image:url(' + this.currentMusic.al.picUrl + ')' : ''
            },
            percentMusic() {
                return this.currentTime && this.duration ? this.currentTime / this.duration : 0
            },
            ...mapGetters([
                'audioEle',
                'playing',
                'playlist',
                'currentIndex',
                'currentMusic',
            ])
        },
        watch: {
            playlist(newPlaylist) {
                if (newPlaylist.length === 0) {
                    this.lyric = null;
                    this.lyricIndex = 0
                }
            },
            currentMusic(newMusic, oldMusic) {
                if (newMusic.id === oldMusic.id) {
                    return
                }
                this.$nextTick(() => {
                    this.audioEle.play();
                    this._getLyric(newMusic.id)
                })
            },
            playing(newPlaying) {
                const audio = this.audioEle;
                this.$nextTick(() => {
                    newPlaying ? audio.play() : audio.pause();
                    this.musicReady = true
                })
            },
            currentTime(newTime) {
                let lyricIndex = 0;
                for (let i = 0; i < this.lyric.length; i++) {
                    if (newTime > this.lyric[i].time) {
                        lyricIndex = i;
                    }
                }
                this.lyricIndex = lyricIndex;
            },
        },
        methods: {
            //上一曲
            prev() {
                if (!this.musicReady) {
                    return
                }
                let index = this.currentIndex - 1;
                if (index < 0) {
                    index = this.playlist.length - 1
                }
                this.setCurrentIndex(index);
                this.musicReady = false
            },
            //播放暂停
            play() {
                if (!this.musicReady) {
                    return
                }
                this.setPlaying(!this.playing)
            },
            //下一曲
            next() {
                if (!this.musicReady) {
                    return
                }
                let index = this.currentIndex + 1;
                if (index === this.playlist.length) {
                    index = 0
                }
                this.setCurrentIndex(index);
                this.musicReady = false
            },
            //修改音乐进度
            progressMusic(percent) {
                this.audioEle.currentTime = Math.round(this.duration * percent)
            },
            //修改音量大小
            volumeChange(percent) {
                percent === 0 ? this.isMute = true : this.isMute = false;
                this.volume = percent;
                this.audioEle.volume = percent;
            },
            //是否静音
            switchMute() {
                const audio = this.audioEle;
                this.isMute = !this.isMute;
                this.isMute ? audio.volume = 0 : audio.volume = this.volume
            },
            //获取歌词
            _getLyric(id) {
                getLyric(id).then((res) => {
                    if (res.status === 200) {
                        this.lyric = parseLyric(res.data.lrc.lyric)
                    }
                    //console.log(parseLyric(res.data.lrc.lyric))
                })
            },
            ...mapMutations({
                setPlaying: 'SET_PLAYING',
                setCurrentIndex: 'SET_CURRENTINDEX'
            }),
            ...mapActions([
                'setHistoryList'
            ])
        },
        filters: {
            //时间格式化
            format(value) {
                let minute = Math.floor(value / 60);
                let second = Math.floor(value % 60);
                return `${minute < 10 ? '0' + minute : minute}:${second < 10 ? '0' + second : second}`
            },
        }
    }
    
    //歌词解析
    function parseLyric(lrc) {
        let lyrics = lrc.split("\n");
        let lrcObj = [];
        for (let i = 0; i < lyrics.length; i++) {
            let lyric = decodeURIComponent(lyrics[i]);
            let timeReg = /\[\d*:\d*((\.|\:)\d*)*\]/g;
            let timeRegExpArr = lyric.match(timeReg);
            if (!timeRegExpArr) continue;
            let clause = lyric.replace(timeReg, '');
            for (let k = 0, h = timeRegExpArr.length; k < h; k++) {
                let t = timeRegExpArr[k];
                let min = Number(String(t.match(/\[\d*/i)).slice(1)),
                    sec = Number(String(t.match(/\:\d*/i)).slice(1));
                let time = min * 60 + sec;
                if (clause !== '') {
                    lrcObj.push({time: time, text: clause})
                }
            }
        }
        return lrcObj;
    }
</script>

<style lang="less">
    @import "../../assets/css/var";
    
    .music {
        padding: 50px 25px 25px 25px;
        width: 100%;
        max-width: 1750px;
        margin: 0 auto;
        height: 100%;
        box-sizing: border-box;
        overflow: hidden;
        .music-content {
            display: flex;
            width: 100%;
            height: calc(~'100% - 80px');
            .music-left {
                flex: 1;
                overflow: hidden;
                .music-btn {
                    width: 100%;
                    height: 60px;
                    span {
                        display: inline-block;
                        height: 40px;
                        box-sizing: border-box;
                        margin-right: 6px;
                        padding: 0 23px;
                        border: 1px solid @btn_color;
                        color: @btn_color;
                        border-radius: 2px;
                        font-size: 14px;
                        line-height: 40px;
                        overflow: hidden;
                        cursor: pointer;
                        &:hover, &.active {
                            border-color: @btn_color_active;
                            color: @btn_color_active;
                        }
                    }
                }
                .music-list {
                    height: calc(~'100% - 60px');
                }
            }
            .music-right {
                position: relative;
                width: 310px;
                margin-left: 10px;
            }
        }
        
        /*底部mmPlayer-bar*/
        .music-bar {
            display: flex;
            align-items: center;
            width: 100%;
            height: 80px;
            box-sizing: border-box;
            padding-bottom: 15px;
            &.disable {
                pointer-events: none;
                .btn-prev, .btn-play, .btn-next {
                    opacity: .6;
                }
            }
            .music-bar-btn {
                display: block;
                background-image: url("../../assets/img/player.png");
                cursor: pointer;
            }
            .btn-prev {
                width: 19px;
                min-width: 19px;
                height: 20px;
                background-position: 0 -30px;
            }
            .btn-play {
                left: 76px;
                width: 21px;
                min-width: 21px;
                height: 29px;
                margin: 0 50px;
                background-position: 0 0;
                &.btn-play-pause {
                    background-position: -30px 0;
                }
            }
            .btn-next {
                left: 148px;
                width: 19px;
                min-width: 19px;
                height: 20px;
                background-position: 0 -52px;
            }
            .music-music {
                position: relative;
                flex: 1;
                box-sizing: border-box;
                padding-left: 50px;
                font-size: @font_size_small;
                .music-bar-info {
                    padding: 0 70px 5px 0;
                    overflow: hidden;
                    white-space: nowrap;
                    text-overflow: ellipsis;
                }
                .music-bar-time {
                    position: absolute;
                    top: 0;
                    right: 0;
                }
            }
            .music-bar-volume {
                position: relative;
                width: 100px;
                padding-left: 50px;
                .btn-volume {
                    position: absolute;
                    top: -4px;
                    left: 20px;
                    width: 26px;
                    height: 21px;
                    background-position: 0 -144px;
                    &.btn-volume-no {
                        background-position: 0 -182px;
                    }
                }
            }
        }
        
        /*遮罩*/
        .mmPlayer-mask, .mmPlayer-bg {
            position: absolute;
            top: 0;
            right: 0;
            left: 0;
            bottom: 0;
        }
        
        .mmPlayer-mask {
            z-index: -1;
            background-color: @mask_color;
        }
        
        .mmPlayer-bg {
            z-index: -2;
            background-image: url("http://img.mtnhao.com/bg.jpg");
            background-repeat: no-repeat;
            background-size: cover;
            background-position: 50%;
            filter: blur(10px);
            opacity: .7;
            transform: translateZ(0);
            transition: all .8s;
        }
        
        //当屏幕小于960时
        @media (max-width: 960px) {
            .music-right {
                display: none;
            }
        }
        //当屏幕小于768时
        @media (max-width: 768px) {
            & {
                padding: 50px 15px 15px 15px
            }
            
            .music-content .music-left {
                .music-btn {
                    height: 50px;
                    span {
                        height: 35px;
                        padding: 0 15px;
                        line-height: 35px;
                    }
                }
                .music-list {
                    height: calc(~'100% - 40px');
                    font-size: 14px;
                }
            }
            
            .music-content {
                height: calc(~'100% - 60px');
            }
            
            .music-bar {
                height: 60px;
                padding-bottom: 0;
                .music-bar-volume {
                    display: none;
                }
            }
        }
        //当屏幕小于520时
        @media (max-width: 520px) {
            .music-bar {
                .btn-play {
                    margin: 0 25px;
                }
                .music-music {
                    padding-left: 25px;
                    .music-bar-info span {
                        display: none;
                    }
                }
            }
        }
    }
</style>