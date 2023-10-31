<!--
 * @Author: 羊驼
 * @Date: 2023-10-31 14:21:49
 * @LastEditors: 羊驼
 * @LastEditTime: 2023-10-31 15:08:56
 * @Description: file content
-->
<template>
  <div
    class="root"
    style="width: 100%; height: 100%; margin-top:10px"
  >
    <div
      class="box"
      id="model-viewer"
      :class="fullScreen ? 'fullbox' : 'exitfullbox'"
    >
      <span
        v-if="viewer != 'init'"
        class="fullscreen"
        @click="fullScreenTable"
      >{{
        fullScreen ? '退出' : '全屏'
      }}</span>
      <div
        class="animations-group"
        v-if="animations.length>0&&!hide"
      >
        <el-button @click="stopAnimation">停止</el-button>
        <el-button
          v-for="item in animations"
          :key="item.name"
          @click="playAnimation(item)"
        >{{item.name}}</el-button>
      </div>
      <p
        v-if="viewer == 'init'"
        style="line-height: 70vh; text-align: center"
      >3D模型加载中...</p>
    </div>
  </div>
</template>

<script>
// 此库底层其实就是threeJS
import * as OV from "online-3d-viewer";
import * as THREE from "three";

export default {
  props: ["type", "file", "url"],
  data() {
    return {
      viewer: null,
      fullScreen: false,
      interval: null,
      detaultAspect: 0,
      // 全屏前Div的宽高
      originalWidth: null,
      originalHeight: null,
      // 动画片段
      animations: [],
      mixer: null,
      hide: false,
      // 更新函数
      update: null,
      clock: new THREE.Clock(),
    };
  },
  methods: {
    // 全屏功能
    fullScreenTable() {
      let element = document.querySelector("#model-viewer");
      this.originalWidth = element.clientWidth;
      this.originalHeight = element.clientHeight;
      this.launchIntoFullscreen(element);
    },
    // 兼容各浏览器
    launchIntoFullscreen(element) {
      // 退出全屏
      if (this.fullScreen) {
        if (document.exitFullscreen) {
          document.exitFullscreen();
        } else if (document.webkitCancelFullScreen) {
          document.webkitCancelFullScreen();
        } else if (document.mozCancelFullScreen) {
          document.mozCancelFullScreen();
        } else if (document.msExitFullscreen) {
          document.msExitFullscreen();
        }
        this.fullScreen = false;
      } else {
        // 检测是否还在全屏的情况 不是的话就回复 viewer的画布大小 不然视野位置会变化
        this.interval = setInterval(() => {
          if (
            document.fullscreenElement !=
            document.querySelector("#model-viewer")
          ) {
            this.fullScreen = false;
            this.$nextTick(() => {
              // 重设画布大小
              this.viewer.viewer.Resize(
                this.originalWidth,
                this.originalHeight
              );
            });
            clearInterval(this.interval);
          }
        }, 100);
        // 全屏
        if (element.requestFullscreen) {
          element.requestFullscreen();
        } else if (element.mozRequestFullScreen) {
          element.mozRequestFullScreen();
        } else if (element.webkitRequestFullscreen) {
          element.webkitRequestFullscreen();
        } else if (element.msRequestFullscreen) {
          element.msRequestFullscreen();
        }
        this.fullScreen = true;
      }
    },
    // 模型初始化
    init() {
      // 设置模型代码 判断是本地的还是url的
      const setFbx = () => {
        this.hide = true;
        switch (this.type) {
          case "file":
            this.viewer.LoadModelFromFileList([this.file]);
            break;
          default:
            this.viewer.LoadModelFromUrlList([this.url]);
            break;
        }
      };
      // 如果已经生成了场景 只需要移除原来在场景上的fbx 减少初始化的开销
      if (this.viewer != null) {
        this.viewer.viewer.cameraMode = 2;
        let { scene } = this.viewer.viewer;
        if (scene.children.length > 3) {
          scene.remove(scene.children[3]);
        }
        return setFbx();
      }

      this.viewer = "init";

      setTimeout(() => {
        // 生成在哪个div下
        let parentDiv = document.getElementById("model-viewer");

        // 初始化预览器
        this.viewer = new OV.EmbeddedViewer(parentDiv, {
          // 背景颜色
          backgroundColor: new OV.RGBAColor(51, 51, 51, 255),
          // 当模型加载完毕
          onModelLoaded: () => {
            let { renderer, camera, scene } = this.viewer.viewer;
            // model是 经过插件处理以后 模型材质显示会变化 并且丢失动画
            let model = this.viewer.viewer.mainModel.mainModel.rootObject;
            // 未经过处理的模型 只有这上面我们才能拿到模型的动画
            let original = this.viewer.model.originalModel;

            // 当模型身上有动画的话 我们才使用原来的模型 不然就使用处理后的模型
            if (original.animations.length > 0) {
              // 本质上其实都是threejs生成的对象 可以进行使用它身上的方法
              // 移除渲染后的模型 添加带动画的模型回到场景
              scene.remove(model);
              scene.add(original);
              // 获取模型的动画片段
              const clips = original.animations;
              this.animations = clips;
              // 生成一个动画混合器在这个模型身上 不然无法播放动画
              const mixer = new THREE.AnimationMixer(original);
              // 记录这个混合器 方便后续更改动画以及停止动画
              this.mixer = mixer;
              // 渲染 更新场景 保证动画的播放
              this.update = () => {
                if (!this.mixer) return;
                requestAnimationFrame(this.update);
                mixer.update(this.clock.getDelta());
                renderer.render(scene, camera);
              };
              this.update();
            }
            this.hide = false;
          },
        });
        ///更改源码中 原来我们这一句 = this.viewer.model.originalModel 是拿不到原来加载的模型对象的
        // 修改fbx加载器中 读取模型时 保存一份解析完毕的原始模型到模型加载器上 这样我们后续才能拿到有无动画
        let original =
          this.viewer.modelLoader.importer.importers[11].OnThreeObjectsLoaded;
        this.viewer.modelLoader.importer.importers[11].OnThreeObjectsLoaded =
          function (loadedObject, onFinish) {
            this.GetMainObject = (loadedObject) => {
              return loadedObject;
            };
            this.model.originalModel = loadedObject;
            original.call(this, loadedObject, onFinish);
          };
        // 这句应该没啥用 写了忘记拿来干嘛了
        this.detaultAspect = this.viewer.viewer.camera.aspect;
        setFbx();
      }, 100);
    },
    // 停止动画
    stopAnimation() {
      this.mixer && this.mixer.stopAllAction();
    },
    // 播放动画
    playAnimation(clip) {
      this.mixer && this.mixer.clipAction(clip).play();
    },
    // 清除
    clear() {
      clearInterval(this.interval);
      if (this.viewer) {
        this.viewer.Destroy();
      }
      this.viewer = null;
      this.mixer = null;
      this.animations = [];
      this.interval = null;
    },
  },
  destroyed() {
    this.clear();
  },
};
</script>

<style>
.animations-group {
  position: absolute;
  right: 10px;
  top: 10px;
  display: flex;
  flex-direction: column;
}
.animations-group .el-button {
  margin-left: 0 !important;
  margin-bottom: 10px;
  width: 200px;
}
@media (max-width: 767.98px) {
  .fbx-box {
    width: 96% !important;
    height: 60vh !important;
    margin-bottom: 20px;
  }

  .root,
  .box {
    height: 70vh !important;
  }
}

.box {
  background-color: #333;
  font-size: 30px;
  font-weight: bold;
  color: #fff;
  position: relative;
  flex: 1;
}

.fullscreen {
  font-size: 16px;
  font-weight: normal;
  position: absolute;
  left: 30px;
  bottom: 30px;
  user-select: none;
  cursor: pointer;
}

.fullscreen:hover {
  color: greenyellow;
}

.fullbox {
  width: 100vw !important;
  height: 100% !important;
}

.exitfullbox {
  width: 100% !important;
  height: 100% !important;
}
</style>
