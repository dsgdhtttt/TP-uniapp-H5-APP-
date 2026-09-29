# TP框架+uniapp多商户商城小程序+H5+APP+商家入驻短视频社区种草直播阶梯拼团运营版

源码获取：ym7k.com/18951/TP框架+uniapp多商户商城小程序+H5+APP+商家入驻短视频社区种草直播阶梯拼团运营版

![Uploading 2025122710090591.png…]()
TP框架 + UniApp 多商户商城：全终端直播与短视频社区的工程架构
引子：直播不是功能，而是多商户架构的压力测试
“多商户商城 + 短视频社区 + 种草 + 直播 + 阶梯拼团 + 商家入驻”——当这些功能叠加在一个系统上时，它考验的已经不是“能不能做”，而是多租户架构能否在实时业务的高压下保持隔离与稳定。

直播是其中最严苛的一环。它要求系统同时处理：多商户的独立推流、全终端的播放适配、直播间的实时互动、直播商品的订单归因、直播内容的合规审核。每一个环节都必须在多租户的框架下正确运行——商户 A 的直播不能出现在商户 B 的商城里，商户 A 的订单不能算到商户 B 的账上，商户 A 的违规内容不能影响商户 B 的正常运营。

“突破限制全终端观看直播，H5 外所有终端直播推流”这句话，背后是一整套流媒体协议适配和终端能力降级的工程方案。

本文从多租户直播的架构设计、全终端播放的协议适配、App 与小程序推流的技术实现、直播与商城的业务耦合，到短视频社区与直播的协同，展开这类系统的完整技术分析。

一、多商户直播的架构分层
1.1 三层结构
多商户直播系统的架构可以抽象为三层。

流媒体层。 负责音视频流的接收（推流）、转码、分发（拉流）。核心组件是 RTMP 服务器、转码服务和 CDN。

业务层。 负责直播间的创建、状态管理、商品关联、互动消息、订单归因。这是商城系统与流媒体层的交界面。

终端层。 负责推流和播放。不同终端使用不同的 SDK 和协议。

三层之间的数据流是：主播通过终端推流 → 流媒体层接收并转码 → 业务层记录直播间状态 → 观众通过终端拉流 → 业务层处理互动和订单。

1.2 多租户隔离
多商户系统要求每个商户的直播间完全隔离。隔离维度包括：推流地址隔离、播放地址隔离、商品隔离、订单隔离。

