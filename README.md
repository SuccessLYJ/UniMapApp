# 集卡地图（UniApp + Leaflet）

一个基于 UniApp（Vite 构建）与 Leaflet 的 H5 集卡地图应用，用于在指定景区范围内展示地图、放置打卡点并完成“集卡”互动。示例以开封龙亭区为演示范围。

## 功能特性
- 指定区域显示与限制：视图始终在设定范围内，拖动或缩放都会被约束回边界。
- 打卡点位：在地图上展示多个景点标记，点击标记弹出底部卡片。
- 集卡逻辑：支持“去打卡（定位到点位）”“打卡集卡”，状态持久化到本地存储。
- 点位列表抽屉：查看全部点位并一键定位。
- 轻量 UI：顶部显示已收集数量，标记颜色区分已打卡/未打卡。

## 快速开始
- 环境要求：Node.js >= 18，建议使用 Chrome/Edge/Firefox 现代浏览器。
- 安装依赖：
  ```bash
  npm install
  ```
- 启动开发服务器：
  ```bash
  npm run dev:h5
  # 浏览器访问 http://localhost:5173/
  ```
- 生产构建：
  ```bash
  npm run build:h5
  # 构建产物位于 dist/build/h5
  ```

## 目录结构
```text
UniMapApp/
├── index.html
├── package.json
├── src/
│   ├── App.vue
│   ├── main.js
│   ├── pages.json
│   ├── pages/
│   │   └── index/index.vue   # 主要地图与集卡页面
│   ├── manifest.json
│   └── static/
└── vite.config.js
```

## 关键实现与配置
- 地图与范围限制（`src/pages/index/index.vue`）：
  - 引入 Leaflet：`import L from 'leaflet'; import 'leaflet/dist/leaflet.css'`
  - 指定范围（示例为龙亭近似矩形）：
    ```js
    bounds: [[34.795, 114.352], [34.804, 114.371]]
    ```
  - 初始化地图并限制范围：
    ```js
    this.map = L.map('map', {
      zoomControl: false,
      maxBounds: L.latLngBounds(this.bounds),
      maxBoundsViscosity: 1.0,
    })
    ```
  - 动态最小缩放与边界校正：
    ```js
    const boundsObj = L.latLngBounds(this.bounds)
    const minZoom = this.map.getBoundsZoom(boundsObj, true)
    this.map.setMinZoom(minZoom)
    this.map.fitBounds(boundsObj)
    const keepInside = () => this.map.panInsideBounds(boundsObj, { animate: true })
    this.map.on('moveend', keepInside)
    this.map.on('zoomend', keepInside)
    ```
  - 点位数据与交互（片段）：
    ```js
    poiList: [{ id:'longting-hall', name:'龙亭大殿', latlng:[34.7995,114.3612], ... }, ...]
    collected: new Set()
    // 统一持久化
    uni.setStorageSync('collected-cards', Array.from(this.collected))
    ```

- 切换瓦片源（可选）：
  ```js
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', { ... })
  // 如需国内镜像，可替换为：
  // https://webrd02.is.autonavi.com/appmaptile?style=7&x={x}&y={y}&z={z}
  // 或其他商业/自建瓦片服务（请遵守相应使用条款）。
  ```

## 常见问题
- `net::ERR_ABORTED`：热更新或瓦片请求被取消的提示，一般不影响功能。刷新即可。
- 地图白屏/跨域：检查网络与瓦片源可用性，必要时更换为国内镜像或自建瓦片服务。
- 集卡不持久：确认浏览器允许本地存储，或在隐私模式下可能被清空。

## 自定义与扩展
- 精确边界：如需更精确的景区“多边形”边界，可将行政区 GeoJSON 引入，计算 `minZoom` 仍用其外接矩形，并在移动/缩放事件中校验中心点是否在多边形内，不在则回退到 `polygon.getBounds().getCenter()`。
- 数据来源：将 `poiList` 改为从后端接口或静态 JSON 加载，便于维护与扩展。
- 成就与分享：完成集齐后生成分享海报、成就徽章等玩法。

## 版权与数据使用
- 地图瓦片与地理数据请遵守提供方的使用协议（如 OpenStreetMap/高德/天地图等）。

---
如需我帮你接入真实龙亭区边界、优化瓦片源或完善交互样式，请直接提出需求。