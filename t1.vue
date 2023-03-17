<template>
  <div class="new-battle-page">
    <div class="title-content">
      <div class="title-text" @click="onBack">
        <i class="el-icon-arrow-left"></i>
        <span> {{ addStatus ? "选择边界" : "修改边界" }} </span>
      </div>
      <el-button size="mini" type="primary" @click="onCalculate">
        计算
      </el-button>
    </div>

    <div id="battleMapNew_el" class="map-content"></div>

    <!-- 右侧弹起商圈详情 -->
    <BattleDetail
      :dialogVisible.sync="isShowBattleDetailDialog"
      :detailData="detailData"
    >
      <template slot="btn">
        <el-button size="mini" @click="onReEditBattle">编辑</el-button>
        <el-button size="mini" type="primary" @click="onCompleteBattle">
          完成
        </el-button>
      </template>
    </BattleDetail>
  </div>
</template>

<script>
import { mapActions, mapGetters } from "vuex";
// utils
import banana from "@/plugins/banana";
import {
  LIGHT_OPACITY,
  DEEP_OPACITY,
  DEFAULT_COLOR,
  CIRCLE_COLOR_MAP,
} from "@/constants/battleMap/battleCircleColor";
import { formatPolygonStr, formatPointLngLat } from "@/utils/battleMap";
// class
import MapService from "@/classService/battleDivide/MapService";
import BoundaryBase from "@/classService/battleDivide/BoundaryBase";
import MarkerBase from "@/classService/battleDivide/MarkerBase";
// components
import BattleDetail from "@/views/battleDivide/components/BattleDetail";
// api
import {
  fetchBizCircleList,
  battleCircleCal,
  saveBaseCircle,
} from "@/api/battleMap";

// decorators
import { saTrack, saTrackTraceId } from "@/decorators/sa";
import { blankDecorator } from "@/decorators/battleDivide";
import { trackTraceId } from "@/utils/shence";
import { CITY_CENTER_POINT } from "@/constants/battleMap";

