# Redis实现UV、DAU
+ UV(Unique Visitor)通常是使用IP或Cookie来记录独立的访客。<br/>
+ DAU(Daily Active User)通常就是通过用户id，来记录每日活跃的用户数量。

```Java
@Component
public class DataInterceptor implements HandlerInterceptor {
    @Autowired
    private DataService dataService;

    @Autowired
    private HostHolder hostHolder;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //统计UV
        String ip = request.getRemoteHost();
        dataService.recordUV(ip);
        //统计DAU
        User user = hostHolder.getUser();
        if (user != null) {
            dataService.recordDAU(user.getId());
        }
        return true;
    }
}
```

这里我使用拦截器，在每次操作前获取请求的ip以及用户的id(前提是用户已登录)，分别统计UV和DAU。接下来看看UV和DAU的实现。<br/>

## UV
首先是UV。<br/>
<br/>

```Java
    //将指定的ip计入UV
    public void recordUV(String ip) {
        String redisKey = RedisKeyUtil.getUVKey(df.format(new Date()));
        redisTemplate.opsForHyperLogLog().add(redisKey, ip);
    }
```

ip的存入，在生成对应日期的redisKey后，直接将ip存入HyperLogLog中即可。<br/>
<br/>

```Java
    //统计指定日期范围内的UV
    public long calculateUV(Date startdate, Date enddate) {
        if (startdate == null || enddate == null) {
            throw new IllegalArgumentException("参数不能为空");
        }
        //整理日期范围内的key
        List<String> keyList = new ArrayList<>();
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(startdate);
        while (!calendar.getTime().after(enddate)) {
            String key = RedisKeyUtil.getUVKey(df.format(calendar.getTime()));
            keyList.add(key);
            calendar.add(Calendar.DATE, 1);
        }
        //合并这些数据
        String redisKey = RedisKeyUtil.getUVKey(df.format(startdate), df.format(enddate));
        redisTemplate.opsForHyperLogLog().union(redisKey, keyList.toArray());
        //返回统计的结果
        return redisTemplate.opsForHyperLogLog().size(redisKey);
    }
```

统计指定日期范围内的UV，只需要将对应日期的redisKey存入List，之后再自己创建一个统计范围UV的redisKey2，接着调用redisTemplate.opsForHyperLogLog().union，合并多个redisKey的UV即可。最后该redisKey2对应的HyperLogLog的size即是UV的个数。

## DAU
接下来是DAU。<br/>
<br/>

```Java
    // 将指定用户计入DAU
    public void recordDAU(int userId) {
        String redisKey = RedisKeyUtil.getDAUKey(df.format(new Date()));
        redisTemplate.opsForValue().setBit(redisKey, userId, true);
    }
```

生成对应日期的redisKey，userId作为索引，将第userId个bit设置为true，来说明当天，id为userId的用户活跃过。<br/>
<br/>

```Java
    // 统计指定日期内的DAU
    public long calculateDAU(Date startdate, Date enddate) {
        if (startdate == null || enddate == null) {
            throw new IllegalArgumentException("参数不能为空");
        }
        //整理日期范围内的key
        List<byte[]> keyList = new ArrayList<>();
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(startdate);
        while (!calendar.getTime().after(enddate)) {
            String key = RedisKeyUtil.getDAUKey(df.format(calendar.getTime()));
            keyList.add(key.getBytes());
            calendar.add(Calendar.DATE, 1);
        }
        //进行OR运算
        return (long)redisTemplate.execute(new RedisCallback() {
            @Override
            public Object doInRedis(RedisConnection connection) throws DataAccessException {
                String redisKey = RedisKeyUtil.getDAUKey(df.format(startdate), df.format(enddate));
                connection.bitOp(RedisStringCommands.BitOperation.OR, redisKey.getBytes(), keyList.toArray(new byte[0][0]));
                return connection.bitCount(redisKey.getBytes());
            }
        });
    }
```

同样在整理日期的redisKey后，对多个BitMap进行OR运算，这样就拥有了范围日期的所有DAU。同样的，计算其bit的个数，那就是在此范围日期的DAU数量。

# Redis实现仿GitHub贡献图
![GitHub贡献图](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/20220306194657.png)

