<template>
  <view class="page">
    <view class="header">
      <text class="title">集卡地图 · 龙亭区</text>
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
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

export default {
  data() {
    return {
      map: null,
      showList: false,
      collected: new Set(),
      activePoi: null,
      // 近似龙亭区范围（H5 体验用，实际可替换为更精确边界）
      bounds: [[34.795, 114.352], [34.804, 114.371]],
      poiList: [
        {
          id: 'longting-hall',
          name: '龙亭大殿',
          desc: '龙亭核心景点，大殿巍峨壮观。',
          image: 'https://images.unsplash.com/photo-1528909514045-2fa4ac7a08ba?w=800&q=80',
          latlng: [34.7995, 114.3612],
        },
        {
          id: 'wuyue-stele',
          name: '五岳真形碑',
          desc: '历代传承文物，见证古城文化。',
          image: 'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?w=800&q=80',
          latlng: [34.799, 114.3598],
        },
        {
          id: 'songhu-gate',
          name: '嵩呼门',
          desc: '园区古门坊，亦称叫呼、高呼。',
          image: 'https://images.unsplash.com/photo-1549875146-2096d5b0a6bd?w=800&q=80',
          latlng: [34.7979, 114.3665],
        },
        {
          id: 'east-lake-tea',
          name: '东湖茶舍',
          desc: '湖畔茶香，休憩观景的好去处。',
          image: 'https://images.unsplash.com/photo-1517638851339-4aa32003c11a?w=800&q=80',
          latlng: [34.7972, 114.3679],
        },
        {
          id: 'jiulong-ting',
          name: '九龙亭',
          desc: '古典亭台，龙纹华丽。',
          image: 'https://images.unsplash.com/photo-1520975916090-3105956dac38?w=800&q=80',
          latlng: [34.8002, 114.3583],
        },
        {
          id: 'yudai-bridge',
          name: '玉带桥',
          desc: '拱桥如玉带，横跨碧波。',
          image: 'https://images.unsplash.com/photo-1470071459604-3b5ec3a7fe05?w=800&q=80',
          latlng: [34.7968, 114.3653],
        },
        {
          id: 'food-plaza',
          name: '美食广场',
          desc: '逛园尝鲜，地道开封味道。',
          image: 'https://images.unsplash.com/photo-1543353071-873f17a7a5c9?w=800&q=80',
          latlng: [34.7988, 114.3567],
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
      const center = [
        (this.bounds[0][0] + this.bounds[1][0]) / 2,
        (this.bounds[0][1] + this.bounds[1][1]) / 2,
      ]
      this.map = L.map('map', {
        zoomControl: false,
        maxBounds: L.latLngBounds(this.bounds),
        maxBoundsViscosity: 1.0,
      }).setView(center, 16)

      L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap',
        maxZoom: 19,
      }).addTo(this.map)

      // 根据容器尺寸动态计算让视窗“完全在范围内”的最小缩放级别，防止缩出边界
      const boundsObj = L.latLngBounds(this.bounds)
      const minZoom = this.map.getBoundsZoom(boundsObj, true)
      this.map.setMinZoom(minZoom)
      // 初始视图贴合指定范围
      this.map.fitBounds(boundsObj)

      // 边界可视化（可选）
      L.rectangle(boundsObj, { color: '#2563eb', weight: 1, fillOpacity: 0 }).addTo(this.map)

      // 移动/缩放后强制保持在范围内
      const keepInside = () => this.map.panInsideBounds(boundsObj, { animate: true })
      this.map.on('moveend', keepInside)
      this.map.on('zoomend', keepInside)

      this.addMarkers()
    },
    markerIcon(collected) {
      const color = collected ? '#10b981' : '#f59e0b'
      return L.divIcon({
        className: 'custom-marker',
        html: `<div style="background:${color};color:#fff;padding:6px 10px;border-radius:16px;box-shadow:0 2px 6px rgba(0,0,0,.2);font-size:12px;">打卡</div>`,
        iconSize: [40, 24],
        iconAnchor: [20, 24],
      })
    },
    addMarkers() {
      this.markers.forEach(m => m.remove())
      this.markers = []
      this.poiList.forEach(p => {
        const m = L.marker(p.latlng, { icon: this.markerIcon(this.collected.has(p.id)) })
        m.addTo(this.map)
        m.on('click', () => {
          this.activePoi = p
        })
        this.markers.push(m)
      })
    },
    flyTo(p) {
      this.showList = false
      if (this.map) {
        this.map.flyTo(p.latlng, 18, { duration: 0.8 })
      }
      this.activePoi = p
    },
    collectActive() {
      if (!this.activePoi) return
      this.collected.add(this.activePoi.id)
      this.persistCollected()
      this.addMarkers()
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