export default {
  name: "NewLevel12",
  components: {
    BattleDetail,
  },
  computed: {
    ...mapGetters(["cityCode"]),
  },
  watch: {
    async cityCode(newv, oldv) {
      if (!newv) return;
      if (newv === oldv) return;

      await this.setMapCenter(newv);

      await this.createAllCircles();

      await this.initEditStatus();
    },
  },
  props: {},
  data() {
    return {
      addStatus: true, // 页面状态：新增 or 编辑
      isShowBattleDetailDialog: false, // 展示详情弹窗

      mapService: null, // 地图

      // 新增 or 编辑的商圈基本信息
      circleId: this.$route.query.circleId,
      circleName: this.$route.query.baseBizCircleName,
      bizcircleTag: this.$route.query.bizcircleTag,
      description: this.$route.query.description,

      circleMap: new Map(),

      selectedCircleArr: [],

      // 新增 or 编辑 数据
      centerOverlay: null, // 中心点overlay
      battleOverlay: null, // 商圈overlay

      detailData: null,
    };
  },

  async mounted() {
    this.initPageStatus();
    this.initMap();
  },

  methods: {
    // 初始化编辑数据，编辑的时候调用
    @blankDecorator(function () {
      this?.centerOverlay?.addEventListener("mouseup", (e) => {
        trackTraceId(
          "battleMap_modifyBattlefield",
          "zcity_battlefield_modify_center_point"
        );
      });
    })
    initEditStatus() {
      if (!this.circleId) return;
      let editOverlay = this.circleMap.get(this.circleId + "");
      this.battleOverlay = editOverlay.boundary;
      this.battleOverlay.enableEditing();

      this.centerOverlay = editOverlay.marker.getMarker(); // 使用customLabelMarker 内置了Maerker属性
      this.centerOverlay.enableDragging();
    },

    // 点击详情弹窗的编辑按钮
    @saTrackTraceId(
      "battleMap_modifyBattlefield",
      "zcity_battlefield_modify_re-edit"
    )
    onReEditBattle() {
      this.isShowBattleDetailDialog = false;
      this.initEditStatus();
    },

    // 点击绘制边界的返回
    @saTrackTraceId(
      "battleMap_modifyBattlefield",
      "zcity_battlefield_cancle_click"
    )
    onBack() {
      this.$router.back();
    },

    // 点击弹窗的完成
    @saTrackTraceId(
      "battleMap_modifyBattlefield",
      "zcity_battlefield_modify_completed"
    )
    onCompleteBattle() {
      this.isShowBattleDetailDialog = false;
      // TODO 跳到首页
    },

    _clearData() {
      const overlaylay = this.centerOverlay;
      if (overlaylay) {
        overlaylay.disableDragging();
        overlaylay.removeFromMap(this.mapService);
      }

      const battleOverlay = this.battleOverlay;
      if (this.battleOverlay) {
        this.battleOverlay.disableEditing();
        this.battleOverlay.removeEventListener("click");
        this.mapService.removeOverlay(this.battleOverlay);
      }
    },

    // ---------------------
    initPageStatus() {
      this.addStatus = this.$route.query.circleId == undefined;
    },

    initMap() {
      this.mapService = new MapService("battleMapNew_el");
    },

    setMapCenter(cityCode) {
      // 找到城市中心点
      const centerCity = CITY_CENTER_POINT.find((i) => i.cityCode == cityCode);
      const centerPoint = (centerCity && centerCity.point.split(",")) || [
        116.404, 39.915,
      ];

      this.mapService.setCenter(centerPoint);
    },

    // 生成商圈
    async createAllCircles() {
      const list = await this.fetchAllTongShi();

      // 生成 circleMap
      for (let i = 0; i < list.length; i++) {
        let item = list[i];

        let circleColor = CIRCLE_COLOR_MAP[0];

        // 生成 面
        const itemBoundary = new BoundaryBase({
          info: item,
          points: formatPolygonStr(item.positionBorder),
          styleOptions: {
            strokeColor: "",
            strokeOpacity: 0,
            fillColor: circleColor,
            fillOpacity: item.isJuHe ? DEEP_OPACITY : LIGHT_OPACITY,
          },
        });
        itemBoundary.addEventListener("click", this.onCircleClick);

        // 生成 点
        const itemMarker = new MarkerBase({
          iconColor: "red",
          iconSize: [14, 27],
          labelContent: item.baseBizCircleName,
          markerPoint: formatPointLngLat(item.centerPoint),
        });

        this.circleMap.set(item.baseBizCircleId, {
          circleColor,
          markerService: itemMarker,
          boundaryService: itemBoundary,
        });
      }

      this.drawAllCircles();
    },

    // 画商圈
    drawAllCircles() {
      this.circleMap.forEach((value, key) => {
        this.mapService.addOverlay(value.boundaryService.getBoundary());
        // this.mapService.addOverlay(value.markerService.getMarker());
      });
    },

    onCircleClick(info) {
      const tempCircle = this.circleMap.get(info.baseBizCircleId);
      const tempBoundary = tempCircle.boundaryService.getBoundary();
      const tempColor = tempBoundary.getFillColor();
      if (tempColor === tempCircle.circleColor) {
        // 点选
        tempBoundary.setFillColor(DEFAULT_COLOR);
        tempBoundary.setFillOpacity(DEEP_OPACITY);

        this.selectedCircleArr.push(info);
      } else {
        // 取消点选
        tempBoundary.setFillColor(tempCircle.circleColor);
        tempBoundary.setFillOpacity(info.isJuHe ? DEEP_OPACITY : LIGHT_OPACITY);

        for (let i = 0; i < this.selectedCircleArr.length; i++) {
          const item = this.selectedCircleArr[i];
          if (item.baseBizCircleId === info.baseBizCircleId) {
            this.selectedCircleArr.splice(i, 1);
            break;
          }
        }
      }

      this.onDrawCenter(
        this.selectedCircleArr.map((item) => {
          const ppp = formatPointLngLat(item.centerPoint);

          return new BMapGL.Point(ppp.lng, ppp.lat);
        })
      );
    },

    // 生成商圈中心点
    onDrawCenter(points) {
      this.mapService.removeOverlay(this.centerOverlay);
      this.centerOverlay = null;

      if (points.length === 0) return;

      let markerService = null;
      if (points.length === 1) {
        markerService = new MarkerBase({
          iconColor: "red",
          iconSize: [14, 27],
          labelContent: this.circleName,
          markerPoint: points[0],
        });
      }

      if (points.length > 1) {
        const dddd = new BoundaryBase({
          points,
          styleOptions: {
            strokeColor: "black",
            strokeOpacity: 0,
            fillColor: "black",
            fillOpacity: 0,
          },
        });
        markerService = new MarkerBase({
          iconColor: "red",
          iconSize: [14, 27],
          labelContent: this.circleName,
          markerPoint: dddd.getBoundary().getBounds().getCenter(),
        });
      }

      this.centerOverlay = markerService.getMarker();
      this.centerOverlay.enableDragging();
      this.mapService.addOverlay(this.centerOverlay);
    },

    // 计算
    @saTrackTraceId(
      "battleMap_modifyBattlefield",
      "zcity_battlefield_modify_calc"
    )
    onCalculate() {
      if (!this.battleOverlay) return;

      // 转换商圈坐标为字符串
      let polStr = "";
      this.battlePoints.forEach((item) => {
        polStr += `${item.lng},${item.lat};`;
      });
      polStr = polStr.substring(0, polStr.length - 1);

      // 获取商圈中心点坐标
      const point = this.centerOverlay.getPosition();

      if (!this.area) return;

      this.loading = this.$loading({
        lock: true,
        text: "请求中",
        spinner: "el-icon-loading",
        background: "rgba(0, 0, 0, 0.7)",
      });
      // 调用接口
      battleCircleCal({
        id: this.circleId || "",
        polStr,
        area: this.area,
        circleLevel: this.bizcircleTag,
        centerLng: point.lng + "",
        centerLat: point.lat + "",
      })
        .then((res) => {
          if (+res.code !== 20000) {
            this.loading.close();
            return;
          }
          this.detailData = res.data;

          this.isShowBattleDetailDialog = true;

          this.circleId = res.data.id;
          this.circleMap.set(res.data.id, {
            color: "blue",
            boundary: this.battleOverlay,
            marker: this.centerOverlay,
          });

          this.onAddBattle(this.detailData, point, polStr);
        })
        .catch((err) => {
          this.loading.close();
        });
    },

    // 保存商圈
    onAddBattle(calRes, point, polStr) {
      const userInfo = banana.getUserinfo();
      const user = {
        empCode: userInfo.uid,
        realName: userInfo.nickname,
        role: userInfo.roles,
      };
      let level = this.bizcircleTag;

      const param = {
        level,
        user,
        baseCircleGeneralData: {
          cityCode: this.cityCode,
          centerLng: point.lng + "",
          centerLat: point.lat + "",
          positionBorder: polStr,
          bizcircleArea: this.area,
          name: this.circleName,
          focusStatus: this.focusStatus,
          description: this.description,
          ...calRes,
        },
      };
      saveBaseCircle(param)
        .then((resSave) => {
          this.loading.close();
          if (+resSave.code !== 20000) return;
        })
        .catch(() => {
          this.loading.close();
        });
    },

    // 获取所有通识商圈
    async fetchAllTongShi() {
      let { data } = await fetchBizCircleList(
        {
          cityCode: this.cityCode,
          pageSize: 998,
          pageNum: 1,
          status: 1,
          optionModule: "bizCircle_k1",
          bizcircleTag: "3",
        },
        3
      );

      data.list = data.list.map((item) => {
        item.baseBizCircleL2Id = "";
        if (item.baseBizCircleName.includes("聚合2级98")) {
          item.baseBizCircleL2Id = 98;
        }
        if (item.baseBizCircleName.includes("聚合2级56")) {
          item.baseBizCircleL2Id = 56;
        }

        if (this.bizcircleTag == 2) {
          item.isJuHe = item.baseBizCircleL2Id ? true : false;
        }
        return item;
      });

      console.log("77777", data.list);

      return data.list || [];
    },
  },
};
</script>

