<script setup lang="ts">
import { onLoad, onReachBottom } from "@dcloudio/uni-app";
import { ref, watch } from "vue";

const PLANET_COUNT = 7;
const isMasked = ref(true); // 遮罩层是否存在
const isLoaded = ref(false); // 音频是否全部加载完毕
const isScrollBottom = ref(false); // 是否滚动到底部
const animationComplete = ref(false); // 渐入渐出文字动效等待
const planetLastVolume = ref<number[]>(Array(PLANET_COUNT).fill(1)); // 各星球上次调节音量大小
const audioBufferedStatus = ref<number[]>(Array(PLANET_COUNT).fill(-1)); // 音频加载状态
const audioList: ReturnType<typeof uni.createInnerAudioContext>[] = []; // 音频上下文

interface Planet {
  id: number;
  image: string;
  outline: string;
  src: string;
  volume: number;
}

const planetList = ref<Planet[]>([
  {
    id: 1,
    image:
      "https://i0.hdslb.com/bfs/article/5070bf8512ce1c308fbf2bd832d919e51402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/3be14c053f206c2a12880db8503eb8d61402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731720465519/1.mp3",
    volume: 0.5,
  }, // 余烬双星
  {
    id: 2,
    image:
      "https://i0.hdslb.com/bfs/article/0efb6937678de02bdc1ba2132069bf541402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/9fe31d82efb736179f5336b45fc3b6591402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731720509341/2.mp3",
    volume: 0.3,
  }, // 废岩星
  {
    id: 3,
    image:
      "https://i0.hdslb.com/bfs/article/236623c6a6f527f620aa05fca381ae581402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/10d5133031b805e5b4aae44d3f58e8fb1402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731720509487/3.mp3",
    volume: 0.65,
  }, // 碎空星
  {
    id: 4,
    image:
      "https://i0.hdslb.com/bfs/article/0d0f8a1d1a599ac635d72c1a4d0ec5181402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/e4b078c28deb0120be29be9f024070a71402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731720509639/4.mp3",
    volume: 0.4,
  }, // 外星站
  {
    id: 5,
    image:
      "https://i0.hdslb.com/bfs/article/6fc016f64c490e3a3c478fb5e32b2cdf1402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/f8511547e065b28aef4e4c19b3bd36681402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731720509774/5.mp3",
    volume: 0.7,
  }, // 量子卫星
  {
    id: 6,
    image:
      "https://i0.hdslb.com/bfs/article/1904c9a5e5f2d0fc7d60b49bb1b73d9d1402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/969599e2d5c70e1568fd7e6068c5d2e71402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731721179091/6.mp3",
    volume: 0.9,
  }, // 深巨星
  {
    id: 7,
    image:
      "https://i0.hdslb.com/bfs/article/fb3e5d5b42bbd74cf01da36a91d3f3e71402305269.png@1e_1c.webp",
    outline:
      "https://i0.hdslb.com/bfs/article/85340e698712cfaf3ccb620dd4f39d1c1402305269.png@1e_1c.webp",
    src: "https://fs-im-kefu.7moor-fs1.com/ly/4d2c3f00-7d4c-11e5-af15-41bf63ae4ea0/1731721179356/7.mp3",
    volume: 0.55,
  }, // 黑棘星
]);

// 加载所有音频
const initializeAudio = () => {
  uni.getStorage({
    key: "volume_key",
    // 有上次音量记录则更新，没有缓存则使用初始音量大小
    success: function (res) {
      planetList.value = JSON.parse(res.data);
    },
    complete: function () {
      for (let i = 0; i < PLANET_COUNT; i++) {
        createAudioContext(i);
      }
    },
  });
};

// 音频加载
const createAudioContext = (i: number) => {
  const innerAudioContext = uni.createInnerAudioContext();
  innerAudioContext.src = planetList.value[i].src;
  innerAudioContext.volume = planetList.value[i].volume;
  innerAudioContext.loop = true;
  audioList.push(innerAudioContext);

  innerAudioContext.onCanplay(() => {
    audioBufferedStatus.value[i] = 0;
    if (audioBufferedStatus.value.every(v => v === 0)) {
      isLoaded.value = true;
      setTimeout(() => {
        animationComplete.value = true;
      }, 750);
    }
  });
};

// 开始合奏
const startEnsemble = () => {
  if (!isLoaded.value || !isMasked.value) return;
  isMasked.value = false;
  audioList.forEach((audio, i) => {
    audioList[i].volume = 0; // 初始化为 0
    fadeVolume(i, planetList.value[i].volume, 500);
    audio.play();
  });
};

// 进度条改变音量
const changeVolume = (e: CustomEvent, i: number) => {
  planetList.value[i].volume = e.detail.value; // 列表音量大小，联动透明度
  audioList[i].volume = e.detail.value; // 音频音量
};

// 音量渐变到目标值
const fadeVolume = (
  i: number,
  targetVolume: number,
  duration: number = 1000
) => {
  const step = 20; // 更新频率 ms
  const steps = duration / step;
  let currentStep = 0;
  const startVolume = audioList[i].volume;

  const intervalId = setInterval(() => {
    currentStep++;
    const progress = Math.min(currentStep / steps, 1);
    const currentVolume = startVolume + (targetVolume - startVolume) * progress;
    planetList.value[i].volume = currentVolume;
    audioList[i].volume = currentVolume;

    if (progress >= 1) {
      clearInterval(intervalId);
    }
  }, step);
};