sql
CREATE TABLE `live_room` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `merchant_id` int(11) NOT NULL COMMENT '所属商户',
  `title` varchar(200) NOT NULL,
  `cover` varchar(255) NOT NULL,
  `stream_key` varchar(64) NOT NULL COMMENT '推流密钥',
  `push_url` varchar(500) DEFAULT NULL COMMENT '推流地址',
  `play_url_flv` varchar(500) DEFAULT NULL COMMENT 'HTTP-FLV 播放地址',
  `play_url_hls` varchar(500) DEFAULT NULL COMMENT 'HLS 播放地址',
  `play_url_rtmp` varchar(500) DEFAULT NULL COMMENT 'RTMP 播放地址',
  `status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '0=预告 1=直播中 2=已结束',
  `scheduled_at` datetime DEFAULT NULL,
  `started_at` datetime DEFAULT NULL,
  `ended_at` datetime DEFAULT NULL,
  `viewer_count` int(11) NOT NULL DEFAULT '0',
  `peak_viewer` int(11) NOT NULL DEFAULT '0',
  `like_count` int(11) NOT NULL DEFAULT '0',
  `replay_url` varchar(500) DEFAULT NULL COMMENT '回放地址',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_stream_key` (`stream_key`),
  KEY `idx_merchant_status` (`merchant_id`, `status`),
  KEY `idx_status_scheduled` (`status`, `scheduled_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
stream_key 是推流的核心凭证。它是一个唯一字符串，主播推流时提供这个密钥，服务器据此识别是哪个直播间。merchant_id 保证了每个直播间归属于特定的商户。

1.3 流媒体服务器的选型
对于多商户商城的私有化部署，流媒体服务器的选择有三个方案。

方案一：SRS（Simple Realtime Server）。 开源、轻量、支持 RTMP 推流和 HLS/HTTP-FLV 拉流。部署简单，适合中小型平台。

方案二：Nginx-rtmp-module。 基于 Nginx 的 RTMP 模块，配置灵活，但功能相对基础。

方案三：云直播服务。 阿里云直播、腾讯云直播、声网等。优点是稳定、低延迟、全球覆盖；缺点是成本随流量线性增长。

对于私有化部署的多商户商城，SRS 是最务实的选择。它支持 RTMP 推流、HLS/HTTP-FLV 拉流，且有完善的 HTTP API 支持直播间管理。

二、全终端播放的协议适配
2.1 各终端的播放能力差异
直播播放的核心是协议和播放器的选择。不同终端对协议的支持程度不同。

微信小程序。 支持 live-player 组件播放 RTMP 和 HTTP-FLV，但需要开通小程序的直播权限（类目审核）。HLS 在小程序中延迟较高（10-30 秒），HTTP-FLV 延迟较低（3-5 秒）。

App。 可以使用原生播放器（如 ijkplayer、ExoPlayer）或 UniApp 的原生插件。支持 RTMP、HTTP-FLV、HLS 全协议。

H5。 浏览器的支持最受限。Safari 原生支持 HLS，Chrome 需要 hls.js 库。HTTP-FLV 需要 flv.js 库。RTMP 在浏览器中已不支持。

公众号。 本质上是 H5 运行在微信内置浏览器中，播放能力与 H5 相同。

2.2 播放协议的降级策略
由于各终端的支持程度不同，系统需要提供协议降级：优先使用低延迟协议（HTTP-FLV），不支持时降级到 HLS。

javascript
// utils/live/player.js

export function selectPlayUrl(room) {
  // #ifdef MP-WEIXIN
  return room.play_url_flv || room.play_url_hls;
  // #endif

  // #ifdef APP-PLUS
  return room.play_url_flv || room.play_url_rtmp || room.play_url_hls;
  // #endif

  // #ifdef H5
  if (supportsFLV()) {
    return room.play_url_flv;
  }
  return room.play_url_hls;
  // #endif
}

function supportsFLV() {
  // #ifdef H5
  return typeof window.MediaSource !== 'undefined'
    && window.MediaSource.isTypeSupported('video/mp4; codecs="avc1.42E01E"');
  // #endif
  return false;
}
2.3 H5 端的播放器实现
H5 端的直播播放是最复杂的，需要根据浏览器能力动态加载播放器。

vue
<!-- components/LivePlayer.vue -->
<template>
  <view class="live-player">
    <!-- #ifdef MP-WEIXIN -->
    <live-player
      :src="playUrl"
      mode="live"
      autoplay
      @statechange="onStateChange"
      @error="onError"
    />
    <!-- #endif -->

    <!-- #ifdef H5 -->
    <video
      ref="videoEl"
      class="h5-player"
      controls
      autoplay
      playsinline
    />
    <!-- #endif -->

    <!-- #ifdef APP-PLUS -->
    <video :src="playUrl" autoplay controls />
    <!-- #endif -->
  </view>
</template>

<script>
export default {
  props: { room: Object },
  data() {
    return {
      playUrl: '',
      flvPlayer: null,
      hlsPlayer: null,
    };
  },
  mounted() {
    this.playUrl = selectPlayUrl(this.room);
    this.initPlayer();
  },
  beforeDestroy() {
    this.destroyPlayer();
  },
  methods: {
    async initPlayer() {
      // #ifdef H5
      await this.initH5Player();
      // #endif
    },
    async initH5Player() {
      const videoEl = this.$refs.videoEl;
      if (!videoEl) return;

      if (this.playUrl.includes('.flv')) {
        const flvjs = await import('flv.js');
        if (flvjs.default.isSupported()) {
          this.flvPlayer = flvjs.default.createPlayer({
            type: 'flv',
            url: this.playUrl,
            isLive: true,
          });
          this.flvPlayer.attachMediaElement(videoEl);
          this.flvPlayer.load();
          this.flvPlayer.play();
        }
      } else if (this.playUrl.includes('.m3u8')) {
        if (!videoEl.canPlayType('application/vnd.apple.mpegurl')) {
          const Hls = await import('hls.js');
          if (Hls.default.isSupported()) {
            this.hlsPlayer = new Hls.default();
            this.hlsPlayer.loadSource(this.playUrl);
            this.hlsPlayer.attachMedia(videoEl);
          }
        } else {
          videoEl.src = this.playUrl;
        }
      }
    },
    destroyPlayer() {
      if (this.flvPlayer) { this.flvPlayer.destroy(); this.flvPlayer = null; }
      if (this.hlsPlayer) { this.hlsPlayer.destroy(); this.hlsPlayer = null; }
    },
    onStateChange(e) {},
    onError(e) { console.error('播放错误', e); },
  },
};
</script>
动态加载播放器库 + 协议降级是核心设计。H5 端根据播放地址的格式动态加载 flv.js 或 hls.js，而不是在初始包中引入所有播放器。这降低了首屏加载时间。

三、推流：App 与小程序的技术实现
3.1 微信小程序推流
微信小程序提供了 live-pusher 组件用于推流。它封装了采集、编码和推送的全流程。

vue
<!-- pages/live/push.vue -->
<template>
  <view class="push-page">
    <!-- #ifdef MP-WEIXIN -->
    <live-pusher
      ref="pusher"
      :url="pushUrl"
      mode="HD"
      :aspect="aspect"
      :beauty="beauty"
      @statechange="onPushStateChange"
      @error="onPushError"
    />
    <!-- #endif -->
  </view>
</template>

<script>
export default {
  data() {
    return {
      pushUrl: '',
      aspect: '9:16',
      beauty: 5,
    };
  },
  async onLoad(options) {
    const res = await this.$api.live.createRoom({ title: options.title || '直播' });
    this.pushUrl = res.data.push_url;
  },
  methods: {
    onPushStateChange(e) {
      if (e.detail.code === 1002) {
        this.$api.live.start(this.roomId);
      }
    },
    onPushError(e) { console.error('推流错误', e); },
    startPush() {
      // #ifdef MP-WEIXIN
      this.$refs.pusher.start();
      // #endif
    },
    stopPush() {
      // #ifdef MP-WEIXIN
      this.$refs.pusher.stop();
      // #endif
    },
  },
};
</script>
小程序推流的工程约束是：需要开通小程序的直播权限。微信对 live-pusher 和 live-player 组件有类目限制，只有特定类目（如电商、教育、医疗）的小程序才能使用。

3.2 App 端推流
App 端的推流需要集成原生推流 SDK。UniApp 没有内置推流组件，需要使用原生插件。

javascript
// utils/live/appPusher.js

class AppPusher {
  constructor() {
    this.pusher = null;
  }

  init(options) {
    // #ifdef APP-PLUS
    this.pusher = uni.requireNativePlugin('LivePusher');
    this.pusher.init({
      url: options.pushUrl,
      mode: options.mode || 'HD',
      aspect: options.aspect || '9:16',
      beauty: options.beauty || 5,
    }, (result) => {
      if (result.code === 0) console.log('推流初始化成功');
      else console.error('推流初始化失败', result);
    });
    // #endif
  }

  start() {
    // #ifdef APP-PLUS
    if (this.pusher) this.pusher.start();
    // #endif
  }

  stop() {
    // #ifdef APP-PLUS
    if (this.pusher) this.pusher.stop();
    // #endif
  }

  switchCamera() {
    // #ifdef APP-PLUS
    if (this.pusher) this.pusher.switchCamera();
    // #endif
  }

  destroy() {
    // #ifdef APP-PLUS
    if (this.pusher) { this.pusher.destroy(); this.pusher = null; }
    // #endif
  }
}

export const appPusher = new AppPusher();
3.3 H5 端不支持推流的原因
H5 端在浏览器中运行，无法直接访问设备的摄像头和麦克风进行 RTMP 推流。虽然浏览器提供了 getUserMedia API 访问摄像头和麦克风，但浏览器无法直接将音视频流推送到 RTMP 服务器。

浏览器的推流方案是 WebRTC，但 WebRTC 需要服务端支持，且与 RTMP 不兼容。如果系统需要支持 H5 推流，需要额外的 WebRTC 服务端和协议转换。

对于多商户商城系统，H5 端不做推流是合理的选择。商家用 App 或小程序推流，观众用任意终端观看，这已经覆盖了绝大多数使用场景。

四、直播与商城的业务耦合
4.1 直播间的商品管理
直播间与商品的关联是直播带货的核心。主播在直播过程中可以切换讲解的商品，观众在直播间看到当前讲解的商品并直接下单。

sql
CREATE TABLE `live_product` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `room_id` int(11) NOT NULL,
  `product_id` int(11) NOT NULL,
  `sort` tinyint(4) NOT NULL DEFAULT '0',
  `is_active` tinyint(1) NOT NULL DEFAULT '0' COMMENT '是否正在讲解',
  `active_at` datetime DEFAULT NULL,
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_room_product` (`room_id`, `product_id`),
  KEY `idx_room_sort` (`room_id`, `sort`),
  KEY `idx_active` (`room_id`, `is_active`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
is_active 字段标记当前正在讲解的商品。同一时刻，一个直播间只有一个商品是 is_active = 1。当主播切换商品时，系统更新这个字段，并通过 WebSocket 推送消息给所有观众。

4.2 直播订单的归因
直播订单需要归因到具体的直播间和主播。

sql
ALTER TABLE `order` ADD COLUMN `live_room_id` int(11) DEFAULT NULL 
  COMMENT '直播订单归因的直播间ID' AFTER `merchant_id`;
ALTER TABLE `order` ADD KEY `idx_live_room` (`live_room_id`);
4.3 实时互动：弹幕、点赞、下单通知
直播间的实时互动通过 WebSocket 实现。弹幕、点赞、下单通知都需要实时推送给所有观众。

php
<?php
// live_server.php — 基于 Workerman 的直播 WebSocket 服务

use Workerman\Worker;
use Workerman\Connection\TcpConnection;

$ws = new Worker('websocket://0.0.0.0:8282');
$ws->count = 4;

$rooms = [];

$ws->onMessage = function (TcpConnection $connection, $data) use (&$rooms) {
    $msg = json_decode($data, true);
    if (!$msg || !isset($msg['type'])) return;

    switch ($msg['type']) {
        case 'join':
            $roomId = (int)$msg['room_id'];
            $connection->roomId = $roomId;
            $connection->userId = (int)$msg['user_id'];

            if (!isset($rooms[$roomId])) $rooms[$roomId] = [];
            $rooms[$roomId][$connection->id] = $connection;

            broadcast($rooms[$roomId], [
                'type' => 'viewer_count',
                'count' => count($rooms[$roomId]),
            ]);
            break;

        case 'danmaku':
            if ($connection->roomId) {
                broadcast($rooms[$connection->roomId] ?? [], [
                    'type' => 'danmaku',
                    'user_id' => $connection->userId,
                    'content' => mb_substr($msg['content'], 0, 50),
                    'timestamp' => time(),
                ]);
            }
            break;

        case 'like':
            if ($connection->roomId) {
                broadcast($rooms[$connection->roomId] ?? [], [
                    'type' => 'like',
                    'user_id' => $connection->userId,
                    'count' => $msg['count'] ?? 1,
                ]);
            }
            break;

        case 'product_active':
            if ($connection->roomId) {
                broadcast($rooms[$connection->roomId] ?? [], [
                    'type' => 'product_active',
                    'product_id' => (int)$msg['product_id'],
                ]);
            }
            break;

        case 'order_notify':
            if ($connection->roomId) {
                broadcast($rooms[$connection->roomId] ?? [], [
                    'type' => 'order_notify',
                    'nickname' => $msg['nickname'],
                    'product_name' => $msg['product_name'],
                ]);
            }
            break;
    }
};

$ws->onClose = function (TcpConnection $connection) use (&$rooms) {
    if ($connection->roomId && isset($rooms[$connection->roomId])) {
        unset($rooms[$connection->roomId][$connection->id]);
        if (empty($rooms[$connection->roomId])) unset($rooms[$connection->roomId]);
        else broadcast($rooms[$connection->roomId], [
            'type' => 'viewer_count',
            'count' => count($rooms[$connection->roomId]),
        ]);
    }
};

function broadcast(array $connections, array $message) {
    $json = json_encode($message);
    foreach ($connections as $conn) $conn->send($json);
}

Worker::runAll();
房间隔离是核心设计。每个直播间是一个独立的房间，消息只在房间内广播。这保证了不同商户的直播间不会互相干扰。

五、短视频社区与直播的协同
5.1 内容与直播的互相引流
短视频社区和直播不是独立的功能，它们可以互相引流。短视频可以关联直播间，引导用户进入直播；直播结束后可以生成回放，沉淀为短视频内容。

sql
-- 短视频关联直播间
ALTER TABLE `community_post` ADD COLUMN `live_room_id` int(11) DEFAULT NULL 
  COMMENT '关联的直播间ID';
ALTER TABLE `community_post` ADD KEY `idx_live_room` (`live_room_id`);
当短视频关联了正在直播的直播间时，视频播放页面会显示“直播中”的入口，用户点击后直接进入直播间。

5.2 直播回放的沉淀
直播结束后，流媒体服务器可以自动将直播流录制为视频文件。系统将回放地址写入直播间记录，并触发转码。

php
<?php
// app/common/service/LiveReplayService.php

namespace app\common\service;

use think\facade\Db;
use think\facade\Queue;

class LiveReplayService
{
    public function handleReplay(int $roomId, string $recordPath): void
    {
        Queue::push('app\job\TranscodeReplay', [
            'room_id' => $roomId,
            'record_path' => $recordPath,
        ], 'video');

        Db::name('live_room')->where('id', $roomId)->update([
            'status' => 2,
            'ended_at' => date('Y-m-d H:i:s'),
            'replay_record_path' => $recordPath,
        ]);
    }
}
回放转码为 HLS 格式后，可以像普通短视频一样在种草社区中展示，形成“直播 → 回放 → 短视频”的内容沉淀链路。

六、边界与张力
多商户商城直播系统在工程实践中，有几个需要清醒认识的边界。

第一，小程序的直播权限限制。 微信对 live-pusher 和 live-player 组件有类目限制。不是所有商户的小程序都能开通直播权限。系统需要提供“直播功能降级”方案。

第二，推流的带宽成本。 推流消耗上行带宽，拉流消耗下行带宽。一场 1000 人观看的直播，按 720p 码率计算，每分钟消耗约 11GB 下行流量。多商户平台的带宽成本会随直播场次和观众数快速累积。

第三，直播延迟与互动的矛盾。 HTTP-FLV 的延迟约 3-5 秒，HLS 的延迟约 10-30 秒。观众看到的画面和主播的实际动作有时间差。

第四，多商户的内容审核。 直播是实时内容，审核难度远高于图文。系统需要提供实时审核能力（AI 识别违规画面 + 人工抽检），以及紧急关停机制。

第五，直播录制的存储成本。 每场直播的录制文件可能达到数 GB。多商户平台需要制定存储策略：保留多久、是否转码、是否清理。

结语：全终端直播的工程本质是“协议适配 + 能力降级”
多商户商城直播系统的核心工程挑战，可以归结为一句话：在有限的终端能力下，提供最接近一致的直播体验。

播放侧，系统需要根据终端能力选择协议：小程序和 App 优先 HTTP-FLV（低延迟），H5 根据浏览器能力选择 FLV 或 HLS。播放器动态加载，避免首屏负担。

推流侧，小程序使用 live-pusher 组件，App 使用原生推流 SDK，H5 不支持推流。这是终端能力的客观限制，工程上的应对是“能力降级”而非“强行突破”。

业务侧，直播间、商品、订单、互动消息都需要与多商户架构正确耦合。每个商户的直播间是独立的，商品是隔离的，订单是归属明确的。

这套系统的价值，最终体现在一个朴素的判断上：一个商户用 App 开播，观众用小程序、H5 或 App 都能看到直播并下单，直播结束后回放可以沉淀为种草社区的内容。 这个“都能看到”就是全终端直播的工程底线。
