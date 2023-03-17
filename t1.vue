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

      await this.createAllCircles1();
      // await this.createAllCircles2();

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
    async createAllCircles2() {
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
            fillOpacity: item.isJuHe ? LIGHT_OPACITY : DEEP_OPACITY,
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

    // 生成商圈
    async createAllCircles1() {
      const list = await this.fetchAllTongShi();

      let tempColorMap = new Map();
      let tempColors = [];
      let tempColor = "";
      // 生成 circleMap
      for (let i = 0; i < list.length; i++) {
        let item = list[i];

        let l2id = item.baseBizCircleL2Id;
        if (tempColors.length === 0) {
          tempColors = [...CIRCLE_COLOR_MAP];
        }
        tempColor = tempColorMap.get(l2id) || tempColors.splice(0, 1)[0];

        console.log("oooooooooo", tempColor, l2id);
        tempColorMap.set(l2id, tempColor);

        // 生成 面
        const itemBoundarys = item.children.map((item2) => {
          const itemBoundary = new BoundaryBase({
            info: item2,
            points: formatPolygonStr(item2.positionBorder),
            styleOptions: {
              strokeColor: "",
              strokeOpacity: 0,
              fillColor: tempColor,
              fillOpacity: item2.isJuHe ? LIGHT_OPACITY : DEEP_OPACITY,
            },
          });
          itemBoundary.addEventListener("click", this.onCircleClick);
          return itemBoundary;
        });

        // 生成 点
        // const itemMarker = new MarkerBase({
        //   iconColor: "red",
        //   iconSize: [14, 27],
        //   labelContent: item.baseBizCircleName,
        //   markerPoint: formatPointLngLat(item.centerPoint),
        // });

        this.circleMap.set(item.baseBizCircleL2Id, {
          circleColor: tempColor,
          // markerService: itemMarker,
          boundaryService: itemBoundarys,
        });
      }

      this.drawAllCircles();
    },

    // 画商圈
    drawAllCircles() {
      this.circleMap.forEach((value, key) => {
        this.mapService.addOverlay(value.boundaryService.getBoundary());
        this.mapService.addOverlay(value.markerService.getMarker());
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
        tempBoundary.setFillOpacity(info.isJuHe ? LIGHT_OPACITY : DEEP_OPACITY);

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
      let res2 = {
        code: "20000",
        data: [
          {
            bizcircleGmv: 3.51,
            bizcircleId: 56,
            bizcircleIdL2: 111,
            bizcircleL2Name: "rt",
            centerPoint: "POINT(116.42474276527273 39.90552310636364)",
            layoutNum: 95926,
            name: "崇文门",
            position:
              "POLYGON((116.40954 39.92173,116.44059 39.92256,116.44248 39.90678,116.449850865 39.904058142,116.449888489 39.898920146,116.425167177 39.899197028,116.425043887 39.894388854,116.40494 39.89394,116.40458 39.89831,116.41055 39.89914,116.40954 39.92173))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 67,
            bizcircleIdL2: 111,
            bizcircleL2Name: "fdsa",
            centerPoint: "POINT(114.06720725849007 22.577900698109225)",
            layoutNum: null,
            name: "范德萨",
            position:
              "POLYGON((114.0674078473529 22.578227742921687,114.0666545247348 22.577606771155626,114.06775999224539 22.577573652581922,114.0674078473529 22.578227742921687))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 68,
            bizcircleIdL2: 111,
            bizcircleL2Name: "fass",
            centerPoint: "POINT(114.06635478768912 22.579084525114098)",
            layoutNum: null,
            name: "为服务",
            position:
              "POLYGON((114.06625870271493 22.57933098490496,114.06607568371638 22.57883806455993,114.0666338916619 22.578948546861326,114.06625870271493 22.57933098490496))",
          },
          {
            bizcircleGmv: 0.023,
            bizcircleId: 69,
            bizcircleIdL2: 222,
            bizcircleL2Name: "dsa",
            centerPoint: "POINT(121.5252373416777 31.248844413681923)",
            layoutNum: 1309,
            name: "饿死位夫人",
            position:
              "POLYGON((121.52422768140832 31.24978296816161,121.52624700194725 31.249201897001324,121.5248362012862 31.247906121391892,121.5248362012862 31.247906121391892,121.52422768140832 31.24978296816161))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 86,
            bizcircleIdL2: 222,
            bizcircleL2Name: "sdfdsa",
            centerPoint: "POINT(121.52732801030498 31.2507086771839)",
            layoutNum: null,
            name: "对方v",
            position:
              "POLYGON((121.52481750937446 31.25156380781526,121.52983851123565 31.25074767555031,121.52920523172159 31.24985380822888,121.52663671294721 31.250333427208353,121.52481750937446 31.25156380781526))",
          },
          {
            bizcircleGmv: 0.551,
            bizcircleId: 90,
            bizcircleIdL2: 222,
            bizcircleL2Name: "dsaf",
            centerPoint: "POINT(120.63379127731079 31.304557765346583)",
            layoutNum: 36876,
            name: "苏州站",
            position:
              "POLYGON((120.62885651367938 31.312247223408967,120.61539212549275 31.311012599665265,120.61787584758544 31.30674741008341,120.62062101410892 31.29288419720344,120.65219042912899 31.29613971157086,120.64826876266689 31.316231579464826,120.62885651367938 31.312247223408967))",
          },
          {
            bizcircleGmv: 1.753,
            bizcircleId: 92,
            bizcircleIdL2: 333,
            bizcircleL2Name: "fdsa",
            centerPoint: "POINT(116.38456154174776 39.96561554338498)",
            layoutNum: 31879,
            name: "33333",
            position:
              "POLYGON((116.36831575705864 39.97287417319735,116.37705004990231 39.958356767747894,116.40080732643703 39.968573043684785,116.40080732643703 39.968573043684785,116.36831575705864 39.97287417319735))",
          },
          {
            bizcircleGmv: 0.033,
            bizcircleId: 101,
            bizcircleIdL2: 333,
            bizcircleL2Name: "",
            centerPoint: "POINT(104.10987811303389 30.661289077447414)",
            layoutNum: 2707,
            name: "苏州2级2222",
            position:
              "POLYGON((104.10376206947207 30.663887935595106,104.11603978806018 30.661557956203996,104.1081540988857 30.65871012657184,104.10376206947207 30.663887935595106))",
          },
          {
            bizcircleGmv: 0.062,
            bizcircleId: 129,
            bizcircleIdL2: 333,
            bizcircleL2Name: "asdf",
            centerPoint: "POINT(116.48000075202908 39.85163878845569)",
            layoutNum: 2257,
            name: "测试4.0",
            position:
              "POLYGON((116.46802728577312 39.85989373358529,116.46809047029163 39.8430428079365,116.48363386184299 39.848546603452796,116.49197421828518 39.859309378220196,116.48388659991699 39.86023460523191,116.46802728577312 39.85989373358529))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 186,
            bizcircleIdL2: 444,
            bizcircleL2Name: "cqdf",
            centerPoint: "POINT(114.06720725849007 22.577900698109225)",
            layoutNum: null,
            name: "范德萨",
            position:
              "POLYGON((114.0674078473529 22.578227742921687,114.0666545247348 22.577606771155626,114.06775999224539 22.577573652581922,114.0674078473529 22.578227742921687))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 187,
            bizcircleIdL2: 444,
            bizcircleL2Name: "fsads",
            centerPoint: "POINT(114.06635478768912 22.579084525114098)",
            layoutNum: null,
            name: "为服务",
            position:
              "POLYGON((114.06625870271493 22.57933098490496,114.06607568371638 22.57883806455993,114.0666338916619 22.578948546861326,114.06625870271493 22.57933098490496))",
          },
          {
            bizcircleGmv: 0.023,
            bizcircleId: 188,
            bizcircleIdL2: 444,
            bizcircleL2Name: "safdsad",
            centerPoint: "POINT(121.5252373416777 31.248844413681923)",
            layoutNum: 1309,
            name: "饿死位夫人",
            position:
              "POLYGON((121.52422768140832 31.24978296816161,121.52624700194725 31.249201897001324,121.5248362012862 31.247906121391892,121.5248362012862 31.247906121391892,121.52422768140832 31.24978296816161))",
          },
          {
            bizcircleGmv: 0,
            bizcircleId: 204,
            centerPoint: "POINT(121.52732801030498 31.2507086771839)",
            layoutNum: null,
            name: "对方v",
            position:
              "POLYGON((121.52481750937446 31.25156380781526,121.52983851123565 31.25074767555031,121.52920523172159 31.24985380822888,121.52663671294721 31.250333427208353,121.52481750937446 31.25156380781526))",
          },
          {
            bizcircleGmv: 0.551,
            bizcircleId: 208,
            centerPoint: "POINT(120.63379127731079 31.304557765346583)",
            layoutNum: 36876,
            name: "苏州站",
            position:
              "POLYGON((120.62885651367938 31.312247223408967,120.61539212549275 31.311012599665265,120.61787584758544 31.30674741008341,120.62062101410892 31.29288419720344,120.65219042912899 31.29613971157086,120.64826876266689 31.316231579464826,120.62885651367938 31.312247223408967))",
          },
          {
            bizcircleGmv: 1.753,
            bizcircleId: 210,
            centerPoint: "POINT(116.38456154174776 39.96561554338498)",
            layoutNum: 31880,
            name: "33333",
            position:
              "POLYGON((116.36831575705864 39.97287417319735,116.37705004990231 39.958356767747894,116.40080732643703 39.968573043684785,116.40080732643703 39.968573043684785,116.36831575705864 39.97287417319735))",
          },
          {
            bizcircleGmv: 0.033,
            bizcircleId: 228,
            centerPoint: "POINT(104.10987811303389 30.661289077447414)",
            layoutNum: 2707,
            name: "苏州2级2222",
            position:
              "POLYGON((104.10376206947207 30.663887935595106,104.11603978806018 30.661557956203996,104.1081540988857 30.65871012657184,104.10376206947207 30.663887935595106))",
          },
          {
            bizcircleGmv: 0.062,
            bizcircleId: 245,
            centerPoint: "POINT(116.48000075202908 39.85163878845569)",
            layoutNum: 2257,
            name: "测试4.0",
            position:
              "POLYGON((116.46802728577312 39.85989373358529,116.46809047029163 39.8430428079365,116.48363386184299 39.848546603452796,116.49197421828518 39.859309378220196,116.48388659991699 39.86023460523191,116.46802728577312 39.85989373358529))",
          },
        ],
        message: "success",
        success: true,
      };
      let res1 = {
        code: "20000",
        data: [
          {
            bizcircleGmv: 0.02,
            bizcircleIdL1: 111,
            bizcircleIdL2: 1,
            bizcircleL1Name: "111",
            bizcircleL2Name: "654321",
            centerPoint: "POINT(116.33444728233876 39.93214506365353)",
            layoutNum: 411,
            name: "全奉献-测试通识商圈",
            position: [
              "POLYGON((116.33115863480704 39.93379946839602,116.32956315234114 39.93149324000699,116.33933141233652 39.930490507580814,116.33115863480704 39.93379946839602))",
              "POLYGON((116.41596230084997 39.86277857543709,116.39123676268007 39.84973964606184,116.4263730537636 39.850074008776325,116.41596230084997 39.86277857543709))",
            ],
          },
          {
            bizcircleGmv: 3.042,
            bizcircleIdL1: 111,
            bizcircleIdL2: 2,
            bizcircleL1Name: "111",
            bizcircleL2Name: "asfsa",
            centerPoint: "POINT(117.19568525576278 39.13187768367064)",
            layoutNum: 100323,
            name: "lis42-2",
            position:
              "POLYGON((117.18465222682082 39.13725469162148,117.184474077487 39.129311055365626,117.18883873616583 39.12493993572692,117.19743444152316 39.12337875529183,117.24596925769842 39.12127302138104,117.20407514128894 39.134385023015575,117.19632100818671 39.140376397434366,117.18465222682082 39.13725469162148))",
              "POLYGON((121.52481750937446 31.25156380781526,121.52983851123565 31.25074767555031,121.52920523172159 31.24985380822888,121.52663671294721 31.250333427208353,121.52481750937446 31.25156380781526))",
          },
          {
            bizcircleGmv: 3.51,
            bizcircleIdL1: 222,
            bizcircleIdL2: 3,
            bizcircleL1Name: "222",
            bizcircleL2Name: "崇文门",
            centerPoint: "POINT(116.42474276527273 39.90552310636364)",
            layoutNum: 95926,
            name: "崇文门",
            position:
              "POLYGON((116.40954 39.92173,116.44059 39.92256,116.44248 39.90678,116.449850865 39.904058142,116.449888489 39.898920146,116.425167177 39.899197028,116.425043887 39.894388854,116.40494 39.89394,116.40458 39.89831,116.41055 39.89914,116.40954 39.92173))",
              "POLYGON((120.62885651367938 31.312247223408967,120.61539212549275 31.311012599665265,120.61787584758544 31.30674741008341,120.62062101410892 31.29288419720344,120.65219042912899 31.29613971157086,120.64826876266689 31.316231579464826,120.62885651367938 31.312247223408967))",
              "POLYGON((121.52422768140832 31.24978296816161,121.52624700194725 31.249201897001324,121.5248362012862 31.247906121391892,121.5248362012862 31.247906121391892,121.52422768140832 31.24978296816161))",
          },
          {
            bizcircleGmv: 0,
            bizcircleIdL1: 222,
            bizcircleIdL2: 4,
            bizcircleL1Name: "222",
            bizcircleL2Name: "范德萨",
            centerPoint: "POINT(114.06720725849007 22.577900698109225)",
            layoutNum: null,
            name: "范德萨",
            position:
              "POLYGON((114.0674078473529 22.578227742921687,114.0666545247348 22.577606771155626,114.06775999224539 22.577573652581922,114.0674078473529 22.578227742921687))",
              "POLYGON((116.36831575705864 39.97287417319735,116.37705004990231 39.958356767747894,116.40080732643703 39.968573043684785,116.40080732643703 39.968573043684785,116.36831575705864 39.97287417319735))",
              "POLYGON((114.06625870271493 22.57933098490496,114.06607568371638 22.57883806455993,114.0666338916619 22.578948546861326,114.06625870271493 22.57933098490496))",

          },
          {
            bizcircleGmv: 0,
            bizcircleIdL1: null,
            bizcircleIdL2: 5,
            bizcircleL1Name: "",
            bizcircleL2Name: "为服务",
            centerPoint: "POINT(114.06635478768912 22.579084525114098)",
            layoutNum: null,
            name: "为服务",
            position:
              "POLYGON((114.06625870271493 22.57933098490496,114.06607568371638 22.57883806455993,114.0666338916619 22.578948546861326,114.06625870271493 22.57933098490496))",
              "POLYGON((114.0674078473529 22.578227742921687,114.0666545247348 22.577606771155626,114.06775999224539 22.577573652581922,114.0674078473529 22.578227742921687))",
              "POLYGON((104.10376206947207 30.663887935595106,104.11603978806018 30.661557956203996,104.1081540988857 30.65871012657184,104.10376206947207 30.663887935595106))",
          },
          {
            bizcircleGmv: 0.023,
            bizcircleIdL1: null,
            bizcircleIdL2: 6,
            bizcircleL1Name: "",
            bizcircleL2Name: "饿死位夫人",
            centerPoint: "POINT(121.5252373416777 31.248844413681923)",
            layoutNum: 1309,
            name: "饿死位夫人",
            position:
              "POLYGON((121.52422768140832 31.24978296816161,121.52624700194725 31.249201897001324,121.5248362012862 31.247906121391892,121.5248362012862 31.247906121391892,121.52422768140832 31.24978296816161))",
              "POLYGON((114.19935248468921 30.63812331892105,114.19820265363644 30.59287158960237,114.42069496234778 30.604808185042767,114.19935248468921 30.63812331892105))",
              "POLYGON((116.46802728577312 39.85989373358529,116.46809047029163 39.8430428079365,116.48363386184299 39.848546603452796,116.49197421828518 39.859309378220196,116.48388659991699 39.86023460523191,116.46802728577312 39.85989373358529))",
              "POLYGON((121.52481750937446 31.25156380781526,121.52983851123565 31.25074767555031,121.52920523172159 31.24985380822888,121.52663671294721 31.250333427208353,121.52481750937446 31.25156380781526))",
          },
        ],
        message: "success",
        success: true,
      };

      let data = [];

      if(this.bizcircleTag == 2) {
        data = res2.data
      }

      if(this.bizcircleTag == 1) {
        data = res1.data
      }

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
