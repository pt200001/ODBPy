# ODB++ 前端画布 MVP（React + Vite）

本文给出可直接落地的前端最小实现，目标是对接后端解析 API 并实现：

- 上传 ODB++ ZIP
- 展示图层列表
- 图层可见性切换
- Canvas 缩放 / 平移
- 渲染线路（line）与焊盘（pad）

## 1. 依赖与项目初始化

```bash
npm create vite@latest odb-viewer -- --template react
cd odb-viewer
npm install
npm install axios
```

## 2. API 约定

前端默认调用：

- `POST /api/upload-and-parse`
- `GET /api/layers/{sessionId}/{layerName}`

可通过 `VITE_API_BASE_URL` 配置后端地址。

## 3. 文件结构

```text
src/
  App.jsx
  api.js
  components/
    Toolbar.jsx
    LayerPanel.jsx
    PcbCanvas.jsx
```

## 4. 代码：`src/api.js`

```js
import axios from "axios";

const API_BASE = import.meta.env.VITE_API_BASE_URL || "http://localhost:8000";

export async function uploadAndParse(file) {
  const formData = new FormData();
  formData.append("file", file);
  const { data } = await axios.post(`${API_BASE}/api/upload-and-parse`, formData, {
    headers: { "Content-Type": "multipart/form-data" },
  });
  return data;
}

export async function fetchLayerFeatures(sessionId, layerName) {
  const { data } = await axios.get(`${API_BASE}/api/layers/${sessionId}/${layerName}`);
  return data;
}
```

## 5. 代码：`src/components/Toolbar.jsx`

```jsx
export default function Toolbar({ onFileSelected, loading }) {
  return (
    <div style={{ display: "flex", gap: 12, marginBottom: 12 }}>
      <label>
        <input
          type="file"
          accept=".zip"
          disabled={loading}
          onChange={(e) => {
            const f = e.target.files?.[0];
            if (f) onFileSelected(f);
          }}
        />
      </label>
      {loading ? <span>解析中...</span> : <span>请选择 ODB++ ZIP</span>}
    </div>
  );
}
```

## 6. 代码：`src/components/LayerPanel.jsx`

```jsx
export default function LayerPanel({ layers, visibleMap, onToggle }) {
  return (
    <div style={{ width: 260, borderRight: "1px solid #ddd", padding: 12 }}>
      <h3 style={{ marginTop: 0 }}>图层</h3>
      {layers.map((l) => (
        <label
          key={l.name}
          style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}
        >
          <span>{l.name}</span>
          <input
            type="checkbox"
            checked={!!visibleMap[l.name]}
            onChange={() => onToggle(l.name)}
          />
        </label>
      ))}
    </div>
  );
}
```

## 7. 代码：`src/components/PcbCanvas.jsx`

