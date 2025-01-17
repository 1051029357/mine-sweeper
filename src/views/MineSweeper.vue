<template>
  <div class="minesweeper row">
    <div class="exit-tips" v-if="store.set.fullScreen" @click="exitFullScreen">
      退出全屏[Esc]
    </div>
    <div class="left-menu col md-3 card">
      <Menu v-model:mode="state.mode"></Menu>
    </div>
    <div class="right-content md-9">
      <div class="content-wrap">
        <div class="header-box panel">
          <Timer ref="timerRef"></Timer>
          <div>
            剩余雷数：X{{ state.mineCount - state.flagNum
            }}<img class="mineImg" src="../assets/images/mine.jpeg" />
          </div>
        </div>
        <div class="game-box panel" :style="gameBoxStyle">
          <template v-for="(row, index) in state.mineArr" :key="index">
            <template v-for="(el, index2) in row" :key="index2">
              <div
                class="box"
                :class="[`box${el.num}`]"
                @click="openFlag(index, index2)"
                v-if="el.open"
              >
                <img v-if="el.num === 10" src="../assets/images/mine.jpeg" />
                <span v-else> {{ el.num ? el.num : "" }}</span>
              </div>
              <div
                class="box unopen"
                :class="{ light: el.light && !el.flag }"
                v-else
                @click="open(index, index2)"
                @contextmenu.prevent="setFlag(index, index2)"
              >
                <img v-if="el.flag" src="../assets/images/flag.png" />
              </div>
            </template>
          </template>
        </div>
        <button style="width: 100%" class="btn-warning" @click="restart">
          重新开始
        </button>
      </div>
    </div>

    <Modal
      v-model:visible="state.successModalVisible"
      title="恭喜您"
      content="游戏成功！"
      @confirm="restart"
    ></Modal>
    <Modal
      v-model:visible="state.failModalVisible"
      title="很遗憾"
      content="游戏失败~请再接再厉"
      @confirm="restart"
    ></Modal>
  </div>
</template>

<script lang="ts" setup>
import { reactive, ref, watch, computed } from "vue";
import Modal from "@/components/ModalComp.vue";
import Timer from "@/components/TimerComp.vue";
import Menu from "@/components/MenuComp.vue";
import { useMainStore } from "../store/main";
import { InitData, posType } from "../types/main";
import { primary, intermediate, senior } from "../enums/gradeEnum";
import { getRandom } from "../utils";

const store = useMainStore();
const state: InitData = reactive({
  mineCount: 10, //雷数
  colCount: 8, //列数
  rowCount: 8, //行数
  mineArr: [[]], //存放雷
  flagNum: 0, //标记旗数
  successModalVisible: false,
  failModalVisible: false,
  hasStart: false, //标记游戏是否已开始
  mode: 1, //难度 1:初级 2：中级 3：高级
});

document.addEventListener("keydown", function ({ key }) {
  console.log(key);
  if (key === "Escape") {
    exitFullScreen();
  }
});

watch(
  () => state.hasStart,
  (newV, oldV) => {
    if (newV) {
      //开始游戏
      timerRef.value && timerRef.value.start();
    } else {
      //结束游戏
      timerRef.value && timerRef.value.stop();
    }
    console.log("监听", newV, oldV);
  }
);

//设置雷数和界面大小
function setConfig(type: number) {
  let typeMap: {
    [keyName: number]: {
      MINE_COUNT: number;
      COL_COUNT: number;
      ROW_COUNT: number;
    };
  } = {
    1: primary,
    2: intermediate,
    3: senior,
  };

  const { MINE_COUNT, COL_COUNT, ROW_COUNT } = typeMap[type];

  state.mineCount = MINE_COUNT;
  state.colCount = COL_COUNT;
  state.rowCount = ROW_COUNT;
}

watch(
  () => state.mode,
  (newV) => {
    setConfig(newV);
    restart();
    store.addAlert({
      type: "secondary",
      content: `难度-${newV == 1 ? "初级" : newV == 2 ? "中级" : "高级"}`,
    });
  }
);

const timerRef = ref<InstanceType<typeof Timer>>();

