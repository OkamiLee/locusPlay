<!--
 * @Author: liwanjin
 * @Date: 2021年12月8日08:42:35
 * @Description: 历史轨迹组件
-->
<template>
  <div ref="mapBox" id="play_map" class="qwgl-map qinwu-map-box"></div>
  <div class="play-box flex">
    <em class="close"/>
    <!--<div class="path-box">
      <div class="path-width" :style="{width:nowDuration+'%'}"></div>
    </div>-->
    <div class="left-box flex">
      <div class="slider-box">
        <el-slider :show-tooltip="false" @input="sliderChange" v-model="nowDuration"></el-slider>
      </div>
      <div class="play-icon" @click="play">
        <img :src="playIcon" v-if="!playState" alt=""/>
        <img :src="pauseIcon" v-else alt=""/>
      </div>
      <div class="reset-icon" @click="reset">
        <img :src="resetIcon" alt=""/>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>

  let svgXML =
    `<svg viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
                <path d="M529.6128 512L239.9232 222.4128 384.7168 77.5168 819.2 512 384.7168 946.4832 239.9232 801.5872z" p-id="9085" fill="#ffffff"></path>
            </svg>
            `
  let svgBase64 = 'data:image/svg+xml;base64,' + window.btoa(unescape(encodeURIComponent(svgXML)));

  import {loadGisIcons} from "@/components/map/utils";
  import playIcon from "../assets/play.png"
  import pauseIcon from "../assets/pause.png"
  import resetIcon from "../assets/reset.png"
  import "../assets/minemap.css";
  import "../assets/minemap-edit.css";
  import {drawBorderLine, addMarker, addArrowMarker} from "@/common/qwgl/mapUtils";
  import minemap from "../assets/minemap";
  import * as turf from "@turf/turf";
  import {
    inject,
    onMounted,
    onBeforeUnmount,
    reactive,
    ref,
    computed,
    watch,
  } from "vue";
  import axios, {AxiosInstance} from "axios"
  import mapboxgl, {LngLat} from "mapbox-gl";
  import {useContext} from "../hooks/context";
  import {useLocalStorage, useScriptTag} from "@vueuse/core";
  type HlkMap = minemap.Map | mapboxgl.Map;
  const {url, emitter} = useContext();
  const http = inject("http") as AxiosInstance
  const mapCenter = useLocalStorage<mapboxgl.LngLat>(
    "mapCenter",
    LngLat.convert(url.mapCenter)
  );
  const mapZoom = useLocalStorage<number>("mapZoom", 10);
  const mapPitch = useLocalStorage<number>("mapPitch", 45);
  useScriptTag(
    `${url.minemap.domainUrl}/minemapapi/v2.1.0/plugins/edit/minemap-edit.js`
  );
  let map: HlkMap | any,
    editCtrl: any,
    steps = ref<number>(1000),//播放速度
    playState = ref<boolean>(false),
    nowDuration = ref<number>(0),
    timer = ref<any>(null),
    coordinates = ref<[number,number][]>([[126.75070740835002, 45.711216850475154], [126.75504615855914, 45.72228460189348], [126.74727175282004, 45.72571908205987], [126.74404943370246, 45.73192768071425]]),
    stepsPath = ref<any>([]),
    counter = ref<number>(0);
  onMounted(() => {
    init()
    //initMap()
  })
  function init() {
    http
      .post(`${url.api}/tGpsCar/selectCarGpsData`, {
        "date": "2021-12-07",
        "code": "警A8001警",
        "minute": "5"
      })
      .then(function (res) {
        coordinates.value = res.data.data.coordinates.splice(0, 5);
        steps.value = res.data.data.coordinates.splice(0, 5).length * 100
        initMap()
      })
  }
  //初始化加载地图
  function initMap() {
    minemap.domainUrl = url.minemap.domainUrl;
    minemap.dataDomainUrl = url.minemap.dataDomainUrl;
    minemap.serverDomainUrl = url.minemap.serverDomainUrl;
    minemap.spriteUrl = url.minemap.spriteUrl;
    minemap.serviceUrl = url.minemap.serviceUrl;
    minemap.key = url.minemap.key;
    map = new minemap.Map({
      container: "play_map",
      style: `${minemap.serviceUrl}/solu/style/id/12613`,
      center: mapCenter.value,
      zoom: mapZoom.value,
      pitch: mapPitch.value,
      maxZoom: 17,
      minZoom: 5,
      logoControl: false,
    });
    map.addControl(new minemap.Navigation(), "bottom-right");
    map.addControl(new minemap.Scale(), "bottom-left");
    map.on("load", () => {
      loadGisIcons(map)
      map.flyTo({
        center: coordinates.value[0],
        zoom: 14,
      });
      let arrowIcon = new Image(20, 20)
      arrowIcon.src = svgBase64
      arrowIcon.onload = function () {
        map.addImage('arrowIcon', arrowIcon)
        drawPath()
        addArrowMarker(map, "arrowLayer", {
          'symbol-placement': 'line',
          'symbol-spacing': 10, // 图标间隔，默认为250
          'icon-image': 'arrowIcon', //箭头图标
          'icon-size': 0.5
        }, 'LineString', [])
      }
    });
  }

  //绘制轨迹路径
  function drawPath() {
    let route: any = {
      "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": coordinates.value
      }
    }
    let lineDistance: any = turf.lineDistance(route);
    for (let i = 0; i < lineDistance; i += lineDistance / steps.value) {
      let segment = turf.along(route, i);
      stepsPath.value.push(segment.geometry.coordinates);
    }
    coordinates.value = stepsPath.value;
    drawBorderLine(map, {
      id: 'route',
      lineColor: "#b1c7ff",
      lineWidth: 10,
      borderColor: '#1955e6',
      borderWidth: 1
    }, coordinates.value)

    drawBorderLine(map, {
      id: 'nowPath',
      lineColor: "#4478ff",
      lineWidth: 10,
      borderColor: '#1955e6',
      borderWidth: 1
    }, [])

    addArrowMarker(map, "point", {
      "icon-image": "policeCar",
      "icon-rotate": ["get", "bearing"],
      "icon-rotation-alignment": "viewport",//map
      "icon-allow-overlap": true,
      "icon-ignore-placement": true,
      "icon-offset": [0, -20]
    }, "Point", coordinates.value[0])
    //设置起点和终点
    setStartPoint(stepsPath.value[0], stepsPath.value[stepsPath.value.length - 1]);
  }

  //设置起点和终点
  function setStartPoint(startLngLat: [number, number], endLngLat: [number, number]) {
    /*addMarker(map,{
      id:'startIcon',
      type:'startIcon',
      coordinates:startLngLat,
    })
    addMarker(map,{
      id:'endIcon',
      type:'endIcon',
      coordinates:endLngLat,
    })*/
    let start = document.createElement('div');
    start.className = 'startIcon';
    start.innerHTML = `起`
    let end = document.createElement('div');
    end.className = 'endIcon';
    end.innerHTML = `终`
    let _marker = new minemap.Marker(start, {offset: [-10, -10]}).setLngLat(startLngLat).addTo(map);
    let _marker2 = new minemap.Marker(end, {offset: [-15, -30]}).setLngLat(endLngLat).addTo(map);
  }

  //轨迹播放暂停
  function play() {
    playState.value = !playState.value;
    animate();
  }

  //轨迹清除事件
  function reset() {
    cancelAnimationFrame(timer.value)
    playState.value = true;
    counter.value = 0;
    animate();
  }

  /**
   * @desc 滑块拖动事件
   * @params tmp 当前进度保费比
   * */
  function sliderChange(tmp: number) {
    counter.value = Math.ceil(tmp / 100 * steps.value);
    if (!playState.value) {
      play()
    }
  }

  /**
   * @desc 构建geoJson结构并返回
   * @params data.type geoJson类型
   * @params data.coordinates geoJson坐标
   * @params data.bearing 旋转角度参数，用于实时轨迹中点位方向
   * @return geoJson结构
   * */
  function buildGeoJson(data: { type: string, coordinates: any, bearing?: number }) {
    return {
      "type": "FeatureCollection",
      "features": [{
        "type": "Feature",
        "geometry": {
          "type": data.type,
          "coordinates": data.coordinates
        },
        "properties": {
          "bearing": data.bearing || 0
        }
      }]
    }
  }

  //轨迹播放动画
  function animate() {
    let path = stepsPath.value;
    //设置点位轨迹
    let point: any = buildGeoJson({
      type: 'Point',
      coordinates: path[counter.value],
      bearing: turf.bearing(
        turf.point(path[counter.value >= steps.value ? counter.value - 1 : counter.value]),
        turf.point(path[counter.value >= steps.value ? counter.value : counter.value + 1])
      )
    });
    if (!playState.value || (counter.value === steps.value - 1)) {
      if(counter.value === steps.value - 1){
        playState.value = false;
        counter.value = 0;
        point.features[0].geometry.coordinates = path[counter.value];
        map.getSource('point').setData(point);
      }
      return
    }
    //设置滑块值
    nowDuration.value = counter.value / steps.value * 100;
    //设置实时轨迹
    let nowPath: any = buildGeoJson({
      type: 'LineString',
      coordinates: path.filter((v: any, i: number) => i <= counter.value),
    })
    //实时更新点位坐标 和实时轨迹路径
    map.getSource('point').setData(point);
    map.getSource('nowPath').setData(nowPath);
    map.getSource('arrowLayer').setData(nowPath);
    if (counter.value < steps.value) {
      timer.value = requestAnimationFrame(animate);
    }
    counter.value += 1;
  }
