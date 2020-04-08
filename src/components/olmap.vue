<template>
  <div style="height: 100%;width:100%;">
    <div id="map"></div>
    <div style="position: absolute;top: 50px;left: 50px;">
      <el-checkbox @change="addClick" v-model="checked">新增</el-checkbox>
      <el-select v-model="value" clearable placeholder="请选择" @change="classChange">
        <el-option
          v-for="item in option1"
          :key="item.value"
          :label="item.label"
          :value="item.value"
        ></el-option>
      </el-select>徒手模式:
      <el-select v-model="value1">
        <el-option
          v-for="item in option2"
          :key="item.value"
          :label="item.label"
          :value="item.value"
        ></el-option>
      </el-select>
      <hr />
      <el-checkbox @click="updataClick" v-model="checked1">编辑</el-checkbox>
    </div>
  </div>
</template>
<script>
import { Map, View } from "ol";
import "ol/ol.css";
import Feature from "ol/Feature";
import OSM from "ol/source/OSM";
import Tile from "ol/layer/Tile";
import MultiPolygon from "ol/geom/MultiPolygon";
import { getTransform } from "ol/proj";
import vectorSource from "ol/source/Vector";
import vectorLayer from "ol/layer/Vector";
import GeoJSON from "ol/format/GeoJSON";
import Draw from "ol/interaction/Draw";
import WFS from "ol/format/WFS";
import GML from "ol/format/GML";
import Select from "ol/interaction/Select";
import Modify from "ol/interaction/Modify";
import axios from "axios";
export default {
  data() {
    return {
      option1: [
        {
          value: "1",
          label: "面"
        }
      ],
      option2: [
        {
          value: "true",
          label: "是"
        },
        {
          value: "false",
          label: "否"
        }
      ],
      value: "",
      value1: "",
      map: null,
      polygonSource: null,
      drawInteraction: null,
      checked: false,
      checked1: false
    };
  },
  mounted() {
    //数据源
    this.polygonSource = new vectorSource({
      url:
        "http://localhost:8519/geoserver/postgis/wfs? " +
          "service=WFS&version=1.1.0&" +
          "request=GetFeature&typeName=postgis:city&" +
          "outputFormat=json&srsname=EPSG:4326",
      format: new GeoJSON({
        geometryName: "geom"
      })
    });
    var polygonLayer = new vectorLayer({
      source: this.polygonSource
    });
    var view = new View({
      center: [114.065, 22.549],
      zoom: 9,
      projection: "EPSG:4326"
    });
    this.map = new Map({
      layers: [
        new Tile({
          source: new OSM()
        })
      ],
      target: "map",
      view: view
    });
    this.map.addLayer(polygonLayer);
  },
  methods: {
    //清空操作工具
    classChange: function() {
      this.map.getInteractions().clear();
    },
    //   选择器，返回绘图工具类型以及数据源
    defineType: function() {
      var geom_type;
      var geom_source;
      console.log(this.value);

      if (this.value == "1") {
        geom_type = "Polygon";
        geom_source = this.polygonSource;
        return [geom_type, geom_source];
      }
    },
    freeMode: function() {
      if (this.option2 == "是") {
        return true;
      } else if (this.option2 == "否") {
        return false;
      }
    },
    transact: function(transType, feat, layerName) {
      if (layerName == "") {
        return;
      }
      var node;
      var s;
      var formatWFS = new WFS();
      //   var formatGML = new GML({
      //     featureNS: "http://localhost/test",
      //     featurePrefix: "test",
      //     featureType: layerName,
      //     gmlOptions: { srsName: "EPSG:4326" }
      //   });
      switch (transType) {
        case "insert":
          node = formatWFS.writeTransaction([feat], null, null, {
            featureType: layerName,
            featureNS: "http://localhost/test", // 注意这个值必须为创建工作区时的命名空间URI
            srsName: "EPSG:4326"
          });
          break;
        case "update":
          node = formatWFS.writeTransaction(null, [feat], null, {
            featureType: layerName,
            featureNS: "http://localhost/test", // 注意这个值必须为创建工作区时的命名空间URI
            srsName: "EPSG:4326"
          });
          break;
      }
      s = new XMLSerializer();
      var str = s.serializeToString(node);
      $.ajax("http://localhost:8519/geoserver/test/wfs", {
        type: "POST",
        contentType: "text/xml",
        data: str
      }).done();
    },
    addClick: function() {
      //选中新增按钮
      if (this.checked == true) {
        var type = this.defineType()[0];
        var source = this.defineType()[1];
        var freeMode = this.freeMode();
        this.map.getInteractions().clear();
        this.drawInteraction = new Draw({
          source: source,
          type: type,
          geometryName: "geometry",
          freeMode: freeMode
        });
        this.map.addInteraction(this.drawInteraction);

        this.drawInteraction.on("drawend", e => {
          var layerName = "";
          var geometry = e.feature.getGeometry().clone();
          var newFeature = new Feature();
          geometry.applyTransform(function(
            flatCoordinates,
            flatCoordinates2,
            stride
          ) {
            for (var j = 0; j < flatCoordinates.length; j += stride) {
              var y = flatCoordinates[j];
              var x = flatCoordinates[j + 1];
              flatCoordinates[j] = x;
              flatCoordinates[j + 1] = y;
            }
          });
          if (this.defineType()[0] == "Polygon") {
            newFeature.setGeometryName("geom");

            newFeature.setGeometry(
              new MultiPolygon([geometry.getCoordinates()])
            );
            newFeature.set("id", 4);
            newFeature.set("name", "多边形");
            layerName = "data";
          }
          this.transact("insert", newFeature, layerName);
        });
      } else {
        this.map.removeInteraction(this.drawInteraction);
      }
    },
    updataClick: function() {}
  }
};
</script>
<style>
#map {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
}
</style>