<template>
  <div class = "goods">
    <div class = "menu-wrapper" v-el:menu-wrapper>
      <ul>
        <li v-for = "item in goods" 
            class = "menu-item" 
            :class = "{'current':currenIndex  === $index}"
            @click = "selectMenu($index, $event)">
          <span class = "text border-1px">
            <span v-show = "item.type > 0" class="icon" :class = "classMap[item.type]"></span>
            {{item.name}}
          </span>
        </li>
      </ul>
    </div>
    <div class = "foods-wrapper" v-el:foods-wrapper>
      <ul>
        <li v-for = "item in goods" class = "foods-list foods-list-hook">
            <h1 class="title">{{item.name}}</h1>
            <ul>
              <li v-for = "food in item.foods" class = "food-item">
                <div class="icon"><img width = "57" height = "57" :src = "food.icon" /></div>
                <div class="content">
                  <h2 class="name">{{food.name}}</h2>
                  <p class = "desc">{{food.description}}</p>
                  <div class = "extra">
                    <span class="count">月售 {{food.sellCount}} 份</span>
                    <span>好评率 {{food.rating}}</span>
                  </div>
                  <div class = "price">
                    <span class = "now">${{food.price}}</span><span class = "old" v-show = "food.oldPrice">${{food.oldPrice}}</span>
                  </div>
                  <div class = "cartcontrol-wrapper">
                      <cart-control :food = "food"></cart-control>
                  </div>
                </div>
              </li>
            </ul>
        </li>
      </ul>
    </div>
    <shop-car :delivery-price = "seller.deliveryPrice"
              :min-price = "seller.minPrice"
              :select-foods = "selectFoods"
              v-ref:shopcar></shop-car>
  </div>

</template>

<script>
  import BuildScroll from 'better-scroll';
  import shopCar from 'components/shopcar/shopcar';
  import cartControl from 'components/cartControl/cartControl.vue';
  const ERROR_OK = 0;
  export default {
    props: {
      seller: {
        type: Object
      }
    },
    data() { // 1.为什么要使用returen 2.data 为什么是一个function 而不是一个 json 对象
      return {
        goods: [],
        listHeight: [], // 存储内容列表高度，实现左侧菜单与右侧内容联动
        scorllY: 0
      };
    },
    computed: {
      currenIndex() {
        for (let i = 0; i < this.listHeight.length; i++) {
          let height1 = this.listHeight[i];
          let height2 = this.listHeight[i + 1];
          if (!height2 || (this.scrollY >= height1 && this.scrollY < height2)) {
            return i;
          }
        }
        return 0;
      },
      selectFoods() {
        let foods = [];
        this.goods.forEach((good) => {
          good.foods.forEach((food) => {
            if (food.count) {
              foods.push(food);
            }
          });
        });
        return foods;
      }
    },
    created() {
      this.classMap = ['decrease', 'discount', 'special', 'invoice', 'guarantee'];

      this.$http.get('/api/goods').then((response) => {
        response = response.body;
        if (response.errno === ERROR_OK) {
          this.goods = response.data;
          // 在vue数据层中更改数据，对应的dom会做映射。但是实际上，vue在更新dom上是一个异步的，vue中是通过nexttick // 来执行dom的更新
          /*

            8.如何使用外部插件与原生vue做交互
            声明挂载点
            v-el:className
            获取原生dom
            this.$els.className.getElementsByClassName()
            更新数据
            this.$nextTick()
            #计算一些与dom相关的东西的时候，一定要保证dom已经渲染完毕，虽然vue提供了dom与数据间的自然映射，即我们改变了数据就改变了dom。
            但是dom真正发生变化是在$nextTick之后，所以，在操作dom的时候一定要调用nextTick接口，然后在这个接口的回调函数里做一些事情。如此一来，就能保证，是在dom渲染之后，在进行数据映射。

          */
          this.$nextTick(() => {
            this._initScroll();
          });
        }
      });
    },
    methods: {
      _initScroll() {
        this.menuScroll = new BuildScroll(this.$els.menuWrapper, {
          click: true
        });
        this.foodsScroll = new BuildScroll(this.$els.foodsWrapper, {
          click: true,
          probeType: 3
        });

        this.foodsScroll.on('scroll', (pos) => {
          this.scrollY = Math.abs(Math.round(pos.y));
          this._calculateHeight();
        });
      },
      _calculateHeight() {
        let foodList = this.$els.foodsWrapper.getElementsByClassName('foods-list-hook'); // 获取每种类别下的卖品数量
        let height = 0; // 构建一个区间数组,记录每组卖品的高度是多少
        this.listHeight.push(height);
        for (var i = 0; i < foodList.length; i++) {
          let item = foodList[i];
          height += item.clientHeight;
          this.listHeight.push(height);
        }
      },
      selectMenu(index, event) {
        if (!event._constructed) { // 如果事件由vue派发则返回true,!event._constructed说明点击事件来自浏览器，来自浏览器的点击事件需要屏蔽掉，避免两次点击事件发生
          return;
        }
        let foodList = this.$els.foodsWrapper.getElementsByClassName('foods-list-hook');
        let activeItem = foodList[index];
        this.foodsScroll.scrollToElement(activeItem, 300);
      },
      _drop(target) {
        // 体验优化，异步执行下落动画
        this.$nextTick(() => {
          this.$refs.shopcar.drop(target);
        });
      }
    },
    components: {
      shopCar,
      cartControl
    },
    events: {
      'cart.add'(target) {
        this._drop(target);
      }
    }
  };
