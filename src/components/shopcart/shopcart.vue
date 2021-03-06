<template>
  <div>
    <div class="shopcart">
      <div class="content">
        <div class="content-left" @click.stop.prevent="toggleList" @touchstart.stop.prevent="toggleList">
          <div class="content-wrap">
            <div class="logo-wrap">
              <div class="logo" :class="{'highlight':totalCount>0}">
                <svg class="icon" aria-hidden="true"><use xlink:href="#icon-shopping_cart"></use></svg>
              </div>
              <div class="num" v-show="totalCount>0">{{totalCount}}</div>
            </div>
            <div class="price" :class="{'highlight':totalCount>0}">￥{{totalPrice}}元</div>
            <div class="desc">另需配送费￥{{deliveryPrice}}元</div>
          </div>
        </div>
        <div class="content-right">
          <div class="pay" :class="payClass">
            {{payDesc}}
          </div>
        </div>
      </div>
      <div class="ball-container">
        <div v-for="ball in balls">
          <transition name="drop" @before-enter="beforeEnter" @enter="enter" @after-enter="afterEnter">
            <div class="ball" v-show="ball.show">
              <div class="inner inner-hook"></div>
            </div>
          </transition>
        </div>
      </div>
      <transition name="fold">
        <div class="shopcart-list" v-show="listShow">
          <div class="list-header">
            <h1 class="title">购物车</h1>
            <span class="empty" @touchstart.stop.prevent="empty" @click.stop.prevent="empty">清空</span>
          </div>
          <div class="list-content" ref="listContent">
            <ul>
              <li v-for="food in selectFoods" class="food">
                <span class="name">{{food.name}}</span>
                <div class="price">
                  <span>￥{{food.price*food.count}}</span>
                </div>
                <div class="cartcontrol-wrap">
                  <cartcontrol :food="food"></cartcontrol>
                </div>
              </li>
            </ul>
          </div>
        </div>
      </transition>
    </div>
    <transition name="fade">
      <div class="list-mask" @touchstart.stop.prevent="hideList" @click.stop.prevent="hideList" v-show="listShow"></div>
    </transition>
  </div>
</template>
<script>
  import BScroll from 'better-scroll';
  import cartcontrol from '@/components/cartcontrol/cartcontrol';
  export default {
    props: {
      selectFoods: {
        type: Array,
        default: []
      },
      deliveryPrice: {
        type: Number,
        default: 0
      },
      minPrice: {
        type: Number,
        default: 0
      }
    },
    components: {
      cartcontrol
    },
    data() {
      return {
        balls: [{
          show: false
        }, {
          show: false
        }, {
          show: false
        }, {
          show: false
        }, {
          show: false
        }],
        dropBalls: [],
        fold: true
      }
    },
    methods: {
      drop(el) {
        for (let i = 0, l = this.balls.length; i < l; i++) {
          let ball = this.balls[i];
          if (!ball.show) {
            ball.show = true;
            ball.el = el;
            this.dropBalls.push(ball);
            return;
          } 
        }
      },
      beforeEnter(el) { //找到所以设为true的小球
        let count = this.balls.length;
        while (count--) {
          let ball = this.balls[count];
          if (ball.show) {
            let rect = ball.el.getBoundingClientRect();
            if (rect.top) {
              //返回元素相对于视口偏移的位置
              let x = rect.left - 32; //点击的按钮与小球（fixed）之间x方向的差值
              let y = -(window.innerHeight - rect.top - 22);
              el.style.display = ''; //设置初始位置前，手动置空，覆盖之前的display：none，使其显示
              el.style.webkitTransform = `translate3d(0,${y}px,0)`; //外层元素做纵向的动画，y是变量
              el.style.transform = `translate3d(0,${y}px,0)`;
              let inner = el.getElementsByClassName('inner-hook')[0];
              //内层元素做横向动画,inner-hook（用于js选择的样式名加上-hook，表明只是用     //于js选择的，没有真实的样式含义）
              inner.style.webkitTransform = `translate3d(${x}px,0,0)`;
              inner.style.transform = `translate3d(${x}px,0,0)`;
            }
          }
        }
      },
      enter(el) {
        el.offsetHeight // 触发浏览器重绘，offsetWidth、offsetTop等方法都可以触发
        this.$nextTick(() => { //异步执行
          el.style.webkitTransform = 'translate3d(0,0,0)'; //重置回来
          el.style.transform = 'translate3d(0,0,0)';
          let inner = el.getElementsByClassName('inner-hook')[0];
          inner.style.webkitTransform = 'translate3d(0,0,0)';
          inner.style.transform = 'translate3d(0,0,0)';
        });
      },
      afterEnter(el) {
        let ball = this.dropBalls.shift(); //取到做完动画的球，再置为false，即重置，它还可以接着被利用
        if (ball) {
          ball.show = false;
          el.style.display = 'none';
        }
      },
      toggleList() {
        if (!this.totalCount) {
          return;
        }
        this.fold = !this.fold;
      },
      empty() {
        this.selectFoods.forEach((food) => {
          food.count = 0;
        })
      },
      hideList() {
        this.fold = true;
      }
    },
    computed: {
      totalPrice() {
        // 计算商品价格
        let total = 0;
        this.selectFoods.forEach((food) => {
          total += food.price * food.count;
        });
        return total;
      },
      totalCount() {
        let count = 0;
        this.selectFoods.forEach((food) => {
          count += food.count;
        });
        return count;
      },
      payDesc() {
        if (this.totalPrice === 0) {
          return `￥${this.minPrice}元起送`;
        } else if (this.totalPrice < this.minPrice) {
          let diff = this.minPrice - this.totalPrice;
          return `还差￥${diff}起送`;
        } else {
          return '去结算';
        }
      },
      payClass() {
        if (this.totalPrice < this.minPrice) {
          return 'not-enough';
        } else {
          return 'enough';
        }
      },
      listShow() {
        if (!this.totalCount) {
          this.fold = true;
          return false;
        }
        let show = !this.fold;
        if (show) {
          this.$nextTick(() => {
            if (!this.scroll) {
              this.scroll = new BScroll(this.$refs.listContent);
            } else {
              this.scroll.refresh();
            }
          });
        }
        return show;
      }
    },
    created() {
      // 监听各组件的this.$root.eventHub.$emit('cartadd', event.target);
      // 这样那个组件的可以调用drop的小球动画
      this.$nextTick(() => {
        this.$root.eventHub.$on('cartadd', this.drop);
      });
    }
  }

