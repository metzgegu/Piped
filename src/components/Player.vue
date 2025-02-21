<template>
    <div class="uk-container-expand">
        <div
            ref="container"
            data-shaka-player-container
            style="width: 100%; height: 100%; background: #000"
            :style="!isEmbed ? { 'max-height': '75vh', 'min-height': '250px' } : {}"
        >
            <video
                ref="videoEl"
                data-shaka-player
                class="uk-width-expand"
                :autoplay="shouldAutoPlay"
                :loop="selectedAutoLoop"
            ></video>
        </div>
    </div>
</template>

<script>
import("shaka-player/dist/controls.css");
const shaka = import("shaka-player/dist/shaka-player.ui.js");
import muxjs from "mux.js";
window.muxjs = muxjs;

export default {
    props: {
        video: {
            type: Object,
            default: () => {
                return {};
            },
        },
        sponsors: {
            type: Object,
            default: () => {
                return {};
            },
        },
        selectedAutoPlay: Boolean,
        selectedAutoLoop: Boolean,
        isEmbed: Boolean,
    },
    data() {
        return {
            $player: null,
            $ui: null,
            lastUpdate: new Date().getTime(),
        };
    },
    computed: {
        shouldAutoPlay: _this => {
            return _this.getPreferenceBoolean("playerAutoPlay", true) && !_this.isEmbed;
        },
        preferredVideoCodecs: _this => {
            var preferredVideoCodecs = [];
            const enabledCodecs = _this.getPreferenceString("enabledCodecs", "av1,vp9,avc").split(",");

            if (
                _this.$refs.videoEl.canPlayType('video/mp4; codecs="av01.0.08M.08"') !== "" &&
                enabledCodecs.includes("av1")
            )
                preferredVideoCodecs.push("av01");
            if (_this.$refs.videoEl.canPlayType('video/webm; codecs="vp9"') !== "" && enabledCodecs.includes("vp9"))
                preferredVideoCodecs.push("vp9");
            if (
                _this.$refs.videoEl.canPlayType('video/mp4; codecs="avc1.4d401f"') !== "" &&
                enabledCodecs.includes("avc")
            )
                preferredVideoCodecs.push("avc1");

            return preferredVideoCodecs;
        },
    },
    mounted() {
        if (!this.shaka) this.shakaPromise = shaka.then(shaka => shaka.default).then(shaka => (this.shaka = shaka));
    },
    activated() {
        import("hotkeys-js")
            .then(mod => mod.default)
            .then(hotkeys => {
                this.hotkeys = hotkeys;
                var self = this;
                hotkeys("f,m,j,k,l,c,space,up,down,left,right", function(e, handler) {
                    const videoEl = self.$refs.videoEl;
                    switch (handler.key) {
                        case "f":
                            self.$ui.getControls().toggleFullScreen();
                            e.preventDefault();
                            break;
                        case "m":
                            videoEl.muted = !videoEl.muted;
                            e.preventDefault();
                            break;
                        case "j":
                            videoEl.currentTime = Math.max(videoEl.currentTime - 15, 0);
                            e.preventDefault();
                            break;
                        case "l":
                            videoEl.currentTime = videoEl.currentTime + 15;
                            e.preventDefault();
                            break;
                        case "c":
                            self.$player.setTextTrackVisibility(!self.$player.isTextTrackVisible());
                            e.preventDefault();
                            break;
                        case "k":
                        case "space":
                            if (videoEl.paused) videoEl.play();
                            else videoEl.pause();
                            e.preventDefault();
                            break;
                        case "up":
                            videoEl.volume = Math.min(videoEl.volume + 0.05, 1);
                            e.preventDefault();
                            break;
                        case "down":
                            videoEl.volume = Math.max(videoEl.volume - 0.05, 0);
                            e.preventDefault();
                            break;
                        case "left":
                            videoEl.currentTime = Math.max(videoEl.currentTime - 5, 0);
                            e.preventDefault();
                            break;
                        case "right":
                            videoEl.currentTime = videoEl.currentTime + 5;
                            e.preventDefault();
                            break;
                    }
                });
            });
    },
    deactivated() {
        this.destroy();
    },
    unmounted() {
        this.destroy();
    },
    methods: {
        async loadVideo() {
            const component = this;
            const videoEl = this.$refs.videoEl;

            videoEl.setAttribute("poster", this.video.thumbnailUrl);

            if (this.$route.query.t) {
                videoEl.currentTime = this.$route.query.t;
            } else if (window.db) {
                var tx = window.db.transaction("watch_history", "readonly");
                var store = tx.objectStore("watch_history");
                var request = store.get(this.video.id);
                request.onsuccess = function(event) {
                    var video = event.target.result;
                    if (video && video.currentTime) {
                        videoEl.currentTime = video.currentTime;
                    }
                };
            }

            const noPrevPlayer = !this.$player;

            var streams = [];

            streams.push(...this.video.audioStreams);
            streams.push(...this.video.videoStreams);

            const MseSupport = window.MediaSource !== undefined;

            const lbry = this.getPreferenceBoolean("disableLBRY", false)
                ? null
                : this.video.videoStreams.filter(stream => stream.quality === "LBRY")[0];

            var uri;
            var mime;

            if (this.video.livestream) {
                uri = this.video.hls;
                mime = "application/x-mpegURL";
            } else if (this.video.audioStreams.length > 0 && !lbry && MseSupport) {
                if (!this.video.dash) {
                    const dash = require("@/utils/DashUtils.js").default.generate_dash_file_from_formats(
                        streams,
                        this.video.duration,
                    );

                    uri = "data:application/dash+xml;charset=utf-8;base64," + btoa(dash);
                } else uri = this.video.dash;
                mime = "application/dash+xml";
            } else if (lbry) {
                uri = lbry.url;
                if (this.getPreferenceBoolean("proxyLBRY", false)) {
                    const url = new URL(uri);
                    url.searchParams.set("host", url.host);
                    url.host = new URL(this.video.proxyUrl).host;
                    uri = url.toString();
                }
                const contentType = await fetch(uri, {
                    method: "HEAD",
                }).then(response => response.headers.get("Content-Type"));
                mime = contentType;
            } else if (this.video.hls) {
                uri = this.video.hls;
                mime = "application/x-mpegURL";
            } else {
                uri = this.video.videoStreams.filter(stream => stream.codec == null).slice(-1)[0].url;
                mime = "video/mp4";
            }

            if (noPrevPlayer)
                this.shakaPromise.then(() => {
                    this.shaka.polyfill.installAll();

                    const localPlayer = new this.shaka.Player(videoEl);
                    const proxyHost = new URL(component.video.proxyUrl).host;

                    localPlayer.getNetworkingEngine().registerRequestFilter((_type, request) => {
                        const uri = request.uris[0];
                        var url = new URL(uri);
                        const headers = request.headers;
                        if (
                            url.host.endsWith(".googlevideo.com") ||
                            (url.host.endsWith(".lbryplayer.xyz") &&
                                (component.getPreferenceBoolean("proxyLBRY", false) || headers.Range))
                        ) {
                            url.searchParams.set("host", url.host);
                            url.host = proxyHost;
                            request.uris[0] = url.toString();
                        }
                        if (url.pathname === "/videoplayback") {
                            if (headers.Range) {
                                url.searchParams.set("range", headers.Range.split("=")[1]);
                                request.headers = {};
                                request.uris[0] = url.toString();
                            }
                        }
                    });

                    localPlayer.configure(
                        "streaming.bufferingGoal",
                        Math.max(this.getPreferenceNumber("bufferGoal", 10), 10),
                    );

                    this.setPlayerAttrs(localPlayer, videoEl, uri, mime, this.shaka);
                });
            else this.setPlayerAttrs(this.$player, videoEl, uri, mime, this.shaka);

            if (noPrevPlayer) {
                videoEl.addEventListener("timeupdate", () => {
                    const time = videoEl.currentTime;
                    this.updateProgressDatabase(time);
                    if (this.sponsors && this.sponsors.segments) {
                        this.sponsors.segments.map(segment => {
                            if (!segment.skipped || this.selectedAutoLoop) {
                                const end = segment.segment[1];
                                if (time >= segment.segment[0] && time < end) {
                                    console.log("Skipped segment at " + time);
                                    videoEl.currentTime = end;
                                    segment.skipped = true;
                                    return;
                                }
                            }
                        });
                    }
                });

                videoEl.addEventListener("volumechange", () => {
                    this.setPreference("volume", videoEl.volume);
                });

                videoEl.addEventListener("ratechange", () => {
                    this.setPreference("rate", videoEl.playbackRate);
                });

                videoEl.addEventListener("ended", () => {
                    if (!this.selectedAutoLoop && this.selectedAutoPlay && this.video.relatedStreams.length > 0) {
                        const params = this.$route.query;
                        let url = this.video.relatedStreams[0].url;
                        const searchParams = new URLSearchParams();
                        for (var param in params)
                            switch (param) {
                                case "v":
                                case "t":
                                    break;
                                default:
                                    searchParams.set(param, params[param]);
                                    break;
                            }
                        const paramStr = searchParams.toString();
                        if (paramStr.length > 0) url += "&" + paramStr;
                        this.$router.push(url);
                    }
                });
            }

            //TODO: Add sponsors on seekbar: https://github.com/ajayyy/SponsorBlock/blob/e39de9fd852adb9196e0358ed827ad38d9933e29/src/js-components/previewBar.ts#L12
        },
        setPlayerAttrs(localPlayer, videoEl, uri, mime, shaka) {
            const url = "/watch?v=" + this.video.id;

            if (!this.$ui) {
                const OpenButton = class extends shaka.ui.Element {
                    constructor(parent, controls) {
                        super(parent, controls);

                        this.newTabButton_ = document.createElement("button");
                        this.newTabButton_.classList.add("shaka-cast-button");
                        this.newTabButton_.classList.add("shaka-tooltip");
                        this.newTabButton_.ariaPressed = "false";

                        this.newTabIcon_ = document.createElement("i");
                        this.newTabIcon_.classList.add("material-icons-round");
                        this.newTabIcon_.textContent = "launch";
                        this.newTabButton_.appendChild(this.newTabIcon_);

                        const label = document.createElement("label");
                        label.classList.add("shaka-overflow-button-label");
                        label.classList.add("shaka-overflow-menu-only");
                        this.newTabNameSpan_ = document.createElement("span");
                        this.newTabNameSpan_.innerText = "Open in new tab";
                        label.appendChild(this.newTabNameSpan_);

                        this.newTabButton_.appendChild(label);
                        this.parent.appendChild(this.newTabButton_);

                        this.eventManager.listen(this.newTabButton_, "click", () => {
                            this.video.pause();
                            window.open(url);
                        });
                    }
                };

                OpenButton.Factory = class {
                    create(rootElement, controls) {
                        return new OpenButton(rootElement, controls);
                    }
                };

                shaka.ui.OverflowMenu.registerElement("open_new_tab", new OpenButton.Factory());

                this.$ui = new shaka.ui.Overlay(localPlayer, this.$refs.container, videoEl);

                const overflowMenuButtons = ["quality", "captions", "picture_in_picture", "playback_rate", "airplay"];

                if (this.isEmbed) {
                    overflowMenuButtons.push("open_new_tab");
                }

                const config = {
                    overflowMenuButtons: overflowMenuButtons,
                    seekBarColors: {
                        base: "rgba(255, 255, 255, 0.3)",
                        buffered: "rgba(255, 255, 255, 0.54)",
                        played: "rgb(255, 0, 0)",
                    },
                };

                this.$ui.configure(config);
            }

            const player = this.$ui.getControls().getPlayer();

            this.$player = player;

            const disableVideo = this.getPreferenceBoolean("listen", false) && !this.video.livestream;

            this.$player.configure({
                preferredVideoCodecs: this.preferredVideoCodecs,
                preferredAudioCodecs: ["opus", "mp4a"],
                manifest: {
                    disableVideo: disableVideo,
                    hls: {
                        useFullSegmentsForStartTime: true,
                    },
                },
            });

            const quality = this.getPreferenceNumber("quality", 0);
            const qualityConds =
                quality > 0 && (this.video.audioStreams.length > 0 || this.video.livestream) && !disableVideo;
            if (qualityConds) this.$player.configure("abr.enabled", false);

            player.load(uri, 0, mime).then(() => {
                if (qualityConds) {
                    var leastDiff = Number.MAX_VALUE;
                    var bestStream = null;
                    player
                        .getVariantTracks()
                        .sort((a, b) => a.bandwidth - b.bandwidth)
                        .forEach(stream => {
                            const diff = Math.abs(quality - stream.height);
                            if (diff < leastDiff) {
                                leastDiff = diff;
                                bestStream = stream;
                            }
                        });
                    player.selectVariantTrack(bestStream);
                }

                this.video.subtitles.map(subtitle => {
                    player.addTextTrackAsync(
                        subtitle.url,
                        subtitle.code,
                        "SUBTITLE",
                        subtitle.mimeType,
                        null,
                        subtitle.name,
                    );
                });
                videoEl.volume = this.getPreferenceNumber("volume", 1);
                player.trickPlay(this.getPreferenceNumber("rate", 1));
            });
        },
        async updateProgressDatabase(time) {
            // debounce
            if (new Date().getTime() - this.lastUpdate < 500) return;
            this.lastUpdate = new Date().getTime();

            if (!this.video.id || !window.db) return;

            var tx = window.db.transaction("watch_history", "readwrite");
            var store = tx.objectStore("watch_history");
            var request = store.get(this.video.id);
            request.onsuccess = function(event) {
                var video = event.target.result;
                if (video) {
                    video.currentTime = time;
                    store.put(video);
                }
            };
        },
        destroy() {
            if (this.$ui) {
                this.$ui.destroy();
                this.$ui = undefined;
                this.$player = undefined;
            }
            if (this.$player) {
                this.$player.destroy();
                this.$player = undefined;
            }
            if (this.hotkeys) this.hotkeys.unbind();
            if (this.$refs.container) this.$refs.container.querySelectorAll("div").forEach(node => node.remove());
        },
    },
};
</script>

<style>
.shaka-text-container > div {
    height: auto !important;
    width: auto !important;
    top: auto !important;
    left: auto !important;
}

.shaka-text-container * {
    background-color: rgba(8, 8, 8, 0.75) !important;
    color: white !important;
}

.shaka-video-container:-webkit-full-screen {
    max-height: none !important;
}
</style>