</script>
<style lang = "stylus" rel = "stylesheet/stylus">
 @import '../../common/stylus/mixin.styl';
  .goods
   position:absolute
   top:174px
   display:flex
   bottom:46px
   width:100%
   overflow:hidden
  .menu-wrapper
   flex:0 0 80px
   width:80px
   background-color:#f3f5f7
   .menu-item
    display:table
    height:54px
    width:100%
    line-height:14px
    font-size:12px
    font-weight:700
    &.current
      position:relative
      maring-top:-1px
      z-index:10
      background:#fff
      font-weight:700
      .text
        border-none()
    .icon
     display:inline-block
     vertical-align:top
     width:12px
     height:12px
     margin-right:2px
     background-size:12px 12px
     background-repeat:no-repeat
     &.decrease
      bg-image('decrease_3')
     &.discount
      bg-image('discount_3')
     &.invoice
      bg-image('invoice_3')
     &.special
      bg-image('special_3')
     &.guarantee
      bg-image('guarantee_3')
    .text
      display:table-cell
      width:56px
      vertical-align:middle
      border-1px(rgba(7,17,27,0.1))
      font-size:12px
      padding: 0 12px
  .foods-wrapper
    flex:1
    .title
      padding-left:14px
      height:26px
      line-height:26px
      border-left:2px solid #d9dde1
      background:#f3f5f7;
      font-size:12px
      color:rgb(147,153,159)
    .food-item
      display:flex
      margin:18px
      border-1px(rgba(7,17,27,0.1))
      padding-bottom:18px
      &.last-child
        border-none()
        margin-bottom:0
      .icon
        flex: 0 0 57px
        margin-right:10px
      .content
        flex: 1
        .name
          margin: 2px 0 8px 0
          height:14px
          line-height:14px
          font-size:14px
          color:rgb(1,17,27)
        .desc,.extra
          line-height:10px
          height:10x
          font-size:10px
          color:rgb(147,153,159)
        .desc
          margin:8px 0;
        .extra
         .count
          margin-right:12px
        .price
          font-weight:700
          line-height:24px
          .now
            margin-right:8px
            font-size:14px
            color:red
          .old
            text-decoration:line-through
            font-size:10px
            color:rgb(147,153,159)
        .cartcontrol-wrapper
          position:absolute
          right:0
          bottom:10px

</style>