//获取在坐标面板内部的坐标
const getPosition = (x: number, y: number): posType[] => {
  const aroundPos: posType[] = [
    [x - 1, y - 1],
    [x - 1, y],
    [x - 1, y + 1],
    [x, y - 1],
    [x, y + 1],
    [x + 1, y - 1],
    [x + 1, y],
    [x + 1, y + 1],
  ];
  return aroundPos.filter((arr) => {
    return (
      arr[0] >= 0 &&
      arr[0] < state.rowCount &&
      arr[1] >= 0 &&
      arr[1] < state.colCount
    );
  });
};

//初始化雷区
const initMine = () => {
  state.mineArr = [[]];
  for (let i = 0; i < state.rowCount; i++) {
    state.mineArr[i] = [];
    for (let j = 0; j < state.colCount; j++) {
      state.mineArr[i][j] = {
        num: 0, //周围雷数
        open: false, //是否已打开
        hasLoop: false, //是否已递归
        flag: false, //是否已标雷
        light: false, //是否高亮
      };
    }
  }

  for (let i = 0; i < state.mineCount; i++) {
    let [row, col] = getRandom(state.rowCount, state.colCount);
    //如果该位置有雷，则重新获取随机坐标
    while (state.mineArr[row][col].num === 10) {
      [row, col] = getRandom(state.rowCount, state.colCount);
    }
    state.mineArr[row][col].num = 10;
    //设置雷之后，把周围一圈非雷方块数字+1

    const pos = getPosition(row, col);

    pos.forEach((item) => {
      const [x, y] = item;
      if (state.mineArr[x][y].num !== 10) {
        state.mineArr[x][y].num++;
      }
    });
  }
  console.log(state.mineArr);
  state.flagNum = 0;
};

//点击方块
const open = (row: number, col: number) => {
  if (!state.hasStart) {
    state.hasStart = true;
  }
  //如果点击的方块已插旗，则不触发
  if (state.mineArr[row][col].flag) {
    return false;
  }
  //点击的方块设置为打开
  state.mineArr[row][col].open = true;
  //点击的为雷
  if (state.mineArr[row][col].num === 10) {
    state.failModalVisible = true;
    //暂停计时器
    timerRef.value && timerRef.value.pause();
    return false;
  } else {
    const pos = getPosition(row, col);
    console.log(pos);
    //遍历坐标，如果发现有一个值为10则停止递归
    loop(pos);
  }
  isFinish();
};

//判断是否游戏结束  成功or失败
const isFinish = () => {
  //判断还有几个方块未打开
  let count = 0;
  for (let x = 0; x < state.mineArr.length; x++) {
    for (let y = 0; y < state.mineArr[x].length; y++) {
      if (!state.mineArr[x][y].open) {
        count = count + 1;
      }
      //只要有一个方块open和雷数10同时出现  则失败
      if (state.mineArr[x][y].open && state.mineArr[x][y].num === 10) {
        state.failModalVisible = true;
        //暂停计时器
        timerRef.value && timerRef.value.pause();
        return false;
      }
    }
  }
  //未打开的方块数<=总雷数 则成功
  if (count <= state.mineCount) {
    state.successModalVisible = true;
    //暂停计时器
    timerRef.value && timerRef.value.pause();
    let map: { [keyName: string]: string } = {
      1: "cj",
      2: "zj",
      3: "gj",
    };
    let record: { [keyName: string]: number } = {};
    if (timerRef.value) {
      record[map[state.mode]] = timerRef.value.allSecond();
    }
    store.refreshRecord(record);
    return false;
  }
};

//标记旗
const setFlag = (x: number, y: number) => {
  if (!state.hasStart) {
    state.hasStart = true;
  }
  if (!state.mineArr[x][y].flag) {
    state.mineArr[x][y].flag = true;
    state.flagNum++;
  } else {
    state.mineArr[x][y].flag = false;
    state.flagNum--;
  }
};

//使用递归打开方块
const loop = (pos: Array<Array<number>>) => {
  //周围方块是否存在雷
  const bool = pos.every((item) => {
    const [x, y] = item;
    return state.mineArr[x][y].num !== 10;
  });

  if (bool) {
    pos.forEach((item) => {
      const [x, y] = item;
      if (!state.mineArr[x][y].hasLoop) {
        state.mineArr[x][y].open = true;
        state.mineArr[x][y].hasLoop = true;
        const _pos = getPosition(x, y);
        loop(_pos);
      }
    });
  } else {
    return false;
  }
};

//重新开始
const restart = () => {
  initMine();
  state.hasStart = false;
  store.addAlert({ type: "success", content: "游戏已重新开始" });
};

