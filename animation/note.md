* 浏览器按照固定频率来为显示器的刷新提供新帧，由于GPU 处理图像的速度很快，所以这个频率是按照显示器的hZ 数来确定的。 所以，只要浏览器1秒钟为显示器提供60帧即可。 即60fps
如果 GPU生产图像的速度过快会导致画面撕裂的问题。所以应该保证GPU的频率和显示器一致。
V-sync 垂直同步技术，让GPU生产图像的频率 保持在显示器二次刷新之间。
但是，用户并不一定开启了V-sync功能，所以浏览器为了保证GPU的频率和显示器一致，1秒钟内最多只会让GPU渲染60帧图像。 即，大概1000/60 ms 去执行一次调用GPU的操作，传递的参数就是当前页面Paint 的位图的数字信号， GPU拿到这个数字信号进行生成图像的操作。
浏览器内部维护了一个渲染队列，这个队列里存在着一些影响layout 和paint 的操作，比如
div.left=30px; div.backgroundColor=red; 当js在执行这些语句的时候，浏览器并不会立即去计算，渲染，调用GPU。 然后把这些操作存储在渲染队列中，知道下一个1000/60ms 到来时才会flush 渲染队列，计算布局，渲染，调用GPU。 这个操作先管它叫，display refresh
* display refresh 为了得到正确的结果，在js线程在执行的时候，本身就不会执行了。因为dom很可能已经变化。 就好像UI线程 和 js线程 是互斥的一样， 伪代码可能是这样
> 
function displayRefresh(){
    1. layout
    2. paint
    3. composite Layers
}

setInterval(function 调度(){
    if(js线程是空闲状态){
        displayRefresh()
    }
},16.7)

> 
// 模拟sleep
function jank(second) {
    var start = +new Date();
    while (start + second * 1000 > (+new Date())) {}
}

div.style.backgroundColor = "red";
jank(5);

div.style.backgroundColor = "blue";

只会看到蓝色。。

* 但是在js 访问 innerWidth，offetTop 。 scroll 这些操作的时候，浏览器为了给出正确的结果，会立即 flush 渲染队列，以变进行layout 。然后再进行repaint
>    div.style.width = 50 + 'px'
     console.log(div.clientWidth)
     div.style.height = 60 + 'px'
     console.log(div.clientWidth)
     div.style.width = 70 + 'px'
     console.log(div.clientWidth)
     div.style.height = 80 + 'px'
     console.log(div.clientWidth)

      4次layout  1次repaint 最新一次的渲染队列。
      耗时直线上升。
* 画面撕裂
画面撕裂问题牵涉到显示器刷新的原理，是图像渲染和屏幕刷新不同步造成的
1.GPU 连续地渲染图像，每渲染图像就放到显存中一个区域，不等到显示到屏幕上就开始渲染下一个图像
2.显卡上一个叫“Ramdac” 的组件不断从显存中取出最新的一个画面，刷新到屏幕上，这个频率就是显示器的频率，一般60HZ
3.Ramdac 如何知道要刷新的画面在显示器中的哪个位置？这自然是GPU告诉它的，GPU每渲染完一个画面都是通知Ramdac
4.GPU渲染的 和Ramdac 刷新显示是不一定同步的，当Ramdac 刷新到某一个画面一半的时候，收到GPU切换画面的通知，就会导致显示器的这一帧上半部分是 GPU的第一个画面，下部部分是GPU 的第二个画面。