// 星球切换静音/播放
const switchVolume = (i: number) => {
  const curVolume = planetList.value[i].volume;
  if (curVolume !== 0) {
    planetLastVolume.value[i] = planetList.value[i].volume;
    fadeVolume(i, 0, 250);
  } else {
    fadeVolume(i, planetLastVolume.value[i], 250);
  }
};

// 音量渐变 不支持 rAF
// const fadeVolume = (i: number, duration: number = 2000) => {
//   const startTime = performance.now();
//   const initialVolume = planetList.value[i].volume;
//   const updateVolume = (currentTime: number) => {
//     const elapsedTime = currentTime - startTime;
//     const progress = Math.min(elapsedTime / duration, 1);
//     planetList.value[i].volume = progress * initialVolume;
//     audioList[i].volume = progress * initialVolume;
//     if (progress < 1) {
//       requestAnimationFrame(updateVolume); // 递归更新
//     }
//   };
//   requestAnimationFrame(updateVolume); // 初始调用
// };

function debounce<T extends (...args: any[]) => void>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: ReturnType<typeof setTimeout> | null = null;

  return function (...args: Parameters<T>) {
    if (timeout !== null) {
      clearTimeout(timeout);
    }

    timeout = setTimeout(() => {
      func(...args);
    }, wait);
  };
}

// 防抖写入存储，防止音量渐变时频繁写入
const debouncedSetStorage = debounce((newVal: Planet[]) => {
  uni.setStorage({
    key: "volume_key",
    data: JSON.stringify(planetList.value),
  });
}, 300);

// 记录用户手动改变音量大小，用于下次进入应用时读取
watch(planetList, newVal => debouncedSetStorage(newVal), { deep: true });

onLoad(initializeAudio);
onReachBottom(() => (isScrollBottom.value = true));
</script>

<template>
  <view v-if="isMasked" class="mask" @click="startEnsemble">
    <!-- uniapp 无法使用 transition -->
    <!-- <transition name="fade" mode="out-in"> -->
    <view>
      <view
        :class="{ fadeOut: isLoaded }"
        v-if="!isLoaded || !animationComplete"
      >
        Preparing Instruments......
      </view>
      <view
        :class="{ fadeIn: isLoaded && animationComplete }"
        v-if="isLoaded && animationComplete"
      >
        <image
          class="ready"
          v-if="isLoaded"
          alt="click"
          src="data:image/png;base64,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"
          style="display: inline-block"
        />
        Let's Play Together
      </view>
    </view>
    <!-- </transition> -->
  </view>

  <view class="logo">
    <image
      src="https://i0.hdslb.com/bfs/article/14e85d799154e852deff463bfc27fcf11402305269.png@1e_1c.webp"
      mode="aspectFit"
    />
  </view>

  <view class="solar-system">
    <view
      v-for="(planet, i) in planetList"
      :key="planet.id"
      :class="['planet', `planet-${planet.id}`]"
    >
      <image
        alt="Planet"
        :src="planet.image"
        :style="{ opacity: planet.volume }"
      />
      <image
        alt="Planet-Outline"
        :src="planet.outline"
        @click="switchVolume(i)"
      />
      <view class="controls">
        <slider
          id="volume"
          :value="planet.volume"
          min="0"
          max="1"
          step="0.01"
          block-size="28"
          @changing="(e: CustomEvent) => changeVolume(e, i)"
          @change="(e: CustomEvent) => changeVolume(e, i)"
        />
      </view>
    </view>
  </view>
  <view class="footer" :style="{ opacity: isScrollBottom ? 1 : 0 }">
    『Still, this encounter feels special. I hope you won't mind if I think of
    you as a friend.』
  </view>
</template>
<style scoped lang="scss">
$animation-duration: 0.75s;
$planet-radius: 18vw;
$background-color: rgba(0, 0, 0, 0.75);
$slider-height: 3vw;

@keyframes fadeOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes blink {
  50% {
    opacity: 0;
  }
}

@keyframes rotate {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.fadeOut {
  animation: fadeOut $animation-duration forwards;
}

.fadeIn {
  animation: fadeIn $animation-duration forwards;
}

.mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  color: white;
  text-align: center;
  line-height: 100vh;
  font-size: 6vw;
  user-select: none;
  background-color: $background-color;
  z-index: 2;

  .ready {
    width: 7vw;
    height: 7vw;
    animation: 1.8s 1s infinite blink;
    margin-top: 0.6vw;
    vertical-align: text-top;

    image {
      width: 100%;
      height: 100%;
    }
  }
}

.logo image {
  width: 100%;
}

.solar-system {
  width: 100vw;
  padding: 5vw;
  margin-top: -7vw;
  background: url("https://i0.hdslb.com/bfs/article/ada071614871444bf252f7b9cf5635ad1402305269.jpg@1e_1c.webp")
    no-repeat right;

  .planet {
    width: $planet-radius * 2;
    height: $planet-radius * 2;
    position: relative;

    image {
      width: 100%;
      height: 100%;
      border-radius: 50%;
      position: absolute;
      top: 0;
      left: 0;
      animation: rotate 20s linear infinite;
    }

    .controls {
      width: 50vw;
      position: absolute;
      top: 50%;
      left: 100%;
      transform: translate(5%, -50%);
    }
  }
}

// 修改 slider 样式
:deep(.wx-slider-handle-wrapper) {
  height: $slider-height !important;
}

:deep(.wx-slider-track) {
  background-image: linear-gradient(to right, #6cd0ca, #133fce) !important;
}

.footer {
  color: #fff;
  width: 70vw;
  opacity: 0;
  font-size: 1.2rem;
  margin: 0 auto;
  padding-bottom: 10vw;
  transition: opacity 1s;
}
</style>
