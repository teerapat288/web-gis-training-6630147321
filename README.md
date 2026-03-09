Wedgis
<!-- 1. วาดโครงสร้าง html -->

<!DOCTYPE html>
<html lang="th">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web GIS Training</title>

    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

    <link rel="stylesheet" href="https://unpkg.com/@geoman-io/leaflet-geoman-free@latest/dist/leaflet-geoman.css" />
    <script src="https://unpkg.com/@geoman-io/leaflet-geoman-free@latest/dist/leaflet-geoman.min.js"></script>

    <style>
        /* ส่วนการจัดหน้าจอ (Layout Styling) */
        body {
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, sans-serif;
            display: flex;
        }

        /* พื้นที่แสดงผลแผนที่หลัก */
        #map {
            flex-grow: 1;
            height: 100vh;
        }
        /* Sidebar สำหรับเก็บเครื่องมือควบคุมชั้นข้อมูล */
        #sidebar {
            width: 320px;
            height: 100vh;
            background: #fff;
            border-right: 1px solid #ddd;
            z-index: 2000;
            overflow-y: auto;
            transition: 0.3s;
            flex-shrink: 0;
        }

        #sidebar.collapsed {
            margin-left: -320px;
        }

        .panel-content {
            padding: 20px;
        }

        .layer-group {
            margin-bottom: 25px;
            clear: both;
        }

        /* หัวข้อกลุ่ม Layer ใน Sidebar */
        .section-title {
            font-weight: bold;
            border-bottom: 2px solid #3498db;
            padding-bottom: 5px;
            margin-bottom: 10px;
            display: block;
            color: #2c3e50;
        }
        /* ขยับปุ่ม Geoman (เครื่องมือวาด) ลงมาเล็กน้อยเพื่อไม่ให้ซ้อนทับกับปุ่มเมนูของเรา */
        .leaflet-top.leaflet-left {
            margin-top: 60px !important;
        }

        /* ปรับสไตล์ Layer Control ของ Leaflet ให้มาแสดงเป็นรายการใน Sidebar */
        .leaflet-control-layers {
            position: relative !important;
            float: none !important;
            box-shadow: none !important;
            border: none !important;
            margin: 0 !important;
            display: block !important;
        }
/* ปุ่ม Hamburger สำหรับเปิด-ปิดแถบเมนูด้านซ้าย */
        #toggle-btn {
            position: absolute;
            top: 10px;
            left: 330px;
            z-index: 2001;
            padding: 10px 15px;
            background: white;
            border: 1px solid #ccc;
            border-radius: 4px;
            cursor: pointer;
            font-weight: bold;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }

        /* ปุ่มสีเขียวสำหรับส่งออกไฟล์ข้อมูล */
        .export-btn {
            width: 100%;
            padding: 12px;
            background: #27ae60;
            color: white;
            border: none;
            cursor: pointer;
            font-weight: bold;
            border-radius: 4px;
        }
    </style>

</head>

<body>

    <div id="sidebar">
        <div class="panel-content">
            <h2 style="margin-top: 0;">GIS Tools</h2>

            <div class="layer-group">
                <div class="section-title">1. Basemap Layers</div>
                <div id="basemap-ctrl"></div>
            </div>

            <div class="layer-group">
                <div class="section-title">2. WMS Layers</div>
                <div id="wms-ctrl"></div>
            </div>

            <div class="layer-group">
                <div class="section-title">3. GeoJSON Data</div>
                <input type="file" id="fileInput" accept=".geojson">
                <div id="geojson-ctrl"></div>
            </div>

            <div class="layer-group">
                <div class="section-title">Drawing Tools</div>
                <button class="export-btn" onclick="exportGeoJSON()">📥 Export GeoJSON</button>
            </div>
        </div>
    </div>

    <button id="toggle-btn" onclick="toggleSidebar()">☰ Layers</button>

