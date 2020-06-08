### Redis Geo

Redis 3.2 版本才有Geo，使用 geohash 保存地理位置的坐标，使用有序集合（zset）保存地理位置的集合

- 常用命令
    - GEOADD：增加某个地理位置的坐标
    - GEOPOS：获取某个地理位置的坐标
    - GEODIST：获取两个地理位置的距离
    - GEORADIUS：根据给定地理位置坐标获取指定范围内的地理位置集合
    - GEORADIUSBYMEMBER：根据给定地理位置获取指定范围内的地理位置集合
    - GEOHASH：获取某个地理位置的 geohash 值

[官方案例](https://redis.io/commands/geoadd)


#### spring data redis 操作

`vo对象`
```java
public class CityInfo {

    /** 城市 */
    private String city;

    /** 经度 */
    private Double longitude;

    /** 纬度 */
    private Double latitude;
}
```

```java
    /** redis 客户端 */
    @Autowired
    private final StringRedisTemplate redisTemplate;

    


    // 把城市信息保存到 Redis 中
    public Long saveCityInfoToRedis(Collection<CityInfo> cityInfos) {


        GeoOperations<String, String> ops = redisTemplate.opsForGeo();

        Set<RedisGeoCommands.GeoLocation<String>> locations = new HashSet<>();
        cityInfos.forEach(ci -> locations.add(new RedisGeoCommands.GeoLocation<String>(
                ci.getCity(), new Point(ci.getLongitude(), ci.getLatitude())
        )));

        log.info("done to save city info.");

        return ops.add(GEO_KEY, locations);
    }

    // 获取给定城市的坐标
    public List<Point> getCityPos(String[] cities) {

        GeoOperations<String, String> ops = redisTemplate.opsForGeo();

        return ops.position(GEO_KEY, cities);
    }

    // 获取两个城市之间的距离
    public Distance getTwoCityDistance(String city1, String city2, Metric metric) {

        GeoOperations<String, String> ops = redisTemplate.opsForGeo();

        return metric == null ?
            ops.distance(GEO_KEY, city1, city2) : ops.distance(GEO_KEY, city1, city2, metric);
    }

    // 根据给定地理位置坐标获取指定范围内的地理位置集合
    public GeoResults<RedisGeoCommands.GeoLocation<String>> getPointRadius(
            Circle within, RedisGeoCommands.GeoRadiusCommandArgs args
    ) {

        GeoOperations<String, String> ops = redisTemplate.opsForGeo();

        return args == null ?
                ops.radius(GEO_KEY, within) : ops.radius(GEO_KEY, within, args);
    }

    // 根据给定地理位置获取指定范围内的地理位置集合
    public GeoResults<RedisGeoCommands.GeoLocation<String>> getMemberRadius(
            String member, Distance distance, RedisGeoCommands.GeoRadiusCommandArgs args
    ) {

        GeoOperations<String, String> ops = redisTemplate.opsForGeo();

        return args == null ?
                ops.radius(GEO_KEY, member, distance) : ops.radius(GEO_KEY, member, distance, args);
    }


```