//当鼠标在已打开方块上按下时，周围未打开方块显示效果
//const showAround = (x, y) => {
//  const pos = getPosition(x, y);
//  pos.forEach((item) => {
//    const [x, y] = item;
//    if (!state.mineArr[x][y].open) {
//      state.mineArr[x][y].light = true;
//    }
//  });
//};
////当鼠标在已打开方块上按下时，周围未打开方块显示效果
//const unShowAround = (x, y) => {
//  const pos = getPosition(x, y);
//  pos.forEach((item) => {
//    const [x, y] = item;
//    if (!state.mineArr[x][y].open) {
//      state.mineArr[x][y].light = false;
//    }
//  });
//};

//已标雷的情况下，点击未打开方块时，如果数字等于周围一圈标雷数量，则自动打开周围一圈未标雷的方块
const openFlag = (_x: number, _y: number) => {
  const pos = getPosition(_x, _y);
  let flagCount = 0;
  //统计周围一圈标了几个旗
  pos.forEach((item) => {
    const [x, y] = item;
    if (state.mineArr[x][y].flag) {
      flagCount = flagCount + 1;
    }
  });
  //如果点击的数字等于周围旗数
  if (state.mineArr[_x][_y].num === flagCount) {
    //遍历周围一圈，把未插旗的方块打开
    pos.forEach((item) => {
      const [x, y] = item;
      if (!state.mineArr[x][y].flag) {
        state.mineArr[x][y].open = true;
        //打开一个方块就判断一次
        return isFinish();
      }
    });

    //去除已插旗的方块
    pos.forEach((item, index) => {
      if (state.mineArr[item[0]][item[1]].flag) {
        pos.splice(index, 1);
      }
    });
    console.log(pos);
    //递归点击的方块
    loop(pos);
  }

  isFinish();
};

//游戏区的动态样式
const gameBoxStyle = computed(() => {
  let gridStyle = {
    gridTemplateColumns: `repeat(${state.colCount}, ${store.set.size}px)`,
    gridTemplateRows: `repeat(${state.rowCount}, ${store.set.size}px)`,
  };
  let fullScreenStyle = {
    position: "fixed",
    top: 0,
    left: 0,
    width: "100vw",
    height: "100vh",
    paddingTop: "70px",
  };
  if (store.set.fullScreen) {
    return Object.assign({}, gridStyle, fullScreenStyle);
  } else {
    return gridStyle;
  }
});

//退出全屏
const exitFullScreen = () => {
  store.set.fullScreen = false;
};

initMine();
</script>
<style scoped lang="scss">
.minesweeper {
  padding: 0 20px;
  align-items: flex-start;
  justify-content: center;
  .exit-tips {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    background-color: rgb(59, 56, 56);
    cursor: pointer;
    transition: all 0.3s ease;
    color: #fff;
    z-index: 2;
    padding: 8px 10px;
    font-size: 12px;
    &:hover {
      opacity: 0.8;
    }
  }
  .left-menu {
    background-color: #fff;
  }
  .right-content {
    .header-box {
      display: flex;
      justify-content: space-between;
      .mineImg {
        border: none;
        width: 20px;
        height: 20px;
        display: inline-block;
      }
    }
  }
  @media screen and (min-width: 992px) {
    .content-wrap {
      margin: 0 120px;
    }
  }

  .game-box {
    display: grid;
    justify-content: center;
    .box {
      display: flex;
      align-items: center;
      justify-content: center;
      background-color: rgb(197, 194, 194);
      border: 1px solid rgb(255, 255, 255);
      cursor: pointer;
      font-size: 20px;
      font-weight: 600;
      transition: all 0.2s ease;
      img {
        border-radius: 0;
        border: none;
      }
      &10 {
        background-color: #f00;
      }
      &1 {
        color: blue;
      }
      &2 {
        color: green;
      }
      &3 {
        color: red;
      }
      &4 {
        color: rgb(102, 4, 167);
      }
      &5 {
        color: orange;
      }
      &6 {
        color: #555;
      }
      &7 {
        color: #444;
      }
      &7 {
        color: #000;
      }
      &.unopen {
        background-color: rgb(111, 111, 112);
      }
      &:hover {
        opacity: 0.8;
      }
      &.light {
        background-color: #555;
      }
    }
  }
}
</style>