<div id="map"></div>

    <script>

        // [ขั้นตอนที่ 2] การสร้างแผนที่และการจัดการลำดับชั้น (Map & Z-Index)
        // 2.1 กำหนดค่าเริ่มต้นแผนที่ (พิกัดกรุงเทพฯ, ซูมระดับ 6)
        const map = L.map('map').setView([13.75, 100.5], 6);

        // 2.2 สร้าง 'Pane' พิเศษสำหรับข้อมูล Vector เพื่อบังคับให้ข้อมูลจุด/เส้น/พื้นที่ อยู่ทับข้อมูลภาพ (WMS) เสมอ
        map.createPane('vectorPane');
        map.getPane('vectorPane').style.zIndex = 650; // ค่ามาตรฐานของภาพคือ 200-400, เราจึงตั้ง 650
        map.getPane('vectorPane').style.pointerEvents = 'none'; // ป้องกันการบังการคลิกของ Layer อื่น

        // [ขั้นตอนที่ 3] การจัดการแผนที่ฐาน (Basemaps)
        const osm = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '© OpenStreetMap'
        }).addTo(map); // แสดง OSM เป็นค่าเริ่มต้น

        const satellite = L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', {
            attribution: '© Esri'
        });

        // รวบรวม Basemap ไว้ใน Object เดียวเพื่อใช้กับ Control
        const baseMaps = { "OpenStreetMap": osm, "Satellite Imagery": satellite };

        // [ขั้นตอนที่ 4] การจัดการข้อมูล WMS (Web Map Service)
        // 4.1 ข้อมูลการใช้ประโยชน์ที่ดินจาก ESA
        const landCover = L.tileLayer.wms("https://services.terrascope.be/wms/v2?", {
            layers: 'WORLDCOVER_2020_MAP', format: 'image/png', transparent: true
        });
        // 4.2 ข้อมูลภูมิประเทศโลกจาก GMRT
        const gmrt = L.tileLayer.wms("https://www.gmrt.org/services/mapserver/wms_merc?", {
            layers: 'gmrt', format: 'image/png', transparent: true
        });

        const wmsLayers = { "Land Cover (ESA)": landCover, "Topography (GMRT)": gmrt };

        // 4.3 เทคนิคการจัดการลำดับภาพ: เมื่อเปิดชั้นข้อมูลใหม่ ให้ขยับมาอยู่ด้านหน้าสุดของกลุ่มภาพ
        map.on('overlayadd', function (e) {
            if (e.layer instanceof L.TileLayer.WMS) {
                e.layer.bringToFront();
            }
        });

        // [ขั้นตอนที่ 5] การจัดการข้อมูล GeoJSON และการนำเข้าไฟล์
        // 5.1 สร้างกลุ่มว่างสำหรับเก็บชั้นข้อมูลที่ผู้ใช้นำเข้า
        const geojsonGroup = L.layerGroup().addTo(map);
        const geojsonLayer = { "Show Imported File": geojsonGroup }

        // 5.2 ฟังก์ชันสำหรับย้ายเครื่องมือควบคุมของ Leaflet ไปใส่ไว้ใน Sidebar ของเรา
        function mountControl(layers, containerId, isBase = false) {
            const ctrl = isBase ? L.control.layers(layers, null, { collapsed: false }) : L.control.layers(null, layers, { collapsed: false });
            ctrl.addTo(map);
            document.getElementById(containerId).appendChild(ctrl.getContainer());
        }

        // ติดตั้งตัวเลือก Layer ลงในช่องที่เราเตรียมไว้ใน Sidebar
        mountControl(baseMaps, 'basemap-ctrl', true);
        mountControl(wmsLayers, 'wms-ctrl');
        mountControl(geojsonLayer, 'geojson-ctrl');
        
        // 5.3 ฟังก์ชันสลับสถานะ Sidebar (เปิด/ซ่อน)
        function toggleSidebar() {
            const sidebar = document.getElementById('sidebar');
            const btn = document.getElementById('toggle-btn');
            sidebar.classList.toggle('collapsed');
            // ปรับตำแหน่งปุ่ม Toggle ตาม Sidebar
            btn.style.left = sidebar.classList.contains('collapsed') ? '10px' : '330px';
            // สั่งให้แผนที่คำนวณขนาดพื้นที่ใหม่หลังจาก Sidebar เลื่อน เพื่อป้องกันแผนที่แหว่ง
            setTimeout(() => { map.invalidateSize(); }, 300);
        }

        // 5.4 ฟังก์ชันประมวลผลการอ่านไฟล์ GeoJSON จากเครื่องผู้ใช้
        document.getElementById('fileInput').addEventListener('change', function (e) {
            const file = e.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = function (event) {
                try {
                    const data = JSON.parse(event.target.result); // แปลงข้อความในไฟล์เป็น JSON Object
                    geojsonGroup.clearLayers(); // ล้างข้อมูลเก่าออกก่อน
// สร้างชั้นข้อมูลใหม่และกำหนดให้อยู่ใน 'vectorPane' ที่เราสร้างไว้ในขั้นตอนที่ 2
                    const layer = L.geoJSON(data, {
                        style: { color: '#e67e22', weight: 3, fillOpacity: 0.4 },
                        pane: 'vectorPane'
                    }).addTo(geojsonGroup);

                    map.fitBounds(layer.getBounds()); // ขยายแผนที่ไปยังขอบเขตของข้อมูลที่นำเข้า
                } catch (err) { alert("ไฟล์ไม่ใช่ GeoJSON ที่ถูกต้อง"); }
            };
            reader.readAsText(file);
        });

        // [ขั้นตอนที่ 6] ระบบการวาดและการส่งออกข้อมูล (Geoman & Data Export)
        // 6.1 ติดตั้งเครื่องมือวาดบนมุมซ้ายบนของแผนที่
        map.pm.addControls({
            position: 'topleft',
            drawCircleMarker: false,
            rotateMode: false
        });

        // 6.2 เมื่อมีการสร้างวัตถุใหม่เสร็จสิ้น ให้ยกมาไว้หน้าสุดเสมอ
        map.on('pm:create', function (e) {
            e.layer.bringToFront();
        });

        // 6.3 ฟังก์ชันรวบรวมสิ่งที่วาดทั้งหมดแล้วแปลงเป็นไฟล์ดาวน์โหลด
        function exportGeoJSON() {
            // ดึงข้อมูล Geoman Layers ออกมาในรูปแบบมาตรฐาน GeoJSON
            const data = map.pm.getGeomanLayers(true).toGeoJSON();
            if (data.features.length === 0) return alert("กรุณาวาดรูปทรงบนแผนที่ก่อนส่งออกครับ");

            // สร้างไฟล์จำลอง (Blob) และสั่งให้บราวเซอร์ดาวน์โหลดออกไป
            const blob = new Blob([JSON.stringify(data)], { type: "application/json" });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = "my_map_data.geojson"; // ตั้งชื่อไฟล์เริ่มต้น
            a.click(); // จำลองการคลิกเพื่อดาวน์โหลด
        }

        // 7. การเพิ่มหมุด
        // ฟังก์ชันสร้างหมุด
        function createMarker(color) {

            return L.divIcon({
                className: "",
                iconSize: [30, 42],
                iconAnchor: [15, 42],
                html: `
        <svg width="30" height="42" viewBox="0 0 24 36">
          <path d="M12 0C6 0 1 5 1 11c0 8 11 24 11 24s11-16 11-24C23 5 18 0 12 0z"
                fill="${color}" stroke="black" stroke-width="1"/>
          <circle cx="12" cy="11" r="4" fill="white"/>
        </svg>
        `
            });

        }

        // 1 เพิ่ม marker
        var marker_a = L.marker([15.00000, 100.000000], {
            icon: createMarker("red")
        }).addTo(map);

        marker_a.bindPopup(
    "<b>=ชื่อสถานที่:</b> Pretchy<br>" +
    "<b>รายละเอียด:</b> พิกัด 15,100<br>" +
    "<img src='file:///C:/Users/ComEcc/Downloads/Bubbles_15.webp' width='200'>"
);