</script>

<style scoped lang="scss">
  .flex {
    display: flex;
    align-items: center;
  }

  .qwgl-map {
    position: relative;
    width: 100%;
    height: 100%;
    user-select: none;
  }

  .play-icon, .reset-icon {
    cursor: pointer;
    width: 18px;
    height: 18px;
    margin-right: 15px;

    img {
      width: 100%;
      height: 100%;
    }
  }

  .play-box {
    position: fixed;
    bottom: 50px;
    left: 50%;
    margin-left: -360px;
    padding: 24px 30px;
    box-sizing: border-box;
    width: 720px;
    height: 120px;
    background: rgba(10, 27, 115, 0.8);
    border: 1px solid #2A70E9;
    border-radius: 20px;

    .close {
      position: absolute;
      right: 0;
      top: 0;
      width: 36px;
      height: 36px;
      background-image: url(".././assets/qwgl/mask_close.png");
      cursor: pointer;
    }

    .path-box {
      width: 576px;
      height: 24px;
      background: rgba(42, 114, 245, .5);
      border-radius: 5px;
      margin-bottom: 15px;

      .path-width {
        height: 24px;
        background: rgba(42, 114, 245, 1);
        border-radius: 5px 0px 0px 5px;
      }
    }

    .slider-box {
      width: 576px;
      margin-right: 30px;
    }
  }

</style>
