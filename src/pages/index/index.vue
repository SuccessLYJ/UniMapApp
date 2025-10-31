<template>
  <view class="page">
  <view class="header">
      <text class="title">集卡地图 · 云台山</text>
      <text class="count">已收集：{{ collectedCount }}/{{ poiList.length }}</text>
    </view>

    <div id="map" class="map"></div>

    <view class="actions">
      <button class="btn" @click="toggleList">全部打卡点位</button>
    </view>

    <!-- 点位列表抽屉 -->
    <view v-if="showList" class="drawer">
      <view class="drawer-header">
        <text>点位列表（点击定位）</text>
        <text class="close" @click="toggleList">关闭</text>
      </view>
      <view class="drawer-body">
        <view v-for="p in poiList" :key="p.id" class="poi-item" @click="flyTo(p)">
          <view class="poi-left">
            <text class="poi-name">{{ p.name }}</text>
            <text class="poi-sub">{{ collected.has(p.id) ? '已打卡' : '未打卡' }}</text>
          </view>
          <view class="poi-right">→</view>
        </view>
      </view>
    </view>

    <!-- 底部卡片弹层 -->
    <view v-if="activePoi" class="card">
      <image class="card-img" :src="activePoi.image" mode="aspectFill"></image>
      <view class="card-content">
        <text class="card-title">{{ activePoi.name }}</text>
        <text class="card-desc">{{ activePoi.desc }}</text>
        <view class="card-actions">
          <button class="btn secondary" @click="flyTo(activePoi)">去打卡</button>
          <button class="btn primary" :disabled="collected.has(activePoi.id)" @click="collectActive">{{ collected.has(activePoi.id) ? '已收集' : '打卡集卡' }}</button>
        </view>
      </view>
    </view>
  </view>
  
</template>

<script>
import AMapLoader from '@amap/amap-jsapi-loader'

export default {
  data() {
    return {
      map: null,
      showList: false,
      collected: new Set(),
      activePoi: null,
      // 云台山近似范围（lat,lng），高德需转为 [lng,lat]
      bounds: [[35.4200, 113.3400], [35.4450, 113.3800]],
      poiList: [
        {
          id: 'hongshixia',
          name: '红石峡（温盘峪）',
          desc: '丹霞地貌代表，外旷内幽，泉瀑溪潭相汇。',
          image: 'https://images.unsplash.com/photo-1501785888041-af3ef285b470?w=800&q=80',
          latlng: [35.4315, 113.3578],
        },
        {
          id: 'tanpoxia',
          name: '潭瀑峡',
          desc: '太行秀水，瀑、泉、溪、潭密布，步步生景。',
          image: 'https://images.unsplash.com/photo-1502082553048-f009c37129b9?w=800&q=80',
          latlng: [35.4305, 113.3645],
        },
        {
          id: 'quanpoxia',
          name: '泉瀑峡（云台天瀑）',
          desc: '单级落差约314米，枯水期有补水装置增强观赏。',
          image: 'https://images.unsplash.com/photo-1513836279014-a89f7a76ae86?w=800&q=80',
          latlng: [35.4285, 113.3680],
        },
        {
          id: 'zhuyufeng',
          name: '茱萸峰',
          desc: '云台山主峰，玄帝宫与玻璃栈道所在，登高望远。',
          image: 'https://images.unsplash.com/photo-1500534314209-a25ddb2bd429?w=800&q=80',
          latlng: [35.4420, 113.3490],
        },
        {
          id: 'mihougu',
          name: '猕猴谷',
          desc: '太行猕猴群落栖居地，趣味性强的动物观赏区。',
          image: 'https://images.unsplash.com/photo-1558981359-9e3e0c9f6254?w=800&q=80',
          latlng: [35.4250, 113.3520],
        },
        {
          id: 'zifanghu',
          name: '子房湖',
          desc: '云台山东区最大湖泊，湖光山色相映，环境幽静。',
          image: 'https://images.unsplash.com/photo-1506421547861-8ffcb08fe5a6?w=800&q=80',
          latlng: [35.4375, 113.3720],
        },
        {
          id: 'diecaidong',
          name: '叠彩洞',
          desc: '沿山体开凿的盘山公路洞群，景致独特。',
          image: 'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?w=800&q=80',
          latlng: [35.4390, 113.3605],
        },
        {
          id: 'wanshansi',
          name: '万善寺',
          desc: '明代古寺，宗教文化底蕴浓厚，环境清幽。',
          image: 'https://images.unsplash.com/photo-1520975916090-3105956dac38?w=800&q=80',
          latlng: [35.4385, 113.3465],
        },
      ],
      markers: [],
    }
  },
  computed: {
    collectedCount() {
      return this.collected.size
    },
  },
  onLoad() {
    this.initCollected()
  },
  mounted() {
    this.initMap()
  },
  methods: {
    initCollected() {
      try {
        const saved = uni.getStorageSync('collected-cards')
        if (saved && Array.isArray(saved)) {
          this.collected = new Set(saved)
        }
      } catch (e) {}
    },
    persistCollected() {
      uni.setStorageSync('collected-cards', Array.from(this.collected))
    },
    initMap() {
      const key = '6afe8ff5aec5b44d1ba9c08235bba145'
      const centerLat = (this.bounds[0][0] + this.bounds[1][0]) / 2
      const centerLng = (this.bounds[0][1] + this.bounds[1][1]) / 2
      const gaodeCenter = [centerLng, centerLat]
      const gaodeBounds = new Array(2)
      gaodeBounds[0] = [this.bounds[0][1], this.bounds[0][0]]
      gaodeBounds[1] = [this.bounds[1][1], this.bounds[1][0]]

      AMapLoader.load({
        key,
        version: '2.0',
        plugins: ['AMap.Scale', 'AMap.ToolBar'],
      }).then(AMap => {
        this.map = new AMap.Map('map', {
          viewMode: '2D',
          zoom: 16,
          center: gaodeCenter,
        })

        const bounds = new AMap.Bounds(gaodeBounds[0], gaodeBounds[1])
        this.map.setLimitBounds(bounds)

        // 可视化边界（可选）
        const rect = new AMap.Rectangle({ bounds, strokeColor: '#2563eb', strokeWeight: 1, fillOpacity: 0 })
        this.map.add(rect)

        // 让视图贴合边界并据此计算“最小允许缩放”
        this.map.setFitView([rect])
        const minZoom = this.map.getZoom()
        this.map.setZooms([minZoom, 20])

        this.addMarkersGaode(AMap)
      }).catch(e => {
        console.error('AMapLoader error:', e)
      })
    },
    markerContent(collected) {
      const color = collected ? '#10b981' : '#f59e0b'
      return `<div style="background:${color};color:#fff;padding:6px 10px;border-radius:16px;box-shadow:0 2px 6px rgba(0,0,0,.2);font-size:12px;">打卡</div>`
    },
    addMarkersGaode(AMap) {
      // 清理旧标记
      this.markers.forEach(m => this.map.remove(m))
      this.markers = []
      this.poiList.forEach(p => {
        const pos = [p.latlng[1], p.latlng[0]] // 转换为 [lng,lat]
        const marker = new AMap.Marker({
          position: pos,
          anchor: 'bottom-center',
          content: this.markerContent(this.collected.has(p.id)),
          title: p.name,
        })
        marker.on('click', () => { this.activePoi = p })
        this.map.add(marker)
        this.markers.push(marker)
      })
    },
    flyTo(p) {
      this.showList = false
      if (this.map) {
        const pos = [p.latlng[1], p.latlng[0]]
        this.map.setZoomAndCenter(18, pos)
      }
      this.activePoi = p
    },
    collectActive() {
      if (!this.activePoi) return
      this.collected.add(this.activePoi.id)
      this.persistCollected()
      this.addMarkersGaode(AMap)
    },
    toggleList() {
      this.showList = !this.showList
    },
  },
}
</script>

