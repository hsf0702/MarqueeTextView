支持跑马灯效果，并且与ViewPager同时使用时不会影响ViewPager的左右滑动手势

转载请注明下出处就好，需要使用，可自行看实现或者copy过去用就好。

## 场景

0. 业务有一个页面为三个tab，使用ViewPager来实现，中间页面的中间歌名显示部分由于可能文字过长，所以回限定最大宽度，超过之后就设置marquee效果。
   到这里，应该都还是比较简单的，网上也是各种资料，重写TextView中的focus，并且写死返回true即可。

1. 第一步实现了之后发现问题了，如果在上面的文字区域左右滑动，ViewPager滑动不了了

基于这个场景对于TextView的Marquee效果对ViewPager的影响进行了一番分析，Stack Overflow上也有类似问题，但都没有很好的的解决这个问题，
所以在此分析下具体逻辑，给有需要的提供个解决这种情况的方法。

## 分析：

### 猜测一   事件被`TextView`拦截消费了
0. 当发现触摸设置了marquee效果的 `TextView` 区域就无法滑动 `ViewPager` 时，最开始的想法就是，事件传递给TextView后，肯定是被TextView拦截了，
   所以ViewPager没法再接收到这个事件。
1. 初步认为应该就是比较简单的事件冲突问题，沿着这个思路，首先的想法就是，那这样的话只要让TextView不去接收处理事件就好了，所以开始试验就是
   想到将 `TextView` 的 `clickable` 和focus设置为false，但是无限循环滚动又需要focus为true，所以这个路子貌似走不下去了。
2. 那既然初步确定是事件被拦截的问题，又不能通过设置 clickable 和 focus 设置为 `false` 来解决，那就在 `TextView` 外面再套一层 `ViewGroup`，并且在  `ViewGroup`
   中拦截这个事件不让往下传就好了。想想还是比较简单，满心欢喜的写了个拦截事件的类，套一下 `TextView`。
3. 经过2分钟编译......
4. 运行查看效果
5. 尴尬😓，在那个区域还是无法左右滑动
6. 这个表层初步的猜测以失败告终
7. 不得不开始单步断点调试了，调试的结果就是，Move事件压根没有到下层

### 猜测二   事件上层（ViewPager）自己没处理
0. 既然`Move`事件下层没有处理，那事件肯定就是回去了`ViewPager`，那么只有看看`ViewPager`的`onInterceptTouchEvent`和`onTouchEvent`里面到底干了啥了/(ㄒoㄒ)/~~
1. 经过一系列无聊的调试，发现了 `onInterceptTouchEvent` 中 `MotionEvent.ACTION_MOVE` 事件时 `return` 了 `false`，这个就很尴尬了，`ViewPager` 明明
   有三个item，怎么可能不滑动了，再细细探究这个事件里面的处理，发现这个事件处理调用了个方法：
   
   
   `
      protected boolean canScroll(View v, boolean checkV, int dx, int x, int y) {
           if (v instanceof ViewGroup) {
               final ViewGroup group = (ViewGroup) v;
               final int scrollX = v.getScrollX();
               final int scrollY = v.getScrollY();
               final int count = group.getChildCount();
               // Count backwards - let topmost views consume scroll distance first.
               for (int i = count - 1; i >= 0; i--) {
                   // TODO: Add versioned support here for transformed views.
                   // This will not work for transformed views in Honeycomb+
                   final View child = group.getChildAt(i);
                   if (x + scrollX >= child.getLeft() && x + scrollX < child.getRight()
                           && y + scrollY >= child.getTop() && y + scrollY < child.getBottom()
                           && canScroll(child, true, dx, x + scrollX - child.getLeft(),
                                   y + scrollY - child.getTop())) {
                       return true;
                   }
               }
           }

           return checkV && v.canScrollHorizontally(-dx);
       }
    `
    
2. 看到这里就清晰了，这里干的是什么事情呢，就是 `ViewPager` 递归的去问它所有的子 `View`，然后调用子 `View` 的 `canScrollHorizontally()` 方法
3. 可算是柳暗花明了，最后就是要验证，是不是在这种设置了marquee的 `TextView` 上触摸并且左右滑动时，`TextView` 的 `canScrollHorizontally()`
   返回了true呢？
4. 继续调试，证实了3的猜测
5. 好了到了这里就好办了，重写 `TextView` 的 `canScrollHorizontally`，返回false好了
6. 自此，大功告成


PS：源码面前，没有秘密