```jsx
import { useEffect, useRef, useState } from "react";

const COLORS = ["#ff6b6b", "#4dabf7", "#51cf66", "#ffd43b", "#845ef7", "#20c997"];

export default function PcbCanvas({ layers, visibleMap, featuresByLayer }) {
  const canvasRef = useRef(null);
  const [view, setView] = useState({ scale: 4, tx: 100, ty: 100 });
  const [dragging, setDragging] = useState(false);
  const [last, setLast] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const c = canvasRef.current;
    const ctx = c.getContext("2d");
    ctx.clearRect(0, 0, c.width, c.height);
    ctx.fillStyle = "#111";
    ctx.fillRect(0, 0, c.width, c.height);

    const drawPt = (p) => ({ x: p.x * view.scale + view.tx, y: -p.y * view.scale + view.ty });

    let colorIdx = 0;
    for (const layer of layers) {
      if (!visibleMap[layer.name]) continue;
      const data = featuresByLayer[layer.name];
      if (!data) continue;

      const color = COLORS[colorIdx % COLORS.length];
      colorIdx += 1;

      ctx.strokeStyle = color;
      ctx.fillStyle = color;
      ctx.lineWidth = 1.2;

      for (const ln of data.lines || []) {
        const s = drawPt(ln.start_mm);
        const e = drawPt(ln.end_mm);
        ctx.beginPath();
        ctx.moveTo(s.x, s.y);
        ctx.lineTo(e.x, e.y);
        ctx.stroke();
      }

      for (const pad of data.pads || []) {
        const p = drawPt(pad.coords_mm);
        const r = Math.max(1.6, 1.8 * (pad.symbol?.resize_factor || 1));
        ctx.beginPath();
        ctx.arc(p.x, p.y, r, 0, Math.PI * 2);
        ctx.fill();
      }
    }
  }, [layers, visibleMap, featuresByLayer, view]);

  const onWheel = (e) => {
    e.preventDefault();
    const zoom = e.deltaY > 0 ? 0.9 : 1.1;
    setView((v) => ({ ...v, scale: Math.max(0.2, Math.min(30, v.scale * zoom)) }));
  };

  return (
    <canvas
      ref={canvasRef}
      width={1200}
      height={760}
      onWheel={onWheel}
      onMouseDown={(e) => {
        setDragging(true);
        setLast({ x: e.clientX, y: e.clientY });
      }}
      onMouseUp={() => setDragging(false)}
      onMouseLeave={() => setDragging(false)}
      onMouseMove={(e) => {
        if (!dragging) return;
        const dx = e.clientX - last.x;
        const dy = e.clientY - last.y;
        setLast({ x: e.clientX, y: e.clientY });
        setView((v) => ({ ...v, tx: v.tx + dx, ty: v.ty + dy }));
      }}
      style={{ display: "block", cursor: dragging ? "grabbing" : "grab" }}
    />
  );
}
```

## 8. 代码：`src/App.jsx`

```jsx
import { useMemo, useState } from "react";
import { uploadAndParse, fetchLayerFeatures } from "./api";
import Toolbar from "./components/Toolbar";
import LayerPanel from "./components/LayerPanel";
import PcbCanvas from "./components/PcbCanvas";

export default function App() {
  const [loading, setLoading] = useState(false);
  const [sessionId, setSessionId] = useState("");
  const [layers, setLayers] = useState([]);
  const [visibleMap, setVisibleMap] = useState({});
  const [featuresByLayer, setFeaturesByLayer] = useState({});

  const onFileSelected = async (file) => {
    setLoading(true);
    try {
      const summary = await uploadAndParse(file);
      setSessionId(summary.session_id);
      setLayers(summary.layers || []);
      const vis = {};
      for (const l of summary.layers || []) vis[l.name] = true;
      setVisibleMap(vis);

      const entries = await Promise.all(
        (summary.layers || []).map(async (l) => [
          l.name,
          await fetchLayerFeatures(summary.session_id, l.name),
        ])
      );
      setFeaturesByLayer(Object.fromEntries(entries));
    } finally {
      setLoading(false);
    }
  };

  const onToggle = (name) => {
    setVisibleMap((m) => ({ ...m, [name]: !m[name] }));
  };

  const title = useMemo(() => {
    if (!sessionId) return "ODB++ Viewer MVP";
    return `ODB++ Viewer MVP · session ${sessionId.slice(0, 8)}`;
  }, [sessionId]);

  return (
    <div style={{ fontFamily: "Inter, system-ui, sans-serif", color: "#111" }}>
      <h2>{title}</h2>
      <Toolbar onFileSelected={onFileSelected} loading={loading} />
      <div style={{ display: "flex", border: "1px solid #ddd" }}>
        <LayerPanel layers={layers} visibleMap={visibleMap} onToggle={onToggle} />
        <div style={{ flex: 1, overflow: "auto" }}>
          <PcbCanvas layers={layers} visibleMap={visibleMap} featuresByLayer={featuresByLayer} />
        </div>
      </div>
    </div>
  );
}
```

## 9. 运行

```bash
npm run dev
```

如果后端不是 `localhost:8000`：

```bash
# .env
VITE_API_BASE_URL=http://127.0.0.1:8000
```

## 10. 后续增强建议

- 分层懒加载：只在图层首次打开时拉取
- WebGL 渲染：适合超大板子
- 几何预处理：后端计算 bbox / 每层统计 / 简化数据
- Google Drive 选择器接入（Picker + OAuth）