我在[https://gitee.com/catwings/bitmap-demo](https://gitee.com/catwings/bitmap-demo)的基础上，将读取8个bit后拼接成String的操作，改成了直接读取一个个字节，之后直接返回给前端int类型的数值即可。

```Java
    // 将发帖次数存入DPR
    public void recordDPR(int userId, Date date) {
        //减一是为了令第一天存储在0的位置
        int dayOfYear = DayOfYear.dayOfYear(date) - 1;
        String redisKey = RedisKeyUtil.getDPRKey(String.valueOf(userId));
        //每天的PR次数用8个bit存储
        boolean whether_add = true;
        for (int i = 7; i >= 0 && whether_add == true; i--) {
            boolean tmp = redisTemplate.opsForValue().getBit(redisKey, dayOfYear * 8 + i);
            if (tmp == true) {
                tmp = false;
            }else {
                tmp = true;
                whether_add = false;
            }
            redisTemplate.opsForValue().setBit(redisKey, dayOfYear * 8 + i, tmp);
        }
    }
```

在BitMap中，我们每次使用8个bit，也就是1个字节来表示一天的PR次数。<br/>
所以当有PR操作时，我们找到对应的bit位置，从最右边开始进行+1操作，向左迭代，直到bit是由false变为true为止。<br/>
<br/>

```Java
    //获取发帖的次数
    public List<Integer> calculateDPR(int userId) {
        String redisKey = RedisKeyUtil.getDPRKey(String.valueOf(userId));
        List<Integer> res = new ArrayList<>(366);
        for(int i=0; i < 366; i++){
            String str = redisTemplate.opsForValue().get(redisKey, i, i);
            if (str == null || str.length() < 1) {
                break;
            }
            res.add((int)str.charAt(0));
        }
        return res;
    }
```

当获取PR次数时，我们直接读取对应天数的长度为1的String即可，因为长度为1的String只包含1个大小为1字节的char。将此char直接转换为int数据类型返回给前端即可。<br/>
<br/>

```html
	<script th:src="@{/js/echarts.min.js}"></script>
		<div id="main" style="width: 512px;height:250px;margin: 0 auto;">

		</div>
	<script th:src="@{/js/leetcode.js}"></script>
```

在html中配置好对应的标签和位置，别忘了引入echarts的js和自己的js。<br/>
<br/>

```JS
var chartDom = document.getElementById('main');
var myChart = echarts.init(chartDom);
var option;

function getVirtulData(year) {
    var date = +echarts.number.parseDate(year + '-01-01');
    var end = +echarts.number.parseDate(+year + 1 + '-01-01');
    var dayTime = 3600 * 24 * 1000;
    var data = [];
    var userId = $("#userId").val()
    console.log(userId);
    $.ajaxSettings.async = false;
    $.post(
        CONTEXT_PATH + "/data/dpr",
        {
            "userId": userId,
        },
        function(d){
            d = $.parseJSON(d);
            console.log(d.code)
            for (let time = date,k=0; time < end && k < d.data.length; time += dayTime,k++) {
                data.push([
                    echarts.format.formatTime('yyyy-MM-dd', time),
                    d.data[k]
                ]);
            }
        }
    )
    $.ajaxSettings.async = true;
    return data;

}

option = {
    title: {
        top: 70,
        left: 'left',
        text: 'BitMap Demo'
    },
    tooltip: {},
    visualMap: {
        min: 0,
        max: 32,
        type: 'piecewise',
        orient: 'horizontal',
        left: 'right',
        top: 220,
        pieces: [
            {min: 0, max: 0,label:"less"},
            {min: 1, max: 10,label:" "},
            {min: 1, max: 20,label:" "},
            {min: 21, max: 40,label:" "},
            {min: 41, max: 64,label:"more"},
        ],
        inRange: {
            color: [ '#EAEDF0', '#9AE9A8', '#41C363', '#31A14E', '#206D38' ],//颜色设置
            colorAlpha: 0.9,//透明度
        }
    },
    calendar: {
        top: 120,
        left: 30,
        right: 30,
        cellSize: 13,
        range: '2022',
        splitLine: { show: false },//不展示边线
        itemStyle: {
            borderWidth: 0.5
        },
        yearLabel: { show: false }
    },
    series: {
        type: 'heatmap',
        coordinateSystem: 'calendar',
        data: getVirtulData("2022")
    }
};

myChart.setOption(option);
```

前端通过Ajax将用户id发送给后端，后端返回对应的List<Integer> list存储着用户一年的PR次数。之后的操作就是固定操作了。<br/>
需要注意的是就是Ajax需要是同步操作，不然在data调用getVirtulData("2022")初始数据前，myChart.setOption(option)就会执行完毕，那么此时的贡献图就是空白的了。