// จุดที่ 1 กรุงเทพ
var marker1 = L.marker([13.7563, 100.5018], {
    icon: createMarker("red")
}).addTo(map);

marker1.bindPopup(`
<b>ชื่อสถานที่:</b> กรุงเทพมหานคร<br>
<img src="https://static.thairath.co.th/media/dFQROr7oWzulq5Fa4MDnIPhZPsqmRTyDeuiDAt6yLwmE0DanaG3wbxTKtKTMLCKfjEb.jpg" width="220">
`);


// จุดที่ 2 เชียงใหม่
var marker2 = L.marker([18.7883, 98.9853], {
    icon: createMarker("blue")
}).addTo(map);

marker2.bindPopup(`
<b>ชื่อสถานที่:</b> เชียงใหม่<br>
<img src="https://upload.wikimedia.org/wikipedia/commons/e/ec/Near_the_Top_of_Doi_Inthanon_2014.jpg" width="220">
`);


// จุดที่ 3 ภูเก็ต
var marker3 = L.marker([7.8804, 98.3923], {
    icon: createMarker("green")
}).addTo(map);

marker3.bindPopup(`
<b>ชื่อสถานที่:</b> ภูเก็ต<br>
<img src="https://blog.bangkokair.com/wp-content/uploads/2024/04/phuket-scaled.jpeg" width="220">
`);


// จุดที่ 4 อยุธยา
var marker4 = L.marker([14.3532, 100.5689], {
    icon: createMarker("orange")
}).addTo(map);

marker4.bindPopup(`
<b>ชื่อสถานที่:</b> พระนครศรีอยุธยา<br>
<img src="https://upload.wikimedia.org/wikipedia/commons/0/0a/%E0%B8%A7%E0%B8%B1%E0%B8%94%E0%B9%84%E0%B8%8A%E0%B8%A2%E0%B8%A7%E0%B8%B1%E0%B8%92%E0%B8%99%E0%B8%B2%E0%B8%A3%E0%B8%B2%E0%B8%A1_%E0%B8%97%E0%B8%B4%E0%B8%A8%E0%B8%95%E0%B8%B0%E0%B8%A7%E0%B8%B1%E0%B8%99%E0%B8%AD%E0%B8%AD%E0%B8%81.jpg" width="220">
`);


// จุดที่ 5 ฉะเชิงเทรา (วัดโสธร)
var marker5 = L.marker([13.6884, 101.0779], {
    icon: createMarker("purple")
}).addTo(map);

marker5.bindPopup(`
<b>ชื่อสถานที่:</b> วัดโสธรวรารามวรวิหาร<br>
<b>จังหวัด:</b> ฉะเชิงเทรา<br>
<img src="https://jk-living.com/wp-content/uploads/2022/07/254556942_310892354207959_5374381316811758431_n-1024x583.jpg" width="220">
`);
    </script>
</body>

</html>