<style lang="scss" scoped>
.new-battle-page {
  position: relative;

  .title-content {
    width: 100%;
    height: 44px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: #ffffff;
    padding: 0 17px;
    box-sizing: border-box;
    .title-text {
      font-family: PingFangSC-Regular;
      font-size: 14px;
      font-weight: normal;
    }
  }

  .draw-panel {
    z-index: 999;
    position: absolute;
    top: 64px;
    left: 20px;

    height: 33px;

    display: flex;
    flex-direction: row;
    align-items: center;

    cursor: pointer;
    font-size: 14px;
    font-weight: normal;
    font-family: PingFangSC-Regular;
    color: rgba(0, 0, 0, 0.86);

    .draw-btn {
      border-radius: 4px;
      box-shadow: 0px 2px 6px 0px rgba(0, 0, 0, 0.08);
      background-color: #fff;
      padding: 0 10px;
      height: 33px;
      line-height: 33px;
    }

    .draw-tip {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      align-items: center;
      width: 246px;
      height: 100%;
      border-radius: 4px;
      background: #ffeeee;
      box-sizing: border-box;
      border: 1px solid #ffbcbc;
      padding: 0 10px;
      margin-left: 14px;
      color: #3d3d3d;
    }
  }

  .map-content {
    overflow: hidden;
    width: 100%;
    height: 840px;
    margin: 0;
    font-family: "微软雅黑";
  }
}
</style>
