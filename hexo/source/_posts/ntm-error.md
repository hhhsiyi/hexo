---
title: 缓存加载问题引发的故障复现
date: 2023-01-18 18:49:13
tags: 故障复盘
---

# 背景描述

---

项目组的底层通用模块代码引用了总行提供的框架,框架本身有一点小小的问题,排查下来定位到了问题,坐在回家的高铁上没事干,就把自己脑子掏空,花时间在自己电脑上复盘了一下

# 测试环境搭建

---

后端+前端+jmeter模拟高并发场景进行压测.通过利用部分工具来观察资源占用情况来确定是否整个过程出现了问题.

故障起因是管理员在业务作业时间在系统前台点击了刷新码表缓存的按钮,随后引发的流量拥塞,请求淤积,进而导致服务不可用

## 后端代码:

controller层,代码很简单,总共三个按钮:获取某一码表值\刷新码表\清空码表

~~~java
/**
 * 2023/1/18
 *
 * @author Hewen
 * 云想衣裳花想容，春风拂槛露华浓
 * 最是人间留不住，朱颜辞镜花辞树
 */
/*
api层,负责进行转发
 */
@RestController
@Slf4j
public class TestController {
    @RequestMapping("hello")
    public String test() {
        return "hello World";
    }

    /**
     * @描述 前端调用后端获取指定码表值的接口
     * @访问网址 http://localhost:8080/getCache
     */
    @RequestMapping("getCache")
    public String getCache() {
        /*
         * 1.查询指定码表值,如果redis里没有,就触发刷新码表功能,去加载全部的缓存(黑人问号脸,为什么要这么写,我还是一脸懵逼)
         */
        String a = "myKey1";
        String o = (String) redisTemplate.opsForValue().get(a);
        if (null == o || StringUtils.isEmpty(o)) {
            reloadCache();
            return "正在获取";
        } else {
            return o;
        }
    }

    @Autowired
    private TestService service;
    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * @描述 刷新缓存按钮
     * @访问网址 http://localhost:8080/reloadCache
     */
    @RequestMapping("reloadCache")
    public String reloadCache() {
        Set keys = redisTemplate.keys("*");
        System.out.println(keys.size());
        reloadCache();
        return "刷新成功";
    }

    /**
     * @描述 清空缓存按钮
     * @访问网址 http://localhost:8080/clearCache
     */
    @RequestMapping("clearCache")
    public String clearCache() {
        Set<String> keys = redisTemplate.keys("*");
        if (redisTemplate.delete(keys) > 0) {
            return "清除成功";
        } else {
            return "清除失败";
        }
    }
}

~~~

service层:就一个方法**loadCache**,主要作用为删除缓存,并重新塞入缓存.但该方法上好巧不巧加了一个**synchronized**同步锁.

为了模拟生产环境可能出现的操作响应时间,设置了2秒的延迟.

~~~java
/**
 * 2023/1/18
 *
 * @author Hewen
 * 云想衣裳花想容，春风拂槛露华浓
 * 最是人间留不住，朱颜辞镜花辞树
 */
/*
service层,负责进行业务逻辑编写
 */
@Component
@Slf4j
public class TestService {
    @Autowired
    private RedisTemplate redisTemplate;
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    public synchronized void loadCache() {
        stringRedisTemplate.delete(stringRedisTemplate.keys("myKe*"));
        log.warn("进入加载缓存方法");
        for (int i = 0; i < 100; i++) {
            redisTemplate.opsForValue().set("myKey"+i, "妈妈, 我想离开浪浪山！！！" + i);
            try {
                Thread.sleep(20L);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
        String mykey = (String) redisTemplate.opsForValue().get("myKey1");
        log.warn("完成加载缓存方法");
        System.out.println(mykey);
    }
}

~~~

至此,后端环境已经搭建完毕,redis全部默认配置6379能用就行,外层随便套一个springboot启动类就能提供服务.

## 前端代码:

随便找三个前端按钮,即可完成前端搭建

{% asset_img 2.png 前端按钮 %}

按需启动redis\后端\前端\打开jmeter开始模拟测试.

## JMETER配置:

{% asset_img 线程组配置.png 线程组配置 %}

{% asset_img http请求配置.png http请求配置 %}

# 开始测试

测试基于对数据位于redis的码表查询接口,设置并发量为200,压测一段时间之后观测吞吐量稳定值.

## 1.流量稳定时的情况

稳定请求时吞吐量在1w+,自己的笔记本没有插电,CPU性能可能发挥不佳.

{% asset_img 未插电,正常请求时.png 未插电,正常请求时 %}

{% asset_img 正常部分线程监控.png 正常部分线程监控 %}

## 2.流量爆降时的情况

在稳定时,去点击刷新缓存按钮,会因为同步锁将并发请求排队独木桥式的处理,且因为getCache方法中特殊(反人类)的判断前移,导致所有的请求都淤积在同步方法前.

{% asset_img 点击完刷新缓存后.png 点击完刷新缓存后 %}

{% asset_img 吞吐量从1w降到了9k.png 吞吐量从1w降到了9k %}

{% asset_img 吞吐量爆降.png 吞吐量爆降 %}

{% asset_img 线程拥塞之后,吞吐量几乎没有了.png 线程拥塞之后,吞吐量几乎没有了 %}

吞吐量会瞬间爆降.

再利用阿里巴巴出品的arthas工具进行监控,可以看到大量的线程基本上都处于BLOCK阻塞状态.

{% asset_img 线程几乎全部陷入block死锁状态.png 线程几乎全部陷入block死锁状态 %}

且服务已无法正常提供服务,所有请求全部超时

{% asset_img 再去点击获取缓存,请求阻塞.png 请求均无响应 %}

出现此种情况,只能切断前端请求,等待进程处理完全部的请求,或重启阻塞的服务,快速释放线程资源.

## 3.做细微改动后的情况

经过分析,应该是getCache接口中前置的redis中数据判断太过提前,导致到reloadCache的方法均是被判定为拿不到缓存数据的情况,进入方法之后又会进行(清空缓存-加载缓存-该线程结束后,下一个线程进来继续清空缓存)的无限循环状态中.

还记得之前的**synchronized**关键字吗.,将其去掉,开始复测

正常状态下的时候,吞吐量好像还高了一点

{% asset_img 去掉同步锁后感觉吞吐量还能再上来一丢丢.png 去掉同步锁后感觉吞吐量还能再上来一丢丢 %}

改掉之后进行测试,响应会慢,但是不会堵塞

{% asset_img 改掉后继续压测.png 改掉后继续压测 %}

{% asset_img 会慢,但不会堵塞其他请求.png 会慢,但不会堵塞其他请求 %}

# 总结:

---

啥都不用想,把脑子搬空,再往里重新注水.

这次的事件,我愿称之为: **一把锁,把我困在这里**

假如没有同步锁\redis中数据的判空可以后置,均可以避免惨剧的发生,有点可惜了.

1:别人的轮子不一定有多优雅;

2:所有诡异事件的发生均是有迹可循的,不是内存指针地址空间之类的编译偶发错误都能找到触发原因;

3:我是真的菜狗,这么简单的写在源码里的bug我竟然在没出事的时候没发现,练习时长两年半真的是白练了;

> -结尾:
>
> -年更人在一月就完成了23年的第一更
>
> -希望今年有机会搭个k8s,验证一下jvm对容器的内存大小进行感知并进行动态调整和内存分配,我们有一个服务内存占用一直缓慢增长,两天时间排查仍未找到任何可疑点.不如腾出不加班的晚上干点别的.