<style>
.page { width: 100%; height: 100vh; display: flex; flex-direction: column; }
.header { padding: 12px 16px; display: flex; justify-content: space-between; align-items: center; background: #ffffff; border-bottom: 1px solid #eee; }
.title { font-size: 16px; font-weight: 600; }
.count { font-size: 14px; color: #666; }
.map { flex: 1; }
.actions { position: absolute; left: 12px; bottom: 92px; z-index: 400; }
.btn { background: #2563eb; color: #fff; border: none; border-radius: 20px; padding: 8px 12px; font-size: 14px; }
.btn.primary { background: #10b981; }
.btn.secondary { background: #f59e0b; }

.drawer { position: absolute; left: 0; right: 0; bottom: 0; max-height: 50vh; background: #fff; border-top-left-radius: 14px; border-top-right-radius: 14px; box-shadow: 0 -6px 16px rgba(0,0,0,.08); z-index: 500; }
.drawer-header { display:flex; justify-content: space-between; align-items: center; padding: 12px 16px; border-bottom: 1px solid #eee; }
.close { color: #2563eb; }
.drawer-body { overflow-y: auto; padding: 8px 12px; }
.poi-item { display:flex; justify-content: space-between; align-items: center; padding: 10px 8px; border-bottom: 1px dashed #eee; }
.poi-name { font-size: 14px; }
.poi-sub { font-size: 12px; color: #666; }

.card { position: absolute; left: 12px; right: 12px; bottom: 16px; background: #fff; border-radius: 12px; box-shadow: 0 8px 24px rgba(0,0,0,.12); display:flex; overflow: hidden; z-index: 450; }
.card-img { width: 36%; height: 120px; }
.card-content { flex: 1; padding: 10px 12px; }
.card-title { font-weight: 600; font-size: 16px; }
.card-desc { display:block; margin-top: 6px; font-size: 12px; color: #555; }
.card-actions { margin-top: 10px; display:flex; gap: 8px; }

/* Leaflet 容器尺寸修复 */
.leaflet-container { width: 100%; height: 100%; }
</style>