</script>

<style lang="scss">
  @import "../../common/sass/mixin";
  .shopcart {
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 50;
    width: 100%;
    height: 48px;
    background: #141b27;
    .content {
      display: flex;
      background: #141b27;
      font-size: 0;
      color: rgba(255, 255, 255, .4);
      .content-left {
        flex: 1;
        .logo-wrap {
          display: inline-block;
          position: relative;
          top: -10px;
          margin: 0 12px;
          padding: 6px;
          width: 56px;
          height: 56px;
          box-sizing: border-box;
          vertical-align: top;
          border-radius: 50%;
          background: #141b27;
          .logo {
            width: 100%;
            height: 100%;
            border-radius: 50%;
            background: #2b343c;
            font-size: 24px;
            text-align: center;
            color: #aaa;
            line-height: 40px;
            &.highlight {
              background: rgb(0, 160, 220);
              color: #fff;
              transition: all .2s linear .4s;
            }
          }
          .num {
            position: absolute;
            top: 0;
            right: 0;
            width: 24px;
            height: 16px;
            line-height: 16px;
            text-align: center;
            border-radius: 16px;
            font-size: 9px;
            font-weight: 700;
            color: #fff;
            background: rgb(240, 20, 20);
            box-shadow: 0 4px 8px 0 rgba(0, 0, 0, .4);
          }
        }
        .price {
          display: inline-block;
          vertical-align: middle;
          line-height: 24px;
          margin-top: 12px;
          box-sizing: border-box;
          padding-right: 12px;
          border-right: 1px solid rgba(255, 255, 255, .1);
          font-size: 16px;
          font-weight: 700;
          &.highlight {
            color: #fff;
          }
        }
        .desc {
          display: inline-block;
          vertical-align: middle;
          margin: 12px 0 0 12px;
          line-height: 24px;
          font-size: 10px;
        }
      }
      .content-right {
        flex: 0 0 105px;
        width: 105px;
        .pay {
          height: 48px;
          line-height: 48px;
          text-align: center;
          font-size: 12px;
          font-weight: 700;
          transition: all .5s;
          &.not-enough {
            background: #2b333b;
          }
          &.enough {
            background: #00b43c;
            color: #fff;
          }
        }
      }
    }
    .ball-container {
      .ball {
        position: fixed;
        left: 32px;
        bottom: 22px;
        z-index: 200;
        border-radius: 50%;
        &.drop-enter,
        &.drop-enter-active {
          transition: all 0.5s cubic-bezier(0.49, -0.29, 0.75, 0.41);
          opacity: 1;
          .inner {
            width: 16px;
            height: 16px;
            border-radius: 50%;
            background: rgb(0, 160, 220);
            transition: all 0.5s linear;
          }
        }
      }
    }
    .shopcart-list {
      position: absolute;
      top: 0;
      left: 0;
      z-index: -1;
      width: 100%;
      transform: translate3d(0, -100%, 0);
      &.fold-enter-active,
      &.fold-leave-active {
        transition: all .5s;
      }
      &.fold-enter,
      &.fold-leave-to {
        transform: translate3d(0, 0, 0);
      }
      .list-header {
        height: 40px;
        line-height: 40px;
        padding: 0 18px;
        background: #eee;
        border-bottom: 1px solid rega(7, 17, 27, .1);
        .title {
          float: left;
          font-size: 16px;
          font-weight: 500;
          color: rgb(0, 160, 220);
        }
        .empty {
          float: right;
          font-size: 12px;
          color: rgb(0, 160, 220);
        }
      }
      .list-content {
        padding: 0 18px;
        max-height: 217px;
        overflow: hidden;
        background: #fff;
        .food {
          position: relative;
          padding: 12px 0;
          box-sizing: border-box;
          @include border-1px(rgba(7, 17, 27, .1));
          .name {
            line-height: 24px;
            font-size: 14px;
            color: rgb(7, 17, 27);
          }
          .price {
            position: absolute;
            right: 90px;
            bottom: 12px;
            line-height: 24px;
            font-size: 14px;
            font-weight: 700;
            color: rgb(240, 20, 20);
          }
          .cartcontrol-wrap {
            position: absolute;
            right: 0;
            bottom: 6px;
          }
        }
      }
    }
  }

  .list-mask {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 40;
    /*这里写显示时的效果*/
    opacity: 1;
    backdrop-filter: blur(10px);
    background: rgba(7, 17, 27, .6);
    &.fade-enter-active,
    &.fade-leave-active {
      /*这里写过度效果*/
      transition: all .5s;
    }
    &.fade-enter,
    &.fade-leave-to {
      /*这里写结果和开始效果*/
      opacity: 0;
      background: rgba(7, 17, 27, 0);
    }
  }

</style>
